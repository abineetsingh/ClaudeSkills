# Reference

## `.claude/launch.json` schema

```json
{
  "version": "0.0.1",
  "configurations": [
    {
      "name": "myapp",
      "runtimeExecutable": "./script/dev",
      "runtimeArgs": [],
      "cwd": ".",
      "env": {},
      "autoPort": true,
      "port": 32100
    }
  ]
}
```

| field | meaning |
|---|---|
| `name` | label shown in the Preview dropdown |
| `runtimeExecutable` | command or script path (relative to repo root) Desktop runs to start the app |
| `runtimeArgs` | args passed to `runtimeExecutable` (e.g. `["run","dev"]` for `npm`) |
| `program` | alternative to `runtimeExecutable` for a direct node script |
| `args` | args for `program` |
| `cwd` | working dir relative to repo root (defaults to repo root) |
| `env` | extra environment variables |
| `port` | preferred port; Desktop opens an embedded browser here |
| `autoPort` | `true` → pick a free port if `port` is taken and inject it as `$PORT`; `false` → fail on conflict; unset → ask the user |

**How port handoff works:** when `autoPort` selects a different free port, Desktop passes it to your process via the **`PORT` env var**. So the wrapper should do `PORT="${PORT:-<derived>}"` — honor an injected value, fall back to the deterministic one. Set `port` in launch.json to the deterministic app-port value so the preferred port and the app's bound port match on the common case.

**Triggering preview:** Desktop starts the server automatically after edits, or the user can start/stop it from the **Preview** dropdown. There is no `/preview` slash command.

## Worktrees in Desktop

- Desktop creates a fresh git worktree per session under `.claude/worktrees/`.
- Worktrees branch from `origin/HEAD` by default; auto-cleaned on exit unless dirty.
- **`.worktreeinclude`** (repo root) lists gitignored files Desktop copies into each new worktree — the clean way to get `.env` into worktrees.
- **`SessionStart` hook** (`.claude/settings.json`) fires when a session begins/resumes. Whether it fires at the exact moment Desktop *creates* a worktree isn't formally documented — so make the setup script idempotent and cheap to re-run, and don't rely on it having run before the first preview. (The wrapper deriving its own ports means the app boots correctly even if the hook hasn't regenerated launch.json yet.)

## The port-hashing helper (copy verbatim)

```bash
hash_of() { printf '%s' "$1" | cksum | cut -d' ' -f1; }
WORKTREE_ROOT="$(git rev-parse --show-toplevel)"   # axis 1: which worktree
APP="web"                                          # axis 2: which app (monorepo); constant for single-app
PORT=$(( 30000 + $(hash_of "$WORKTREE_ROOT:$APP")          % 10000 ))
VITE=$(( 40000 + $(hash_of "$WORKTREE_ROOT:$APP:vite")     % 10000 ))
REDIS_DB=$((       $(hash_of "$WORKTREE_ROOT:$APP:redis")   % 16 ))
```

- **`cksum`** is POSIX — present on macOS and every Linux. No language runtime needed, so the same scheme works for any stack.
- **Anchor on the worktree root**, never `$PWD` (changes per subdir) or the branch name (changes on rename, and two worktrees can share a branch).
- **Two salt axes: worktree × app.** The app id keeps sibling apps in one monorepo worktree off each other's ports; the worktree root keeps parallel worktrees apart. For a single-app repo the app id is a harmless constant — same formula, no special-casing.
- **Then salt per slot** (`":vite"`, `":redis"`, `":worker"`) so services within one app don't collude into the same number.
- **Band the ranges** (app 30000–39999, frontend 40000–49999) so a glance at a port tells you what it is, and the two bands can't overlap.

## Monorepo launch.json (one config per app)

```json
{
  "version": "0.0.1",
  "configurations": [
    { "name": "web",       "runtimeExecutable": "web/script/dev",       "autoPort": true, "port": 32375 },
    { "name": "marketing", "runtimeExecutable": "marketing/script/dev", "autoPort": true, "port": 47120 }
  ]
}
```

Desktop lists every config in the Preview dropdown; the user picks which app to preview (or runs several). Each `name` is also the app's port salt — keep it equal to the value baked into that app's wrapper. The whole file is regenerated per worktree by the top-level SessionStart setup script (see `templates/setup-worktree-monorepo`).

## Native / non-web apps

iOS, macOS, and CLI targets have no dev-server port and aren't web pages, so they don't go in `launch.json` — leave them out of the configurations array entirely. The per-worktree win for native apps is build isolation instead (e.g. an Xcode `-derivedDataPath` inside the worktree so parallel sessions don't share stale build output), which is a build-tooling concern handled outside this skill.

## Hard-won gotchas (generalized from a production multi-service web app)

1. **Force UTF-8.** Desktop's launcher has a minimal env with no `LANG`. Tools that read files in the system encoding (Ruby's `File.read`, some Python paths) default to ASCII and crash on non-ASCII bytes in `.env`. `export LANG="${LANG:-en_US.UTF-8}"` + `LC_ALL` at the top of the wrapper.

2. **Per-worktree env must win at runtime.** dotenv-style loaders do NOT override a variable already set in the parent shell. If the user has `DATABASE_URL` in their global shell, it silently beats the worktree's value and the app connects to the wrong DB. Source the per-worktree file (`.env.local`) explicitly in the wrapper AND, if the framework has a load step, force an overload early in app boot (Rails: `Dotenv.overload(".env.local")` in `config/application.rb`).

3. **Background workers need their OWN Redis DB slot.** Cache / pub-sub / rate-limit clients namespace their keys, so they can share one DB safely. Job queues (Sidekiq, Resque, BullMQ, Celery) use UNNAMESPACED keys (`queue:*`, `retry`, `dead`) — two workers on the same DB steal each other's jobs. Give the worker a separately-salted slot.

4. **Default Redis only has 16 DBs.** `% 16` is safe out of the box but collision-prone across many worktrees. For heavy parallel use, set `databases 10000` in `redis.conf` (`brew services restart redis`) and hash `% 10000`. The default 16 will throw `ERR DB index is out of range` if you hash into a higher index without raising the limit.

5. **Never hardcode `localhost:6379/0`** (or `:3000`, or a fixed DB name) anywhere in the app. Always read from the env var the wrapper sets. One hardcoded port/DB defeats the whole isolation scheme.

6. **Regenerate `launch.json` per worktree.** Its `port` is worktree-specific, so the setup script must rewrite it on every run. Gitignore it. (Keep `.claude/settings.json` and the scripts committed — they're shared.)

7. **Locate the main checkout via git, not a hardcoded path.** `MAIN="$(dirname "$(git rev-parse --git-common-dir)")"` works for both `git worktree` and Desktop worktrees — `.git` lives in the main checkout for both.
