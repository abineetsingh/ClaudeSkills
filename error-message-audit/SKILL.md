---
name: error-message-audit
description: "Audit user-facing error messages in any codebase against an error-writing rubric. Walks through findings one at a time and proposes rewrites. Use this whenever the user mentions auditing error messages, reviewing error copy, fixing \"Something went wrong\" messages, error UX, error wording, error tone, or anything about how errors are worded — even if they don't say the word \"audit.\""
---

You are auditing the user-facing error messages in this codebase against an error-writing rubric. Your job is to find error strings that real users see, judge them against the rubric, and walk the user through fixes one at a time — proposing a rewrite and asking before editing.

The point of this skill is *quality of communication*, not bug hunting. You're looking at how errors are *worded*, not whether the error handling logic is correct.

## Mode

- If `$ARGUMENTS` is a path (file or directory), scan only that path.
- Otherwise, scan the whole repo, excluding: `node_modules`, `.git`, `dist`, `build`, `out`, `.next`, `.nuxt`, `vendor`, `target`, `__pycache__`, `.venv`, lock files, minified assets, and anything in `.gitignore` that looks like build output.
- Always announce the scope before scanning so the user can redirect you.

---

## 1. Detect user-facing error strings

You're looking for strings the *end user* will read in the UI or in an API response that gets surfaced to them. Not log lines, not internal exception messages that get reformatted before reaching anyone.

There's no single grep that finds these reliably across stacks, so work through this hierarchy and use judgment. When a hit could be either user-facing or dev-only, read enough surrounding code to decide.

### Strong signals (almost certainly user-facing)

- **Toast / notify / snackbar / alert calls** — `toast.error`, `toast(...)`, `notify.error`, `Alert.alert`, `enqueueSnackbar`, `useToast`, `showAlert`, `addToast`, `flash[:error]`, Rails `flash.now[:alert]`, Django `messages.error`, Phoenix `put_flash`
- **i18n error keys** — `t('errors.*')`, `i18n.t('error.*')`, `$t('error.*')`, `gettext("...")`, JSON/YAML files like `errors.json`, `en.json`, `en.yml` with keys matching `error*|fail*|invalid*|cannot*|unable*`
- **Form validation messages** — `setError`, `errors.<field>.message`, Zod/Yup/Joi `.message("...")`, `react-hook-form` `rules.required` strings, Rails `validates ..., message: "..."`, Django `ValidationError("...")`
- **Error UI components** — strings rendered inside `<Alert>`, `<ErrorMessage>`, `<Toast>`, `<Banner variant="error">`, `<FlashMessage>`, `<Notification type="error">`, `<Snackbar>`, error modal/dialog components named `Error*`, `*ErrorDialog`, `*ErrorModal`
- **HTTP error response bodies destined for the client** — `res.status(4xx|5xx).json({ error: "..." })`, `res.status(...).send("...")`, FastAPI `HTTPException(detail="...")`, Rails `render json: { error: "..." }, status: 4xx`, GraphQL `throw new GraphQLError("...")`
- **Error boundary fallbacks** — strings inside React `ErrorBoundary` fallback UI, Vue error handlers, Next.js `error.tsx`/`global-error.tsx`

### Weak signals (read context before including)

- `throw new Error("...")` — only count it if it bubbles to a user-visible boundary (an error boundary, an API error formatter, a global handler that renders the message). If it's caught and reformatted before the user sees it, it's a dev string — skip.
- String literals near `catch` blocks — only if the catch renders or surfaces them.
- Raw `Error` subclasses — `class FooError extends Error { constructor() { super("...") } }` — check whether the `.message` is shown to users anywhere.

### Skip (developer-only, not user-facing)

- `console.*`, `console.error`, `console.warn`
- Logger calls: `logger.*`, `log.error`, `winston.*`, `pino.*`, `Rails.logger.*`, Python `logging.*`, `Sentry.captureMessage`, `Sentry.captureException`, `debug(...)`
- Comments and JSDoc
- Test files: anything under `__tests__`, `tests/`, `spec/`, or matching `*.test.*`, `*.spec.*`, `*_test.go`, `test_*.py`
- Internal `Error()` throws that get caught and reformatted

When in doubt: read the file. Spending 30 seconds to confirm a string actually reaches users beats flagging 100 dev-only strings the user has to ignore.

### Group duplicates

The same error string often appears in many call sites, or the same i18n key gets used everywhere. Group these into one finding (with all locations listed) so a single rewrite fixes them all.

---

## 2. The rubric — score each finding

Each finding gets a severity based on which principles it violates. A single message can violate several at once — list them all in `Issues:`.

### Critical (definitely bad — fix first)

| Principle | What to look for | Bad → Good |
|---|---|---|
| **Generic / no information** | "Something went wrong", "An error occurred", "Failed", "Oops", standalone "Error" | "Something went wrong" → "We couldn't save your post due to a technical issue on our end. Please try again." |
| **Technical jargon** | "fetch", "credentials", "null", "undefined", "exception", "parsing", "schema", "request failed", endpoint paths | "Failed to fetch user data" → "We couldn't load your account" |
| **Leaks internals** | Raw HTTP codes ("Error 500", "401 Unauthorized"), stack traces, exception class names, error IDs without explanation, SQL fragments. These belong in logs, not the UI. A short opaque ref code (e.g. `ref: A8C3`) is OK *alongside* a real message. | "NullPointerException at line 42" → "We couldn't load your dashboard. Try again in a moment. (ref: A8C3)" |
| **Collapses distinct causes** | One generic message catching multiple known causes (network, validation, permissions). If the code can tell which it is, the message must too. Reserve generics for genuinely unknown failures. | One catch-all "Couldn't save" for permission, network, *and* validation errors → branch into "You don't have permission to edit this", "We lost connection — your changes are still here", "Title is required" |
| **Blames the user** | "You entered an invalid…", "You forgot to…", "Wrong password" with no help. Reframe around the *problem*, not the user's action. | "You entered an invalid email" → "That email address doesn't look right — check for typos" |
| **Blames a third party** | "Stripe isn't responding", "GitHub is down", "The API timed out". Even if true, the user came to *your* product. | "Stripe isn't responding" → "We're having trouble connecting to Stripe. Please try again in a moment." |

### Serious (should fix)

| Principle | What to look for | Bad → Good |
|---|---|---|
| **Inappropriate tone** | "Whoops!", "Yikes!", "Uh oh!", exclamation points, ALL CAPS, the literal word "Error" as a title, emoji in serious flows, cutesy/jokey language when stakes are high (payments, data loss, account access) | "Whoops! Payment failed 💸" → "We couldn't process your payment" |
| **Unclear / not specific enough** | Real words, but the user can't tell what to do or what's actually wrong. "Please enter a valid email" doesn't say what's invalid. Name the actual missing or malformed piece. | "Please enter a valid email" → "Add an @ — emails look like name@example.com" · "Allow the requested permissions" → "We need camera access. Open Settings → Privacy → Camera and turn it on." |
| **No fix path** | Tells the user something broke but not what to do next | Add a concrete step: retry, refresh, check X, contact support |
| **No way out** | For unrecoverable errors, no retry / contact / support link | Add support contact, "Try again" button, link to help docs |
| **Buried lede** | The most important info isn't first. Users scan. | "To proceed, please make sure all required fields are completed before submitting" → "Email is required" |
| **Weak CTA verb** | The button says "OK" / "Got it" / "Dismiss" on a recoverable error — closes the dialog without helping. CTAs should name the next action. | "OK" on a payment error → "Try a different card" · "Got it" on a connection error → "Try again" |
| **Standalone announcement won't make sense** | If the message is rendered inside a toast, `aria-live`, or a `role="alert"` region, it's read without visual context. "Required" announced alone is meaningless — include the field name and the fix. | toast: "Required" → "Email is required" |

### Moderate (consider fixing)

| Principle | What to look for | Bad → Good |
|---|---|---|
| **No reassurance** | When relevant, doesn't say what was *preserved* (draft saved, payment not charged, etc.) | "Couldn't send" → "Your draft is saved. We couldn't send it just now — try again in a few minutes." |
| **Missing empathy** | High-stakes situation, message reads cold; a "please" or acknowledgment would warrant it | "Cannot connect" → "We can't connect right now, please try again" |
| **Inconsistent voice** | Tone, capitalization, or terminology varies across error messages in the same product | Normalize toward the rest of the product's voice |
| **Reading level too high** | Aim for ~7th–8th grade. If the message uses words a non-native speaker or hurried user has to parse, simplify. | "Authentication credentials could not be validated" → "Wrong email or password" |
| **The word "invalid"** | Accusatory and vague — name the specific problem instead. | "Invalid input" → "Phone number must be 10 digits" |
| **Stacked apologies** | "Sorry! Oops! We apologize for the inconvenience…" Multiple apologies in one message read hollow. | One "We're sorry — …" is plenty. Drop the rest. |
| **Title case for body text** | Error body should be sentence case, not Title Case Like A Headline. | "Card Was Declined By Your Bank" → "Your card was declined by the bank" |

The rubric is a guide, not a checklist to mechanically apply. The reason these principles exist is empathy for someone who has just hit a wall — every rewrite should make their next 30 seconds easier.

---

## 3. Report and hand off

This skill produces a report. It does not invent its own interactive command loop. The host tool (Claude Code, Codex, Cursor, etc.) already has primitives for confirming destructive changes — use them, don't simulate them.

### Step 1: Produce the report

After scanning, output a single report grouped by severity. Sort: Critical → Serious → Moderate. Within a severity, sort by frequency (a string used in 14 call sites is one finding with 14 locations — listed once, not 14 times).

Use this shape, in plain markdown so it renders in any host:

```markdown
# Error message audit

**Scope:** `<path or "whole repo">`
**Found:** N error messages — C critical · S serious · M moderate
**Top offenders:** "Something went wrong" (14 sites), "Failed to fetch" (8 sites)

---

## Critical (N)

### 1. "Failed to fetch user data. Try again later."  · 3 locations
- **Issues:** generic ("failed"), jargon ("fetch"), no fix path, no way out
- **Suggested:** "We couldn't load your account due to a technical issue on our end. Please try again. If this keeps happening, contact support."
- **Locations:**
  - `src/api/users.ts:142`
  - `src/components/Profile.tsx:88`
  - `src/hooks/useUser.ts:45`

### 2. ...

## Serious (N)
...

## Moderate (N)
...

## Locale siblings (if any)
- `en.json` keys touched: 12 — paired keys exist in `es.json`, `fr.json` (will need re-translation)
```

If the report would be very long, lead with the summary line and the critical section, then say "scroll for serious + moderate" — don't truncate.

### Step 2: Hand off to the host tool's confirmation primitive

After the report, ask the user what they want done — using **whatever native question/confirmation primitive the host environment provides.** Don't roll your own `[y/n/edit/skip]` syntax: that hijacks the host's UX and breaks in tools that render answers as buttons (Claude Code's `AskUserQuestion`), as inline suggestions (Cursor), or as plan-mode confirmations (Codex).

Use these in priority order:

1. **If a structured question tool is available** (e.g. Claude Code's `AskUserQuestion`), use it. Offer concrete choices: "Apply all critical fixes", "Apply all (critical + serious + moderate)", "Pick specific findings to apply", "Don't apply anything — I'll review the report".
2. **If the host has a plan/approval mode** (e.g. Codex plan mode), present the proposed edits as a plan and let the host's approval flow handle confirmation.
3. **Otherwise, ask in plain text.** End the report with a single, clear question: *"Want me to apply any of these? Reply with the finding numbers (e.g. 'fix 1, 3, 5'), 'all critical', 'all', or 'no'."*

Do not pretend a pseudo-CLI prompt is real interactivity. The user should see one report + one question, not a fake loop.

### Step 3: If the user says yes, edit normally

Once the user has chosen what to fix, apply the changes using the host's standard editing tools (`Edit`, `MultiEdit`, or whatever the platform exposes). One finding may touch many files — group those into one edit batch per finding.

Show what changed at the end as a brief summary: which findings were applied, which files were touched, and any locale files that now need re-translation. Do not run tests, lint, or type-check unless the user asks — those aren't this skill's job.

If the user picks a subset, treat the un-picked findings as deliberately deferred — don't nag, don't reprint them, just complete the chosen ones.

---

## 4. How to write a good rewrite

A good message has up to 5 parts. Aim for at least 3, in this priority order:

1. **Say what happened** — be specific about what the user was trying to do that didn't complete
2. **Say why** — even "a technical issue on our end" is better than no reason
3. **Help them fix it** — concrete next step (retry, check X, open settings, contact support)
4. **Reassure** — when relevant, say what was *not* lost ("Your draft is saved", "You weren't charged")
5. **Give a way out** — for unrecoverable errors, a support link or contact

### Voice rules

- **Talk to a friend, but don't be cutesy when stakes are high.** Payment failures, data loss, and account access aren't the time for "Whoops!" — keep the warmth, lose the exclamation.
- **Own the problem.** "On our end" / "We couldn't" beats blaming the user or a third party. The user came to *your* product; absorbing the blame is part of the deal.
- **Use "please" sparingly.** Save it for situations that warrant real empathy (the user can't fix it themselves, they're going to be frustrated). If you're using it on every message it loses meaning. Don't use "please" to scold ("please enter a valid email").
- **Lead with the most important thing.** Users scan. Put the noun and the problem first, the reasoning and apology after. "Email is required" beats "In order to continue, please make sure to fill in this required field."
- **Sentence case, not title case.** "Your card was declined" not "Your Card Was Declined."
- **One apology max.** "Sorry, oops, we apologize for the inconvenience" reads hollow. Pick one or none.
- **Replace jargon and accusatory language with plain language.** Some useful swaps:
  - "fetch" → "load"
  - "credentials" → "sign-in" / "password"
  - "invalid" / "invalid input" → name the actual issue ("Phone number must be 10 digits", not "Invalid phone")
  - "Please enter a valid X" → name the missing piece ("Add an @ — emails look like name@example.com")
  - "request failed" → "we couldn't reach our server"
  - "401 / 403 / Unauthorized" → "you're signed out" / "you don't have access to this"
  - "404 / Not found" (when user-facing) → "we couldn't find that page / item"
  - "500 / Internal Server Error" → "a technical issue on our end"
- **Match length to severity.** A toast that flashes for 3 seconds gets one tight sentence. A blocking modal can carry two sentences plus a link.
- **Preserve interpolation.** If the original is `"Couldn't load ${name}"`, the rewrite must keep the same template variables — don't silently drop them. If you remove or rename a variable, that's a code change, not a copy change — flag it to the user.

### Action buttons (the CTA next to the message)

If the message ships with a button or link, the verb on it is part of the message. Audit it too.

- **Name the next action.** "Try again", "Use a different card", "Edit address", "Reconnect", "Open settings". Avoid "OK" / "Got it" / "Dismiss" on recoverable errors — they close the dialog without resolving anything.
- **For unrecoverable errors, give an escape hatch.** "Contact support" or "Back to home" beats a lone "Close."
- **Two buttons: primary = forward, secondary = back/cancel.** Don't make "Cancel" the primary action on an error.

### Things that look like a fix but aren't

- Adding "Please try again later" without saying *what* failed → still generic.
- Apologizing more (`"We're so sorry!"`) without adding information → tone change, no real improvement.
- Replacing "Error" with "Oops" → makes it worse.
- Replacing one generic catch-all with a slightly nicer generic catch-all when the code can actually distinguish causes → still failing the user.

---

## 5. Form and inline-validation errors

Form errors are their own beast — placement and timing affect what the wording needs to say.

- **Inline error next to the field can be terse.** The field label provides context. "Required." or "Must be 10 digits." is fine *if* the user can see which field it's attached to.
- **Toasts and `aria-live` announcements need to self-locate.** They're read without visual context. "Email is required" beats "Required."
- **Multi-error forms need a summary.** On submit, render an error summary at the top with the same wording as each inline error and links to each field. Screen reader and keyboard users depend on it.
- **Don't validate too eagerly.** Validating on every keystroke or unconditionally on blur fires before the user is done — and screen readers tab through fields, triggering blur events the user didn't mean. Validate on submit, or after meaningful interaction (user stopped typing, then moved away). The wording assumes appropriate timing — "Email is required" reads as an accusation if it fires while they're still typing.
- **Don't clear the form on error.** Preserve what the user typed, even invalid values, so they can edit instead of retype.
- **Name the actual rule, not just "invalid."** "Password must be at least 8 characters" beats "Invalid password."

---

## 6. Internationalization

If the codebase has i18n (locale files, `t()` calls, `gettext`, etc.), the *source string* needs to be written in a way that translators and the i18n library can work with.

- **No string concatenation.** `"Couldn't load " + name + " from server"` breaks for languages with different word order. Use full sentences with named placeholders: `t('couldnt_load', { name })`.
- **Use ICU MessageFormat (or framework equivalent) for plurals.** English's "1 item / 2 items" is one of dozens of plural forms — Arabic has six, Russian has three. Don't do `count === 1 ? "item" : "items"` inline.
- **Don't bake gender into source strings.** "He couldn't be reached" — translate to a name or "they."
- **Flag locale siblings.** When fixing a string in `en.json`, the same key in `es.json`, `fr.json`, etc. is now stale. Note this so the user can queue a translation update — don't try to translate it yourself.
- **Preserve existing interpolation tokens exactly.** `{{name}}`, `${name}`, `%{name}`, and `{0}` are framework-specific. If you change the syntax you'll break the build.

---

## Guidelines

- **Read the file before flagging.** Context matters. A `console.error` near a toast call might be paired with a user-facing message, or it might be dev-only — you can't tell from the grep hit alone.
- **Don't flag dev-only strings.** Logger output, Sentry breadcrumbs, internal throws that get caught and reformatted — these are noise to the user. If they have to filter through 50 logger lines to get to 5 real findings, the audit failed.
- **Group duplicates.** Same string in 14 call sites = one finding with 14 locations, one rewrite. Don't make the user say `y` 14 times.
- **Never edit silently.** Always show original + rewrite + one-line reason and wait for confirmation. The whole point is the user gets to drive the voice of their product.
- **Stop trying to be helpful past the rubric.** Don't refactor the error handling, don't suggest adding new error states, don't reorganize the i18n file. Audit the words; that's the job.

