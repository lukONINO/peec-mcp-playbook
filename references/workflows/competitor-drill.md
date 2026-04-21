# Workflow — Competitor Head-to-Head Drill

Target call budget: **3-6 MCP calls.**

**Schema-verified 2026-04-18.** Uses flat `start_date`/`end_date`, `filters[]` for brand selection (not top-level `brand_ids`), no `metrics[]` (all returned automatically), and no `order_by` (sort client-side). URL filtering uses `mentioned_brand_id` (singular) with `in`/`not_in`, or the native `gap` operator.

## When to use

- A new competitor is surfacing in reports (triggered from daily/weekly integrity flags).
- Sales needs ammunition for a specific ONINO-vs-X conversation.
- A competitor launched new content / funding / product and we need to size the AI-visibility impact.

## Inputs

- `<competitor_name>` — the target (e.g. Tokeny, Cashlink, Stokr, Brickken)
- Date range: default past 28 days; tune to context (product launch window, etc.)

## Steps

### Step 1 — Resolve brand ids — 1 call

```
list_brands(project_id=<onino>)
  → filter by name = <competitor_name> → <competitor_brand_id>
  → filter by is_own = true → <onino_brand>
```

If the competitor isn't in the tracked brand set, this is the finding — add it before doing anything else.

### Step 2 — Head-to-head leaderboard — 1 call

Use recipe R4:

```json
get_brand_report({
  project_id: "<onino_project>",
  start_date: "<today - 28d>",
  end_date: "<today>",
  dimensions: ["brand_id", "tag_id", "model_id"],
  filters: [
    { field: "brand_id", operator: "in", values: ["<onino_brand>", "<competitor_brand_id>"] }
  ]
})
```

All metrics (visibility, share_of_voice, sentiment, position) come back automatically. Post-process client-side: compute `onino - competitor` delta per `(tag_id, model_id)` cell; sort by magnitude; the largest negative deltas are the action list.

### Step 3 — Where are they winning that we're not? — 1 call

Use the native `gap` filter operator plus a competitor-scoping filter (there is no top-level `mentioned_brand_ids` — filter uses singular `mentioned_brand_id` or the native `gap` operator):

```json
get_url_report({
  project_id: "<onino_project>",
  start_date: "<today - 28d>",
  end_date: "<today>",
  dimensions: ["tag_id"],
  filters: [
    { field: "gap", operator: "gt", value: 0 },
    { field: "mentioned_brand_id", operator: "in", values: ["<competitor_brand_id>"] }
  ],
  limit: 30
})
```

`classification` is a returned column (not a filter field) — sort / group client-side by `classification` after the response. Also sort by `citation_count` desc client-side; there is no `order_by` parameter.

### Step 4 — Read the top 3 winning URLs — 3 calls (drill-down)

```
get_url_content(url) × top 3 from Step 3
```

This tells you:
- What the competitor's positioning looks like where they win.
- Whether the article mentions ONINO at all (and if so, how).
- Whether we could plausibly get included via outreach to the author / editor.

### Step 5 (optional) — Sample their representative AI responses — 2 calls

```json
list_chats({
  project_id: "<onino_project>",
  start_date: "<today - 28d>",
  end_date: "<today>",
  limit: 20
})
```

`list_chats` doesn't accept a `brand_id` filter — filter the returned rows client-side for chats where the competitor appears on MOFU/BOFU prompts, then:

```json
get_chat({ project_id: "<onino_project>", chat_id: "<id>" })
```

(Note: parameter is `chat_id`, not `conversation_id`.) Read 2 representative chats to understand exactly what the assistant is saying about the competitor.

See exactly what the assistant is saying about them.

## Output

Write a short competitive brief (Slack or Notion doc) with:

- **Current position**: "ONINO is at Xpp visibility vs. [competitor] at Ypp on MOFU prompts."
- **Where they win**: Top 5 URL clusters (with URL, citing engine, and a one-line why).
- **Tone of coverage**: Positive / neutral / negative for them, based on the 2 chats read.
- **One action**: The single highest-leverage intervention (content update, editor outreach, product page change).

## Budget audit

| Step | Calls | Cumulative |
|---|---|---|
| 1. Resolve | 1 | 1 |
| 2. H2H leaderboard | 1 | 2 |
| 3. Their winning URLs | 1 | 3 |
| 4. Read top 3 URLs | 3 | 6 |
| 5. Sample chats (opt) | 2 | 8 |

Target: ≤6 without optional chat reading, ≤8 with.
