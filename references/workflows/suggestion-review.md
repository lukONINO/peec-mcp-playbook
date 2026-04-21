# Workflow — Weekly Opportunity Review (get_actions)

**Goal:** Keep ONINO's tracked prompt set, content coverage, and competitive posture aligned with the highest-leverage opportunities Peec surfaces. Replaces the pre-2026 "pending suggestion inbox" paradigm — Peec did not ship `list_prompt_suggestions` / `accept_*_suggestion`; it shipped `get_actions`, a server-scored recommendations engine.

**Cadence:** Every Monday, 20 minutes, before the pipeline sync meeting.
**Target:** 4–6 MCP read calls + up to 5 write calls (only when user explicitly approves). Most weeks end with ≤6 calls total.
**Preconditions:** `get_actions` available (confirmed live 2026-04-18). `list_brands`, `list_tags` cached from an earlier session call, or pull fresh at step 0.

---

## Why `get_actions` replaces the old workflow

The anticipated suggestion inbox never shipped. Instead, Peec rolled the "what should we do?" question into `get_actions`, which is more efficient because:

1. **One call gives a ranked navigation surface** (`scope=overview`) — no need to pull separate prompt/topic suggestion queues.
2. **Opportunity-scored, not time-sequenced** — instead of "new suggestions this week", you get the top opportunities across the full rolling window. No stale-backlog accumulation.
3. **Drill-down is typed** — `owned | editorial | reference | ugc` map cleanly to four different response strategies (write our own page, pitch an editor, add to Wikipedia/reference sites, seed UGC).
4. **Grounded in chats automatically** — the recommendations reference specific `url_classification` / `domain` values you can verify via `get_url_report` or `get_url_content` in one follow-up call.

Old 17-call workflow → new ≤6-call workflow. Stay disciplined about the call budget — `get_actions` is efficient only when you resist the urge to drill into every scope.

---

## Step 0 — Session bootstrap (cache once, reuse)

```json
Call A: list_projects            // if project_id not cached
Call B: list_brands              // to resolve own-brand + competitor ids
```

Skip both if already cached earlier in the session. 0 calls if warm.

---

## Step 1 — Pull the overview

**Always start here.** This gives you the ranked opportunity surface for the last 30 days.

```json
Tool: get_actions
Args: {
  "project_id": "<onino_project>",
  "scope": "overview"
  // start_date/end_date optional — default last 30 days, which is the right window for weekly review
}
```

Returns opportunity rollups grouped by `action_group_type × (url_classification | domain)`. This is navigation metadata — use it to pick the top 1–3 slices to drill.

---

## Step 2 — Triage the overview (client-side, 0 calls)

Rank the returned rows by opportunity score. Apply the ONINO ICP filter:

| Slice | Keep if | Drop if |
|---|---|---|
| `OWNED` | Any url_classification where we could publish (LISTICLE/COMPARISON/CATEGORY_PAGE/PRODUCT_PAGE/HOW_TO_GUIDE) | HOMEPAGE/PROFILE when we already have coverage |
| `EDITORIAL` | Classification that maps to a real editor pitch (LISTICLE, COMPARISON, ARTICLE) — especially if the opportunity is in DACH-relevant domains | Retail-investor publications, crowdinvesting editorials |
| `REFERENCE` | `wikipedia.org`, `bundesbank.de`, `bafin.de`, `investopedia.com`, industry association sites | retail-crypto reference sites |
| `UGC` | `reddit.com/r/Finanzen`, `reddit.com/r/eupersonalfinance`, `news.ycombinator.com`, YouTube channels covering B2B fintech | TikTok, Instagram, retail-crypto YouTube |

Pick the **top 3 slices** by opportunity score that pass ICP filtering. If only 1–2 pass, that's fine — smaller action list means higher execution probability.

---

## Step 3 — Drill into each selected slice (1 call per slice)

For each of the top 3 slices, make one follow-up `get_actions` call with the required extras:

```json
// If slice is OWNED (any url_classification):
{ "project_id": "<onino>", "scope": "owned" }
// or scoped to one classification:
{ "project_id": "<onino>", "scope": "owned", "url_classification": "LISTICLE" }

// If slice is EDITORIAL (url_classification required):
{ "project_id": "<onino>", "scope": "editorial", "url_classification": "COMPARISON" }

// If slice is REFERENCE (domain required):
{ "project_id": "<onino>", "scope": "reference", "domain": "wikipedia.org" }

// If slice is UGC (domain required):
{ "project_id": "<onino>", "scope": "ugc", "domain": "reddit.com" }
```

Each call returns textual recommendations in the `text` column, markdown-formatted with links to targets or examples. Read them verbatim into the output — don't paraphrase.

**Budget:** 3 drill calls max. If you're tempted to drill into 5 scopes, you're rehunting; 3 is the discipline.

---

## Step 4 — Ground one recommendation before acting (2 calls, optional but strongly recommended)

For the single highest-priority recommendation you intend to act on this week, verify it's real:

```json
Call P: get_url_report
Args: {
  "project_id": "<onino>",
  "start_date": "<last 30d>",
  "end_date": "<today>",
  "filters": [
    { "field": "url", "operator": "in", "values": ["<URL from get_actions text>"] }
  ]
}
// Confirms the URL is actually cited and by which chats/models

Call Q: get_chat
Args: { "project_id": "<onino>", "chat_id": "<one chat_id from Call P>" }
// Read the full AI response to confirm the recommendation lands in real buyer context
```

Skip this step only when the recommendation is obvious from the overview alone (e.g. "we have no ONINO content on `tokenized bonds EU` and two competitors do"). If the recommendation is subtle (e.g. "reframe positioning"), always ground it.

---

## Step 5 — Execute decisions

The only write actions available are the CRUD tools (create/update/delete on brand/prompt/tag/topic). Map each recommendation to the right write — or to a non-Peec deliverable.

| Recommendation pattern | Peec write action | Non-Peec deliverable |
|---|---|---|
| "Add tracking for <new prompt angle>" | `create_prompt` (with 5-tag overlay, see Step 6) | — |
| "Retire/replace <prompt X>" | `update_prompt` to add `retired` tag; optionally `delete_prompt` (cascades to chats — prefer retire) | — |
| "Add competitor <brand>" | `create_brand` + optional `regex` | — |
| "Correct brand alias drift" | `update_brand` (mind the recalc race — one call at a time) | — |
| "Consolidate tag case-duplicates" | `update_tag` then `delete_tag` (see `prompt-set-consolidation.md`) | — |
| "Publish LISTICLE on <topic>" | — | Asana task to Content, linked to the `get_actions` output |
| "Pitch editor at <domain>" | — | Linear/Asana task to Growth, with the exact URL gap as context |
| "Seed Reddit discussion on <topic>" | — | Asana task to Community, with the fanout query + top competitor URL |

**Each CRUD write requires explicit user approval.** Print the proposed call, wait for a clear "yes" before executing. No speculative loops.

---

## Step 6 — 5-tag overlay on every new prompt

If Step 5 produces a `create_prompt`, every ONINO prompt carries these 4 tags + native `country_code`:

| Family | Required value | How to decide |
|---|---|---|
| `funnel_stage:*` | `TOFU` / `MOFU` / `BOFU` | Is the searcher scoping (TOFU), comparing (MOFU), or vendor-evaluating (BOFU)? |
| `icp_segment:*` | `asset-manager` / `bank` / `platform-operator` / `club-coop` / `financing-consultant` | Read the prompt text; pick the primary buyer persona |
| `branded:*` | `branded` / `non-branded` | Does the prompt contain "ONINO"? If yes, `branded`; otherwise `non-branded` |
| `language:*` | `de` / `en` | Language of the prompt text |
| (native) `country_code` | ISO code like `DE`, `AT`, `CH`, `FR` | Required field on `create_prompt` — no `EU` meta-code |

Notes:
- The `region_target:` tag family is **deprecated** as of 2026-04-18 because `country_code` is now native on the prompt. Don't create new `region_target:*` tags. Existing `region_target:*` tags can be left in place or cleaned up during quarterly consolidation.
- Resolve each tag name against a fresh `list_tags` lookup. If a required tag doesn't exist, stop and `create_tag` first — do not guess tag IDs.

---

## Step 7 — Log and post

After decisions are executed:

- Tally: recommendations surfaced (by scope), actions taken (Peec writes + external tasks).
- Post to the Peec Slack channel:

  > **Mon 2026-MM-DD — Peec opportunity review**
  > Top opportunities this week:
  > 1. [owned/editorial/reference/ugc] — \<one-line summary\>
  > 2. …
  > Peec writes: X prompts created, Y brand updates
  > External tasks: Z created in Asana (links)

---

## Guardrails

- **Budget discipline.** 4–6 calls typical. If a session climbs to 15+, you've stopped using `get_actions` as the index and started re-hunting in `get_brand_report` / `get_url_report`. Step back.
- **Never accept a recommendation you can't ground.** If the overview surfaces something you can't trace to a specific chat or URL, the action is speculative.
- **Retire, don't delete.** When a recommendation is "replace prompt X", use `update_prompt` to add a `retired` tag rather than `delete_prompt`. Chat history is preserved for future baseline comparisons.
- **One destructive call at a time.** If consolidation recommendations stack (e.g. delete a topic + delete 3 orphan tags), that's the quarterly consolidation workflow, not the weekly review. Defer.
- **Ignore recommendations that route to retail/crowdinvesting surface.** If `get_actions` surfaces a UGC opportunity on `r/cryptocurrency`, drop it — that's not our ICP even if the opportunity score is high.

---

## Call budget

| Action | Calls | Typical weeks |
|---|---|---|
| Step 0 — bootstrap | 0–2 | 0 if cached |
| Step 1 — overview | 1 | 1 |
| Step 2 — triage | 0 | 0 (client-side) |
| Step 3 — drill | 3 | 3 |
| Step 4 — grounding | 2 | 2 (highly recommended) |
| Step 5 — writes | 0–5 | 1–2 typical |
| **Total typical week** | **4–6 reads + 1–2 writes** | **≤8 calls** |

Compare to the pre-2026 pending-inbox workflow (~17 calls). ≥50% reduction in MCP usage per weekly pass.
