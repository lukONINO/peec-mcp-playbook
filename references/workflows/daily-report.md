# Workflow — Daily AI Visibility Report

Target call budget: **≤8 MCP calls** + 2-3 optional drill-downs. Roughly 80% fewer calls than the pre-playbook v3.1 workflow.

**Schema-verified 2026-04-18.** Payloads below use the flat `start_date`/`end_date` (not `date_range`), `filters[]` (not `brand_ids`), and no `metrics[]` / `order_by` (both are non-existent — all metrics are always returned; sorting is client-side).

## Inputs

- Date: today (or the most recent day with finalized scrapes — typically T-1)
- Comparison window: previous 14 days for rolling baseline

## Steps

### Step 0 — Resolve identifiers (once per day, cache) — 2 calls

```
Call 1: list_brands(project_id=<onino>) → extract brand with is_own=true → <onino_brand>
Call 2: list_models(project_id=<onino>) → check is_active; flag any expected engine that's inactive
```

### Step 1 — Single-call leaderboard + funnel + region rollup — 1 call

Use recipe R1:

```json
get_brand_report({
  project_id: "<onino_project>",
  start_date: "<today>",
  end_date: "<today>",
  dimensions: ["model_id", "tag_id", "country_code"]
})
```

This returns the full cross-tab — all engines × all tags × all countries — in one response. Every metric (visibility, mention_count, share_of_voice, sentiment, position, plus raw `*_count`/`*_total` fields) is returned automatically.

### Step 2 — Branded vs. non-branded split — 1 call

Use recipe R2. Confirms that today's visibility movement is driven by non-branded (pipeline-valuable) queries, not brand searches.

### Step 3 — ICP-segment audit — 1 call

Use recipe R10. Surfaces which ICP segment we're most invisible to today. Often the most actionable output of the daily.

### Step 4 — Rolling baseline for anomaly detection — 1 call

Use recipe R9 with `date_range = (today - 28d, today)`.

Client-side compute:
- Rolling 14-day μ and σ (excluding today)
- Flag today if `|x - μ| > 2σ` on any of `visibility`, `share_of_voice`, `mention_count`

### Step 5 — Content-gap scan (MOFU/BOFU only) — 1 call

Use recipe R3 with `tag_id in ["funnel_stage:MOFU", "funnel_stage:BOFU"]` and `classification in ["LISTICLE", "COMPARISON", "CATEGORY_PAGE"]`.

Take the top 10 results by `citation_count`.

### Step 6 — Engine-completeness integrity — 1 call

Use recipe R5. Confirm all active engines actually ran today's prompts. Flag any engine × prompt combo with `visibility_total = 0`.

### Step 7 (optional) — Drill-down on anomalies — 1-3 calls

For any anomaly flagged in Step 4 or any new top-3 gap from Step 5:

```
list_chats(prompt_id=<X>, date=today, limit=5) → pick 2-3 representative conversation_ids
get_chat(conversation_id) × 2-3
get_url_content(url) on the top cited URL if applicable
```

## Output structure (Notion page + Slack TL;DR)

Ordered by commercial priority, not metric magnitude:

1. **ICP segment alert** (from Step 3) — "Asset managers: visibility dropped to 22% — see gap prompt #8."
2. **MOFU content-gap headline** (from Step 5) — "New competitor surfacing on 'best white-label tokenization platforms in Europe 2026' — top citing URL is X."
3. **Aggregate line** (from Step 1) — "Overall visibility: 33% (+1pp DoD). SoV: 68%."
4. **Anomaly flags** (from Step 4) — only if σ>2 on any metric; otherwise omit.
5. **Integrity** (from Step 6) — only if anything failed; otherwise omit.

**Do NOT include:**
- TOFU-only findings unless sentiment-related
- Retail/crowdinvesting-prompt movements (out-of-scope — see `anti-patterns.md` §1)
- SoV figures on clusters with `mention_count < 5` (noise)

## Call budget audit

| Step | Calls | Cumulative |
|---|---|---|
| 0. Resolve | 2 | 2 |
| 1. Leaderboard | 1 | 3 |
| 2. Branded split | 1 | 4 |
| 3. ICP audit | 1 | 5 |
| 4. Baseline | 1 | 6 |
| 5. Gap scan | 1 | 7 |
| 6. Integrity | 1 | 8 |
| 7. Drill (opt) | 1-3 | 9-11 |

vs. pre-playbook v3.1: ~38 MCP calls.
