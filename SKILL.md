---
name: peec-mcp-playbook
description: >
  Peec.ai MCP playbook for ONINO B2B AI-visibility work. Use for daily/weekly visibility reports, opportunity-scored recommendations (get_actions), competitor drills, content-gap hunts, query-fanout analysis, prompt-set audits, and Brand/Prompt/Tag/Topic CRUD. Encodes the verified 27-tool live MCP surface (dimensions[], gap filter, classification, is_own, country_code, search_queries, full CRUD, get_actions), the ICP-aligned prompt taxonomy (asset managers, banks, platform operators, clubs/coops, financing consultants across TOFU/MOFU/BOFU), and anti-patterns (ignore retail/crowdinvesting, don't tight-loop update_brand, retire-don't-delete prompts). Trigger on "Peec", "peec.ai", "AI visibility", "share of voice", "SoV", "GEO analysis", "AI search tracking", "generative engine optimization", "ChatGPT visibility", "Perplexity visibility", "query fanout", "prompt suggestions", "get_actions", or any `mcp__a7063981-...__*` Peec tool. Use before touching Peec so ICP framing is applied from the start.
version: 1.2
last_updated: 2026-04-18
---

# Peec MCP Playbook — ONINO B2B Infrastructure Edition

You are working against the Peec.ai MCP server (prefix `mcp__a7063981-0048-4421-9faa-e30e47784e46__`) to track and improve ONINO's visibility in AI-generated answers across ChatGPT, Claude, Perplexity, Gemini, Google AI Overviews, Microsoft Copilot, Grok, DeepSeek, and Llama. Every query you run should be grounded in the ONINO B2B ICP — **asset managers, banks, investment platform operators, investment clubs & cooperatives, financing consultants in real estate / renewables** — NOT retail investors, NOT crowdinvesting audiences.

## Before you start

1. **Read the ICP filter.** ONINO is white-label financing infrastructure for B2B buyers. Prompts, competitors, and gap analyses must be grounded in that universe. If a metric is driven by a retail/crowdinvesting prompt, flag the prompt set itself as misaligned — don't chase the gap. See `references/icp-prompt-library.md`.
2. **Know the fastest path for this task.** The 2026 update made one call replace ~5 for most weekly work — `get_actions(scope=overview)` is now the entry point for "what should we do next?" Before writing a loop, check the "Fastest paths" block below and the recipes in `references/query-recipes.md`. The `dimensions[]` multi-axis pattern, `gap` filter operator, `classification` column, `is_own` flag, and `country_code` filter each collapse 5-40 calls into 1.
3. **Plan by recipe, not by tool.** `references/query-recipes.md` has copy-paste MCP payloads for the 11 most common read tasks, verified against the live MCP on 2026-04-18. Starting from a recipe prevents the 40-call-expansion pattern and avoids the broken-schema mistakes (date_range nesting, metrics[], brand_ids top-level, order_by — none exist).
4. **Gate every destructive call individually.** `delete_*` tools soft-delete but cascade (chats for prompts, topic-level suggestions for topics). `update_brand` triggers a background recalc that races subsequent edits. Every destructiveHint call requires explicit per-call user approval. See Write-ops discipline below.

## Core principles

**Efficiency.** One well-shaped call using `dimensions: [model_id, tag_id, date, country_code]` replaces a loop of per-engine × per-stage × per-region calls. `get_actions(scope=overview)` replaces the old 5-call "what should I do?" pattern. Target budgets: ≤5 calls for "what should we do this week?", ≤8 calls for a daily report, ≤15 for a full weekly report, ≤12 for the monthly fanout pass. Anything above that is a signal you've missed a `dimensions` opportunity, a native filter operator, or the `get_actions` shortcut.

**ICP alignment.** Every metric must answer a question an ONINO B2B buyer would ask — or a question we need to answer *about* them. "Visibility in DACH retail-investor prompts" is noise. "Visibility when an asset manager searches for white-label SPV platforms" is signal.

**Integrity.** `list_prompts` silently truncates at 50 rows despite advertising `maximum: 10000` — always prefer `get_brand_report` with `dimensions: [prompt_id]` for the authoritative prompt universe. `visibility_count = 0, visibility_total > 0` means scraped-but-not-mentioned; `visibility_total = 0` means the engine didn't run. Never conflate. `list_search_queries` has a hard 1000-row cap regardless of the description's stated maximum.

**Transparency.** When something looks anomalous (σ>2 spike, new competitor surfacing, sentiment drop), pull `get_chat` on a representative chat ID and `get_url_content` on the top citing URL before writing the summary. Report what you actually saw, not what the aggregate implied.

## Fastest paths (memorize these five)

| Question | Tool + key args | Replaces |
|---|---|---|
| "What should we do this week?" | `get_actions({scope:"overview"})` | 5-8 chained report calls |
| "Full brand leaderboard across engines + ICP segments + DACH" | `get_brand_report({dimensions:["model_id","tag_id","country_code"]})` | 15+ per-slice calls |
| "Where are competitors winning and we aren't?" | `get_url_report({filters:[{field:"gap",operator:"gt",value:0}]})` | Manual `mentioned_brand_count>0 + not_in [onino]` construction |
| "Authoritative list of every tracked prompt" | `get_brand_report({dimensions:["prompt_id"]})` | `list_prompts` (broken — silently caps at 50) |
| "What queries does the model actually issue internally for our prompts?" | `list_search_queries({prompt_id, limit:1000})` — no `model_id` filter = engine-blind | Nothing — new capability 2026-04 |

See `references/query-recipes.md` for the full 11 recipes with parameter details.

## Decision tree — "I need to…"

```
I need to know what to work on this week (highest-leverage question)
   → R11 in query-recipes.md: get_actions(scope=overview) → drill into top 3 slices
   → workflows/suggestion-review.md (≤8 calls total)

I need to produce today's/this-week's AI visibility report
   → workflows/daily-report.md  (≤8 calls)
   → workflows/weekly-report.md (≤15 calls, with trend + anomaly detection)

I need to find where competitors are winning and we're not
   → R3 in query-recipes.md: get_url_report with native `gap` filter
   → For upstream (fanout-driven) gap hunt: workflows/fanout-discovery.md

I need to go head-to-head against one specific competitor
   → R4 in query-recipes.md (paired leaderboard)
   → workflows/competitor-drill.md

I need to audit / rebuild the Peec prompt set
   → references/icp-prompt-library.md + references/anti-patterns.md §1
   → For consolidation using CRUD tools: workflows/prompt-set-consolidation.md

I need to understand what the model is *actually* searching internally
   → workflows/fanout-discovery.md (monthly, ≤12 calls, uses list_search_queries)

I need to clean up accumulated taxonomy drift (alias-split brands, case-dupe tags, orphan topics)
   → workflows/prompt-set-consolidation.md (quarterly, ~28 calls, gated on user approval)

I want to understand why our score changed
   → R6 (sentiment drill-down) + R9 (rolling baseline) in query-recipes.md
   → always back with `get_chat` + `get_url_content`, never just aggregates

I need to create/rename/delete/retire a brand, prompt, tag, or topic
   → references/mcp-tool-reference.md §CRUD (12 CRUD tools — all gated on per-call user confirmation)
   → never call destructiveHint tools without explicit per-call approval
   → prefer retire-via-tag over delete for anything with baseline history

I'm unsure whether a tool/feature exists
   → references/mcp-tool-reference.md (single source of truth — 27 live tools)
   → references/peec-mcp-update-2026.md (changelog of what shipped vs. what didn't)
```

## Peec MCP tool surface — 27 live tools (verified 2026-04-18)

All tools below are live in the Cowork-hosted MCP (`mcp__a7063981-0048-4421-9faa-e30e47784e46__*`). For full schemas with parameter details and edge cases, see `references/mcp-tool-reference.md`.

### Reads (15)

| Tool | Use for | Key parameter you probably missed |
|---|---|---|
| `get_brand_report` | Brand-level visibility, SoV, sentiment, position, mention_count | `dimensions[]` (multi-axis rollup in one call); raw `*_count`/`*_total` fields |
| `get_url_report` | Per-URL citation counts, retrieval rate, url classification | `classification` returned column; `gap` filter operator |
| `get_domain_report` | Per-domain leaderboard (which sites are citing whom) | `classification` returned column (CORPORATE/EDITORIAL/UGC/COMPETITOR/OWN/…) |
| `get_chat` | Full assistant response with brand positions + cited sources | Parameter is `chat_id`, not `conversation_id` |
| `get_url_content` | Raw scraped markdown of an indexed URL | Filter on `url` string, not `url_id` |
| `get_actions` | **Opportunity-scored recommendations engine** — one-call ranked surface | Two-step: `scope:"overview"` then drill into `owned`/`editorial`/`reference`/`ugc` |
| `list_brands` | All tracked brands with `is_own` flag | `is_own` is read-only — never hardcode "ONINO" |
| `list_models` | Available engines + `is_active` per-project | Reveals which engines are actually running |
| `list_prompts` | Tracked prompts | **Broken — caps silently at 50**; use `get_brand_report` with `dimensions:[prompt_id]` |
| `list_tags` | Taxonomy (funnel-stage, branded, ICP-segment) | Watch for case-duplicates — merge defensively |
| `list_topics` | Topic clustering | Use as dimension for category rollups |
| `list_chats` | Enumerate recent conversations | Takes singular `prompt_id`, `model_id` — loop if you need multiple |
| `list_projects` | Available Peec projects | Rare — only at session start |
| `list_search_queries` | **AI-search query fanout** per prompt × engine × date | Singular `prompt_id`, `model_id`, `model_channel_id`; 1000-row hard cap |
| `list_shopping_queries` | Shopping/product query fanout | Low relevance for B2B ONINO; skip by default |

### Writes — Brand / Prompt / Tag / Topic CRUD (12)

| Family | Tools | Key schema fact |
|---|---|---|
| Brand | `create_brand`, `update_brand`, `delete_brand` | `is_own` is read-only (can't set via MCP). `regex` field accepted for alias matching. `update_brand` has a recalc race — don't tight-loop. |
| Prompt | `create_prompt`, `update_prompt`, `delete_prompt` | `country_code` required on create (ISO-3166, no `EU`). `text` max 200 chars. `update_prompt` does NOT accept text changes — retire-and-recreate. `tag_ids[]` REPLACES existing tags on update. `delete_prompt` cascades to chats. |
| Tag | `create_tag`, `update_tag`, `delete_tag` | First-class `color` enum (22 values). |
| Topic | `create_topic`, `update_topic`, `delete_topic` | Name max 64 chars. `delete_topic` detaches prompts + deletes topic-level suggestions. |

All CRUD tools carry `destructiveHint: true` where relevant and require per-call user confirmation.

### Announced but NOT shipped (don't plan around these)

`list_model_channels`, `list_prompt_suggestions`, `list_topic_suggestions`, `accept_prompt_suggestion`, `reject_prompt_suggestion`, `accept_topic_suggestion`, `reject_topic_suggestion` — none live as of 2026-04-18. The suggestion-inbox paradigm was replaced by `get_actions`. See `references/peec-mcp-update-2026.md` for the full changelog and substitute patterns.

## Write-ops discipline

When using any CRUD tool:

1. **Every destructive call requires per-call user confirmation.** Never batch-delete brands/prompts/tags/topics. Print the target row (from a fresh `list_*`) and wait for explicit go.
2. **Prefer retire over delete.** For prompts with baseline history, use `update_prompt` to add a `retired` tag rather than `delete_prompt` (which cascades to chats and erases historical continuity). `delete_*` is reserved for genuinely orphaned items or brand-new entries with no baseline.
3. **Prompt rephrase = retire + create.** `update_prompt` does NOT accept `text` changes (silently ignored). To fix a mis-phrased prompt: add `retired` tag, then `create_prompt` with the new text. See `workflows/fanout-discovery.md` §5.
4. **`update_prompt.tag_ids[]` REPLACES, not merges.** Always pass the complete intended tag list; partial writes strip the rest.
5. **4-tag + country_code overlay on every new prompt.** `funnel_stage + icp_segment + branded + language` as tags, plus native `country_code` on the prompt. The old `region_target:*` tag family is deprecated (redundant with native country_code).
6. **One `update_brand` at a time.** Background metric recalc races subsequent writes. Serialize; wait 30-60s; verify via `list_brands` before the next write.
7. **Downstream awareness.** Renaming a tag referenced in `onino-ai-visibility-daily` silently breaks it. Any rename must come with a scheduled-task SKILL.md update in the same pass.

## Tags / taxonomy ONINO uses

These tag families must be maintained in Peec for the playbook's workflows to compose correctly. If a tag is missing or case-duplicated, fix it (quarterly consolidation, see workflow) before running reports.

| Tag family | Purpose | Values |
|---|---|---|
| `funnel_stage:*` | Buyer-journey segmentation | `TOFU`, `MOFU`, `BOFU` |
| `icp_segment:*` | ICP-aligned prompt grouping | `asset-manager`, `bank`, `platform-operator`, `club-coop`, `financing-consultant` |
| `branded:*` | Distinguish brand-term queries | `branded`, `non-branded` |
| `language:*` | German / English split | `de`, `en` |
| (native) `country_code` | Geo of the prompt — required on `create_prompt` | ISO-3166: `DE`, `AT`, `CH`, `FR`, `NL`, `ES`, `IT`, ... |

**Deprecated 2026-04-18:** `region_target:DACH | EU | Global` — redundant with native `country_code`. Do not create new ones. Existing tags are harmless; clean up in quarterly consolidation.

See `references/icp-prompt-library.md` for the full 50-prompt taxonomy and how each prompt maps to these tags.

## Out of scope — do NOT chase these

- **DACH retail-investor prompts** ("Wie investiere ich 10 000 € in Immobilien?"). ONINO doesn't sell to retail — visibility there is not commercial signal.
- **Crowdinvesting category prompts** ("best crowdinvesting platforms DE"). ONINO is infrastructure that powers these platforms; it competes with Tokeny/Cashlink/Stokr/Bitbond, not with Companisto/Seedrs/Exporo.
- **Blockchain / crypto hype prompts** ("best crypto coins 2026"). Tokenization is a delivery efficiency on ONINO, not the pitch.
- **Investor-sourcing / fundraising prompts** ("how to find investors for my startup"). ONINO provides the platform, not the investors.
- **`get_actions` recommendations that route to retail/crowdinvesting surfaces.** If the overview surfaces a UGC opportunity on r/cryptocurrency, drop it even if the opportunity score is high.

Full list and corrective framing: `references/anti-patterns.md`.

## Reference map

| File | When to read |
|---|---|
| `references/mcp-tool-reference.md` | Single source of truth for all 27 live tool schemas. Check before every loop. |
| `references/query-recipes.md` | 11 copy-paste MCP payloads for the most common tasks. Start here. |
| `references/icp-prompt-library.md` | The 50-prompt ICP taxonomy + tag structure + competitor set. |
| `references/anti-patterns.md` | Mistakes to avoid — ICP drift, broken schemas, CRUD footguns. |
| `references/peec-mcp-update-2026.md` | Changelog: what shipped on 2026-04-18 vs. what didn't. Provenance only. |
| `references/workflows/suggestion-review.md` | **Weekly** opportunity review via `get_actions` (≤8 calls). |
| `references/workflows/fanout-discovery.md` | **Monthly** query-fanout pass (≤12 calls). |
| `references/workflows/prompt-set-consolidation.md` | **Quarterly** taxonomy cleanup (~28 calls, gated). |
| `references/workflows/daily-report.md` | Daily AI visibility report (≤8 calls). |
| `references/workflows/weekly-report.md` | Weekly report with trend + anomaly detection (≤15 calls). |
| `references/workflows/competitor-drill.md` | Head-to-head against one named competitor. |
| `references/workflows/content-gap-hunt.md` | Gap-driven content brief generation. |

## Handoff to the daily-report scheduled task

The `onino-ai-visibility-daily` scheduled task runs an older pre-playbook workflow (~38 calls, inferred URL types, manual gap filter). When updating that task's SKILL.md, copy the workflow from `references/workflows/daily-report.md` and reference the relevant recipe IDs — do not re-derive it. Consider folding in `get_actions(scope=overview)` as step 0 to front-load the "what matters today?" framing.

## Change log

- **v1.2 (2026-04-18)** — Verified against live hosted MCP. Unified tool surface to the 27 tools that actually shipped. Collapsed the anticipated "Gen 1 / Gen 2" split. `get_actions` promoted to top-level decision-tree entry and Fastest-paths table. Deprecated `region_target:*` tag family in favor of native `country_code`. Added Write-ops discipline updates (recalc race, retire-over-delete, `update_prompt.tag_ids` replace semantics, text-not-updatable). Removed 6 workflow branches that depended on unshipped tools (`accept_*_suggestion`, `list_*_suggestions`, `list_model_channels`). Tightened call budgets: weekly opportunity review 17 → 8, fanout 20 → 12, consolidation 33 → 28.
- **v1.1 (2026-04-17)** — Incorporated anticipated Peec MCP 2026 update. Added `references/peec-mcp-update-2026.md`, three new workflow files, write-ops discipline section, 3 new decision-tree branches. Pre-verification — later found to overclaim 6 unshipped tools.
- **v1.0 (2026-04-17)** — Initial release after cross-verifying 12 MCP tool schemas and walking back 9 earlier mischaracterizations. Encoded deep-ICP prompt taxonomy, verified feature list, and ≤8-call daily / ≤15-call weekly workflows.
