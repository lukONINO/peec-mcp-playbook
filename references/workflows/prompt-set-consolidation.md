# Workflow — Prompt-Set Consolidation (Quarterly Clean-up)

**Goal:** Use CRUD tools to eliminate accumulated taxonomy drift — alias-split brands, case-duplicate tags, orphan topics, prompts missing tag-family coverage. This is the only workflow that routinely uses `destructiveHint: true` tools; safety rules below are non-negotiable.

**Cadence:** Quarterly. Also run once right after first enabling CRUD writes (as an inaugural clean-up).
**Target:** Session-scoped; typically 20–35 MCP calls depending on drift depth.
**Preconditions:** CRUD tools available (confirmed live 2026-04-18). User is in-session and can approve destructive calls one-by-one.

---

## Non-negotiable safety rules

1. **Every `delete_brand`, `delete_prompt`, `delete_tag`, `delete_topic` requires explicit per-call user confirmation.** Print the row being deleted (from a fresh `list_*` call) and wait for an explicit "yes delete" before proceeding.
2. **Prefer update over delete.** Consolidate via `update_*` whenever history is involved. Delete only when the target has zero attached history.
3. **`delete_prompt` cascades to chats.** The chat history tied to the prompt is soft-deleted along with it. If the prompt has any baseline value, use the **retire-via-tag** pattern instead — add a `retired` tag via `update_prompt`.
4. **Renames that could break scheduled tasks are forbidden until the downstream SKILL.md is updated in the same session.** Check `onino-ai-visibility-daily` for references to any tag/topic name you're about to rename.
5. **`update_brand` has a background recalc race.** After changing `name`, `aliases`, or `regex`, a repeat update call may fail until recalc completes. Space out consolidation calls — one at a time, no tight loops.
6. **Never run this workflow unattended.** Do not schedule it. Human-in-the-loop on every destructive step.

---

## Schema reality — reference before you plan

The playbook's earlier consolidation doc assumed schemas that don't match reality. Confirmed live as of 2026-04-18:

| Field/behavior | Reality | Impact |
|---|---|---|
| `create_brand` | Does NOT accept `is_own` | Can't fix mis-flagged own-brand via MCP. Log for Peec UI / support. |
| `create_brand` | Accepts `regex` for text matching | Use to detect ONINO variants with one pattern |
| `update_brand` | Background recalc after name/regex/alias changes; tight loops fail | Space calls, handle retries |
| `update_prompt` | Does NOT accept `text`; only `topic_id` and `tag_ids` | Rephrase = retire-tag + create new prompt |
| `update_prompt` | `tag_ids[]` REPLACES existing set (not merge) | Always pass full desired tag list |
| `delete_prompt` | Cascades to chats (soft-delete) | Prefer retire-via-tag |
| `delete_topic` | Detaches prompts (they survive), deletes topic-level suggestions | Safe-ish |
| `create_prompt` | Requires `country_code`; no `EU` meta-code | Pick a specific country |
| `create_tag` | First-class `color` enum | Apply consistent colors per namespace |

---

## Phase 1 — Discovery (4 reads)

### 1.1 Pull the current state

```json
Call A: list_brands      Args: { "project_id": "<onino_project>" }
Call B: list_tags        Args: { "project_id": "<onino_project>" }
Call C: list_topics      Args: { "project_id": "<onino_project>" }
Call D: get_brand_report
Args: {
  "project_id": "<onino_project>",
  "start_date": "<today - 90d>",
  "end_date": "<today>",
  "dimensions": ["prompt_id", "tag_id"]
}
```

Call D is the **authoritative prompt universe** (works around the `list_prompts` 50-row truncation). It enumerates every prompt that produced data in the window, with its current tag set.

### 1.2 Classify drift client-side

Produce a single drift report with four sections:

**Section 1 — Brand drift.** For each brand name, look for near-duplicates (Levenshtein distance ≤2; same-case-different-whitespace; common suffix variants like "Solutions"/"Inc"). Also flag brands with `is_own = false` whose domains overlap with an `is_own = true` brand (possible mis-flag — CAN'T be fixed via MCP, log for UI fix).

**Section 2 — Tag drift.** Group tags by lowercased-trimmed name. Any group with >1 member is a case-duplicate. Also flag:
- Tags not following `family:value` convention (e.g. bare `MOFU` instead of `funnel_stage:MOFU`).
- **`region_target:*` tags** — these are deprecated as of 2026-04-18 because `country_code` is now native on the prompt. Flag for removal.
- Tags with inconsistent colors within a namespace (e.g. two `funnel_stage:*` tags with different colors).

**Section 3 — Topic drift.** Count prompts per topic using Call D output. Topics with 0 prompts are orphans. Topic pairs with heavy prompt-overlap (same prompt IDs could live in either) are merger candidates.

**Section 4 — Prompt tag-coverage.** For every `prompt_id` in Call D, check all 4 tag families are present: `funnel_stage`, `icp_segment`, `branded`, `language`. Any missing family is a flag. (Note: `region_target` is no longer required since `country_code` is native — you can pull each prompt's `country_code` from `list_prompts` in small batches if you need to audit country coverage.)

### 1.3 Present the drift report to the user

Print as one summary before any write:

```
Drift report — <YYYY-MM-DD>

Brand drift:
  - [CONSOLIDATE] "Tokeny" + "Tokeny Solutions" → canonical "Tokeny Solutions"
  - [MIS-FLAGGED] "ONINO AG" has is_own=false (Peec UI fix only — no MCP support)
  - [MISSING REGEX] "ONINO" brand lacks regex; "ONINO GmbH" mentions may be undercounted

Tag drift:
  - [CASE DUP]    "branded" + "Branded" → canonical "branded"
  - [NAMING]      "MOFU" is missing family prefix → rename to "funnel_stage:MOFU"
  - [DEPRECATED]  "region_target:DACH" — superseded by native country_code; remove

Topic drift:
  - [ORPHAN] "Regulation crypto 2024" has 0 prompts
  - [MERGE]  "EU DLT" + "DLT Pilot Regime" → canonical "EU DLT Pilot Regime"

Prompt tag-coverage (country_code audit):
  - 12 prompts have no `language:*` tag
  - 4 prompts have no `branded:*` tag
  - 2 prompts have no `icp_segment:*` tag
```

Stop here. Ask the user which sections to action. Do not proceed to Phase 2 without explicit go.

---

## Phase 2 — Execution (writes, one category at a time)

Process in this order — each step builds on the previous.

### 2.1 Fix brand definitions (excluding `is_own`)

For each row where domains or regex need tightening:

```json
Tool: update_brand
Args: {
  "project_id": "<onino>",
  "brand_id": "<id>",
  "domains": ["onino.io", "onino.com", "docs.onino.io"],
  "regex": "(?i)\\bonino(?:\\s+(?:ag|gmbh))?\\b"
}
```

**Recalc race:** after each `update_brand`, wait ~30 seconds before the next update to the same or related brand. Don't batch these.

For `is_own` mis-flags, log them for Peec UI intervention — no MCP support.

### 2.2 Consolidate brand aliases

For each `CONSOLIDATE` pair:

1. Pick the canonical brand (usually the one with more history).
2. Call `update_brand` on the canonical: add the other's name into `aliases[]`, merge `domains`.
3. Wait for recalc.
4. Ask the user: "Delete the non-canonical dupe? It will orphan its historical mentions." Only `delete_brand` after explicit approval. (Note: delete is soft, not hard, but `get_brand_report` will no longer surface the deleted brand's mentions in rollups.)

### 2.3 Add missing tags

For each tag family that needs a new value (e.g. a new ICP segment, or re-creating a canonical after a case-dup cleanup):

```json
Tool: create_tag
Args: {
  "project_id": "<onino>",
  "name": "funnel_stage:BOFU",
  "color": "violet"
}
```

Record the returned `tag_id` for use in 2.4 and 2.6. Apply the color convention from `mcp-tool-reference.md` §22.

### 2.4 Rename / consolidate existing tags

For each `CASE DUP`, `NAMING`, or `DEPRECATED` row:

1. Decide canonical (or removal target).
2. **Before any rename**, grep the skill and scheduled-task SKILLs for references to the non-canonical name — fix downstream SKILL.md first.
3. Call `update_tag` on the canonical if its name needs adjustment:
   ```json
   { "project_id": "<onino>", "tag_id": "<id>", "name": "funnel_stage:MOFU", "color": "indigo" }
   ```
4. For each prompt currently carrying the non-canonical tag, call `update_prompt` to move its tag set to the canonical `tag_id`. **Remember `tag_ids[]` fully replaces** — pass the complete desired tag list:
   ```json
   {
     "project_id": "<onino>",
     "prompt_id": "<id>",
     "tag_ids": ["funnel_stage:MOFU", "icp_segment:asset-manager", "branded:non-branded", "language:en"]
   }
   ```
5. Once the non-canonical tag has zero attached prompts, ask user: "Delete the non-canonical tag?" Only `delete_tag` after explicit approval.

For **`region_target:*` deprecations**: follow the same pattern — update each attached prompt's `tag_ids[]` to drop the deprecated tag (don't replace with country_code tag — country_code is already native), then delete the tag.

### 2.5 Merge / delete topics

For each `MERGE` pair:

1. Decide canonical.
2. For each prompt attached to the non-canonical, call `update_prompt` to move it to the canonical `topic_id`. **Don't forget to also include the full `tag_ids[]` list** — if you only pass `topic_id`, the tags are preserved automatically (only `tag_ids` replacement is explicit).
3. Once the non-canonical has zero prompts, ask: "Delete empty topic?" `delete_topic` after explicit approval.

For each `ORPHAN`: ask: "Topic X has 0 attached prompts. Delete?" Only `delete_topic` on approval. (Soft-delete; prompts would be detached anyway.)

### 2.6 Fill missing prompt-tag families

For each prompt flagged in Section 4 of the drift report:

1. Infer the correct tag values from the prompt text + existing tags + `country_code` (pull from `list_prompts` per small batch if needed).
2. If inference isn't obvious → leave for Alex; don't guess.
3. For clear cases, call `update_prompt` with the extended `tag_ids` list. **Always pass the complete desired tag set** — `tag_ids[]` replaces, doesn't merge.

**Batch discipline:** no more than 5 prompt updates in a row before pausing to verify. Tag inference mistakes compound.

---

## Phase 3 — Verification (4 reads)

After all writes, re-pull:

```json
Call X: list_brands    (expect dedupe)
Call Y: list_tags      (expect canonical set only, region_target:* gone)
Call Z: list_topics    (expect orphans gone)
Call W: get_brand_report (same dimensions as Call D)
         (expect every prompt to carry 4 tag families)
```

Diff X/Y/Z/W vs. A/B/C/D. Any unexpected change = rollback point.

Post to the Peec Slack channel:

```
Quarterly consolidation — <YYYY-MM-DD>
Brands: N updated, M deleted (dupes)
Tags: P renamed, Q deleted (case-dupes + deprecated region_target:*)
Topics: R merged, S deleted (orphans)
Prompts: T re-tagged for full 4-family coverage
Next review: <+90 days>
```

---

## Rollback considerations

Peec's deletes are **soft**, which means restoration is *theoretically* possible (contact Peec support). That said, the following remain practically irreversible in-session:

- `delete_brand` on a brand with history → `get_brand_report` stops surfacing it (until restored).
- `delete_prompt` → cascade to chats; baseline comparisons break.
- `update_brand`/`update_tag` renames → any reports generated in the interim show old names until re-rendered.

For this reason, Phase 2 is staged: idempotent updates first (cheap to reverse), destructive deletes last and only with confirmation.

---

## Guardrails (repeated for emphasis)

- **One destructive call at a time. User approves each.**
- **Updates before deletes. Always.**
- **`tag_ids[]` replaces — pass the complete desired set.**
- **`update_brand` recalc race — no tight loops.**
- **Downstream scheduled-task awareness on every rename.**
- **If the drift report itself seems off** (e.g. flags a tag you know is correct) — stop, investigate Call B's output, don't proceed.
- **Do not action Section 4 (tag-family coverage) in bulk.** Batches of 5.

---

## Call budget

| Phase | Calls |
|---|---|
| Phase 1 | 4 reads |
| Phase 2 | ~20 writes (varies with drift depth — brand consolidation 2-4, tag consolidation 6-10, topic consolidation 2-4, prompt tag-fills 5-10) |
| Phase 3 | 4 reads |
| **Typical total** | **~28** |

Down from the pre-2026 estimate of 33 calls — the `region_target:*` deprecation trims a tag family, and the retire-via-tag pattern replaces destructive delete loops on most misfit prompts.
