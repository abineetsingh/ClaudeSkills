---
name: security-audit
description: Audit a codebase for table-stakes security hygiene (the basics that would be embarrassing to miss) and verify that the security/privacy promises in legal pages, marketing copy, and in-product UI actually hold up in code. Use when the user mentions "security audit", "security review", "security check", "harden this app", "are we doing the security basics", "audit our privacy claims", "do we actually do what our privacy policy says", or asks about specific basics (secrets in repo, password hashing, security headers, cookie flags, CORS, CSRF, rate limiting, dependency CVEs). Skill is harness-agnostic — works in Claude Code, Codex, Cursor, and any other agent host.
---

# Security Audit

You are auditing a codebase against **table-stakes security hygiene** — the obvious basics where, if a journalist looked at the code, they'd write a piece. You are also checking whether the **security and privacy promises the product makes to users** (in legal pages, marketing copy, the UI itself) are actually backed by code.

The bar is "would this be embarrassing on Hacker News?", not "would this pass a SOC 2 audit." A small SaaS doesn't need enterprise hardening, but it does need to not commit `.env` to git, not log passwords, and not ship `dangerouslySetInnerHTML(userInput)`. Match the bar of the surrounding code (same rule as adversarial code review): don't demand exhaustive input validation in a 200-line side project.

**Functionality is sacred.** Never propose a fix that breaks a working flow without flagging the breakage. Prefer additive changes (add a header, set a flag, hash a column on the next write) over restrictive ones that could lock users out (forcing 2FA, tightening CORS, rotating session cookies without a migration plan). Every finding includes a `Breakage risk:` line.

The audit has two halves and produces one report:

1. **Code/config hygiene** — the table-stakes checklist applied to source, configuration, and infra-as-code.
2. **Promise audit** — read what the product says it does about security/privacy and verify the code backs it up.

## Mode and scope

- If `$ARGUMENTS` is a path (file or directory), scan only that path.
- Otherwise, scan the whole repo, excluding: `node_modules`, `.git`, `dist`, `build`, `out`, `.next`, `.nuxt`, `vendor`, `target`, `__pycache__`, `.venv`, lock files, minified assets, and anything in `.gitignore` that looks like build output.
- Always announce the scope before scanning so the user can redirect you.
- Detect the stack early (look at manifest files: `package.json`, `Gemfile`, `requirements.txt`/`pyproject.toml`, `go.mod`, `Cargo.toml`, `composer.json`, `pubspec.yaml`, `mix.exs`). The stack drives which patterns matter and which dependency scanner to run.

---

## 1. Code/config checklist

Each item below is what to look for, why it matters, and a "bad → good" example where useful. A finding's severity comes from this section. Use `grep` / `rg` / `find` / `git log` — they work in every host. If you suspect a finding involves other files, grep for callers before flagging; speculation isn't a finding.

### Critical (the "embarrassing on HN" tier)

| Issue | Detection hint | Bad → Good |
|---|---|---|
| **Secrets committed to git** | Check `.env*` files tracked by git (`git ls-files \| grep -E '^\.env'`); scan working tree and history for high-entropy strings, `sk_live_`, `xoxb-`, `AKIA[0-9A-Z]{16}`, `-----BEGIN.*PRIVATE KEY-----`, `ghp_`, `glpat-`, `AIza[0-9A-Za-z_-]{35}`. Run `git log -p -S` on suspicious patterns. | `STRIPE_SECRET_KEY=sk_live_…` in `.env` → key rotated, `.env` removed from history, `.env.example` used as the tracked template. |
| **Plaintext passwords or weak hashing** | Grep for password column writes; confirm `bcrypt`/`argon2`/`scrypt`/`pbkdf2`. Flag `md5(`, `sha1(`, `===` against a `password` field, `plaintext_password`. | `User.password = md5(input)` → `User.password_digest = BCrypt::Password.create(input)`. |
| **SQL built by string concatenation/interpolation with user input** | Grep for query strings containing `${`, `+ req.`, `" % `, `f"…{` near `execute`/`query`/`raw`. Distinguish parameterized queries that use template strings for static SQL from genuine interpolation of untrusted input. | `db.query("SELECT * FROM users WHERE id = " + req.params.id)` → parameterized: `db.query("SELECT * FROM users WHERE id = $1", [req.params.id])`. |
| **XSS sinks with user input** | `innerHTML =`, `dangerouslySetInnerHTML`, `v-html`, Mustache `{{{ }}}`, `document.write`, `Element.outerHTML`, Rails `raw(`/`html_safe`, Django `\|safe`, Jinja `\|safe`. | `el.innerHTML = userBio` → text node + sanitizer (DOMPurify, sanitize-html, Rails `sanitize`). |
| **Code-execution sinks on user data** | `eval(`, `new Function(`, `setTimeout("string"`, `vm.runInNewContext`, `child_process.exec(` with interpolation, `pickle.loads`, `Marshal.load`, `unserialize(`, YAML `load` (not `safe_load`). | `eval(req.body.expr)` → a real parser, or remove the feature. |
| **Auth missing on admin/internal routes** | Find admin route files (`admin/`, `internal/`, `/api/admin`) and check each handler has auth middleware applied. Grep for `before_action :authenticate` / `requireAuth` / `@login_required` and find the gaps. | `app.post('/api/admin/users/:id/impersonate', handler)` → same line wrapped in `requireAdmin` middleware. |
| **IDOR — direct object reference without authorization** | Handlers that take a resource ID from params and query without scoping by the current user/tenant. Look for `Model.find(params[:id])` patterns. | `Invoice.find(params[:id])` → `current_user.invoices.find(params[:id])`. |
| **JWT with `algorithm: 'none'`, hardcoded weak secret, or `verify: false`** | Grep `jwt.verify`, `jwt.decode`, `algorithms`, `alg: 'none'`, `verify: false`, `secret = "..."` in JWT config. | `jwt.verify(token, secret, { algorithms: ['none', 'HS256'] })` → `algorithms: ['RS256']` (or single strong alg), secret from env. |
| **Open storage ACLs / public buckets** | In IaC (Terraform, CloudFormation, Pulumi), check S3/GCS bucket policies for `PublicRead`, `*` principals, `allUsers`. Signed URLs with no expiry. | `acl = "public-read"` on a user-uploads bucket → `acl = "private"` + presigned URLs with `Expires`. |
| **Debug / verbose errors enabled in production** | `DEBUG=True` in prod env, `app.debug = True`, `config.consider_all_requests_local`, framework error pages reachable. Look at prod env files and CI/deploy config. | `DEBUG=True` shipped → false in prod, generic error page returned, full trace only in logs. |
| **`Access-Control-Allow-Origin: *` with `Allow-Credentials: true`** | Grep CORS config for both together. Browsers block this combo, but APIs sometimes try to force it. | `origin: '*', credentials: true` → explicit allowlist of origins. |
| **File uploads with no extension/MIME/size check, served from auth origin** | Find upload handlers, check for `multer` limits / Rails `content_type` validation / size caps. Check the serving path: are uploads on the same origin as the app, no `Content-Disposition: attachment`? | Unbounded `<input type="file">` → size + MIME allowlist + served from a separate origin or with `Content-Disposition: attachment`. |

### Serious (should fix)

| Issue | Detection hint | Bad → Good |
|---|---|---|
| **Cookies missing `Secure`, `HttpOnly`, `SameSite`** | Grep cookie set sites: `Set-Cookie`, `res.cookie`, `cookies.signed`, `session_cookie`. | `res.cookie('sid', token)` → `res.cookie('sid', token, { httpOnly: true, secure: true, sameSite: 'lax' })`. |
| **Missing CSRF tokens** | Stacks that don't autoconfigure CSRF (raw Node, FastAPI, Flask without `flask-wtf`). Check that state-changing POSTs require an unguessable token. | Unprotected POST `/transfer` → CSRF middleware + token in form. |
| **Missing security headers** | `Strict-Transport-Security`, `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `X-Frame-Options` or CSP `frame-ancestors`. Look at server config / middleware (Helmet, `secure_headers`, Django `SecurityMiddleware`). | No `Strict-Transport-Security` → `max-age=31536000; includeSubDomains; preload`. |
| **HTTPS not enforced** | No HSTS, no HTTP→HTTPS redirect at the edge or in the app. | Plain `http://` redirect handler missing → 308 to `https://`. |
| **Sensitive data in logs** | Logger calls and request-logging middleware passing through password fields, tokens, full card numbers, SSNs, full email bodies. Look at Rails `filter_parameters`, Express morgan tokens, Sentry `beforeSend`. | `logger.info({ password })` → redact at the middleware level. |
| **Rate limiting absent** | No throttle on login, signup, password reset, OTP/email verification, webhook endpoints, public APIs. Look for `rack-attack`, `express-rate-limit`, `slowapi`, custom counters. | Unthrottled `/login` → IP + email throttle, lockout after N failures with backoff. |
| **Login enumeration** | Different responses for "wrong email" vs "wrong password"; user-existence leaks from `/forgot-password`. | "No account with that email" → "If that email matches an account, we sent a reset link." |
| **Password reset tokens** | Tokens with no expiry, no single-use enforcement, predictable (sequential or timestamp-based), sent over insecure channel. | 24h expiry, single-use, generated with a CSPRNG. |
| **SSRF** | Server-side `fetch`/`requests.get`/`Net::HTTP` on URLs from user input without an allowlist. Especially bad if it can reach the metadata service (`169.254.169.254`) or internal hosts. | Free-form `url` param → allowlist of hosts, block private IP ranges. |
| **Path traversal** | `path.join(__dirname, userInput)`, `File.read(params[:file])`, `open(user_path)` without normalization. | `path.join(uploads, filename)` → `path.resolve` + check it starts with the uploads dir. |
| **Unsigned webhooks** | POST endpoints accepting from external services (Stripe, GitHub, Slack) without verifying the signature header. | Raw body accepted → verify `Stripe-Signature` / `X-Hub-Signature-256` / `X-Slack-Signature`. |
| **Catastrophic-backtracking regexes on user input** | Obvious cases: nested quantifiers on user-controlled strings (`(.*)*`, `(a+)+`). | Replace with a non-backtracking engine or a bounded regex. |
| **Default/sample admin credentials shipped** | Seeds with `admin@example.com / password`, README walking through default creds. | Force a setup step that picks credentials before the app accepts traffic. |

### Moderate (consider fixing)

- TLS version pinned to 1.0/1.1 anywhere (load balancer, in-app HTTP clients).
- Permissive `Permissions-Policy` / `Feature-Policy`.
- Subresource Integrity missing on third-party `<script src>`.
- Auth tokens in `localStorage` where an `HttpOnly` cookie would do.
- `target="_blank"` without `rel="noopener noreferrer"`.
- Email sent over plain SMTP without STARTTLS.
- Outdated dependencies — see the dependency section.

---

## 2. Dependency scanning

Run the appropriate scanner for the stack — but only ones already installed. Never `npm install -g` or otherwise mutate the user's global env.

| Stack signal | Command | Notes |
|---|---|---|
| `package.json` + lockfile | `npm audit --json` / `pnpm audit --json` / `yarn npm audit --json` | Use the lockfile's package manager. |
| `Gemfile.lock` | `bundle exec bundler-audit check --format json` | Fall back to `bundle outdated` if bundler-audit isn't installed. |
| `requirements.txt` / `poetry.lock` / `Pipfile.lock` | `pip-audit --format=json` | Fall back to `safety check --json`. |
| `go.mod` | `govulncheck ./...` | |
| `Cargo.lock` | `cargo audit --json` | |
| `composer.lock` | `composer audit --format=json` | |

**Rules:**
- Parse the scanner's JSON. Surface only High/Critical CVEs in the main report. Medium/Low go in a collapsible block.
- Use the upgrade path the scanner already provides. Don't invent versions.
- If a scanner is missing, print the one-liner to install it (`npm install -g pip-audit` etc.) and move on — don't run installs yourself.
- If `npm audit` etc. fails for a reason other than findings (e.g. private registry not reachable), report the failure rather than swallowing it.

---

## 3. Promise audit

This half is the part that turns a generic security scan into something real. The goal: find every specific security/privacy claim the product makes to users, and check the code does it.

### Step A — Find the claims

In priority order:

**1. In-codebase legal/security/marketing pages.** Look for routes/files matching `/privacy`, `/privacy-policy`, `/terms`, `/terms-of-service`, `/security`, `/trust`, `/dpa`, `/gdpr`, `/ccpa`, `/legal/*`. Common locations:

- Next.js / Nuxt / Astro: `app/privacy/page.{tsx,mdx}`, `pages/privacy.*`, `content/legal/*.md`.
- Rails: `app/views/pages/privacy.*`, route entries in `config/routes.rb`.
- Django: `templates/legal/*`, URL conf entries.
- Phoenix/Laravel/Rails APIs paired with a marketing SPA: check both repos if accessible.
- Static sites: `public/privacy.html`, `static/legal/*`.
- Monorepos: `apps/marketing/`, `sites/landing/`, `content/legal/`.

**2. In-product copy.** Search templates, components, and marketing copy for security/privacy claim strings:

```
bcrypt, argon2, scrypt, pbkdf2, AES-256, AES-GCM, TLS, SSL, "end-to-end encrypted", "end to end encrypted", "encrypted at rest", "encrypted in transit", "industry-standard encryption", "bank-level security", "military-grade", "zero-knowledge", "we never sell", "we don't sell", "we will never sell", "your data is yours", "you own your data", 2FA, "two-factor", "two factor", MFA, SSO, "single sign-on", SAML, "audit log", "audit trail", "delete your data", "export your data", "right to be forgotten", SOC 2, SOC2, HIPAA, GDPR, CCPA, ISO 27001, PCI DSS
```

**3. External marketing site (only if not found in-codebase).** Sometimes the marketing site lives in a separate repo. Discover candidate URLs from:

- `package.json` `homepage`, `repository.url`.
- `README.md` first-paragraph links, badges, screenshots.
- `.env*.example` for `NEXT_PUBLIC_SITE_URL`, `APP_URL`, `MARKETING_URL`, `PUBLIC_URL`.
- `<meta property="og:url">`, canonical `<link>` tags in shared layout files.
- `robots.txt` Sitemap directive, `sitemap.xml`.
- Email sender domains in transactional email config.

If you find a candidate site (e.g. `https://example.com`), show the URLs you'd fetch (`/privacy`, `/terms`, `/security`, `/trust`) **and ask the user before fetching.** Use the host environment's structured question primitive (e.g. Claude Code's `AskUserQuestion`, Codex plan mode, Cursor inline suggestions). Otherwise ask in plain text. Once confirmed, fetch with `curl -sL --max-time 10 <url>`, or the host's fetching tool if available.

If no promise sources turn up at all, say so plainly and skip the promise audit — don't fabricate claims.

### Step B — Extract the claims and test them

Pull out concrete, testable promises. Each row below is a claim type the audit should test against code evidence:

| Claim | What "backed by code" means |
|---|---|
| "Passwords are encrypted" / "hashed with bcrypt" | Password column writes go through `bcrypt`/`argon2`/`scrypt`. No `md5`, `sha1`, plaintext compare. |
| "All data is encrypted at rest" | DB-level or column-level encryption configured. Look for Active Record `encrypts`, Django field-level encryption, `pgcrypto`, cloud KMS usage, infra-as-code that enables encrypted volumes. At minimum, the production DB has encryption-at-rest enabled by the host — confirm and document. |
| "Encrypted in transit" / "uses TLS" | HSTS header set, HTTP→HTTPS redirect at the edge, no hardcoded `http://` for own domain. |
| "End-to-end encrypted" | A strong claim. Verify keys are generated/held client-side and the server can't decrypt. If TLS is the only encryption, this claim is **wrong** — flag as Critical. |
| "2FA / MFA" | A real 2FA setup flow exists in account settings, not a stub. TOTP enrollment, backup codes, recovery flow. |
| "SSO / SAML / SCIM" | Find SAML/OIDC code paths and identity-provider config. |
| "Audit log" / "audit trail" | An audit-log write happens on sensitive events (login, permission changes, data export, billing changes). |
| "We never sell your data" | List every third-party SDK that ships user data off-platform (analytics, ads, session replay, support chat). Don't accuse — list and let the user judge whether any of those vendors monetize what they receive. |
| "GDPR / CCPA compliant" / "export and delete your data" | Data-export endpoint or job exists. Data-delete endpoint or job exists and actually deletes (not just soft-deletes / anonymizes — check what the claim actually says). |
| "SOC 2 / HIPAA / ISO 27001 / PCI DSS" | These are organizational certifications, not code. Note as out-of-scope for a code audit, but flag if claimed without the supporting infra a real audit would require (centralized audit logging, MFA, access controls, encryption). |
| "Bank-level encryption" / "military-grade" | Vague marketing phrase. The closest code-backable version is TLS 1.3 + AES-256 at rest. Flag as Moderate if either is missing. |

### Step C — Flag mismatches

Each promise → code-evidence finding goes under the `## Promise audit` section of the report. Severity rules:

- Claim with **no code backing whatsoever** → **Critical** (this is the highest-blast-radius category — broken promises about security are exactly what makes the news).
- Claim with **partial or weak backing** → **Serious**.
- Claim that's true but **could be stronger or better documented** → **Moderate**.

---

## 4. Report format

One report, severity sections sorted Critical → Serious → Moderate, then dependency CVEs, then the promise audit. Plain markdown so it renders in any host.

```markdown
# Security audit

**Scope:** `<path or "whole repo">`
**Stack detected:** Node 20 + Postgres + Stripe + Sentry
**Found:** N findings — C critical · S serious · M moderate · D dependency CVEs
**Top risk:** <one sentence on the worst finding>

---

## Critical (N)

### 1. `.env` committed to git with live Stripe secret key
- **Why it matters:** Anyone with repo read access has your Stripe live secret. Public repo or a contractor offboarding = key compromised.
- **Evidence:** `.env:14` — `STRIPE_SECRET_KEY=sk_live_…`; tracked since commit `a1b2c3d` (2024-08-01).
- **Fix:** (1) Rotate the key in Stripe dashboard now. (2) Remove `.env` from the working tree, add to `.gitignore`. (3) Purge from history with `git filter-repo --path .env --invert-paths` (one-time op; coordinate if others have clones).
- **Breakage risk:** None for the rotate itself. History rewrite requires every collaborator to re-clone.

### 2. ...

## Serious (N)
...

## Moderate (N)
...

## Dependency CVEs

### High / Critical (X)
| Package | Installed | Patched | CVE | Advisory |
|---|---|---|---|---|
| lodash | 4.17.15 | 4.17.21 | CVE-2021-23337 | https://… |

<details><summary>Medium / Low (Y) — click to expand</summary>

| Package | Installed | Patched | CVE | Advisory |
|---|---|---|---|---|
| … | … | … | … | … |

</details>

---

## Promise audit

### Claims found
- `/privacy` (in-codebase, `app/privacy/page.mdx`): "encrypted at rest", "we never sell your data"
- `/security` (fetched from `example.com/security` with user consent): "SOC 2 Type II", "2FA available on all plans"
- In-app copy (`components/Pricing.tsx:42`): "bank-level encryption"

### Mismatches
- **Critical — "2FA available on all plans"** but no 2FA enrollment flow exists in `app/settings/`. Either build it or remove the claim from `/security`.
- **Serious — "encrypted at rest"** has no code-level evidence. Production DB encryption-at-rest is determined by the hosting provider — confirm it's on in your DB host config and either (a) note that in the privacy policy as the source of the encryption claim, or (b) add column-level encryption for PII fields.
- **Moderate — "bank-level encryption"** is vague. TLS 1.3 is in place (good); at-rest claim is unbacked (see above). Either remove the phrase or define what it means specifically.
```

If the report would be very long, lead with the summary line + the Critical section, then say "scroll for serious, moderate, deps, promises" — don't truncate.

---

## 5. Hand off

This skill produces a report. It does **not** invent its own interactive command loop. The host tool already has primitives for confirming destructive changes — use them, don't simulate them.

After the report, ask the user what they want done — using whatever native question/confirmation primitive the host environment provides. Use these in priority order:

1. **If a structured question tool is available** (e.g. Claude Code's `AskUserQuestion`), use it. Offer concrete choices: "Apply all critical fixes", "Apply all (critical + serious + moderate)", "Pick specific findings to apply", "Don't apply anything — I'll review the report".
2. **If the host has a plan/approval mode** (e.g. Codex plan mode), present the proposed edits as a plan and let the host's approval flow handle confirmation.
3. **Otherwise, ask in plain text.** End the report with a single, clear question: *"Want me to apply any of these? Reply with the finding numbers (e.g. 'fix 1, 3, 5'), 'all critical', 'all', or 'no'."*

Don't pretend a pseudo-CLI prompt is real interactivity. The user should see one report + one question, not a fake loop.

### What's safe to auto-apply

If the user picks fixes, apply them using the host's standard editing tools. Some changes are safe to apply in one shot; some require the user to act outside the skill.

**Safe to apply:**
- Adding security headers (HSTS, CSP starter, `X-Content-Type-Options`, `Referrer-Policy`, `X-Frame-Options` / CSP `frame-ancestors`).
- Setting cookie flags (`Secure`, `HttpOnly`, `SameSite`).
- Adding `rel="noopener noreferrer"` to `target="_blank"` links.
- Tightening `.gitignore` to exclude `.env*` going forward.
- Replacing `md5(password)` with a proper KDF call **only if** the surrounding code is greenfield (no existing users with md5 hashes). Otherwise this is a migration, not an edit — flag it and stop.
- Adding webhook signature verification when the upstream signature header is already documented.
- Replacing string-concatenated SQL with parameterized queries when the change is mechanical and the test surface covers it.

**Not safe to auto-apply — call out and hand to the user:**
- Rotating leaked secrets (the actual rotation happens in the third party's dashboard).
- `git filter-repo` / history rewrites.
- DB schema changes (hashing legacy passwords, adding encryption to existing columns).
- Building new flows (2FA enrollment, data export, audit log).
- Changes to production infrastructure / IaC that touch live systems.
- CSP that isn't a permissive starter (a real CSP needs a report-only rollout).
- Tightening CORS, forcing TLS, changing session handling — these can lock users out mid-session.

Whenever you skip a fix because of breakage risk, say so in one line and move on. Don't try to half-fix.

If the user picks a subset, treat the un-picked findings as deliberately deferred — don't nag, don't reprint them, just complete the chosen ones.

After applying, show a one-block summary: which findings were applied, which files were touched, and what's still outstanding (rotate-secret, history-rewrite, etc.) so the user has a clean to-do.

---

## 6. Guidelines

- **Don't perform thoroughness.** A clean audit with no critical findings is a real outcome. Don't manufacture findings to fill out severity buckets.
- **Match the bar of the surrounding code.** A 500-line personal project doesn't need CSP. A B2B SaaS handling customer data does. Same rule as the rest of this user's audit skills.
- **Don't propose breakage silently.** Every finding includes a `Breakage risk:` line. If a fix would log out all users, invalidate sessions, lock out admins, force a re-auth, or change a public API — say so.
- **Don't moralize.** "You should really care about security" is noise. The finding is the message.
- **Don't roll your own confirmation loop.** Hand off to the host's primitive.
- **No theater.** Every finding has to be defensible — name the actual attack, the actual data leak, or the actual broken promise. "Consider adding more validation" is not a finding.
- **Don't refactor.** Fix the security issue and stop. Don't reorganize the auth module on the way out.
- **Read the file before flagging.** A `dangerouslySetInnerHTML` on a literal string is fine. An `eval` on a config-file constant is fine. Context decides; the grep hit doesn't.
- **Group duplicates.** Same finding in 14 call sites = one finding with 14 locations, one fix batch. Don't make the user say `y` 14 times.
- **Promise audit is the highest-blast-radius half.** A code bug embarrasses you. A broken promise embarrasses you on the front page. Treat it that way.
