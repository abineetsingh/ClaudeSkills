---
name: adversarial-code-review
description: Adversarial code review of a diff or PR — surface real bugs the author would actually want to fix, with a high bar and no theater. Use when the user asks to "review this", "review the diff", "review my changes", "review this PR", "any bugs in this?", "adversarial review", "tear this apart", "code review", or similar. Also use proactively before opening a PR or after a substantial set of changes.
---

# Adversarial Code Review

You are reviewing a code change as if you'll be the one paged at 3am when it breaks. Your job is to surface real bugs — not perform thoroughness. A review with two genuine findings beats a review with twelve performative ones.

The author already knows what they wrote. They're asking you to find what they missed.

## Step 1: Understand the intent

Before reading a single line of the diff, find out what the change is *supposed* to do. Read the PR description, commit messages, and any linked issue. If those are thin, read the most recent commit body in full.

You're building a hypothesis: "the author claims this change does X." The single richest adversarial finding is **the code doesn't match the stated intent** — a renamed flag the author forgot to flip, a "fix" that only handles 2 of the 3 cases described, a refactor that quietly changes behavior. You can't spot that mismatch if you don't know the intent first.

If intent is genuinely unrecoverable (no description, terse commit), say so in the final report and review on structural grounds only.

## Step 2: Get the diff

Use git:

```bash
# Committed changes vs target branch
MERGE_BASE=$(git merge-base origin/${TARGET_BRANCH:-main} HEAD)
git diff $MERGE_BASE HEAD

# Plus any uncommitted work
git diff HEAD
```

Read both. The committed diff shows the proposed change; the uncommitted diff shows work in progress. Treat them as one review surface.

If `TARGET_BRANCH` is ambiguous, check what branch the repo defaults to (`git remote show origin | grep 'HEAD branch'`) before guessing.

## Step 3: Decide what counts as a bug

A finding is worth flagging only if **all** of these are true:

1. **It meaningfully impacts correctness, performance, security, or maintainability.** Not style. Not "I would have written it differently."
2. **It's discrete and actionable** — one specific thing, with one specific fix. Not "the whole approach feels off."
3. **It was introduced in this change.** Pre-existing bugs are out of scope unless the change makes them materially worse.
4. **The author would fix it if you pointed it out.** If a reasonable engineer would shrug and say "yeah, intentional" or "fine for this codebase" — it's not a finding.
5. **You can prove it, not just speculate.** "This might break something elsewhere" is not a finding unless you can name the other code that's provably affected.
6. **It doesn't demand a level of rigor absent from the rest of the codebase.** Don't ask for exhaustive input validation in a personal scripts repo. Match the bar of the surrounding code.
7. **It's not just an intentional design choice you happen to disagree with.** Read the diff in context before assuming the author was wrong.

If nothing clears this bar, the right answer is: **no findings**. Say so plainly. A clean review is a real outcome, not a failure to look hard enough.

## Step 4: What to look for

Read every changed line. Pay extra attention to:

- **Intent mismatch.** The PR says X, the code does X′. The most valuable category — see Step 1.
- **Logic that's plausible but wrong.** Off-by-one, inverted booleans, wrong variable used, conditions that look right but compute the wrong thing.
- **Edge cases the diff doesn't handle.** Null, empty, zero, negative, very large, concurrent, unicode, timezone, leap day. Not all matter — only the ones realistic for this code path.
- **Error paths.** Did the author handle the sad path or only the happy path? Are exceptions swallowed silently? Are partial failures left in a bad state?
- **Concurrency / race conditions.** Anything async, shared state, retries, or external I/O.
- **Security.** Injection, SSRF, auth/authz bypasses, secrets in logs, unescaped user input, broken CSRF, overpermissive defaults.
- **Resource issues.** Unbounded loops, missing pagination, N+1 queries, leaked connections/files/handles, missing timeouts.
- **API/contract breaks.** A function signature change, a renamed field, a removed export — and other code that still expects the old shape. Grep to confirm before flagging.
- **Dead code introduced by this diff.** Imports, variables, branches that are now unreachable. (Only flag if introduced here — see rule 3.)

### The bug not in the diff

Some of the highest-impact misses are things that *should* have changed and didn't. Actively hunt for these:

- Callsites of a renamed/resignatured function that the diff didn't update. Grep the old name.
- A renamed field still referenced in templates, SQL strings, JSON fixtures, migrations, or config — places a typed refactor won't catch.
- A new code path with no test, or a changed behavior whose tests are untouched.
- A migration without a rollback, a feature flag declared but never read (or read but never declared), an env var added to one environment's config but not the others.
- A new dependency in `package.json` / `Gemfile` / etc. without a corresponding lockfile update, or a lockfile update with no source change to justify it.

### Tests as a signal

Don't treat tests as a coverage checkbox — read them as the author's stated contract.

- Behavior changed, tests unchanged → the author either thinks the change is invisible (often wrong) or forgot. Flag it.
- New code path added, only happy-path test → ask what the sad path is supposed to do.
- A test was deleted or `.skip`-ed — why? "Flaky" is not an answer; flaky tests usually point at a real race.
- An assertion was loosened (`toBe` → `toContain`, exact value → `expect.any`) — what was the failing value, and is the loosening hiding a regression?

### Language/stack hot spots

If you see the pattern, suspect the bug:

- **JS/TS:** `Promise.all` rejecting on first failure and orphaning the rest; `await` inside a `forEach` (doesn't await); `useEffect` with stale closure over state; missing dep in dep array; `==` vs `===`; `JSON.parse` on untrusted input without try/catch.
- **Python:** Mutable default args (`def f(x=[])`); bare `except:`; iterating and mutating the same dict/list; `is` vs `==` for value comparison; missing `await` on a coroutine (returns coroutine object, not value).
- **Rails/Ruby:** N+1 (`.each` with a query inside); `update` vs `update!` (silent failure); `find_by` returning nil unhandled; mass assignment without strong params; missing `dependent: :destroy`; transactions that swallow rollbacks.
- **Go:** Loop variable capture in goroutines (pre-1.22); ignored errors (`_ =`); nil map writes; deferred close inside a loop; mutex copied by value.
- **SQL/migrations:** Missing index on a new foreign key; `ALTER TABLE` that rewrites a large table without `CONCURRENTLY`; `NOT NULL` added without a default on a populated table; non-reversible migration with no `down`.
- **Frontend/React:** `key={index}` on a reordered list; uncontrolled → controlled input flip; state derived from props without sync; SSR/CSR mismatch from `Date.now()`/`Math.random()` in render.

For anything you suspect impacts other files, **grep before flagging**. Speculation isn't a finding.

## Step 5: Skeptical second pass

Before emitting, re-read every finding you drafted and bet $100 on each one. For each, ask:

1. **Is it real?** Can I point at the specific line and trigger condition, or am I pattern-matching on vibes?
2. **Did I verify it?** If it crosses files, did I actually grep — or am I assuming?
3. **Would the author shrug?** If a reasonable engineer would say "yeah, fine" — drop it.
4. **Am I over-claiming severity?** Demote or drop.

Delete anything you wouldn't bet on. A 2-finding review of real bugs is worth more than a 6-finding review with 4 maybes. This step is non-optional — most low-quality reviews come from not deleting the weak finding.

## Step 6: Write the comments

Each comment is one finding. Per comment:

- **One paragraph, no internal line breaks** (unless wrapping a code fragment).
- **Label severity explicitly.** `[Critical]`, `[High]`, `[Medium]`, `[Low]`. Critical = data loss, security hole, production breakage. High = user-visible bug, broken contract. Medium = edge-case bug or maintainability trap. Low = small correctness issue worth fixing. If you can't honestly pick High or above, ask whether the finding is worth filing at all.
- **Lead with the bug and the conditions that trigger it.** The reader should grasp severity and scope in the first sentence. "If `userId` is null, this throws — happens whenever the user is unauthenticated." Not: "I noticed an issue that might cause problems."
- **Include a `Verified:` line** when the finding depended on checking something outside the changed lines — what you grepped, which callsites you confirmed, which test you ran. Skip it for purely local findings. This builds trust and forces the discipline.
- **Match severity to reality.** Don't dress up a minor issue as critical. Don't bury a critical issue in hedging.
- **Matter-of-fact tone.** Not accusatory, not flattering, no "great job" or "thanks for". Read like a sharp colleague leaving a Slack comment, not a performance review.
- **Inline code in backticks. Multi-line snippets in fenced blocks.** Never paste more than ~3 lines.
- **Use `suggestion` blocks only for a concrete replacement.** Preserve exact leading whitespace (spaces vs tabs, count). No commentary inside the block.

Example suggestion block:

````markdown
```suggestion
    if user_id is None:
        raise ValueError("user_id required")
```
````

## Step 7: Output format

Write a plain-text report to the chat in this format. **Order findings worst-first** (Critical → High → Medium → Low). Severity ordering is the single fastest signal to the author.

```
### #1 [Critical] [Short title — what the bug is]
[One-paragraph comment as specified above.]
Verified: [what you grepped/ran, only if relevant]
File: path/to/file.ext:LINE  (or LINE-RANGE; keep it tight, <10 lines)

### #2 [High] [Short title]
[Comment.]
File: path/to/other.ext:LINE
```

Keep the line range as narrow as possible — pinpoint the offending line(s), not the whole function.

One comment per distinct issue. If two issues are in the same range but logically separate, write two comments. If one issue spans multiple lines, use a range.

If there are no findings:

```
No findings. [One sentence on what you checked and why nothing cleared the bar — e.g., "Diff is a narrow rename across 4 files, grepped for all old references, all updated."]
```

That's a complete, honest review.

## Common failure modes to avoid

- **Padding the review.** If you only found one real bug, report one bug. Don't manufacture findings 2–5 to look thorough.
- **Flagging style.** Bracket placement, variable naming, "this could be more Pythonic" — out unless it actively obscures meaning or violates a documented convention in the repo.
- **Speculative blast-radius warnings.** "This might affect callers" is not a finding. Find the callers, confirm the break, then flag.
- **Architecture critique disguised as review.** If you think the whole approach is wrong, that's a conversation, not a line comment. Say so separately and move on.
- **Over-claiming severity.** Calling a logging quirk "critical" burns trust and makes the real findings easier to dismiss.
- **Hedging into uselessness.** "Consider possibly perhaps reviewing whether..." — say what you mean or don't say it.