# Peec MCP 2026 Update — Verification Changelog

**Purpose:** Capture what actually shipped vs. what was announced, so we know which playbook workflows depend on unshipped tools.

**For tool specs:** always read `mcp-tool-reference.md`. That file is the single source of truth. This file is kept only as a provenance trail.

---

## What shipped (verified live 2026-04-18)

All tools below are exposed in the hosted Peec MCP (`mcp__a7063981-0048-4421-9faa-e30e47784e46__*`). 15 new tools bring the total from 12 to 27.

### New reads (3)

| Tool | Purpose | Notes |
|---|---|---|
| `get_actions` | Opportunity-scored recommendations engine (two-step: `scope=overview` → drill into `owned|editorial|reference|ugc`) | Replaces the anticipated suggestion-inbox paradigm. Single biggest efficiency gain in the update. |
| `list_search_queries` | AI-search query fanout per prompt × engine × date | Filters are **singular** (`prompt_id`, `model_id`). 1000-row hard cap. |
| `list_shopping_queries` | Shopping/product query fanout | Low relevance for ONINO B2B. Max 10000 rows but treat 1000 as safe default. |

### CRUD — Brands, Prompts, Tags, Topics (12)

`create_*`, `update_*`, `delete_*` for all four entity types. All deletes are **soft-delete**. All carry a server-side "confirm with user" instruction.

Key schema surprises vs. the 2026-04-17 draft:

| Tool | Schema reality | Impact on playbook |
|---|---|---|
| `create_brand` | Does NOT accept `is_own` (read-only flag derived by Peec) | Update `prompt-set-consolidation.md` §2.1 — can't fix mis-flagged own-brand via MCP |
| `create_brand` | Accepts `regex` field for brand matching | Use to consolidate ONINO alias detection in one pattern |
| `update_brand` | Background metric recalc; repeat attempts during recalc fail | Don't loop tightly over alias consolidation |
| `create_prompt` | `country_code` **required**; text max 200 chars; no `EU` meta-code | Partially obsoletes `region_target:` tag family |
| `update_prompt` | Does NOT accept `text` updates — only `topic_id`, `tag_ids` | To rephrase a prompt: delete + recreate, or use Peec UI |
| `update_prompt` | `tag_ids[]` fully REPLACES existing tag set (not merge) | Always pass the complete desired tag list |
| `delete_prompt` | Cascades to chats (soft-delete) | Prefer retire-via-tag over delete for historical continuity |
| `create_tag` | First-class `color` enum (22 values) | Cosmetic; use consistent colors per namespace for UI scanning |
| `create_topic` | Name max 64 chars; optional `country_code` | Tighter cap than prompt text |
| `delete_topic` | Detaches prompts, but DELETES topic-level suggestions | Confirms suggestions exist server-side but aren't exposed |

### Naming drift — resolved in favor of singular

The community MCP server (`thein-art/mcp-server-peecai`) used plural report names. The hosted Peec MCP kept the singular originals. Canonical:

- `get_brand_report` (not `get_brands_report`)
- `get_url_report` (not `get_urls_report`)
- `get_domain_report` (not `get_domains_report`)
- `get_chat` (not `get_chat_content`)

---

## What was announced but did NOT ship

These tools are referenced in earlier skill docs but are **not present** in the live hosted MCP. Do not plan workflows around them without re-verifying.

| Anticipated tool | Status | Closest live substitute |
|---|---|---|
| `list_model_channels` | Not shipped | `model_channel_id` is still selectable as a dimension / filter on reports; use `get_brand_report(dimensions:[model_channel_id])` for channel-level coverage |
| `list_prompt_suggestions` | Not shipped | None direct. `get_actions(scope=overview)` provides prioritized opportunities but not a pending-suggestion queue |
| `list_topic_suggestions` | Not shipped | None direct. Derive from `get_brand_report(dimensions:[topic_id])` thin clusters |
| `accept_prompt_suggestion` / `reject_prompt_suggestion` | Not shipped | Accept = `create_prompt`; reject = no-op (just don't create) |
| `accept_topic_suggestion` / `reject_topic_suggestion` | Not shipped | Accept = `create_topic`; reject = no-op |

**Workflow implication:** the old `suggestion-review.md` (pending-inbox model) has been rewritten around `get_actions`. See that file for the live weekly workflow.

---

## Open questions Peec hasn't confirmed

1. Do deletes ever purge irreversibly, or does soft-delete hold forever?
2. `list_search_queries` caps at 1000 rows but `list_shopping_queries` caps at 10000 — intentional or a schema mismatch on Peec's side?
3. Is `update_brand`'s recalc race surfaced as a specific error code, or just a generic retry failure?
4. Does `delete_prompt`'s chat cascade also affect `get_chat` lookups by ID, or just aggregation?

Answer these by running small tests against a non-production project, or ask Peec directly. Until confirmed, treat assumptions conservatively.

---

## Change log

- **2026-04-18** — Verified against live hosted MCP. 15 of the announced 21 tools shipped; 6 (suggestion inbox + actions + `list_model_channels`) did not. Reduced this doc to a changelog; specs moved to `mcp-tool-reference.md`.
- **2026-04-17** — Initial draft anticipating the update based on the community MCP README.
