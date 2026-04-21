# Workflow — Weekly AI Visibility Report

Target call budget: **≤15 MCP calls** + 3-5 drill-downs. Produces a richer narrative than the daily: trend, competitor movement, per-ICP rollup, content-gap prioritization, citation-source analysis.

Cadence: every Monday morning, covering the previous ISO week (Mon-Sun).

**Schema-verified 2026-04-18.** Uses flat `start_date`/`end_date` fields, not `date_range:{start,end}`. All metrics are returned automatically — there is no `metrics[]` parameter.

**Consider starting with `get_actions(scope=overview)`** (recipe R11) before this workflow. The overview often surfaces the week's priority faster than the full 15-call derivation, and the derivation becomes grounding rather than hunting. See `suggestion-review.md` for the ≤8-call fast path.

## Steps

### Step 0 — Resolve + prior-week comparison — 2 calls

Same as daily Step 0.

### Step 1 — Full 7-day leaderboard with prior-week comparison — 2 calls

```json
Call A: get_brand_report({
  project_id: "<onino_project>",
  start_date: "<monday of this week>",
  end_date: "<sunday of this week>",
  dimensions: ["date", "model_id", "tag_id", "country_code"]
})

Call B: get_brand_report({
  project_id: "<onino_project>",
  start_date: "<monday of prior week>",
  end_date: "<sunday of prior week>",
  dimensions: ["model_id", "tag_id", "country_code"]
})
```

Compute WoW delta per (model × tag × country) cell client-side.

### Step 2 — Per-ICP-segment weekly trend — 1 call

Use recipe R10 with the week's date range. Identify which of the 5 ICP segments has the weakest trend and the strongest.

### Step 3 — Branded vs. non-branded split — 1 call

Recipe R2 over the week.

### Step 4 — Top competitor head-to-head — 1 call

Recipe R4 against the top 2 competitors (from `list_brands` sorted by SoV).

### Step 5 — Content-gap hunt (full surface) — 1 call

Recipe R3 without the TOFU exclusion on the weekly (TOFU gap data is useful as long-term input even if not immediate action).

### Step 6 — Domain leaderboard by classification — 1 call

Recipe R7. Weekly view is where editorial-coverage trends become visible.

### Step 7 — Per-country weekly split — 1 call

Recipe R8. Focus on DE / AT / CH vs. rest-of-EU.

### Step 8 — Prompt-level integrity — 1 call

Recipe R5 over the week. Ensures no prompts silently dropped out.

### Step 9 — Anomaly-detection with 28-day baseline — 1 call

Recipe R9.

### Step 10 — Drill-downs — 3-5 calls

On findings from the above:
- Biggest ICP-segment decline → `list_chats` + `get_chat` to read 2-3 representative responses.
- New competitor URL surfacing → `get_url_content` on the URL to understand what it says.
- Sentiment movement → `get_chat` on sampled negative-sentiment chats.

## Output structure (Notion page)

1. **Executive narrative (3-5 bullets)** — the story of the week, not the numbers.
2. **ICP segment WoW trend (chart or table)** — visibility per segment this week vs. prior. Call out the weakest.
3. **MOFU content-gap priority list (top 10 URLs)** — sorted by `citation_count` on non-ONINO competitors. These feed SEO/content backlog.
4. **Competitor head-to-head** — ONINO vs. top 2, per engine, per ICP segment.
5. **Citation-source health** — editorial vs. UGC vs. own. Flag over-reliance on any single category.
6. **Regional split (DACH / rest-of-EU)** — visibility per country.
7. **Anomaly flags** — σ>2 events; with drill-down notes.
8. **Integrity flags** — engines missing, prompts dropped, tag case-duplicates.

## Slack TL;DR format (posted to #growth-team)

```
🗓 AI Visibility — Week of <start> to <end>

Headline: <the one ICP finding that matters most>

MOFU gap: <top URL + competitor winning it>

Overall: <visibility% (±pp WoW)>, <SoV% (±pp)>

Notion page: <url>
```

Under 200 words. Link the Notion page.

## Call budget audit

| Step | Calls | Cumulative |
|---|---|---|
| 0. Resolve | 2 | 2 |
| 1. Leaderboard × 2 | 2 | 4 |
| 2. ICP trend | 1 | 5 |
| 3. Branded split | 1 | 6 |
| 4. Competitor H2H | 1 | 7 |
| 5. Gap hunt | 1 | 8 |
| 6. Domain classification | 1 | 9 |
| 7. Regional | 1 | 10 |
| 8. Integrity | 1 | 11 |
| 9. Baseline | 1 | 12 |
| 10. Drill | 3-5 | 15-17 |

Target: ≤15. Acceptable: ≤18 with heavy drill-down.
