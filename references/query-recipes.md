# Query Recipes

Copy-paste MCP payloads for the 11 most common Peec tasks. Each recipe is optimized for minimum call count using `dimensions[]`, native filter operators, and (post-2026) `get_actions`.

**All schemas verified against live hosted MCP on 2026-04-18.** The earlier version of this file used `date_range: {start, end}` nesting, `metrics: [...]`, `brand_ids: [...]` top-level, and `order_by: {...}` — none of which exist. Those payloads would have failed at runtime.

All recipes assume:
- `<onino_project>` = the ONINO Peec project id (resolve once via `list_projects` at session start, then cache)
- `<onino_brand>` = resolved via `list_brands` where `is_own = true` (never hardcode "ONINO")
- Date placeholders: `<start>`, `<end>` in `YYYY-MM-DD` — **flat top-level fields, NOT nested**
- All report tools return all metrics automatically; there is no `metrics[]` parameter
- Sorting is client-side; there is no `order_by` parameter

---

## R1 — Full brand leaderboard (one call, all engines, all dimensions)

**Purpose:** Daily/weekly top-line snapshot. Replaces 8-15 per-engine / per-funnel-stage calls.

```json
Tool: get_brand_report
Args: {
  "project_id": "<onino_project>",
  "start_date": "<start>",
  "end_date": "<end>",
  "dimensions": ["date", "model_id", "tag_id", "country_code"]
}
```

**Returned per row:** brand_id, brand_name, visibility, mention_count, share_of_voice, sentiment, position, raw aggregation fields, plus every selected dimension column.

**Post-process client-side:**
- Group by `tag_id` where name starts `funnel_stage:` → TOFU/MOFU/BOFU rollup.
- Group by `tag_id` where name starts `icp_segment:` → per-ICP rollup.
- Group by `country_code` → DACH split (DE/AT/CH) vs. rest-of-EU.
- Sort by whichever metric matters; there is no server-side sort.

---

## R2 — Branded vs. non-branded decomposition (one call)

**Purpose:** Understand whether ONINO's visibility comes from brand searches ("ONINO reviews") or category searches ("best white-label tokenization platform"). The latter is what matters for pipeline.

```json
Tool: get_brand_report
Args: {
  "project_id": "<onino_project>",
  "start_date": "<start>",
  "end_date": "<end>",
  "dimensions": ["tag_id", "model_id"],
  "filters": [
    { "field": "brand_id", "operator": "in", "values": ["<onino_brand>"] },
    { "field": "tag_id", "operator": "in", "values": ["branded:branded", "branded:non-branded"] }
  ]
}
```

**Interpretation:**
- `branded` visibility trending up without `non-branded` moving = brand searches only, not category wins.
- `non-branded` visibility up = real pipeline-driving visibility.

---

## R3 — Content-gap hunt (native `gap` filter)

**Purpose:** Find URLs where competitors appear and ONINO doesn't. Replaces manual `mentioned_brand_count > 0 + not_in [onino]` construction.

```json
Tool: get_url_report
Args: {
  "project_id": "<onino_project>",
  "start_date": "<start>",
  "end_date": "<end>",
  "dimensions": ["tag_id"],
  "filters": [
    { "field": "gap", "operator": "gt", "value": 0 },
    { "field": "tag_id", "operator": "in", "values": ["funnel_stage:MOFU", "funnel_stage:BOFU"] }
  ],
  "limit": 50
}
```

**Post-filter client-side** by `classification IN ["LISTICLE", "COMPARISON", "CATEGORY_PAGE"]` (classification is a returned column, not a filter field). Sort by `citation_count` desc.

**Why this filter combination:** Gaps on TOFU content matter less than gaps on MOFU/BOFU. Gaps on LISTICLE/COMPARISON/CATEGORY pages are commercially decisive — these are the articles actual ICP buyers read during vendor shortlisting.

---

## R4 — Competitor head-to-head (paired leaderboard)

**Purpose:** Compare ONINO against a specific named competitor across all dimensions.

```json
Tool: get_brand_report
Args: {
  "project_id": "<onino_project>",
  "start_date": "<start>",
  "end_date": "<end>",
  "dimensions": ["brand_id", "tag_id", "model_id"],
  "filters": [
    { "field": "brand_id", "operator": "in", "values": ["<onino_brand>", "<competitor_brand_id>"] }
  ]
}
```

**Post-process:** Compute deltas `onino - competitor` per `(tag_id, model_id)` cell. Sort by magnitude. The largest negative deltas are your action list.

---

## R5 — Engine-coverage integrity check (authoritative prompt universe)

**Purpose:** Confirm which prompts × which engines actually ran in the period. Works around the `list_prompts` pagination bug.

```json
Tool: get_brand_report
Args: {
  "project_id": "<onino_project>",
  "start_date": "<start>",
  "end_date": "<end>",
  "dimensions": ["prompt_id", "model_id"]
}
```

**Interpretation using raw fields:**
- `visibility_total = 0` → engine didn't run for this prompt in the period.
- `visibility_total > 0, visibility_count = 0` → engine ran, brand not mentioned.
- `visibility_total > 0, visibility_count > 0` → engine ran, brand mentioned.

Use this to flag any engine that's silently dropping prompts before the report ships.

---

## R6 — Sentiment drill-down on a specific prompt

**Purpose:** A prompt's sentiment moved. Find out why.

```json
Step 1: list_chats
Args: {
  "project_id": "<onino_project>",
  "start_date": "<start>",
  "end_date": "<end>",
  "prompt_id": "<target_prompt>",
  "limit": 20
}

Step 2: get_chat (on 2-3 representative chat_ids from Step 1)
Args: { "project_id": "<onino_project>", "chat_id": "<id>" }

Step 3: get_url_content (on top cited URL if a source is driving the shift)
Args: { "project_id": "<onino_project>", "url": "<exact url from get_url_report>" }
```

**Never** summarize a sentiment movement from aggregates alone — always read at least one chat and one source before writing the finding.

**Note:** `list_chats` takes singular `prompt_id` and `model_id`. If you want two engines, make two calls.

---

## R7 — Domain leaderboard by editorial vs. corporate vs. UGC

**Purpose:** Understand *what kind* of source is driving brand mentions — are we winning on editorial coverage (good for pipeline), our own property (expected), or UGC (fragile)?

```json
Tool: get_domain_report
Args: {
  "project_id": "<onino_project>",
  "start_date": "<start>",
  "end_date": "<end>",
  "dimensions": ["domain"],
  "filters": [
    { "field": "mentioned_brand_id", "operator": "in", "values": ["<onino_brand>"] }
  ],
  "limit": 50
}
```

**Post-process client-side** by `classification`:
- High `retrieval_rate` on `OWN` + `EDITORIAL` = healthy.
- Heavy reliance on `UGC` (Reddit, forums) = fragile; may disappear after a model update.
- Any `COMPETITOR` classification appearing for our brand = likely mis-classification; spot-check via `get_url_content`.

---

## R8 — Per-country visibility split (DACH strategy check)

**Purpose:** ONINO is DACH-primary. Confirm visibility per country.

```json
Tool: get_brand_report
Args: {
  "project_id": "<onino_project>",
  "start_date": "<start>",
  "end_date": "<end>",
  "dimensions": ["country_code", "model_id"],
  "filters": [
    { "field": "brand_id", "operator": "in", "values": ["<onino_brand>"] },
    { "field": "country_code", "operator": "in", "values": ["DE", "AT", "CH", "FR", "NL", "ES", "IT"] }
  ]
}
```

---

## R9 — Anomaly detection baseline (rolling window)

**Purpose:** Build a rolling-mean baseline so today's number can be judged in context. Client-side math, single MCP call.

```json
Tool: get_brand_report
Args: {
  "project_id": "<onino_project>",
  "start_date": "<today - 28d>",
  "end_date": "<today>",
  "dimensions": ["date", "model_id"],
  "filters": [
    { "field": "brand_id", "operator": "in", "values": ["<onino_brand>"] }
  ]
}
```

**Client-side:**
- Rolling 14-day μ and σ (excluding today).
- Flag today's row if `|x - μ| > 2σ` on any metric.
- Cross-reference with scrape volume (`list_chats` count) — large volume anomalies usually explain the metric anomaly.

---

## R10 — ICP-aligned narrative audit (which segments are we invisible to?)

**Purpose:** The highest-value question. Which of the 5 ICP segments is ONINO failing to surface for?

```json
Tool: get_brand_report
Args: {
  "project_id": "<onino_project>",
  "start_date": "<start>",
  "end_date": "<end>",
  "dimensions": ["tag_id", "model_id"],
  "filters": [
    { "field": "brand_id", "operator": "in", "values": ["<onino_brand>"] },
    { "field": "tag_id", "operator": "in", "values": [
      "icp_segment:asset-manager",
      "icp_segment:bank",
      "icp_segment:platform-operator",
      "icp_segment:club-coop",
      "icp_segment:financing-consultant"
    ]}
  ]
}
```

**This is the report Alex should see every week.** The ICP segment with the lowest visibility is the highest-ROI content/SEO/GEO priority.

---

## R11 — "What should we do next?" (get_actions overview) — NEW 2026

**Purpose:** Single-call prioritized recommendation surface. Replaces the pre-2026 pattern of chaining R1 + R3 + R7 + manual triage to reach "what's the top opportunity?"

```json
Tool: get_actions
Args: {
  "project_id": "<onino_project>",
  "scope": "overview"
  // start_date/end_date optional — defaults to last 30 days, which is the right window for weekly review
}
```

Returns opportunity rollups grouped by `action_group_type × (url_classification | domain)`.

**Drill into the top 1–3 slices (one call each):**

```json
// OWNED (our own content opportunities):
{ "project_id": "<onino>", "scope": "owned" }

// EDITORIAL (url_classification required):
{ "project_id": "<onino>", "scope": "editorial", "url_classification": "LISTICLE" }

// REFERENCE (domain required):
{ "project_id": "<onino>", "scope": "reference", "domain": "wikipedia.org" }

// UGC (domain required):
{ "project_id": "<onino>", "scope": "ugc", "domain": "reddit.com" }
```

**Budget:** 1 overview + 1–3 drills = ≤4 calls to reach an actionable list.

---

## Recipe budgets (target call counts)

| Task | Recipes | Target MCP calls |
|---|---|---|
| **"What should we do this week?"** | **R11 + R9** | **≤5** (new fastest path) |
| Daily report | R1, R2, R9 | 3 + optional drill-downs |
| Weekly report | R11 first, then R1, R3, R5, R7, R10 as needed | 6–8 + 2–3 drill-downs |
| Competitor drill | R4 + drill-downs | 1 + 2–4 |
| Content gap hunt | R3 + drill-downs | 1 + 3–5 |
| Integrity check | R5 | 1 |
| ICP audit (monthly) | R10 + R5 + R9 | 3 |
| Monthly fanout review | see `workflows/fanout-discovery.md` | ≤12 |
| Weekly opportunity review | R11 + drill + ground | ≤8 (see `workflows/suggestion-review.md`) |
| Quarterly consolidation | see `workflows/prompt-set-consolidation.md` | ~28 |

Anything significantly above these counts is a signal to re-check whether `dimensions[]`, `get_actions`, or a native filter operator can collapse the workload.

---

## Common schema gotchas (verified live 2026-04-18)

| Mistake | Reality |
|---|---|
| `date_range: { start, end }` | Flat top-level: `start_date`, `end_date` |
| `metrics: ["visibility", "sentiment"]` | No such param — all metrics always returned |
| `brand_ids: ["<id>"]` top-level | Use `filters: [{field: "brand_id", operator: "in", values: [...]}]` |
| `order_by: { field, direction }` | No server-side sort — post-process client-side |
| `url_id: "<id>"` filter | Filter on `url` (string) or `domain`, not `url_id` |
| `conversation_id` in `get_chat` | Parameter is `chat_id` |
| `prompt_ids[]` on fanout tools | Singular `prompt_id` only — loop N times |
| `classification: "LISTICLE"` as a filter | `classification` is a returned column, not a filter field |
| `limit: 10000` on `list_search_queries` | Hard cap 1000 — description rejects 10000 |
