---
name: seo-sprint
description: "When the user wants to build organic search traffic from scratch on a site or app — a multi-phase SEO sprint that ships programmatic landing pages (alternatives, use-cases, comparisons, playbooks) backed by real keyword research. Also use when the user mentions 'SEO sprint,' 'build organic traffic,' 'rank in Google,' 'build an SEO machine,' 'alternatives pages,' 'comparison pages,' '/for/ pages,' 'programmatic SEO playbook,' 'competitor alternatives,' or vague asks like 'help me with SEO' or 'we need traffic' when there's no existing roadmap — this skill creates one and then executes it phase by phase. Distinguish from seo-audit (one-off diagnostic) and programmatic-seo (page-template scaling only) — this skill owns research, planning, page generation, internal linking, off-page checklist, and the persistent phase tracker across the whole sprint."
---

# SEO Sprint

An end-to-end playbook for building organic search traffic on any app or marketing site, from day-0 setup through ongoing programmatic page expansion. Conversion-first (alternatives and comparison pages before blogs), publish-early (a thin page indexed now beats a perfect page indexed later), and battle-tested across dozens of real production phases.

The skill has two modes:

1. **Initialize** — first run on a new repo. Detects stack and brand, runs keyword research via Ahrefs, audits technical foundations, generates a persistent roadmap doc and link inventory.
2. **Resume** — subsequent runs. Reads the existing roadmap, picks up the next pending phase (or one the user names), executes it end-to-end with quality gates.

The user does not need to remember which mode they're in. On invocation, **detect mode by checking whether `docs/seo-sprint.md` exists at repo root** (or wherever the user chose to put it last time — see `.seo/config.json` if present). No file → Initialize. File exists → Resume.

---

## Hard prerequisites

Before doing anything, verify:

1. **Working directory is a git repo** — `git rev-parse --is-inside-work-tree`. If not, stop and say: "This skill writes a persistent roadmap to your repo. Initialize git first, or run from inside an existing repo."
2. **A stack is detectable** — at least one of `package.json`, `Gemfile`, `composer.json`, `requirements.txt`, `astro.config.*`, `next.config.*`, `nuxt.config.*`, `gatsby-config.*`, `_config.yml`, `config.toml`, `pyproject.toml`. If none, ask the user what stack they're on before continuing.
3. **Ahrefs MCP availability check** — try a small ping: `mcp__ahrefs__subscription-info-limits-and-usage` (no args). If it returns data, you're good. If it errors, the skill works in **manual research mode** — see `references/manual-research.md`. Don't refuse to run; just adjust.

If Ahrefs is available but no `project_id` is on file, prompt the user for one. Store it in `.seo/config.json`. You'll reuse it dozens of times.

---

## Initialize mode — first run

Goal: end this run with a committed (or at least written) `docs/seo-sprint.md` that contains keyword research, a phase tracker grouped by pattern, and the technical audit findings. Plus `.seo/brand.md` (product context) and `.seo/link-inventory.md` (every internal link target available).

### Step 1 — Detect stack + frontend convention

Read `references/stacks/detection.md` for the full signal table. Identify:

- **Framework family**: Rails+Inertia, Next.js (App Router vs Pages), Astro, Nuxt, Remix, SvelteKit, Hugo, Jekyll, plain HTML, or "unknown → markdown fallback."
- **Routing convention**: file-based (Next/Astro/Nuxt) or controller-based (Rails/Django/Laravel).
- **Component language**: TSX, JSX, `.astro`, `.svelte`, `.vue`, `.erb`, or plain HTML.
- **Existing marketing pages**: `git ls-files | grep -iE 'marketing|landing|pages/(home|about|pricing)'`. Note what exists — you'll link to it.

Confirm with the user using `AskUserQuestion` if any signal is ambiguous. Save to `.seo/config.json`.

### Step 2 — Detect brand + product context (hybrid)

Read every signal first, propose `.seo/brand.md`, then ask only about gaps. Signals to read:

- `CLAUDE.md`, `README.md`, `README` — name, one-liner, audience hints
- `package.json` or `Gemfile.lock` — framework version (informs the stack adapter)
- `tailwind.config.*` or design-token CSS file — accent color, fonts
- `app/views/marketing/*`, `pages/index.*`, `src/pages/index.*` — existing hero copy
- `pricing` page (any path) — plan structure, price points

Then use a single `AskUserQuestion` call (3-4 questions) to fill gaps. Required to know:

- **Product one-liner** (≤20 words)
- **Primary persona** (e.g. "B2B SaaS founder," "indie agency owner," "ecommerce ops manager")
- **3-7 direct competitors** (by name; you'll Ahrefs them next)
- **Brand voice tags** (e.g. "honest, technical, no-jargon, slightly irreverent")
- **Whether the product has a free tier** (drives "is [brand] free" keyword strategy)
- **Anti-positioning** — what you do NOT do (used for honest comparison sections later)

Write to `.seo/brand.md` using `assets/brand-template.md` as the skeleton.

### Step 3 — Run keyword research

If Ahrefs MCP is available: follow `references/ahrefs-recipes.md`. The exact recipes:

- **A. Domain rating + baseline.** `mcp__ahrefs__site-explorer-domain-rating` + `site-explorer-metrics` on the user's domain. Record DR. **This is the single most important number** — it caps which keywords are winnable. KD ≤ DR + 5 is the heuristic; if DR < 10, restrict to KD ≤ 30 until your DR climbs.
- **B. Competitor reverse-lookup.** For each competitor name the user gave: `site-explorer-organic-keywords` (top 50 by volume, filtered to KD ≤ DR+20). This is where most of your `/alternatives/[competitor]` candidates come from. Capture `volume`, `kd`, `traffic_potential`, and the SERP for the competitor brand term itself.
- **C. Use-case keyword sweep.** For the persona + product one-liner: `keywords-explorer-matching-terms` with seeds like "[product category] for [audience]", "[verb] [object]" patterns. Filter to KD ≤ DR+10, volume ≥ 30. These become Pattern B/C candidates.
- **D. Comparison volume.** For each competitor pair where both are in the user's list: `keywords-explorer-overview` on "[a] vs [b]". Volumes are usually small (30-500) but conversion intent is the highest of any pattern.
- **E. Striking-distance audit (existing sites only).** `mcp__ahrefs__gsc-keywords` filtered to position 5-20. These are pages one push from rank-3. Each becomes a boost task.

If Ahrefs MCP is unavailable: follow `references/manual-research.md` — ask the user to paste data from their existing Ahrefs/Semrush UI, or do a web-search-based competitor reverse-lookup.

Save the raw research output to `.seo/keyword-research.json` (you'll reference it from every phase). Save the curated, decision-ready summary to the **Keyword Research Appendix** section of `docs/seo-sprint.md`.

### Step 4 — Run the technical foundations audit

Read `references/technical-audit.md` and run the four checks (sitemap, robots, meta-tag uniqueness, schema). Use `scripts/tech_audit.py` if you have shell access — it's deterministic and faster than eyeballing.

Whatever it finds (missing sitemap, duplicate meta titles, no JSON-LD schema, blocked routes) becomes **Phase 0** in the roadmap. Don't gloss over this — the day-0 crawl shapes Google's understanding of the site for months. Retroactive fixes are harder than getting it right the first time.

### Step 5 — Generate the roadmap doc

Take `assets/roadmap-template.md` and fill in:

- **Site facts** (domain, DR, stack, brand colors, fonts) from steps 1-2
- **Reference data** section (paths to controllers/page files the skill will edit)
- **Keyword Research Appendix** (the curated output from step 3, grouped by pattern)
- **Phase Status Tracker** — auto-populate from research:
  - Phase 0: Technical foundations fixes (whatever the audit found)
  - Phase 1+: One row per page candidate in priority order
  - Group by pattern. Within a pattern, order by `traffic_potential` desc, then `volume` desc, then `KD` asc
  - Striking-distance boosts go near the front of the tracker — they're the fastest wins on existing sites
  - Off-page checklist (directories + outreach targets) gets one tail-end phase per category

Write to the configured path (default `docs/seo-sprint.md`). Print the path back to the user and recommend they review it before kicking off Phase 0.

### Step 6 — Generate `.seo/link-inventory.md`

This is the file every phase reads to pick internal links. Use `assets/link-inventory-template.md`. Pre-populate from what the repo already has — any URL the skill itself might link to (existing features, tools, pricing, blog posts). Each phase will append to it as new pages ship.

### Step 7 — Hand off

End the Initialize run with a clear status message:

```
✓ Initialize complete

  Roadmap:        docs/seo-sprint.md
  Brand context:  .seo/brand.md
  Keyword cache:  .seo/keyword-research.json
  Link inventory: .seo/link-inventory.md
  Config:         .seo/config.json

Next: review docs/seo-sprint.md, then run me again to execute Phase 0 (technical foundations).
```

Do not auto-execute Phase 0. The user should eyeball the roadmap first — pattern priorities and competitor lists are decisions worth a human pass.

---

## Resume mode — subsequent runs

Goal: pick a phase from the tracker, execute it end-to-end with quality gates, hand back so the user can commit/PR.

### Step 1 — Read state

- `docs/seo-sprint.md` (or the configured path) — load the full doc into context
- `.seo/brand.md` — voice + positioning + anti-positioning
- `.seo/link-inventory.md` — internal link targets
- `.seo/config.json` — stack + Ahrefs project_id + path config
- Locate the **Phase Status Tracker**. Identify the next `pending` phase with the lowest number. Print it back to the user along with the 2 phases after it.

### Step 2 — Confirm scope

Use `AskUserQuestion` once:

- "Continue with Phase N: [title]?"
- Options: "Yes, start Phase N" / "Pick a different phase" / "Re-audit (re-run research)" / "Just show me the tracker"

If they pick "different," show pending phases as options and let them choose. If "re-audit," loop back into Initialize-mode steps 3-5 (refresh keyword data, regenerate tracker). If "just show," print the tracker and stop.

### Step 3 — Execute the phase

The execution pattern depends on the phase type. Phases fall into one of these buckets:

| Phase type | Reference |
|---|---|
| Technical foundations (Phase 0) | `references/technical-audit.md` |
| Pattern A — `/alternatives/[competitor]` | `references/patterns/alternatives.md` |
| Pattern B/C — `/for/[use-case]` or `/for/[audience]` | `references/patterns/use-case.md` |
| Pattern D — `/compare/[a]-vs-[b]` | `references/patterns/compare.md` |
| Pattern E — `/playbooks/[topic]` long-form | `references/patterns/playbooks.md` |
| Striking-distance boost | `references/striking-distance.md` |
| Off-page checklist phase | `references/off-page.md` |
| Internal-link spine audit | `references/quality-bars.md` (link-audit section) |

For page-generating phases, the flow is always:

1. **Re-research** what's freshly needed (current competitor pricing, feature changes) — Ahrefs MCP if available, web search otherwise. Don't trust 60-day-old cached data on commercial intent terms.
2. **Generate the page payload** following the pattern reference. The output format depends on the stack — see `references/stacks/<framework>.md` for native-code output, or `references/stacks/markdown-fallback.md` for portable markdown.
3. **Verify** against the quality bar (word count, internal links, schema, honesty section for alts). Run `scripts/word_count.py` and `scripts/link_audit.py`. If a check fails, fix it — don't ship under-spec work.
4. **Update `.seo/link-inventory.md`** with the new page so future phases can link to it.
5. **Update the tracker row** in `docs/seo-sprint.md` — status `completed`, with PR ref or commit SHA. Do this in the same edit batch as the page work so reviewers see both in one diff.

### Step 4 — Hand off (do NOT auto-commit unless asked)

End with a concrete next-step message:

```
✓ Phase N complete: [title]

Files changed:
  app/controllers/marketing_controller.rb  (added entry)
  app/frontend/pages/Alternatives/Show.tsx (no change — uses existing layout)
  docs/seo-sprint.md                       (tracker updated)
  .seo/link-inventory.md                   (new page registered)

Quality gates:
  ✓ Word count: 712 / 600 min
  ✓ Internal links: 2 alts, 1 feature, 1 tool
  ✓ FAQ JSON-LD attached
  ✓ Honesty section present (3 rows)

Suggested commit: "SEO Phase N: ship /alternatives/[slug]"

Next phase pending: Phase N+1 — [title]
```

Ask whether to open a PR only if the user has expressed they want that cadence. Otherwise let them drive git.

---

## Interactive principles

This skill is in front of the user for a long time. Every phase = another interaction. Make it feel like a thoughtful collaborator, not a checklist drone.

- **Ask before you guess.** When picking which competitor to fan out next, or which use-case to write up next, surface the top 3 candidates with their volume/KD/TP numbers and let the user pick. Their gut on positioning often matters more than the data.
- **Show the data.** Don't say "I'm going to target [keyword] because the data is good." Say "Targeting [keyword] — vol 400, KD 12, traffic potential 1,800. Closest competitor in top-10 is [DR 24 site] which we can outrank." Concrete numbers build trust.
- **Be honest about what won't work.** If a user picks a head term they can't rank for (KD 80 with DR 8), say so plainly and offer the closest winnable alternative. Don't humor a doomed phase.
- **Don't lecture.** The user does not need a recap of SEO theory every phase. State the action and the why-this-matters in one sentence each, then proceed.
- **Surface tradeoffs, not opinions.** When the user could go either way (e.g. "do we ship 5 thin alternatives pages or 2 deep ones first?"), state the tradeoff in one line and ask. Don't sermonize.

---

## When to use vs adjacent skills

- **seo-audit** is for a one-off diagnostic on an existing site. If the user just wants "what's wrong with my SEO," use that. This skill is for forward execution.
- **programmatic-seo** is the page-templating-only subset. If the user already has a roadmap and just wants help building one specific template, that's a better fit. This skill *contains* programmatic SEO as Patterns A/B/C/D.
- **content-strategy** is for content planning that isn't search-driven. This skill is search-driven; if the user wants to plan content around brand themes regardless of search demand, content-strategy is the better fit.
- **blog-article** writes individual blog posts. This skill explicitly does not write blogs — for any phase that needs a blog post, hand off to blog-article and resume after.
- **ai-seo** is for LLM-citation optimization (entity mentions, FAQ formatting for AI overviews). Worth running after this sprint reaches ~10 indexed pages.

---

## Anti-patterns (do NOT do these)

- **Don't target head terms when DR is low.** "Social media management tool" at DR 8 is wasted work. Stick to KD ≤ DR+10 while your DR is still low. The roadmap should reflect this.
- **Don't ship pages without internal links from existing pages.** A new page nobody links to is an island. Always add ≥2 inbound links from existing pages in the same phase.
- **Don't ship without the "honesty" section on alternative pages.** Three honest tradeoffs (where the competitor wins) is non-negotiable. It's a brand signal AND a Google quality signal. Pages without it underperform measurably.
- **Don't skip the schema markup.** FAQPage JSON-LD captures featured snippets. Article JSON-LD for playbooks. SoftwareApplication for the homepage and use-case pages. Quality gate fails without them.
- **Don't auto-commit.** Always show the diff summary and let the user commit. They may have local conventions (signed commits, hook config, branch naming) you don't know about.
- **Don't restart Initialize mode if a partial state exists.** If `.seo/config.json` exists but the roadmap doesn't, prompt the user — they may have started and aborted, and you should pick up rather than wipe.

---

## File map

| File | Purpose |
|---|---|
| `references/methodology.md` | The why-this-works theory + lessons from 49 real phases |
| `references/ahrefs-recipes.md` | Exact MCP queries for every research task |
| `references/manual-research.md` | Ahrefs-free fallback workflow |
| `references/patterns/alternatives.md` | Pattern A spec, data shape, quality bar, example |
| `references/patterns/use-case.md` | Pattern B+C spec |
| `references/patterns/compare.md` | Pattern D spec |
| `references/patterns/playbooks.md` | Pattern E spec (2,500-word bar) |
| `references/stacks/detection.md` | Stack detection signal table |
| `references/stacks/rails-inertia.md` | Native code patterns (this skill's reference implementation) |
| `references/stacks/nextjs.md` | Next.js App Router adapter |
| `references/stacks/astro.md` | Astro content-collection adapter |
| `references/stacks/markdown-fallback.md` | Universal markdown output format |
| `references/technical-audit.md` | Phase 0 audit recipes (sitemap, robots, meta, schema) |
| `references/striking-distance.md` | GSC pos 5-20 audit + boost recipe |
| `references/off-page.md` | Backlink checklist + Ahrefs competitor-referring-domains research |
| `references/quality-bars.md` | Verification spec per pattern |
| `scripts/word_count.py` | Strip markup → word count |
| `scripts/link_audit.py` | Verify internal-link minimums per page |
| `scripts/tech_audit.py` | Sitemap.xml + robots.txt + meta tag scanner |
| `assets/roadmap-template.md` | `docs/seo-sprint.md` skeleton |
| `assets/brand-template.md` | `.seo/brand.md` skeleton |
| `assets/link-inventory-template.md` | `.seo/link-inventory.md` skeleton |
