# peec-mcp-playbook

A Claude skill for working efficiently against the **[Peec.ai](https://peec.ai) MCP server** — tracking and improving a brand's visibility in AI-generated answers across ChatGPT, Claude, Perplexity, Gemini, Google AI Overviews, Microsoft Copilot, Grok, DeepSeek, and Llama.

Encodes the 27-tool live MCP surface (verified 2026-04-18), copy-paste query recipes for the 11 most common tasks, schema gotchas, write-ops discipline, and workflow skeletons for daily/weekly/monthly/quarterly passes — with call-count budgets that replace 40-call loops with single `dimensions[]` or `get_actions` calls.

- **Version:** 1.2
- **Peec MCP verified:** 2026-04-18
- **Target runtime:** [Claude Code](https://docs.claude.com/en/docs/claude-code), [Cowork](https://claude.ai/cowork), [Claude Agent SDK](https://docs.claude.com/en/docs/agent-sdk)
- **License:** MIT

---

## Is this a drop-in skill for my company? No — it's a reference implementation.

Read this before installing.

This repo is the working playbook used at **ONINO** (B2B tokenized-financing infrastructure) to drive weekly AI-visibility work via Peec. It's published because the **Peec MCP technical layer** — tool schemas, `dimensions[]` patterns, schema gotchas, `get_actions` workflow, write-ops discipline — is universally useful to anyone using Peec, regardless of industry. But the skill has an **ONINO-specific business overlay** braided through it: ICP segments, competitor set, out-of-scope list, 50-prompt library, DACH regional framing.

You can use the skill two ways:

1. **As a Peec-MCP reference.** Read `references/mcp-tool-reference.md`, `references/query-recipes.md`, and `references/anti-patterns.md §2` — these are ~90% industry-agnostic. Ignore the ONINO business framing.
2. **As a template to fork.** Copy the skill into your own repo, then swap the ICP layer (see [Adapting this skill](#adapting-this-skill-for-your-company) below). Expect a one-afternoon rewrite to re-home it to your company.

If you want a version with no ONINO content at all, open an issue — happy to collaborate on a generalized fork.

---

## Why this skill exists

Peec.ai's hosted MCP shipped a large tool surface in 2026, but the *efficient* path through it is non-obvious. Naïve usage produces 40-call loops where 1 call would do. This skill encodes:

- **Five "fastest paths"** that collectively replace ~80% of report work
- **11 copy-paste query recipes** (R1–R11) with MCP payloads verified against the live server
- **Schema gotchas** — nine parameters that look like they should exist but don't (`date_range` nesting, `metrics[]`, `brand_ids` top-level, `order_by`, `conversation_id`, `url_id` filter, etc.)
- **Write-ops discipline** for the 12 CRUD tools (e.g., `update_prompt.tag_ids[]` replaces rather than merges; `update_brand` triggers a recalc race; `list_prompts` silently truncates at 50 rows)
- **Anti-patterns** harvested from real misfires (building the prompt universe from the broken `list_prompts`, hand-constructing the content-gap filter, hardcoding the own-brand name, etc.)
- **Workflow skeletons** for daily reports, weekly reports, opportunity reviews, competitor drills, content-gap hunts, quarterly taxonomy consolidation, and monthly query-fanout passes — each with target call budgets

---

## File inventory

Files are labeled by how much adaptation they need for non-ONINO use.

| File | Contents | Adaptation needed |
|---|---|---|
| `SKILL.md` | Top-level playbook + decision tree + fastest paths | **Medium** — rewrite ICP framing in intro, "out of scope" section, tag taxonomy table |
| `references/mcp-tool-reference.md` | 27-tool schema reference | **None** — fully generic |
| `references/query-recipes.md` | 11 copy-paste MCP payloads | **Light** — replace `<onino_project>` / `<onino_brand>` placeholders with your own |
| `references/anti-patterns.md` | §1 ICP anti-patterns + §2 MCP usage anti-patterns | **§1: Heavy** (ONINO ICP-specific). **§2: None** (fully generic) |
| `references/icp-prompt-library.md` | 50-prompt taxonomy tied to 5 ICP segments | **Full rewrite** — ONINO-specific prompts, segments, competitor set |
| `references/peec-mcp-update-2026.md` | Changelog of what shipped vs. what was announced | **None** — factual record of the 2026-04-18 MCP update |
| `references/workflows/daily-report.md` | ≤8-call daily workflow | **Light** — ICP-aware output structure needs adapting |
| `references/workflows/weekly-report.md` | ≤15-call weekly workflow | **Light** |
| `references/workflows/suggestion-review.md` | ≤8-call `get_actions` opportunity review | **Light** — ICP triage tables reference ONINO segments |
| `references/workflows/competitor-drill.md` | Head-to-head against one competitor | **Light** — example competitors are ONINO's |
| `references/workflows/content-gap-hunt.md` | Gap-driven content brief generation | **Light** |
| `references/workflows/fanout-discovery.md` | Monthly query-fanout pass | **Light** |
| `references/workflows/prompt-set-consolidation.md` | Quarterly taxonomy cleanup | **None** — generic taxonomy hygiene |

---

## Installing

Skills are plain folders. To install:

**Claude Code / Agent SDK:** drop the `peec-mcp-playbook/` folder into your project's `.claude/skills/` directory (or the user-level `~/.claude/skills/`). Claude will auto-surface it via the `description` field in `SKILL.md` when relevant queries come in.

**Cowork:** drop into your Cowork skills folder (path varies by install — check `~/Library/Application Support/Claude/...` on macOS).

**Prerequisite:** the Peec.ai MCP server must be connected to your Claude runtime. See [peec.ai docs](https://docs.peec.ai/mcp/introduction) for the MCP install. Your Peec account needs at least one active project with brands and prompts configured.

### Heads-up: the MCP prefix varies per install

Throughout this skill and its references, Peec MCP tools are referenced as `mcp__a7063981-0048-4421-9faa-e30e47784e46__*`. That UUID is **the Cowork-assigned prefix on the author's install** — not a universal Peec constant. Your runtime will assign a different UUID.

After installing, run your MCP tool-list command (or ask Claude "what MCP tools are available?") to find your actual Peec prefix, then either:
- Do a find/replace on the UUID across all files, or
- Leave the docs as-is and trust that Claude will map the intent to your local prefix (works fine in practice — the tool names after the prefix are stable).

---

## Adapting this skill for your company

If you're forking this, here's a concrete path from ONINO → your company. Roughly an afternoon of work.

### 1. Define your ICP layer (30 min)

Open `references/icp-prompt-library.md`. Replace the "5 ICP segments × 3 funnel stages" table with your own. Example mappings:

- ONINO → you. "Asset managers / banks / platform operators / clubs-coops / financing consultants" → *your* 3–6 buyer segments.
- Keep the funnel-stage split (TOFU / MOFU / BOFU) — this is portable across B2B.
- Define your `icp_segment:*`, `funnel_stage:*`, `branded:*`, `language:*` tag families in Peec before running any workflows.

### 2. Rewrite the out-of-scope list (15 min)

Open `SKILL.md` §"Out of scope" and `references/anti-patterns.md` §1. ONINO's list ("don't chase retail, don't chase crowdinvesting, don't lead with blockchain/crypto") is specific to B2B infra in a space adjacent to retail crypto. Your out-of-scope list will be different — for a marketing-analytics SaaS targeting CMOs, it might be "don't chase small-business owner prompts, don't chase agency-buyer prompts unless agency-specific." Write it out; the workflows use it to filter `get_actions` recommendations.

### 3. Rewrite the prompt library (60–90 min)

The 50 prompts in `icp-prompt-library.md` are ONINO-specific. Keep the **structure** (prompts grouped by ICP segment × funnel stage, carrying `icp_segment`, `funnel_stage`, `branded`, `language`, `country_code` tags) but rewrite every prompt. For each segment × stage cell, draft 3–5 prompts that a real buyer in that segment would type into ChatGPT/Perplexity.

### 4. Swap placeholders in query recipes (15 min)

Open `references/query-recipes.md`. Every recipe has placeholders like `<onino_project>` and `<onino_brand>`. These are already meant to be replaced — just rename to `<your_project>` / `<your_brand>`. Tag-name references (e.g., `funnel_stage:BOFU`) only need rewriting if you're using different tag families.

### 5. Update the competitor set (10 min)

Search the repo for "Tokeny | Cashlink | Stokr | Brickken | Bitbond | Finexity" and replace with your competitors. `references/anti-patterns.md` §1.2 calls out who is *not* a competitor (out-of-category) — replicate this distinction for your space.

### 6. Strip integrations you don't use (10 min)

The workflows reference Slack, Asana, Notion, and Linear in the "log and post" steps. Remove or retarget these to your stack.

### 7. Optional: remove DACH-specific framing

ONINO is DACH-primary, so the skill emphasizes German-language prompts and DE/AT/CH `country_code` filtering. If you're not DACH-focused, delete the DACH-specific recipe (R8) and reframe the language/country split around your markets.

### What you don't need to touch

- `references/mcp-tool-reference.md` — the entire 27-tool reference is industry-agnostic. Only update the version/verified-date stamp if you re-verify against a newer Peec MCP release.
- `references/anti-patterns.md` §2 — the MCP usage anti-patterns (broken `list_prompts`, native `gap` filter, `classification` column, hardcoded own-brand name) are universal.
- `references/peec-mcp-update-2026.md` — factual changelog of the 2026-04-18 MCP update. Leave as-is.
- Call-budget targets throughout — these reflect real MCP efficiency, not ONINO specifics.

---

## What's in the Peec MCP surface (as of 2026-04-18)

Full schemas: `references/mcp-tool-reference.md`.

**Reads (15):** `get_brand_report`, `get_url_report`, `get_domain_report`, `get_chat`, `get_url_content`, `get_actions`, `list_brands`, `list_models`, `list_prompts`, `list_tags`, `list_topics`, `list_chats`, `list_projects`, `list_search_queries`, `list_shopping_queries`.

**Writes (12, all soft-delete + per-call user confirmation):** `create_brand`, `update_brand`, `delete_brand`, `create_prompt`, `update_prompt`, `delete_prompt`, `create_tag`, `update_tag`, `delete_tag`, `create_topic`, `update_topic`, `delete_topic`.

**Announced but not shipped (don't plan around):** `list_model_channels`, `list_prompt_suggestions`, `list_topic_suggestions`, `accept_prompt_suggestion`, `reject_prompt_suggestion`, `accept_topic_suggestion`, `reject_topic_suggestion`. The suggestion-inbox paradigm was replaced by `get_actions`.

---

## Five "fastest paths" to memorize

These five patterns collapse roughly 80% of typical report work:

1. `get_actions({scope: "overview"})` — single call replaces 5–8 chained report calls for "what should we do this week?"
2. `get_brand_report({dimensions: ["model_id", "tag_id", "country_code"]})` — full cross-tab leaderboard in one call.
3. `get_url_report({filters: [{field: "gap", operator: "gt", value: 0}]})` — native content-gap filter in one expression.
4. `get_brand_report({dimensions: ["prompt_id"]})` — authoritative prompt universe (works around `list_prompts` 50-row truncation).
5. `list_search_queries({prompt_id, limit: 1000})` — AI-search query fanout per prompt (new capability, 2026-04).

---

## Versioning

This skill versions against the **Peec MCP surface it was verified against**, not against semver of the skill content itself. A bump in the version string means the tool schemas, filter operators, or edge cases were re-audited against the live hosted MCP on a specific date.

### Change log

**v1.2 — 2026-04-18**
Verified against live hosted MCP. Unified tool surface to the 27 tools that actually shipped. Collapsed the anticipated "Gen 1 / Gen 2" split. `get_actions` promoted to top-level decision-tree entry and fastest-paths table. Deprecated `region_target:*` tag family in favor of native `country_code`. Added write-ops discipline updates (recalc race, retire-over-delete, `update_prompt.tag_ids` replace semantics, text-not-updatable). Removed 6 workflow branches that depended on unshipped tools (`accept_*_suggestion`, `list_*_suggestions`, `list_model_channels`). Tightened call budgets: weekly opportunity review 17 → 8, fanout 20 → 12, consolidation 33 → 28.

**v1.1 — 2026-04-17**
Incorporated the anticipated Peec MCP 2026 update. Added `references/peec-mcp-update-2026.md`, three new workflow files, write-ops discipline section, three new decision-tree branches. Pre-verification — later found to over-claim six unshipped tools; corrected in v1.2.

**v1.0 — 2026-04-17**
Initial release after cross-verifying 12 MCP tool schemas and walking back 9 earlier mischaracterizations. Encoded deep-ICP prompt taxonomy, verified feature list, and ≤8-call daily / ≤15-call weekly workflows.

---

## Contributing

Contributions welcome. A few ground rules:

- **Verify before documenting.** The "schema gotchas" table in `references/mcp-tool-reference.md` exists because an earlier version of this skill documented parameters that didn't exist (`date_range` nesting, `metrics[]`, `order_by`, etc.). If you add a recipe or edit a tool schema, verify it against the live Peec MCP and cite the verification date in your PR.
- **Update the version/verified-date stamp** in `SKILL.md` and `README.md` whenever you re-verify against a newer Peec MCP release.
- **Keep the call-budget targets.** The ≤8-call daily / ≤15-call weekly / ≤28-call quarterly budgets are there because bloated workflows waste API quota and slow the human in the loop. If a workflow grows past its budget, the first question is whether a `dimensions[]` opportunity or native filter operator got missed.
- **Don't over-generalize the ICP layer.** If you contribute a new workflow, it's fine (encouraged) to include a worked example with concrete ICP segments, as long as the generic skeleton is separable.
- **ONINO-specific content lives in this fork, not upstream.** If a generalized fork ever materializes, the ONINO business overlay will stay here and the generic playbook will move there.

---

## License

MIT. See `LICENSE`.

---

## Links

- [Peec.ai](https://peec.ai) — the product this skill wraps
- [Claude Code](https://docs.claude.com/en/docs/claude-code)
- [Claude Agent SDK](https://docs.claude.com/en/docs/agent-sdk)
- [Cowork](https://claude.ai/cowork)
# peec-mcp-playbook
