---
name: claude-desktop-setup
description: Make any repository work cleanly with Claude Code Desktop's preview + parallel git worktrees, regardless of language or framework. Generates .claude/launch.json, a SessionStart setup hook, a dev-server wrapper that gives each worktree collision-free ports, and isolated backing services (Redis DB index, database, frontend dev server, background worker). Use when the user wants to "make this repo Claude Code Desktop compatible", set up worktree previews, fix port collisions across worktrees, configure unique-port / per-worktree ports, or get the Preview button working in Desktop.
---

# Claude Code Desktop setup

Make the current repo boot a working preview in **Claude Code Desktop**, and — when it has more than one process or any stateful backing service — give every git worktree its own collision-free ports and isolated services so multiple Desktop sessions run in parallel without stepping on each other.

## Mental model — two layers

**Layer 1 (always): a working preview.** Desktop reads `.claude/launch.json` to know how to start the app and which port to open. The minimum viable setup is this one file.

**Layer 2 (when needed): per-worktree isolation.** Desktop creates a fresh git worktree per session (under `.claude/worktrees/`). If two worktrees both bind `:3000`, or share one Redis DB / one Postgres database, they collide. The fix is a tiny boot wrapper that derives **deterministic ports + service slots from the worktree's own path** — so the setup script and the boot script independently compute identical values, and no two worktrees ever overlap.

Skip Layer 2 for a single-process app with no backing services (a static site, a lone Next.js/Vite/Rails server) — Desktop's `autoPort` handles the one port. Add Layer 2 the moment there's a second port or a stateful service.

## The core trick (this is the whole idea)

```bash
hash_of() { printf '%s' "$1" | cksum | cut -d' ' -f1; }
WORKTREE_ROOT="$(git rev-parse --show-toplevel)"   # differs per worktree
APP="web"                                          # differs per app in a monorepo; any stable id for a single-app repo
export PORT=$(( 30000 + $(hash_of "$WORKTREE_ROOT:$APP")        % 10000 ))   # app server
export VITE_PORT=$(( 40000 + $(hash_of "$WORKTREE_ROOT:$APP:vite") % 10000 ))  # frontend
export REDIS_URL="redis://localhost:6379/$(( $(hash_of "$WORKTREE_ROOT:$APP:redis") % 16 ))"
```

`cksum` is everywhere (POSIX), stable for a given path, and differs across worktrees. The salt has **two axes**: the worktree root (so parallel worktrees don't collide) and the app id (so sibling apps in one monorepo worktree don't collide either) — plus the service name so each slot is independent. Anchor on the **worktree root** (`git rev-parse --show-toplevel`) — never `$PWD` or the branch name — so any script in any subdir agrees on the numbers. For a single-app repo `$APP` is just a constant; it costs nothing and keeps one formula for both cases.

---

## Procedure

### Step 1 — Detect the project (do this before asking anything)

Read, don't guess. Look for and read:

- **Dev command + port:** `package.json` scripts (`dev`/`start`), `Procfile.dev`, `Procfile`, `bin/dev`, `Makefile`, `Gemfile` (Rails → `bin/dev`/`bin/rails server`), `manage.py` (Django → `runserver`), `go.mod`/`main.go`, `Cargo.toml`, `docker-compose.yml`, `mix.exs`.
- **How the framework reads its port:** most read `$PORT` (Rails, Next.js via `-p`, Express, Django needs an arg, Vite via `--port`/config). Note the exact mechanism — it determines how the wrapper passes the port.
- **Backing services:** `docker-compose.yml`, `config/database.yml`, `config/cable.yml`, any `redis`/`REDIS_URL`/`DATABASE_URL` references, a separate frontend dev server (Vite/webpack), background workers (Sidekiq/Celery/BullMQ/Resque).
- **Secrets:** is there a `.env` / `.env.local`? Is it gitignored?
- **First-run needs:** dependency install (`npm install`, `bundle install`, `pip install`, `go mod download`), DB create/migrate/seed.
- **Existing Claude config:** read `.claude/settings.json` and `.claude/launch.json` if they exist — you'll MERGE, not clobber.
- **Monorepo?** Multiple app dirs each with their own manifest (e.g. `web/Gemfile`, `marketing/package.json`, `ios/`), a workspace file (`pnpm-workspace.yaml`, `turbo.json`, npm/yarn workspaces, Cargo workspace, Nx/Lerna), or obvious top-level app folders. If so, treat each previewable app separately — see "Monorepos" below.

Form a concrete hypothesis (e.g. "monorepo: `web/` is a Rails app with Postgres+Redis+a worker, `marketing/` is a Next.js app, `ios/` is native; `.env` at root is gitignored"). Then confirm only the parts you can't be sure of.

### Step 2 — Ask the consequential unknowns (AskUserQuestion)

Ask only what changes the output and you can't reliably infer. Typical batch:

1. **Dev command** — present your detected command(s) as options; confirm or correct. (Skip if there's an unambiguous single answer like an existing `bin/dev`.)
2. **Which services need per-worktree isolation?** (multiSelect): `Frontend dev server`, `Redis / cache / queue`, `Database (separate DB per worktree)`, `Background worker`, `None — single process`. This decides which slots the wrapper derives and whether you build Layer 2 at all.
3. **Secrets** — only if a gitignored `.env` exists: "Copy `.env` into each worktree?" Options: `Yes, via .worktreeinclude (Desktop copies it)` / `Yes, symlink to a single source of truth (edits propagate)` / `No secrets needed`.
4. **First-run setup** — only if non-trivial: confirm the install/migrate/seed commands a fresh worktree needs. (Often inferable; confirm rather than open-ended ask.)

Don't ask about the hashing scheme, base port numbers, or file locations — those have sane defaults. Pick them and say what you picked.

### Step 3 — Generate the files

Templates live in `templates/` next to this skill — read them, then write adapted copies into the repo. Replace every `{{PLACEHOLDER}}`.

**Always write `.claude/launch.json`** (`templates/launch.json`):
- `name`: the app/repo name.
- `runtimeExecutable`: path to the boot script relative to repo root (Layer 2 → your wrapper, e.g. `./script/dev`; Layer 1 → the dev command directly, e.g. `npm`).
- `runtimeArgs`: args if `runtimeExecutable` is a bare command (e.g. `["run","dev"]`).
- `autoPort: true` and a `port` (the deterministic app-port value for the main checkout). `autoPort` lets Desktop pick a free port and inject it as `$PORT` if the preferred one is taken — the wrapper honors an injected `$PORT`.

**Always merge `.claude/settings.json`** to add a `SessionStart` hook (`templates/settings.json`) that runs your setup script. If the file exists, merge into `hooks.SessionStart` — don't overwrite other hooks.

**Layer 2 only — write the boot wrapper** (`templates/dev-wrapper`) into e.g. `script/dev` (or `bin/dev` if the framework convention fits). Keep only the service slots the user selected; delete the rest. End with `exec <dev command>`. `chmod +x` it.

**Layer 2 only — write the setup script** (`templates/setup-worktree`) into e.g. `script/setup-worktree`. It regenerates `launch.json` with THIS worktree's port (so a new worktree's preview opens the right port), then runs first-run install/DB setup guarded by an idempotency marker. `chmod +x` it.

**For per-worktree databases**, the simplest portable pattern is a DB **name** derived from the hash (`myapp_dev_<hash>`); the setup script `createdb`/migrates it. (Managed branch databases like Neon/PlanetScale are possible but provider-specific — only build that if the user is already on one.)

### Step 4 — Wire secrets into worktrees

If the user chose to copy `.env`:
- **`.worktreeinclude`** (preferred, native): one gitignored path/glob per line. Desktop copies these into each new worktree. Write `.env` (and any other needed-but-gitignored files) into it. See `templates/worktreeinclude`.
- **Symlink** (single source of truth): in the setup script, locate the main checkout via `MAIN="$(dirname "$(git rev-parse --git-common-dir)")"` and `ln -sf "$MAIN/.env" .env`. Edits to the main `.env` then show up in every worktree.

### Step 5 — `.gitignore`

Ensure generated/per-worktree artifacts are ignored: `.claude/launch.json` (regenerated per worktree), `.claude/worktrees/`, `.env.local`, and any per-worktree DB-name file. Do NOT gitignore `.claude/settings.json` or the scripts — those are shared, committed config.

### Step 6 — Verify it actually boots (Always Works™)

Do not declare success on file creation alone.
1. Run the setup script once: `bash script/setup-worktree` — it should complete clean and write `launch.json`.
2. Boot the wrapper in the background and confirm the app answers on the derived port: `PORT= bash script/dev &` then `curl -sf -o /dev/null -w '%{http_code}' localhost:$PORT` (compute the port the same way the script does). A 2xx/3xx/even 4xx means it bound; connection-refused means it didn't.
3. Kill the test process. Report the actual port(s) and service URLs you observed.
4. If you genuinely can't run it (missing system deps, no Redis), say so explicitly and tell the user the exact command to run themselves — don't imply you verified it. In a monorepo, verify each previewable app on its own port.

---

## Monorepos (multiple apps in one repo)

Desktop's `launch.json` holds an **array** of configurations — one per app — and shows them all in the Preview dropdown. That's the whole monorepo story: N apps → N entries in one `launch.json`.

What changes from the single-app flow:

- **Port salt gains the app axis.** Each app's slots are `hash_of("$WORKTREE_ROOT:$APP_ID:...")` where `APP_ID` is the app's subdir (`web`, `marketing`, ...). Two axes — worktree × app — so neither parallel worktrees nor sibling apps in the same worktree collide. Keep each app's `APP_ID` identical in its `dev-wrapper` and wherever its port is written into `launch.json`, or the preview opens the wrong port.
- **One wrapper per app**, living in that app's subdir (e.g. `web/script/dev`, `marketing/script/dev`), each `cd`-ing into its own dir but resolving the same `WORKTREE_ROOT` via git. Give each only the service slots it needs (the Rails app gets Redis + DB; a static marketing site gets neither).
- **One top-level setup script**, not one per app. Use `templates/setup-worktree-monorepo`: it lists the apps, regenerates the single combined `launch.json` (one config per app), and runs each app's first-run setup guarded by its own marker. Wire THIS to the `SessionStart` hook.
- **Ask which subdirs are previewable apps.** Detection finds candidates; confirm the set and each one's dev command + type via AskUserQuestion before generating. A `docs/` or `packages/shared` dir is usually not a previewable app.

**Native / non-web apps (iOS, macOS, CLI):** these have no dev-server port, so they don't belong in `launch.json` (Desktop's preview is a web view). Leave them out of the configurations array. They can still benefit from worktree isolation in a different form — e.g. building into a worktree-local dir so parallel sessions don't fight over shared build output — but that's a build-tooling concern, not a Desktop preview. Note it to the user and move on; don't force a fake port onto them.

## Reference

Full launch.json schema, the generalized port-hashing helper, and the hard-won gotchas (UTF-8 forcing, dotenv override order, unnamespaced worker queues, Redis 16-DB limit) are in `reference.md`. Read it before writing the wrapper if the app has a worker or Redis.

## Publishing

This is a personal skill at `~/.claude/skills/claude-desktop-setup/`. To share it with others, the user can run `/skill-sync` (don't push to a public repo without asking).
