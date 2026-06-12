# Ahrefs MCP Recipes

Every research task this skill performs maps to a specific MCP tool call (or short pipeline of calls). The exact parameters matter — Ahrefs results swing wildly on `mode`, `country`, `volume`/`difficulty` filters. These recipes are the working set the skill should reuse.

**Project ID** lives in `.seo/config.json` as `ahrefs_project_id`. Many endpoints don't need it (site-explorer takes a `target` domain directly), but a few do (`gsc-*`, `rank-tracker-*`, `site-audit-*`).

Before running any recipe for the first time in a session, call:
```
mcp__ahrefs__doc (with the tool name) — get current parameter docs
```
The Ahrefs API evolves; cached parameter knowledge goes stale fast.

---

## Recipe A — Baseline: where are we starting?

**Goal:** know the domain's DR and ranking baseline before doing anything. DR caps which keywords are reachable.

```
1. mcp__ahrefs__site-explorer-domain-rating
   target: <user-domain>
   → captures current DR
2. mcp__ahrefs__site-explorer-metrics
   target: <user-domain>
   mode: subdomains  (or domain if no subdomain)
   → captures organic_traffic, organic_keywords, referring_domains
3. mcp__ahrefs__site-explorer-organic-keywords
   target: <user-domain>
   limit: 100
   order_by: traffic:desc
   → captures what the site already ranks for (baseline for striking-distance)
```

Save to `.seo/keyword-research.json` under `baseline.*`. Print summary back to user: "Your domain is DR X with N keywords ranking. Highest-traffic page is /<path> for [keyword] at position Y."

**Heuristic:** if DR < 10, restrict all future research to KD ≤ 30. If DR 10-20, KD ≤ 40. If DR 20-30, KD ≤ 50. State this cap to the user and apply it as a filter on all subsequent recipes.

---

## Recipe B — Competitor reverse-lookup

**Goal:** find which keywords each named competitor ranks for that you could also target. Most `/alternatives/[competitor]` candidates come from here, plus a lot of `/for/[use-case]` ideas.

For each competitor name the user provided in `.seo/brand.md`:

```
mcp__ahrefs__site-explorer-organic-keywords
target: <competitor-domain>
limit: 100
order_by: traffic:desc
where: kd <= <DR_CAP>  AND  volume >= 30
country: us
```

Then optionally:

```
mcp__ahrefs__site-explorer-top-pages
target: <competitor-domain>
limit: 30
order_by: traffic:desc
```

This reveals which page templates the competitor invested in. If they have 50 `/integrations/*` pages and that pattern matches your product, consider adding it.

**Output to capture per competitor:**
- Brand-term volume + KD (the keyword that's just their brand)
- Top 20 non-brand keywords they rank for (skip brand variations like `[brand] login`, `[brand] pricing` UNLESS pricing volume is high)
- Top 10 pages by traffic (informs which patterns to copy)

Save to `.seo/keyword-research.json` under `competitors.<name>.*`.

---

## Recipe C — Use-case / audience keyword sweep

**Goal:** find Pattern B and Pattern C candidates — "[verb] [object]" and "[product category] for [audience]" keywords.

```
mcp__ahrefs__keywords-explorer-matching-terms
keywords: ["<seed-1>", "<seed-2>", ...]
limit: 200
where: kd <= <DR_CAP>  AND  volume >= 30  AND  parent_topic_volume >= 100
country: us
order_by: traffic_potential:desc
```

Seed strategy:
- Take the product one-liner: e.g. "social media monitoring tool"
- Generate seeds: ["social media monitoring", "twitter monitoring", "social listening", "brand monitoring", "competitor monitoring"]
- For audience seeds: ["[category] for agencies", "[category] for ecommerce", "[category] for freelancers"]

Run two queries: one for use-case seeds, one for audience seeds. Capture `volume`, `kd`, `traffic_potential` (TP), `parent_topic`. **Traffic potential is the single most useful metric here** — a KD 22 keyword with TP 2,500 is a better target than a KD 8 keyword with TP 200.

Group results by parent_topic. Each parent_topic cluster is roughly one `/for/*` page candidate.

Save to `.seo/keyword-research.json` under `use_cases.*` and `audiences.*`.

---

## Recipe D — Comparison volume check

**Goal:** validate Pattern D candidates. Run before adding any `/compare/[a]-vs-[b]` row to the tracker.

For each candidate competitor pair (both from the brand list):

```
mcp__ahrefs__keywords-explorer-overview
keywords: ["<a> vs <b>", "<b> vs <a>"]
country: us
```

Capture `volume` and `cpc` (CPC is a strong commercial-intent proxy here). Comparison volumes are typically 30-500; anything ≥30 is worth a page since intent is so high. **Build one page per canonical pair** (alphabetical: `a-vs-b`) — both directional searches will rank for the same URL.

Save to `.seo/keyword-research.json` under `comparisons.*`.

---

## Recipe E — Striking-distance audit

**Goal:** find pages ranking position 5-20 in GSC that could be boosted to top-5 with content depth + internal links. Often the highest-leverage phases on any existing site.

```
mcp__ahrefs__gsc-keywords
project_id: <project_id>
date_from: <90-days-ago>
date_to: <today>
where: position >= 5 AND position <= 20 AND impressions >= 50
limit: 100
order_by: impressions:desc
```

For each result, also pull the page:

```
mcp__ahrefs__gsc-pages
project_id: <project_id>
date_from: <90-days-ago>
date_to: <today>
where: position >= 5 AND position <= 20
order_by: clicks:desc
limit: 50
```

Cross-reference to identify which **page** owns which **keyword** at striking distance. Each (page, keyword) pair becomes a candidate boost phase.

**Output:** prioritized list of striking-distance opportunities, each with:
- URL
- Keyword
- Current position
- Impressions (last 90d)
- Estimated traffic at position 3 (use the rough CTR table: pos 1 ≈ 30%, pos 3 ≈ 11%, pos 10 ≈ 2%, pos 15 ≈ 1%)
- Boost strategy: more depth, FAQ section, new H2, or new internal links from sibling pages

Save to `.seo/keyword-research.json` under `striking_distance.*`. **Front-load these in the tracker** — they're faster than shipping new pages and use existing link equity.

---

## Recipe F — Backlink target research (off-page phase)

**Goal:** find directories, guest-post candidates, and link-trade prospects ranking for the niche. The Ahrefs "backlinks competitors rank for" pattern.

For each top competitor:

```
mcp__ahrefs__site-explorer-referring-domains
target: <competitor-domain>
limit: 100
where: domain_rating >= 30 AND traffic_dofollow >= 100
order_by: domain_rating:desc
```

Filter results manually for:
- **Directory candidates** — site name matches "[category] directory", "[category] tools", "alternatives", "list", "vs"
- **Guest post candidates** — looks like a blog (e.g. `*.com/blog/...` paths), DR 30+
- **Tool roundup candidates** — sites that rank for "best [category]" keywords
- **Skip** — generic press release sites, paid-link networks, irrelevant niches

For each candidate, capture: domain, DR, traffic, what they link to about the competitor (informs the pitch angle).

Save to `.seo/backlink-targets.json` and use it to populate the off-page checklist in the roadmap.

---

## Recipe G — SERP overview (for any keyword you're targeting)

**Goal:** before targeting a keyword, look at who's ranking and whether you can realistically displace them.

```
mcp__ahrefs__serp-overview
keyword: "<target keyword>"
country: us
```

Read the top 10. Heuristics:
- If 5+ of top 10 are DR > 60 and they're not generic pages but real tool/comparison content → skip, you can't displace them
- If 3+ of top 10 are DR < 30 → strong signal you can rank
- If top 3 are all Reddit/Quora threads → strong signal, you'll outrank them with a real page
- If top 1-2 are direct brand pages (Buffer.com for "buffer") → that's table-stakes, the question is about positions 4-10

**Run this for any tracker phase before starting work.** Catches doomed phases early.

---

## Tool-result rendering

Many Ahrefs MCP tool results include `render_with` metadata. **You must call the specified render tool** to display the data — don't summarize raw JSON. Common ones:
- `mcp__ahrefs__render-data-table` for keyword lists
- `mcp__ahrefs__render-time-series-chart` for DR / traffic history
- `mcp__ahrefs__render-scorecard` for top-level metric callouts

Show the rendered output back to the user. They want to see the data, not your interpretation of it.

## Common cost-saving notes

- Ahrefs MCP has rate/cost limits — check usage with `mcp__ahrefs__subscription-info-limits-and-usage` if requests start failing.
- Cache keyword-research results to `.seo/keyword-research.json` and **don't re-query within 30 days** unless the user explicitly says "re-research."
- For competitor analysis, the `site-explorer-organic-keywords` endpoint is the most expensive per call. Batch all competitors in step B in a single research pass at Initialize time; don't re-run per phase.
- USD cent units: Ahrefs returns `value`, `org_cost`, `paid_cost`, `traffic_value` in cents. Divide by 100 to display dollars.
