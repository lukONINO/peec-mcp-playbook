# Workflow — Query-Fanout-Driven Prompt Discovery

**Goal:** Use `list_search_queries` (the intermediate queries AI models issue while answering our tracked prompts) to find: (1) prompt-phrasing leaks into retail space, (2) content-gap targets with the exact queries competitors are winning, (3) competitor positioning language. This is the single highest-leverage new read capability in the 2026 update.

**Cadence:** Monthly, or after any major AI engine release (GPT-5, Perplexity v2, Gemini Deep Research, etc.).
**Target:** ≤12 MCP calls for the standard 5-prompt pass. `list_shopping_queries` included only when the narrative needs it (rare for ONINO).

---

## Why this workflow exists

Standard gap hunt (`get_url_report` with `gap > 0`) tells you which *published URLs* competitors are cited on and we aren't. That's downstream — by the time a URL is cited, the retrieval race is over. Query fanouts show the **upstream retrieval layer**: the model is generating query X internally; that query retrieves these URLs; those URLs shape the final answer. If we can see X, we can:

- Write a piece that directly targets X.
- Rewrite our Peec-tracked prompt if X is leaking into a retail/adjacent query we shouldn't be in anyway.
- Learn the *language* competitors have anchored in the model's retrieval layer.

---

## Schema reality — read this before building loops

Fanout filters are **SINGULAR, NOT arrays**. One call filters by one `prompt_id`, one `model_id`, one `model_channel_id`. There is no batching across prompts or engines.

Implications for the workflow:

- 5 prompts × 4 engines = **20 fanout calls**, minimum, if you want per-engine resolution.
- 5 prompts (engine-blind, no `model_id` filter) = **5 calls** but rows come back mixed across engines — group client-side.
- **Default to engine-blind** for the monthly pass; drop to per-engine only when you need to confirm a specific engine's fanout pattern.
- **`list_search_queries` has a hard 1000-row cap.** Wide-date-range calls on high-volume prompts can blow through this. Narrow to 30 days and paginate with `offset` if needed.

---

## Step 1 — Pick the 5 highest-leverage prompts (1 call)

Criteria for selection (apply client-side to the output):

- Prompt has **high competitor visibility** (`mentioned_brand_count` or equivalent surfaces >2 non-ONINO brands averaged across the month).
- Prompt has **low or zero ONINO visibility** (`visibility_count = 0` or near it).
- Prompt is **in ICP** (funnel_stage ∈ {MOFU, BOFU}; icp_segment ∈ ONINO's 5; `branded:non-branded`).

One multi-axial `get_brand_report` call gives the ranking surface:

```json
Tool: get_brand_report
Args: {
  "project_id": "<onino_project>",
  "start_date": "<today - 30d>",
  "end_date": "<today>",
  "dimensions": ["prompt_id", "brand_id", "tag_id"],
  "filters": [
    { "field": "tag_id", "operator": "in", "values": ["funnel_stage:MOFU"] },
    { "field": "tag_id", "operator": "in", "values": ["funnel_stage:BOFU"] },
    { "field": "tag_id", "operator": "in", "values": ["branded:non-branded"] }
  ]
}
```

Note — the three tag filters stack as AND; values within each entry are OR'd. MOFU OR BOFU, AND non-branded.

Client-side: rank prompts by `(sum_competitor_mentions - onino_mentions)` descending; take top 5.

**Skip this call if you already have a recent gap-analysis output** (e.g. from a daily/weekly report within the last week). Reuse it.

---

## Step 2 — Pull fanouts per prompt (engine-blind, 5 calls)

For each of the 5 prompts, make one `list_search_queries` call without `model_id` — this returns rows across all active engines in one call:

```json
Tool: list_search_queries
Args: {
  "project_id": "<onino_project>",
  "prompt_id": "<one prompt id>",
  "start_date": "<today - 30d>",
  "end_date": "<today>",
  "limit": 1000
}
```

Returns: `prompt_id, chat_id, model_id, model_channel_id, date, query_index, query_text`.

**When to switch to per-engine calls** (+4 calls per prompt): if the engine-blind output shows fanout queries from only 1–2 engines, that's an engine-coverage artifact — pin to each active engine to confirm or add `model_id`-scoped calls where you suspect sparse coverage. Most weeks don't need this.

**Pagination:** if `rowCount == 1000` (hit the cap), narrow the date window to 14 days or paginate with `offset: 1000`. Don't request `limit: 10000` on this tool — the description explicitly rejects it.

---

## Step 3 — Triage the fanout queries (client-side, 0 calls)

For each prompt's fanout rows, group by `query_text` (case-insensitive) and count frequency across dates and models. Then classify each unique query into one of three buckets:

| Bucket | Definition | Action |
|---|---|---|
| **Leak** | Query contains retail / crowdinvesting / consumer-framing signals ("private investors", "small capital", "crowdinvesting", "retail", "invest as individual") | Flag the *tracked prompt* as mis-phrased. The prompt itself is pulling the model into retail space. Candidate for retire-and-recreate (see §5). |
| **Opportunity** | Query is B2B-aligned AND zero ONINO citation coverage in the fanout results AND at least one competitor cited | Content-gap target. Commission a piece targeting this query phrasing. |
| **Competitor anchor** | Query is B2B-aligned AND dominated by a single competitor across ≥3 chats | Positioning insight. Competitor X owns narrative Y. Either write a direct comparison or reposition our own content to use their language. |

**Triage discipline:**
- A query must appear in **≥3 distinct chats** (not just 3 rows — check `chat_id` uniqueness) before classifying as a stable pattern. Fanouts are stochastic; single appearances are noise.
- `query_index` tells you ordering within one chat (index 0 = first sub-query). Lower indices are higher-priority retrieval intents for the model.

---

## Step 4 — Optional: verify fanout-to-citation linkage (1–2 calls)

Before acting on a high-leverage opportunity, confirm the fanout query actually drives citations:

```json
Call X: get_chat
Args: { "project_id": "<onino>", "chat_id": "<chat_id from Step 2>" }
// Inspect the full AI response — what URLs did the fanout actually cite?

Call Y: get_url_content
Args: { "project_id": "<onino>", "url": "<top cited URL from Call X>" }
// Read the structure: headings, length, data tables, citation patterns
// Informs the content brief for our competing version
```

Skip for leak classifications (don't need to verify a prompt is mis-phrased; just fix it). Always run for opportunity/anchor classifications before committing content spend.

---

## Step 5 — Execute prompt fixes for Leak classifications

`update_prompt` **does not accept `text` updates** (verified 2026-04-18). To fix a mis-phrased prompt you must delete-and-recreate. Because `delete_prompt` cascades to chats (loses history), the retire pattern is preferred when historical baselines matter:

**Option A — Retire and replace (preserves history):**

```json
Call 1: update_prompt
Args: {
  "project_id": "<onino>",
  "prompt_id": "<old_prompt_id>",
  "tag_ids": ["funnel_stage:MOFU", "icp_segment:asset-manager", "branded:non-branded", "language:en", "retired"]
}
// Keeps old prompt tracked but marks as retired for reports to exclude

Call 2: create_prompt
Args: {
  "project_id": "<onino>",
  "text": "<new B2B-anchored prompt text>",
  "country_code": "DE",
  "topic_id": "<same topic>",
  "tag_ids": ["funnel_stage:MOFU", "icp_segment:asset-manager", "branded:non-branded", "language:en"]
}
// Create new version; fanout in ~7 days should show cleaner B2B queries
```

**Option B — Delete and replace (clean break, loses old chat history):**

```json
Call 1: delete_prompt
Args: { "project_id": "<onino>", "prompt_id": "<old_prompt_id>" }
// Cascades to chats — no going back to old baseline

Call 2: create_prompt (as above)
```

Prefer Option A unless you explicitly don't care about historical comparability (e.g. the prompt was drafted three days ago and has no baseline).

**Per-call user confirmation:** Each delete/retire requires explicit go. No speculative loops.

---

## Step 6 — Monitor and close the loop

Rerun `list_search_queries` on the recreated prompt **7 days later**:

```json
Tool: list_search_queries
Args: {
  "project_id": "<onino>",
  "prompt_id": "<new_prompt_id>",
  "start_date": "<creation date>",
  "end_date": "<today>",
  "limit": 1000
}
```

If fanout queries shifted out of retail space → the rephrase worked. Log it.
If queries still leak → rephrase further, or mark the prompt for retirement (maybe the ICP doesn't search this way at all).

---

## Step 7 — Log and post

Write the full output to `/Users/lukaswipf/Documents/Claude/Projects/ONINO GTM/peec-fanout-<YYYY-MM>.md` so the monthly set accumulates. Structure:

```
Monthly fanout review — <YYYY-MM>

Prompts analyzed: 5
  1. "<prompt text>" → [Leak / Opportunity / Anchor dominant]
     Top fanout queries:
       - "<query>" (×N chats, models: X,Y,Z) — <bucket>
     Action: <retire & replace / commission content / competitor teardown>
  2. ...

Peec writes this month: X retires, Y creates
External deliverables: Z content briefs (links to Asana)
```

Post a 3-line Slack summary to the Peec channel.

---

## Guardrails

- **Do not conclude from a single day's fanouts.** Require ≥3 distinct `chat_id`s before classifying.
- **Don't commission content off a single fanout instance.** Opportunity needs ≥3 appearances across ≥2 models.
- **Retail-query fanouts from a B2B prompt always indicate prompt-phrasing drift, not audience opportunity.** Fix the prompt; don't write retail content.
- **`list_search_queries` does not reveal URLs — only queries.** If you need URL-level gap data, use `get_url_report`; fanouts are strictly the query layer.
- **The 1000-row cap is real.** A 30-day window on a high-volume prompt can exceed 1000 sub-queries. If `rowCount == 1000`, paginate or narrow dates.
- **Engine-blind default.** Per-engine resolution is ~4× the calls. Only pin to `model_id` when engine behavior is the story.

---

## Call budget

| Phase | Calls | Notes |
|---|---|---|
| Step 1 — candidate selection | 0 or 1 | 0 if reusing a recent report |
| Step 2 — fanouts (engine-blind) | 5 | One per prompt |
| Step 4 — fanout-to-citation grounding | 0–2 | Only for actions you'll take |
| Step 5 — prompt fixes | 0–5 writes | One per Leak classification |
| Step 6 — monitor (next month) | 0 | Deferred |
| **Total typical monthly pass** | **≤12** | Writes gated on per-call user approval |

For comparison, a per-engine-resolved pass: 5 prompts × 4 engines = 20 read calls alone. Avoid unless engine-level differentiation is the point of the session.
