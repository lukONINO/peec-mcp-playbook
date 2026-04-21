# Peec MCP — Complete Tool Reference (27 tools)

All tools exposed by the Peec.ai MCP server (prefix `mcp__a7063981-0048-4421-9faa-e30e47784e46__`). Schemas cross-verified 2026-04-18 against the live hosted MCP.

> **Read the edge cases in each section.** A meaningful portion of what looks like "Peec is missing X" is actually "X is a parameter we didn't use."

## Surface at a glance

| Category | Tools | Count |
|---|---|---|
| Reports (read) | `get_brand_report`, `get_url_report`, `get_domain_report` | 3 |
| Drill-down (read) | `get_chat`, `get_url_content` | 2 |
| Lists (read) | `list_brands`, `list_models`, `list_prompts`, `list_tags`, `list_topics`, `list_chats`, `list_projects` | 7 |
| Query Fanouts (read) | `list_search_queries`, `list_shopping_queries` | 2 |
| Recommendations (read) | `get_actions` | 1 |
| Brand CRUD (write) | `create_brand`, `update_brand`, `delete_brand` | 3 |
| Prompt CRUD (write) | `create_prompt`, `update_prompt`, `delete_prompt` | 3 |
| Tag CRUD (write) | `create_tag`, `update_tag`, `delete_tag` | 3 |
| Topic CRUD (write) | `create_topic`, `update_topic`, `delete_topic` | 3 |

All write tools have server-side "confirm with user" instructions baked into their descriptions. All `delete_*` tools are **soft-delete**.

## Fastest paths (what replaces multi-call workflows)

Five calls replace 80% of report work. Memorize these before composing new workflows.

1. **`get_actions({project_id, scope: "overview"})`** — prioritized "what matters most" in one call. Start here for any "what should I do?" question; drill into `scope=owned|editorial|reference|ugc` only for the top-surfaced opportunities.
2. **`get_brand_report({dimensions: ["prompt_id"]})`** — authoritative prompt universe. Skips the `list_prompts` 50-row truncation bug.
3. **`get_brand_report({dimensions: ["date", "model_id", "tag_id", "country_code"]})`** — full cross-tab leaderboard in one call. Replaces per-engine × per-stage × per-region loops (4 axes × ~6 days × ~18 engines × 15 tags × 8 countries = 12,960 cells in one response).
4. **`get_url_report({filters: [{field: "gap", operator: "gt", value: 0}]})`** — native content-gap in one expression. Never hand-build `mentioned_brand_count > 0 AND not_in [self]`.
5. **`get_chat(chat_id) + get_url_content(url)`** — verification pair. Every anomaly summary should be grounded in at least one of each before posting.

---

## Reports (3)

### 1. `get_brand_report` — the workhorse

Returns aggregated brand-level metrics (visibility, mention_count, share_of_voice, sentiment, position) across any combination of dimensions.

**Exact parameters**

- `project_id` (required, string)
- `start_date`, `end_date` (required, `YYYY-MM-DD`) — these are flat top-level fields, NOT nested under `date_range`
- `dimensions[]` — any of: `prompt_id, model_id, model_channel_id, tag_id, topic_id, date, country_code, chat_id`
- `filters[]` — array of `{field, operator, values}` (or `{field, operator, value}` for numeric ops)
- `limit` (default 100, max 10000), `offset` (default 0)

**There is no `brand_ids`, no `metrics`, no `order_by` parameter.** Brand filtering goes through `filters` with `field: "brand_id"`. All metric columns are returned automatically. Sorting is client-side.

**Example call**

```json
{
  "project_id": "<onino project id>",
  "start_date": "2026-04-10",
  "end_date": "2026-04-16",
  "dimensions": ["date", "model_id", "tag_id"],
  "filters": [
    { "field": "brand_id", "operator": "in", "values": ["<onino_brand_id>"] },
    { "field": "tag_id", "operator": "in", "values": ["funnel_stage:BOFU"] }
  ]
}
```

**Returned columns**

- `brand_id`, `brand_name`
- `visibility` (0-1 ratio, e.g. 0.45 = 45% of chats)
- `mention_count`
- `share_of_voice` (0-1 ratio, brand's share of total mentions across tracked brands)
- `sentiment` (0-100 scale; typical range 65-85 per schema doc)
- `position` (avg rank when brand appears; lower = better)
- Raw aggregation fields: `visibility_count`, `visibility_total`, `sentiment_sum`, `sentiment_count`, `position_sum`, `position_count`
- Plus any selected dimension columns

**Filter operators (brand report)**

| Field | Operator | Values |
|---|---|---|
| `model_id` | `in, not_in` | enum of 19 engine ids |
| `model_channel_id` | `in, not_in` | enum of 16 channel ids (`openai-0..2, anthropic-0..1, google-0..3, perplexity-0..1, xai-0..1, deepseek-0, meta-0, microsoft-0`) |
| `tag_id, topic_id, prompt_id, brand_id, chat_id` | `in, not_in` | string[] |
| `country_code` | `in, not_in` | enum of 95+ ISO country codes |

`get_brand_report` does **not** support numeric filter operators (gt/lte/etc) or the `gap` pseudo-field — those are URL/domain-report-only.

**Hidden capabilities most agents miss**

- **`dimensions[]` is multi-axial.** Passing `["date", "model_id", "tag_id", "country_code"]` in one call returns a fully rolled-up cross-tab. This replaces 5-20 separate calls.
- **Raw aggregation fields.** Every row returns raw sums/counts alongside normalized metrics. Use them for:
  - **0 vs. missing distinction.** `visibility_count=0, visibility_total>0` → scraped but not mentioned. `visibility_total=0` → engine didn't run.
  - **Correct averaging across slices.** Re-normalize `sentiment_sum / sentiment_count` instead of averaging pre-normalized scores.
- **`prompt_id` dimension** gives per-prompt breakouts — use this as the **authoritative prompt universe** (see `list_prompts` bug below).

**Filter AND/OR semantics**

```
Within a single filter entry:  values are OR'd
Across separate filter entries: AND'd
```

Example: "branded TOFU in German" =

```json
[
  {"field":"tag_id","operator":"in","values":["branded"]},
  {"field":"tag_id","operator":"in","values":["funnel_stage:TOFU"]},
  {"field":"tag_id","operator":"in","values":["language:de"]}
]
```

**Pitfalls**

- `mention_count` ≠ `citation_count`. `mention_count` is brand name appearances in the assistant's answer; `citation_count` (on `get_url_report`) is URL citations. Don't conflate.
- `sentiment` is 0-100; schema doc says most brands land 65-85, so treat any score below ~60 as meaningfully negative.
- `position` is only meaningful when `mention_count > 0`.

---

### 2. `get_url_report` — per-URL citation metrics

Returns per-URL citation data: how often a URL is cited by AI engines alongside a brand, URL classification.

**Exact parameters**

Same shape as `get_brand_report`: `project_id`, `start_date`, `end_date`, `dimensions[]`, `filters[]`, `limit`, `offset`. No `order_by`, no `metrics`.

**Returned columns**

- `url` — full source URL
- `classification` — 11 values: `HOMEPAGE, CATEGORY_PAGE, PRODUCT_PAGE, LISTICLE, COMPARISON, PROFILE, ALTERNATIVE, DISCUSSION, HOW_TO_GUIDE, ARTICLE, OTHER, null`
- `title` — page title or null
- `channel_title` — channel/author name (e.g. YouTube channel, subreddit) or null
- `citation_count` — total explicit citations across all chats
- `retrievals` — total times this URL was used as a source, whether cited or not
- `citation_rate` — avg inline citations per chat when retrieved (can exceed 1.0; higher = more authoritative)
- `mentioned_brand_ids` — array of brand IDs mentioned alongside this URL
- Plus selected dimension columns

**Filter fields (URL report)**

In addition to the standard dimension-based filters (`model_id, model_channel_id, tag_id, topic_id, prompt_id, country_code, chat_id`), URL report also supports:

- `domain` (`in, not_in`)
- `url` (`in, not_in`)
- `mentioned_brand_id` (`in, not_in`) — **singular, not plural**
- `mentioned_brand_count` (`gt, gte, lt, lte` with numeric `value`)
- **`gap` (`gt, gte, lt, lte` with numeric `value`)** — pseudo-field for gap analysis

**The `gap` filter — use this for content-gap analysis**

```json
{"field": "gap", "operator": "gt", "value": 0}
```

Returns URLs where competitors are mentioned but the own-brand is not. One clean expression replaces manual `mentioned_brand_count > 0 AND mentioned_brand_id not_in [self]` constructions.

**`classification` is a returned column, not a filter field.** Filter broadly, post-process client-side. Typical pattern:

```json
"filters": [
  {"field":"gap","operator":"gt","value":0}
]
```

Post-filter `rows` where `row.classification IN ["LISTICLE","COMPARISON"]`.

**Pitfalls**

- Currently no per-URL or per-citation sentiment.
- `classification` is AI-inferred, not editorial — spot-check via `get_url_content` when decisive.
- No `first_seen_at` / `last_seen_at` fields; if you need freshness info, drill to `get_url_content` for the scrape timestamp.

---

### 3. `get_domain_report` — per-domain leaderboard

Returns per-domain citation leaderboard. Same parameter shape and filter fields as `get_url_report` (incl. `gap` and `mentioned_brand_count` pseudo-fields).

**Returned columns (different from URL report)**

- `domain` — the source domain (e.g. `example.com`)
- `classification` — 8 values: `CORPORATE, EDITORIAL, INSTITUTIONAL, UGC, REFERENCE, COMPETITOR, OWN, OTHER, null`
- `retrieved_percentage` (0-1) — fraction of chats that included at least one URL from this domain
- `retrieval_rate` — avg URLs from this domain pulled per chat (can exceed 1.0)
- `citation_rate` — avg inline citations when this domain is retrieved
- `mentioned_brand_ids` — array

Note: no `citation_count` or `retrievals` absolute counts here — those are URL-level only. At the domain level you work with rates and percentages.

**Typical use**

- "Which editorial domains are driving competitor mentions?" → `gap > 0` filter, then client-side filter rows where `classification == "EDITORIAL"`.
- "Who are we losing UGC surface to?" → `gap > 0` + client-side `classification == "UGC"` filter.
- "Competitor-owned domains surfacing in answers" → client-side `classification == "COMPETITOR"`.

As with URL report: **`classification` is a returned column, not a filter field.** Filter broadly, post-process client-side.

---

## Drill-down reads (2)

### 4. `get_chat` — drill down on a specific assistant response

Returns the full content of one chat — one prompt run against one engine on one date.

**Parameters:** `project_id`, `chat_id` (NOT `conversation_id`).

**Returned fields**

- `messages` — the prompt and the assistant response(s)
- `brands_mentioned` — array with positions
- `sources` — URLs retrieved, with citation counts and positions
- `queries` — search queries the model issued
- `products` — product gallery entries
- `prompt: {id}`, `model: {id}`

**Use when**

- An aggregate looks anomalous and you need to explain why.
- You want to see whether a mention is positive, neutral, negative, or a simple name-drop.
- Validating a content-gap: does the assistant actually recommend a competitor in a buying context, or just mention them in a footnote?

**Pattern**

```
1. get_brand_report or list_chats → find interesting chat id
2. get_chat(project_id, chat_id) → read the actual response
3. if surprising → get_url_content on top cited URL
```

---

### 5. `get_url_content` — scraped markdown of a URL

Returns the scraped markdown Peec pulled for a URL.

**Parameters:** `project_id`, `url` (full URL, copy verbatim from `get_url_report` — trailing slashes and scheme variants matter), optional `max_length` (default 100000 chars).

**Returned fields**

- `url, title, domain, channel_title` — page metadata
- `classification` — domain-level
- `url_classification` — page-level (HOMEPAGE, LISTICLE, COMPARISON, …)
- `content` — markdown (Mozilla Readability + Turndown GFM). `null` if scraped but not yet processed (can take up to 24h).
- `content_length` — pre-truncation character count
- `truncated` — bool
- `content_updated_at` — ISO timestamp of last scrape, or null

**Pitfalls**

- Returns 404 if the URL has never been indexed by any Peec project.
- `url` field must be exact — copy from `get_url_report` rows, don't hand-edit.

---

## Lists (7)

### 6. `list_brands` — with `is_own` flag

Returns tracked brands. Columns: `id, name, domains, is_own, aliases, regex`.

**Critical usage note**

- **Always derive own-brand via `is_own = true`.** Never hardcode "ONINO" — the workflow should be portable across projects.
- `domains` is the list of web domains Peec associates with the brand (e.g. `["onino.io", "onino.com"]`) — useful for cross-referencing with `get_url_report` / `get_domain_report` when you need to distinguish "mentioned our brand name" vs. "cited our owned domain."
- `aliases` is a first-class array on the returned brand (confirmed on `create_brand`/`update_brand`). Use these — don't re-invent a local alias list.
- `regex` — optional server-side regex for brand matching in chat text. If set, Peec uses it in addition to name + aliases. Useful for catching "ONINO GmbH" / "Onino" / "ONINO.io" with one pattern instead of an alias array.

---

### 7. `list_models` — available engines + project-level activation

Returns the AI engines Peec supports and which are active on the current project.

**Known engine list (19 as of 2026-04-18)**

| Vendor | Engine ids |
|---|---|
| OpenAI | `chatgpt-scraper, gpt-4o, gpt-4o-search, gpt-3.5-turbo` |
| Perplexity | `perplexity-scraper, sonar, llama-sonar` |
| Anthropic | `claude-3.5-haiku, claude-haiku-4.5, claude-sonnet-4` |
| Google | `gemini-scraper, gemini-2.5-flash, google-ai-overview-scraper, google-ai-mode-scraper` |
| xAI | `grok-scraper, grok-4` |
| Microsoft | `microsoft-copilot-scraper` |
| Open-source | `deepseek-r1, llama-3.3-70b-instruct` |

**`model_channel_id`** — parallel axis to `model_id`, with IDs `openai-0..2, anthropic-0..1, google-0..3, perplexity-0..1, xai-0..1, deepseek-0, meta-0, microsoft-0`. Represents Peec's different scraping approaches per vendor. Use it when you need sub-engine resolution (e.g. ChatGPT Web vs. ChatGPT Search).

**ONINO project current state (2026-04-17)**

Active: `chatgpt-scraper, google-ai-overview-scraper, microsoft-copilot-scraper` (3 of 19).

**→ Highest-leverage fix: request Peec to enable Perplexity, Claude, Gemini on the ONINO project.** These are the three engines most likely to be used by our actual ICP (asset managers, bank staff, financing consultants research).

---

### 8. `list_prompts` — ⚠️ known pagination bug

Returns tracked prompts. Columns: `id, text, tag_ids` (array of tag ID strings), `topic_id` (string or null), `country_code`.

**Parameters:** `project_id` (required), optional `topic_id`, `tag_id`, `limit`, `offset`. Takes no date range — prompts are not time-scoped.

**Important schema note (post-2026 update):**

`country_code` is now a **first-class field on the prompt** (not just a derived tag). `list_prompts` rows include it. `create_prompt` requires it. Language is still only available as a tag (`language:de`, `language:en`).

**Bug**

- Documented `limit` supports `maximum: 10000, default: 100`.
- **Actually silently truncates at 50** on the ONINO project. A follow-up `offset: 50` returns 0 rows despite the full set being ~150.
- No `total_count`, `has_more`, or warning returned.

**Workaround**

Never treat `list_prompts` as the authoritative prompt universe. Instead use `get_brand_report` with `dimensions: ["prompt_id"]` — the aggregation path surfaces every `prompt_id` that produced data in the date range.

```json
{
  "project_id": "<onino>",
  "start_date": "2026-04-10",
  "end_date": "2026-04-16",
  "dimensions": ["prompt_id"]
}
```

Post-process: merge with `list_prompts` rows to enrich the prompt IDs with `text` and `tag_ids`. If `list_prompts` is truncated, iterate with `tag_id` or `topic_id` filters to slice the universe into sub-50-row chunks.

---

### 9. `list_tags` — flat taxonomy (with color)

Returns tags. Columns: `id, name, color`. There is **no `type` field** — tags are a flat string list. Semantic grouping (funnel stage, ICP segment, language) must be encoded in the tag `name` itself using a convention like `funnel_stage:MOFU` or `icp_segment:asset-manager`.

**Parameters:** `project_id` (required), `limit`, `offset`.

**Post-2026 addition:** `color` field on every tag (one of 22 values — see `create_tag`). Cosmetic only, does not affect filter logic.

**Pitfalls**

- **Case-duplicates are a real problem.** We've seen `branded` and `Branded` as separate tags on the ONINO project. Peec does not deduplicate. Merge defensively in every report: `tag_key = tag.name.lower().strip()`.
- Because there is no `type` field, rely on a **naming convention** (colon-prefixed namespaces like `funnel_stage:`, `icp_segment:`, `language:`, `branded`). See `icp-prompt-library.md` for ONINO's canonical tag structure.
- Every prompt can carry multiple tags — cross-sectional filters (e.g. MOFU × asset-manager × German) stack as separate filter entries, each AND'd with the others.

---

### 10. `list_topics` — topic clustering (now country-aware)

Returns Peec's topic clusters. Columns: `id, name, country_code` (optional). Each prompt belongs to **exactly one topic** (via `topic_id` on `list_prompts`) — topics are folder-like, not tag-like.

**Parameters:** `project_id` (required), `limit`, `offset`.

**Post-2026 addition:** topics can now carry an optional `country_code` (set via `create_topic`). Useful for separating DE-specific topic clusters from EU or global ones.

**Use cases**

- As a dimension in `get_brand_report` to roll up metrics by topic rather than by prompt.
- As a filter in `list_prompts(topic_id=...)` to slice the prompt universe into under-50-row chunks (works around the `list_prompts` truncation bug).
- For prompt-set audit: a topic with a very low prompt count may be under-tracked.

---

### 11. `list_chats` — enumerate recent conversations

Returns one row per AI response (one prompt × one engine × one date). Columns: `id, prompt_id, model_id, date`. **No `tag_ids` and no `brands_mentioned` in the list output** — those are only accessible via `get_chat` on a specific chat id, or via the `get_brand_report`/`get_url_report` aggregation path.

**Parameters:** `project_id`, `start_date`, `end_date` (all required), optional `brand_id` (single, filters to chats that mentioned that brand), `prompt_id` (single), `model_id` (single enum — **not** `in`/`not_in`), `limit` (max 10000), `offset`.

**Use as the index before `get_chat`.** Typical patterns:

- "Chats mentioning a competitor" → `brand_id: <competitor_id>` + date range → pick 2-3 ids → `get_chat` on each.
- "Chats from a specific prompt" → `prompt_id: <X>` + date range → useful when drilling into a MOFU/BOFU prompt where we're invisible.
- "Single-engine drilldown" → `model_id: "chatgpt-scraper"` (single value only — if you want two engines, make two calls).

---

### 12. `list_projects` — account-level

Lists Peec projects the authenticated user has access to. Columns: `id, name, status`.

**Parameters:** none required. Optional `include_inactive: boolean` (default `false`). By default only active projects (`CUSTOMER, PITCH, TRIAL, ONBOARDING, API_PARTNER`) are returned.

**This tool takes no `project_id`** — it's the one endpoint you call *before* you know a project id. Use it once at session start to resolve the ONINO project id, then pass that id into every other tool. Rarely called otherwise — only when switching accounts or auditing which projects exist.

---

## Query Fanouts (2) — NEW (2026-04-18)

Query fanouts expose the intermediate queries an AI engine issues while composing its answer. This is the **upstream retrieval layer** — what the model is *actually* searching, before any URL gets cited. Two distinct channels: general AI search, and shopping/product search.

> **Critical efficiency note:** Fanout filters are **SINGULAR, NOT arrays**. `prompt_id` takes one string. `model_id` takes one string. If you want fanouts across 5 prompts × 4 engines, that's 20 calls — there is no batching. Budget accordingly.

### 13. `list_search_queries` — AI-search fanout

Returns one row per sub-query an engine fanned out to while answering a tracked prompt. Columns: `prompt_id, chat_id, model_id, model_channel_id, date, query_index, query_text`.

**Parameters:**
- `project_id`, `start_date`, `end_date` (required)
- `prompt_id` (optional, singular) — filter to chats from this prompt
- `chat_id` (optional, singular) — filter to this specific chat
- `model_id` (optional, enum of 19 engine ids, singular) — one engine only
- `model_channel_id` (optional, enum of 16 channel ids, singular)
- `topic_id` (optional, singular)
- `tag_id` (optional, singular)
- `limit` (default 100, **hard cap 1000** — the description explicitly says "Do not request 10000"), `offset`

**Pagination:** 1000-row cap is a real ceiling. High-volume prompts over a wide date window can blow through this — either narrow the date range, pin to one engine, or paginate with `offset`.

**How to use efficiently**

Candidate-select with a single `get_brand_report(dimensions:[prompt_id])` call, pick 5 prompts worth the fanout drill, then loop:

```
for prompt_id in top_5:
  for model_id in active_engines:
    list_search_queries({project_id, start_date, end_date, prompt_id, model_id})
```

20 calls to cover 5 prompts × 4 engines. Don't try to batch; there's no multi-filter for these.

### 14. `list_shopping_queries` — shopping/product fanout

Same shape as `list_search_queries`, but returns rows tagged as shopping intent. Columns: `prompt_id, chat_id, model_id, model_channel_id, date, query_text, products[]`. The `products[]` array gives the distinct product names each sub-query returned.

**Parameters:** same as `list_search_queries`. Note: `limit` here is documented max 10000 (inconsistent with `list_search_queries` which caps at 1000). Treat 1000 as the safe default for both until Peec confirms.

**Relevance for ONINO:** low. ONINO is B2B infra, not a consumer product. Useful mainly as a *competitive* signal — if competitors (especially crowdinvesting platforms incorrectly tagged as competitors) show up heavily in shopping fanouts, it's a fast confirmation they're operating in retail space, not B2B.

---

## Recommendations (1) — NEW (2026-04-18)

### 15. `get_actions` — opportunity-scored recommendations engine

**This is the biggest shift in the 2026 MCP update.** Instead of us manually constructing "where are we losing?" via gap filters, Peec now scores opportunities server-side and returns ranked, textual recommendations with markdown-formatted pointers to targets.

**Two-step workflow — always use `scope=overview` first**

**Step 1 — overview** returns opportunity rollups grouped by `action_group_type × (url_classification | domain)`. This is *navigation metadata*, not the recommendations themselves — use it to find which slices have the largest gap.

```json
{
  "project_id": "<onino>",
  "scope": "overview"
  // optional: start_date, end_date (default: last 30 days)
}
```

**Step 2 — drill down** into the high-opportunity slice. Required extras per scope:

| `scope` | Required extra | Returns |
|---|---|---|
| `overview` | — | Opportunity rollups (navigation metadata) |
| `owned` | — | Recommendations for ONINO-owned content |
| `editorial` | `url_classification` (e.g. `LISTICLE`) | Recommendations for editorial placements of this type |
| `reference` | `domain` (e.g. `wikipedia.org`) | Recommendations for a specific reference domain |
| `ugc` | `domain` (e.g. `reddit.com`, `youtube.com`) | Recommendations for UGC on a specific platform |

**Returned columns (drill-down scopes)**

- `text` — the actual recommendation, markdown-formatted with links to targets/examples
- Opportunity score fields (magnitude of gap)
- Dimension columns that situate the recommendation (e.g. `url_classification`, `domain`, model, tag)

**Use this whenever the user asks any of:**

- "What should I do next?"
- "What are the top opportunities?"
- "Based on this data, what actions should I take?"
- "How do I improve visibility?"

**Never hand-roll SEO advice when `get_actions` is available.** The server has context on opportunity magnitude, competitor overlap, and classification that we'd have to re-derive across 3-4 calls.

**Date range default:** last 30 days if omitted. Use a 30-day window unless the user asks otherwise.

**Efficiency pattern**

```
Step 1: get_actions({project_id, scope: "overview"})
        → identify top 3 opportunity slices
Step 2: for each slice, get_actions(scope + required extras)
        → textual recommendations per slice
Step 3: ground the recommendation in a chat drill-down
        (get_chat on a cited chat_id, get_url_content on top source)
        before writing to Slack / Notion
```

Budget: ≤5 calls for a full "what should we do?" pass. Compare to the pre-2026 pattern (3× `get_brand_report` with different dimensions + 2× `get_url_report` with gap + manual triage) which typically ran 8-12 calls for the same output.

---

## Brand CRUD (3)

All write tools carry an explicit "confirm with user before calling" instruction in their server-side descriptions. Soft-delete semantics on every `delete_*`.

### 16. `create_brand`

Create a new brand (competitor or own) tracked in a project. Returns the created brand id.

**Parameters**

- `project_id` (required)
- `name` (required, min length 1)
- `domains[]` (optional)
- `aliases[]` (optional)
- `regex` (optional) — server-side regex for brand matching in chat text

**NOT accepted as a create-time parameter:** `is_own`. That flag is read-only (derived by Peec from project config). If you need to flag a brand as own-brand, contact Peec support or use the Peec UI.

**Before creating:** always `list_brands` first — Peec does not dedupe on name or alias. Duplicates accumulate silently.

```json
{
  "project_id": "<onino>",
  "name": "Tokeny",
  "domains": ["tokeny.com"],
  "aliases": ["Tokeny Solutions"],
  "regex": "(?i)\\btokeny(?:\\s+solutions)?\\b"
}
```

### 17. `update_brand`

Update a brand's name, regex, aliases, or domains. Returns the updated brand.

**Parameters:** `project_id`, `brand_id` (both required); optional `name`, `domains`, `aliases`, `regex` (pass `null` to clear an existing regex).

**Recalc race condition:** Changes to `name`, `regex`, or `aliases` trigger **background metric recalculation**. Repeat attempts during recalculation will fail. Implications:

- Don't loop tightly over `update_brand` calls — space them out or handle the failure.
- After a rename, expect a brief window where reports show stale data.

```json
{
  "project_id": "<onino>",
  "brand_id": "<id>",
  "name": "Tokeny Solutions",
  "domains": ["tokeny.com", "docs.tokeny.com"],
  "aliases": ["Tokeny"]
}
```

### 18. `delete_brand`  `destructiveHint: true`

Soft-delete a brand within a project.

**Parameters:** `project_id`, `brand_id`.

**Soft-delete semantics:** historical attribution is orphaned but data is preserved server-side (not hard-deleted). Still disruptive to aggregations — `get_brand_report` will no longer surface this brand's mentions in rollups.

**Almost never what you want.** Prefer `update_brand` to consolidate aliases into a canonical brand. Only use when a brand was created in error (e.g. a mis-capitalized dupe on day 1 with zero history attached).

---

## Prompt CRUD (3)

### 19. `create_prompt`

Create a new prompt in a project. Returns the created prompt id. **May consume plan credits** per the server description.

**Parameters**

- `project_id` (required)
- `text` (required, **max 200 chars**, min 1)
- `country_code` (required, enum of 95+ ISO codes — `AE, AL, AM, AR, AT, AU, BA, BE, BG, BH, BO, BR, BY, CA, CH, CL, CO, CR, CY, CZ, DE, DK, DO, EC, EE, EG, ES, FI, FR, GB, GE, GH, GR, GT, HN, HR, HU, ID, IE, IL, IN, IQ, IS, IT, JO, JP, KR, KW, LB, LT, LU, LV, MA, MD, ME, MK, MT, MX, MY, NG, NI, NL, NO, NZ, OM, PK, PA, PE, PH, PL, PT, PY, PS, QA, RO, RS, SA, SE, SG, SI, SK, SV, TH, TN, TR, TW, UA, US, UY, VE, VN, ZA`). **No `EU` meta-code.**
- `topic_id` (optional)
- `tag_ids[]` (optional)

**Important for ONINO taxonomy:** `country_code` is now first-class and required. This **partially obsoletes the `region_target:` tag family** — the country is native, not a tag. Language is still only available via `language:de` / `language:en` tags since country ≠ language (CH has DE/FR/IT speakers; BE has NL/FR/DE).

```json
{
  "project_id": "<onino>",
  "text": "Best white-label tokenization platforms for European asset managers",
  "country_code": "DE",
  "topic_id": "<topic_id>",
  "tag_ids": ["funnel_stage:MOFU", "icp_segment:asset-manager", "branded:non-branded", "language:en"]
}
```

### 20. `update_prompt`

Update a prompt's topic and/or tags. **Does not accept `text` updates** — if you need to rephrase a prompt, delete and recreate, or update via the Peec UI.

**Parameters:** `project_id`, `prompt_id` (required); optional `topic_id` (pass `null` to detach), `tag_ids[]` (replaces existing tag set — not merge).

**Critical:** `tag_ids[]` **fully replaces** the prompt's tag set. Pass the complete desired tag list, not just additions.

```json
{
  "project_id": "<onino>",
  "prompt_id": "<id>",
  "topic_id": "<new_topic_id>",
  "tag_ids": ["funnel_stage:MOFU", "icp_segment:asset-manager", "branded:non-branded", "language:en"]
}
```

### 21. `delete_prompt`  `destructiveHint: true`

Soft-delete a prompt and **cascade the deletion to its chats**.

**Parameters:** `project_id`, `prompt_id`.

**Cascade behavior:** unlike brand/tag deletes which orphan attribution but keep chats, `delete_prompt` soft-deletes the prompt AND its chat history. Reports can no longer query over this prompt's historical data.

**Strong preference: retire, don't delete.** Add a `retired` tag via `update_prompt` instead of calling `delete_prompt`. Historical continuity survives.

---

## Tag CRUD (3)

### 22. `create_tag`

Create a new tag in a project. Returns the created tag id.

**Parameters**

- `project_id` (required)
- `name` (required, min 1)
- `color` (optional, default `gray`; enum of 22: `gray, red, orange, yellow, lime, green, cyan, blue, purple, fuchsia, pink, emerald, amber, violet, indigo, teal, sky, rose, slate, zinc, neutral, stone`)

**ONINO naming convention — enforce religiously:**

Tags have no `type` field. Encode namespace in the name itself: `funnel_stage:MOFU`, `icp_segment:asset-manager`, `branded:non-branded`, `language:de`. Any drift breaks the filter logic in `query-recipes.md`. Suggested color conventions:

- `funnel_stage:*` → `blue` (TOFU), `indigo` (MOFU), `violet` (BOFU)
- `icp_segment:*` → `emerald`
- `branded:*` → `amber` (branded), `gray` (non-branded)
- `language:*` → `cyan`
- `retired` → `red`

Colors are cosmetic but help humans visually audit the Peec UI.

### 23. `update_tag`

Rename or recolor a tag. Idempotent.

**Parameters:** `project_id`, `tag_id` (required); optional `name`, `color`.

**Pre-rename check:** grep `onino-ai-visibility-daily` and this skill's reference files for references to the old tag name. Downstream scheduled tasks break silently if a referenced tag is renamed without a matching SKILL.md update.

### 24. `delete_tag`  `destructiveHint: true`

Soft-delete a tag and detach it from all prompts.

**Parameters:** `project_id`, `tag_id`.

History is preserved on the prompt side but historical filter queries by this tag return empty. Rare use — prefer consolidation via `update_tag` rename.

---

## Topic CRUD (3)

### 25. `create_topic`

Create a new topic in a project. Topics group related prompts. Returns the created topic id.

**Parameters**

- `project_id` (required)
- `name` (required, **max 64 chars**, min 1)
- `country_code` (optional, same enum as `create_prompt`)

**Naming discipline:** keep names short and descriptive. 64-char cap is tighter than prompt text (200). Examples: `White-label tokenization infrastructure`, `SPV & private market platforms`, `DACH crowdinvesting adjacency`.

### 26. `update_topic`

Rename a topic within a project.

**Parameters:** `project_id`, `topic_id` (required); optional `name` (max 64 chars).

Does NOT accept `country_code` updates — set at create time only (verify empirically if needed).

### 27. `delete_topic`  `destructiveHint: true`

Soft-delete a topic.

**Parameters:** `project_id`, `topic_id`.

**Cascade behavior:** associated prompts are **detached** (not deleted); prompt **suggestions** on the topic **are deleted**. The detached prompts become "orphan" topic-wise but continue to function.

---

## Filter operator reference (verified against schema)

| Operator | Semantics | Works on |
|---|---|---|
| `in`, `not_in` | membership in `values[]` | categorical fields: `model_id, model_channel_id, tag_id, topic_id, prompt_id, brand_id, mentioned_brand_id, country_code, chat_id, domain, url` |
| `gt`, `gte`, `lt`, `lte` | numeric comparison with `value` (singular) | `mentioned_brand_count`, `gap` — URL/domain report only |

Operators NOT supported (don't use them): `eq`, `neq`, `contains`, `not_contains`, `starts_with`, `ends_with`, `is_null`, `is_not_null`. For equality, use `in` with a single-item array.

Pseudo-fields:
- `gap` — available on `get_url_report` and `get_domain_report` only. Finds rows where competitors are mentioned but own-brand is absent.
- `mentioned_brand_count` — URL/domain report only.

**`classification` is a returned column, not a filter field.** Filter broadly, post-process rows client-side.

---

## Tools anticipated but NOT shipped

The 2026-04-17 announcement listed several additional tools that did NOT make it to the hosted MCP. Do NOT plan workflows around these until Peec ships them:

- `list_model_channels`
- `list_prompt_suggestions` / `list_topic_suggestions` (pending-suggestion inbox)
- `accept_prompt_suggestion` / `reject_prompt_suggestion` / `accept_topic_suggestion` / `reject_topic_suggestion`

The closest live paradigm is `get_actions` (§15) — it surfaces prioritized opportunities server-side rather than exposing a pending-suggestion inbox. See `peec-mcp-update-2026.md` for the verification trail.

---

## Response envelope reminder

Every response currently returns `{data, pagination, metadata}` — no `warnings` array yet. Defensively check:

- Row count matches expected dimensions product (if you requested `dimensions: [model_id, date]` over 5 models × 7 days, expect ≤35 rows; fewer implies silent filtering).
- `visibility_total > 0` on rows where you expect engine coverage — else the engine didn't run for that slice.

---

## Change log

- **2026-04-18** — Expanded from 12 to 27 tools. Added CRUD (brand/prompt/tag/topic × 3), Query Fanouts, `get_actions`. Corrected schemas: `create_prompt` requires `country_code`; `create_brand` does NOT accept `is_own`; fanout filters are singular; `list_search_queries` has a hard 1000 cap; all deletes are soft; `delete_prompt` cascades to chats; `update_brand` has a recalc race. Named fastest paths that replace multi-call workflows. Resolved naming drift — hosted MCP keeps singular report names (`get_brand_report`, not `get_brands_report`).
- **2026-04-17** — Initial 12-tool reference.
