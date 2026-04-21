# Peec MCP — Anti-Patterns & Common Mistakes

Read this before every significant Peec work session. Lessons learned from the 2026-04-17 daily-report run plus schema-drift findings from the 2026-04-18 live-MCP audit.

---

## § 1 — Prompt-set anti-patterns (the single biggest source of bad signal)

### 1.1 Chasing retail-investor visibility gaps

**Anti-pattern:** "ONINO isn't appearing on prompts like 'beste Crowdinvesting-Plattformen 2026'. Let's write content to fix that."

**Why it's wrong:** ONINO's ICP is B2B infrastructure buyers — asset managers, banks, platform operators. Retail investors don't buy ONINO; platforms that serve retail investors might, but they shop as platform operators, not as retail investors. A high-visibility score on a retail prompt has near-zero commercial value.

**Correction:** Reframe as a platform-operator prompt: "Welche Whitelabel-Plattformen betreiben führende Crowdinvesting-Anbieter?" is a good prompt. "Best crowdinvesting platform" is not.

### 1.2 Confusing out-of-category competitors with competitors

**Anti-pattern:** Tracking Companisto, Seedrs, Exporo, Bergfürst as ONINO competitors.

**Why it's wrong:** These are crowdinvesting platforms — they're potential *customers* of ONINO, not competitors. Any dashboard where "ONINO loses to Exporo" is a category-confusion artifact, not a competitive signal.

**Correction:** Competitive set is infrastructure/platform: Tokeny, Cashlink, Stokr, Brickken, Bitbond, Finexity.

### 1.3 Leading prompts with "blockchain" or "crypto"

**Anti-pattern:** Building the prompt set around "best blockchain platforms for finance", "crypto tokenization software".

**Why it's wrong:** ONINO's positioning is "financing infrastructure" — tokenization is an efficiency, not the pitch. B2B buyers search in terms of financing, compliance, investor management, securities issuance, white-label platforms. The prompt set should mirror how buyers search, not how vendors speak.

**Correction:** Lead the prompt set with "financing infrastructure", "white-label platform", "securities issuance", "investor management". Tokenization-first prompts should exist but stay under 20% of the set.

### 1.4 Ignoring language / region tags

**Anti-pattern:** Running a single leaderboard without splitting by language or country.

**Why it's wrong:** ONINO is DACH-primary. A 30% visibility in English US prompts is less valuable than 30% in German DACH prompts. Without `language` and `country_code` dimensions, you can't distinguish them.

**Correction:** Every serious report should split by `language` (de/en) and/or `country_code` (DE/AT/CH/EU). Use recipe R8 in `query-recipes.md`.

---

## § 2 — MCP usage anti-patterns

### 2.1 Building the prompt universe from `list_prompts`

**Anti-pattern:** `list_prompts(limit: 100)` → build full prompt universe from the returned 50 rows.

**Why it's wrong:** `list_prompts` silently truncates at 50 rows on the ONINO project. The advertised `maximum: 10000` is not honored. No warning, no `has_more`, no `total_count`. You'll build the full report on one-third of reality.

**Correction:** Use `get_brand_report` with `dimensions: [prompt_id]` — the aggregation path returns all prompts. Enrich tags via `list_prompts` per small batch if you need them.

### 2.2 Inferring URL classification from URL strings

**Anti-pattern:** Regex-matching `/compare/` or `/vs-/` in URL paths to classify listicles.

**Why it's wrong:** `get_url_report` returns a `classification` column with 11 machine-generated values (`LISTICLE, COMPARISON, CATEGORY_PAGE, PRODUCT_PAGE, HOMEPAGE, PROFILE, ALTERNATIVE, DISCUSSION, HOW_TO_GUIDE, ARTICLE, OTHER`). Your regex will be wrong more often than this classifier.

**Correction:** Always prefer the `classification` column. Spot-check only when classification is decisive (e.g. you're about to pitch an editor based on a "LISTICLE" flag).

### 2.3 Inferring domain type from domain names

**Anti-pattern:** String-matching for ".edu", "reddit.com", "cointelegraph.com" to classify editorial vs. UGC vs. corporate.

**Why it's wrong:** `get_domain_report` returns a `classification` column (`CORPORATE, EDITORIAL, INSTITUTIONAL, UGC, REFERENCE, COMPETITOR, OWN, OTHER`). Use it.

### 2.4 Manually reconstructing the content-gap filter

**Anti-pattern:** `filters: [mentioned_brand_count > 0, mentioned_brand_id not_in [ONINO]]` to find content gaps.

**Why it's wrong:** There's a native `gap` filter operator that does exactly this in one expression. The manual construction is brittle (misses edge cases where own-brand has an alias that's not in the array) and more verbose.

**Correction:** `{field: "gap", operator: "gt", value: 0}`. See recipe R3.

### 2.5 Hardcoding "ONINO" as the own-brand name

**Anti-pattern:** Every filter carries `brand_ids: ["ONINO"]` as a string constant.

**Why it's wrong:** Brittle. Breaks if the brand id gets reassigned in Peec. Non-portable across projects.

**Correction:** `list_brands` → find the row with `is_own: true` → use that id. Cache per session.

### 2.6 Confusing `visibility_count = 0` with "engine didn't run"

**Anti-pattern:** Reporting "Perplexity visibility: 0%" when Perplexity wasn't even active on the project.

**Why it's wrong:** A 0% is a fact; a missing engine is a setup issue. The two have opposite implications for the team reading the report. Currently on the ONINO project, Perplexity / Claude / Gemini are **not active** — any metrics for them are absences, not zeros.

**Correction:** Check `visibility_total` alongside `visibility_count`. `total > 0, count = 0` → scraped-not-mentioned (0%). `total = 0` → engine didn't run (exclude from the report or flag explicitly).

### 2.7 Making 40 sequential calls when `dimensions[]` collapses to 1

**Anti-pattern:** A loop over engines × funnel stages × regions, each making a separate `get_brand_report` call.

**Why it's wrong:** `dimensions: ["model_id", "tag_id", "country_code"]` returns the fully rolled-up cross-tab in a single call. 40→1 is common.

**Correction:** Default to multi-dimensional calls. Only split into separate calls when filter logic genuinely differs per slice (e.g. you need separate date-ranges).

### 2.8 Summarizing from aggregates without drill-down

**Anti-pattern:** "Sentiment dropped 8 points this week — probably the new Perplexity update." Posted to Slack without reading a single chat.

**Why it's wrong:** Sentiment movements often have narrative explanations (a new article got cited, a competitor launched, a product change was reported). Summarizing from aggregates regularly leads to wrong stories.

**Correction:** Before any anomaly is summarized, pull 2-3 representative chats via `get_chat` and the top cited URL via `get_url_content`. Confirm the story. Then write it.

### 2.9 Not merging tag case-duplicates

**Anti-pattern:** Reporting `branded` and `Branded` as separate tag buckets.

**Why it's wrong:** Case-duplicates happen in Peec. A report that lists them separately is just confusing.

**Correction:** Always lowercase + trim tag names before grouping. Flag case-duplicates explicitly in the integrity section so someone can clean them up in the UI.

### 2.10 Chaining separate `get_brand_report` / `get_url_report` / `get_domain_report` calls to reach "what should we do next?"

**Anti-pattern:** Build the weekly opportunity list by running R1 (leaderboard) → R3 (gap hunt) → R7 (domain breakdown) → manual triage. Five to eight reads before the first recommendation.

**Why it's wrong:** `get_actions` (shipped 2026-04-18) is an opportunity-scored recommendations engine. `scope=overview` returns a ranked navigation surface in one call; 1–3 typed drill-downs (`owned | editorial | reference | ugc`) expand the single top opportunity.

**Correction:** Start with `get_actions(scope=overview)`. Only fall back to the raw report tools when you need the underlying metrics `get_actions` doesn't expose (e.g. trend math, per-engine resolution). See recipe R11 in `query-recipes.md` and the weekly workflow in `workflows/suggestion-review.md`.

---

## § 3 — Reporting / narrative anti-patterns

### 3.1 Mixing TOFU and MOFU in a "top movers" section

**Anti-pattern:** "Top 5 visibility wins this week: [3 TOFU, 1 MOFU, 1 BOFU]".

**Why it's wrong:** TOFU visibility has ~5-10× less commercial value per point than MOFU visibility. A "top movers" section should be stage-specific, or stages should be separately weighted.

**Correction:** Split into "MOFU/BOFU wins" and "TOFU wins" — or explicitly weight by stage.

### 3.2 Leading with aggregate visibility

**Anti-pattern:** TL;DR opens with "ONINO visibility: 33% (+2pp WoW)".

**Why it's wrong:** The aggregate hides the story. The real signals are: which ICP segment we're invisible to, which MOFU prompts we lost position on, which new competitor is emerging.

**Correction:** Lead the TL;DR with the segment/prompt finding that matters most, not the aggregate number.

### 3.3 Reporting share-of-voice without sample size

**Anti-pattern:** "ONINO SoV on 'ESG tokenization' prompts: 83%."

**Why it's wrong:** If only 2 chats surfaced for that prompt cluster, 83% is 1.66 mentions out of 2 — statistical noise.

**Correction:** Always report SoV alongside `mention_count` or `chat_count`. Don't cite SoV at all if sample size < 5.

### 3.4 Claiming causation from a single week of data

**Anti-pattern:** "Our new blog post drove visibility up 8pp this week."

**Why it's wrong:** A single week is rarely statistically distinguishable from noise, especially on prompts with moderate sample size. Attribution to content is almost always premature before the 4-week mark.

**Correction:** State movements as observations, not causes, until you have a 4-week baseline. Use recipe R9 for the baseline.

---

## § 4 — Known Peec bugs and workarounds

| Bug | Workaround |
|---|---|
| `list_prompts` silently caps at 50 rows | Use `get_brand_report` with `dimensions:[prompt_id]` |
| Tag case-duplicates (`branded` vs `Branded`) | Lowercase + trim before grouping; flag in integrity section |
| No structured `warnings` array on responses | Defensive row-count checks; compare against expected dimensions product |
| No citation-level sentiment | Drill down via `get_url_content` + manual read |
| No prompt-set changelog | Snapshot `list_prompts` + `dimensions:[prompt_id]` weekly into Notion |
| No scrape-completeness endpoint | Compute from `visibility_total` across `dimensions:[model_id, date]` |
| `update_brand` background recalc race (repeat edits fail during recalc window) | Serialize brand edits; wait between calls; don't tight-loop |
| `list_search_queries` hard cap at 1000 rows (despite description allowing up to 10000) | Narrow date window to 14-30 days; paginate with `offset` when `rowCount == 1000` |
| `update_prompt` does NOT accept `text` changes | Retire via `update_prompt` adding `retired` tag, then `create_prompt` with the new text |
| `delete_prompt` soft-deletes but cascades to chat history | Prefer retire-via-tag pattern; delete only when baseline continuity doesn't matter |

These bugs are tracked for feedback to Peec — see `/Users/lukaswipf/Documents/Claude/Projects/ONINO GTM/peec-mcp-product-feedback-2026-04-17.md` when it exists.

---

## § 5 — 2026 CRUD / fanout surface anti-patterns (verified against live MCP 2026-04-18)

The 15 new write-tools and fanout-read-tools shipped with sharp edges. These are the ones that have already caused or will cause real damage.

### 5.1 Rephrasing a prompt via `update_prompt(text: "...")`

**Anti-pattern:** Call `update_prompt` with a new `text` field to fix a mis-phrased prompt.

**Why it's wrong:** `update_prompt` accepts `topic_id` and `tag_ids` only. Passing `text` is silently ignored or rejected — the prompt keeps its old wording while your dashboard says "updated". The observed retrieval behavior won't match the phrasing in your source of truth.

**Correction:** Use the retire-and-recreate pattern. Call `update_prompt` to add a `retired` tag (preserves historical chats), then `create_prompt` with the new text. If baseline continuity doesn't matter, `delete_prompt` + `create_prompt` is acceptable (chats cascade — see §5.3).

### 5.2 Passing partial `tag_ids[]` to `update_prompt` expecting merge semantics

**Anti-pattern:** `update_prompt(prompt_id, tag_ids: ["retired"])` intending to add the retired tag while keeping existing tags.

**Why it's wrong:** `tag_ids[]` fully REPLACES the tag set — it is not additive. Passing `["retired"]` alone will strip every other tag (funnel_stage, icp_segment, language, branded) off the prompt. Reports will silently lose the prompt from every filtered view.

**Correction:** Always pass the complete intended tag list. Read current tags via `get_brand_report` with `dimensions:[prompt_id, tag_id]`, union with the new tag, then write the full set.

### 5.3 Treating `delete_prompt` as reversible

**Anti-pattern:** "Peec uses soft-delete so I can just delete and restore later if we change our mind."

**Why it's wrong:** `delete_prompt` is soft-delete at the prompt level but CASCADES to all associated chats. The chats disappear from `list_chats` / `get_chat` aggregation. You cannot recover the historical conversation evidence that drove baselines and sentiment trends. Restoration (if Peec supports it at all — unconfirmed) would bring back the prompt but not necessarily the chat linkage.

**Correction:** For prompts with meaningful baseline history, always prefer retire-via-tag. Reserve `delete_prompt` for prompts created in the last week that have no baseline yet.

### 5.4 Creating prompts with `region_target:DACH` tags

**Anti-pattern:** Continuing to add `region_target:DACH`, `region_target:EU`, `region_target:Global` tags on every new prompt per the old 5-tag overlay.

**Why it's wrong:** `create_prompt` now has `country_code` as a **required native field** (95+ ISO-3166 enum: `DE`, `AT`, `CH`, `FR`, `NL`, etc.). The `region_target:*` tag family is redundant with native country_code and has no `EU` meta-value at the native level — meaning `country_code` is strictly per-country. The tag family is deprecated as of 2026-04-18.

**Correction:** Drop `region_target:*` from the required tag overlay. The new required surface is 4 tags + native `country_code`: `funnel_stage`, `icp_segment`, `branded`, `language` + `country_code` on the prompt itself. Existing `region_target:*` tags can be left in place (harmless) or cleaned up in quarterly consolidation.

### 5.5 Passing arrays to singular-only fanout filters

**Anti-pattern:** `list_search_queries(prompt_ids: ["id1", "id2", "id3"])` or `list_search_queries(model_ids: [...])`.

**Why it's wrong:** Fanout tools take SINGULAR filters only — `prompt_id`, `model_id`, `model_channel_id`. Arrays are either rejected or silently treat the first value only. Code that expected batching will silently under-sample.

**Correction:** Loop. One call per prompt × engine combination if you need per-engine resolution, or one call per prompt with no `model_id` (engine-blind — preferred for the monthly pass; ~5 calls for 5 prompts instead of 20). See `workflows/fanout-discovery.md`.

### 5.6 Tight-looping `update_brand` during alias consolidation

**Anti-pattern:** Consolidating ONINO's alias set by calling `update_brand` repeatedly with different partial aliases in rapid succession.

**Why it's wrong:** `update_brand` triggers a background metric recalc. During the recalc window, subsequent `update_brand` calls on the same brand fail (not always with a clean error — sometimes just silent no-op). The alias set ends up partially applied.

**Correction:** Consolidate aliases in a single `update_brand` call with the full final list. If that's not possible (e.g. you're exploring), wait 30-60 seconds between calls and verify the prior write landed via `list_brands` before the next write. Never loop tightly.

### 5.7 Trying to toggle `is_own` via `create_brand` / `update_brand`

**Anti-pattern:** `create_brand(name: "ONINO", is_own: true)` or `update_brand(brand_id, is_own: true)` to fix a mis-classified own-brand.

**Why it's wrong:** `is_own` is a **read-only** derived flag — Peec sets it server-side based on project ownership. The parameter isn't in the accepted schema; passing it is silently ignored.

**Correction:** Mis-flagged own-brand has to be corrected in the Peec UI (project settings), not via MCP. File a ticket with Peec if the ONINO project has this issue.

### 5.8 Chaining `get_actions` drills beyond three slices

**Anti-pattern:** `get_actions(scope=overview)` → drill into owned → drill into editorial → drill into reference → drill into ugc → drill into owned again with a specific classification → ...

**Why it's wrong:** `get_actions` is most efficient when used as an index with disciplined drill. Drilling into 5+ slices in one session defeats the efficiency win — you're back to the ~15-call pattern, just routed through a different tool. The overview already ranked opportunities; drilling into every ranked entry is re-hunting, not executing.

**Correction:** Overview → top 3 slices by score → act on one. If a session needs more than 3 drills, step back — you're probably not triaging, you're browsing. See `workflows/suggestion-review.md` §3.

### 5.9 Using `list_shopping_queries` on B2B prompts

**Anti-pattern:** Including `list_shopping_queries` in the monthly fanout pass for ONINO's B2B prompts.

**Why it's wrong:** Shopping-query fanouts surface product/retail intent. ONINO's ICP doesn't buy through AI shopping surfaces — banks don't add ONINO to a cart. The output is near-universally empty or noise for B2B prompts, and the 10000-row cap is misleading (don't actually request 10000).

**Correction:** Skip `list_shopping_queries` for ONINO's standard prompt set. Only include if a specific prompt is framed with consumer-buying intent (which should itself be flagged as a Leak — see `workflows/fanout-discovery.md` §3).
