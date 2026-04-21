# ICP-Aligned Prompt Library

This is the canonical prompt taxonomy for the ONINO Peec project. Any prompt that doesn't fit one of the five ICP segments × three funnel stages is out-of-scope — see `anti-patterns.md`.

**Updated 2026-04-18:** `region_target:*` tag family deprecated in favor of native `country_code` on `create_prompt`. The required tag overlay is now 4 tag families + native `country_code` (was 5 tag families). See §Tag structure below.

## The 5 ICP segments (ordered by 2026 marketing priority)

| Code | Segment | Buyer role | Decision authority | Deal-size signal |
|---|---|---|---|---|
| `asset-manager` | **Asset Managers** | Head of Private Markets / Product / Digitalization | High | 7-9-figure assets under management, repeat issuance |
| `bank` | **Banks** | Head of Digital Securities / Innovation / Private Banking | High (but long cycle) | Regulatory modernization budget |
| `platform-operator` | **Investment Platforms / Private Market Operators** | CEO / Head of Product | Highest (founder-led) | Recurring offerings, fund-style businesses |
| `club-coop` | **Investment Clubs & Cooperatives** | Managing Director / Vorstand | Medium-High | Many members, smaller tickets each |
| `financing-consultant` | **Financing Consultants (Real Estate, Renewables)** | Managing Partner / Head of Advisory | Medium | Recurring deal pipeline, often pitched on <24h setup |

**Note:** The 2026 plan targets ≥60% of marketing focus on ONE primary ICP (per `onino-gtm-knowledgebase`). That primary ICP is still being finalized — for now, the playbook treats all 5 as in-scope but weights financing consultants and platform operators slightly higher (they're where ONINO's sales machine has the most proof points).

## The 3 funnel stages

| Stage | Buyer state | Prompt intent | Visibility value |
|---|---|---|---|
| **TOFU** | Problem-aware, not vendor-aware | Educational, category-defining | Authority / thought-leadership impression |
| **MOFU** | Solution-aware, evaluating categories | Comparison, "best X for Y" | **Pipeline-decisive** — most commercial prompts live here |
| **BOFU** | Vendor-aware, evaluating ONINO | Brand + product + proof-point | Conversion / shortlist inclusion |

> **MOFU is where the money is.** A 10-point visibility lift on one MOFU prompt is worth more than a 30-point lift on five TOFU prompts.

---

## The 50 current prompts — mapped to segments × stages

Source: `/Users/lukaswipf/Documents/Claude/Projects/ONINO GTM/Peec_AI_Tracking_Prompts.md` (as of 2026-04-17)

Every prompt below should carry these Peec tags: `funnel_stage`, `icp_segment` (1+), `language`, `branded` (branded/non-branded) — plus native `country_code` on the prompt itself (required field on `create_prompt`, ISO-3166 values like `DE`/`AT`/`CH`/`FR`). The old `region_target:*` tag family is deprecated and no longer required on new prompts.

### TOFU (15 prompts — problem-aware education)

| # | Prompt | `icp_segment` | `language` | Notes |
|---|---|---|---|---|
| 1 | What is a white-label financing platform and how does it work? | platform-operator, asset-manager | en | Pure category-definition — ONINO should surface as a live example |
| 2 | Was ist eine Whitelabel-Emissionsplattform? | platform-operator, asset-manager | de | DACH equivalent of #1 |
| 3 | How does digital securities issuance work in the EU? | bank, asset-manager, platform-operator | en | Regulatory framing — BaFin/ECSPR/MiCA natural surface |
| 4 | Wie funktioniert die Tokenisierung von Vermögensanlagen in Deutschland? | asset-manager, financing-consultant | de | Hygiene keyword; do NOT lead with tokenization in our own content |
| 5 | How can companies raise capital from many retail investors in Europe? | platform-operator, club-coop | en | "Many investors" is the ONINO sweet spot |
| 6 | Wie emittiert man Nachrangdarlehen an eine große Investorenbasis? | financing-consultant, club-coop | de | Highly specific to ONINO's instrument stack |
| 7 | What is MiCA regulation and how does it affect digital asset issuance? | bank, asset-manager | en | |
| 8 | Wie digitalisiert man die Investorenverwaltung bei Immobilieninvestments? | financing-consultant (real estate) | de | Direct ICP fit |
| 9 | What is an SPV platform and why do asset managers use one? | asset-manager, platform-operator | en | SPV = first solution pillar |
| 10 | Wie strukturiert man eine Kapitalanlage mit Genussrechten? | financing-consultant, club-coop | de | Genussrechte = ONINO-supported instrument |
| 11 | How does ECSPR crowdfunding regulation work in practice? | platform-operator, club-coop | en | Regulatory TOFU; not a competitive signal |
| 12 | Was ist der Unterschied zwischen Crowdfunding und einer eigenen Emissionsplattform? | platform-operator, financing-consultant | de | Positioning prompt — ONINO's differentiator |
| 13 | How to automate KYC and AML for private market investor onboarding? | all-B2B | en | Compliance pain-point |
| 14 | Wie können Banken ihre Wertpapieremission digitalisieren? | bank | de | Narrow to bank segment |
| 15 | What are the benefits of tokenizing real estate and renewable energy assets? | financing-consultant, asset-manager | en | Sub-segment: RE + renewables |

### MOFU (15 prompts — comparison and evaluation — the commercial core)

| # | Prompt | `icp_segment` | `language` | Commercial value |
|---|---|---|---|---|
| 1 | Best white-label tokenization platforms in Europe in 2026 | platform-operator, asset-manager | en | **Top-5 priority** |
| 2 | Top platforms for private market financing in Germany | platform-operator, financing-consultant | en | **Top-5 priority** |
| 3 | Vergleich: Anbieter für Tokenisierungsplattformen im DACH-Raum | platform-operator | de | **Top-5 priority (DACH)** |
| 4 | Which platforms support BaFin-compliant digital securities issuance? | bank, platform-operator | en | Regulatory-first segment |
| 5 | Best platforms for real estate developers to manage investor capital raises | financing-consultant (RE) | en | **Top-5 priority** |
| 6 | Welche Whitelabel-Investmentplattformen gibt es für Asset Manager? | asset-manager | de | Segment-pure |
| 7 | Platforms for issuing subordinated loans (Nachrangdarlehen) digitally | financing-consultant, club-coop | en | Instrument-specific |
| 8 | Which software helps asset managers scale investor onboarding to thousands of investors? | asset-manager | en | **Top-5 priority** — "many investors" narrative |
| 9 | Welche Plattformen eignen sich für die Emission von Genussrechten? | financing-consultant, club-coop | de | |
| 10 | Best tokenization providers for renewable energy project financing | financing-consultant (renewables) | en | **Top-5 priority** — a named sweet spot |
| 11 | Which platforms offer end-to-end SPV management for private equity firms? | asset-manager, platform-operator | en | **Top-5 priority** |
| 12 | Plattformen für digitale Mitgliederbeteiligungen bei Genossenschaften | club-coop | de | Segment-pure DE |
| 13 | White-label vs custom-built financing platform: which should I choose? | platform-operator | en | Positioning (ONINO wins this framing) |
| 14 | Top EU platforms for issuing tokenized bonds and equity tokens | bank, asset-manager | en | |
| 15 | Welche Anbieter bieten Secondary Market Funktionalität für tokenisierte Wertpapiere? | asset-manager, platform-operator | de | Secondary market is a differentiator |

### BOFU (20 prompts — vendor-specific, proof-point)

All BOFU prompts carry `branded:branded` unless explicitly asking about a competitor.

**Brand & company** (4)
1. What is ONINO and what does the company do?
2. Was macht ONINO GmbH aus Karlsruhe?
3. ONINO review: is it a reliable white-label financing platform?
4. ONINO customer reviews and case studies

**Product & features** (6)
5. ONINO white-label platform features and capabilities
6. How long does it take to launch a white-label platform with ONINO?
7. What asset classes does ONINO support?
8. Does ONINO support secondary market trading for tokenized assets?
9. Which financial instruments (Nachrangdarlehen, Genussrechte, bonds, equity tokens) does ONINO support?
10. Which countries and jurisdictions does ONINO support?

**Pricing & plans** (2)
11. ONINO pricing plans compared (Eigenemission vs Whitelabel vs ONINO Listing)
12. Welcher ONINO-Tarif passt für einen Immobilien-Emittenten?

**Compliance & regulation** (2)
13. Is ONINO BaFin compliant and how does regulation work on the platform?
14. Does ONINO support MiCA and ECSPR compliant issuances?

**Use case / ICP fit** (4)
15. Can ONINO be used for renewable energy project financing?
16. ONINO for banks: how does it help digitize securities operations?
17. Is ONINO suitable for asset managers managing private market products?
18. Eignet sich ONINO für Genossenschaften und Mitgliederbeteiligungen?

**Comparison vs competitors** (2)
19. ONINO vs Finexity vs Tokeny: which is best for tokenized real estate financing? `branded:branded` + `competitor_comparison:yes`
20. ONINO vs Stokr vs Brickken: comparison for white-label SPV platforms `branded:branded` + `competitor_comparison:yes`

---

## Competitor brands to track alongside ONINO

The competitor set should be ICP-aligned — i.e. **infrastructure/platform competitors**, not crowdinvesting platforms.

**Direct infrastructure competitors** (should always be in `list_brands`):
- Tokeny (Luxembourg)
- Cashlink (Germany)
- Stokr (Luxembourg)
- Brickken (Spain)
- Bitbond (Germany — debt-focused)
- Finexity (hybrid: infra + marketplace)

**Adjacent platforms to monitor** (may appear in some MOFU prompts):
- Kapilendo / Invesdor (more crowdinvesting, adjacent)

**Out-of-scope — do NOT add to tracked brand set:**
- Companisto, Seedrs, Exporo, Bergfürst, Econeers — these are crowdinvesting platforms; they are ONINO *customers* in theory, not competitors.

---

## Tag structure to maintain in Peec

Each of the 50 prompts carries a 4-tag overlay plus a native `country_code` on the prompt itself (required field on `create_prompt`):

```
# Required (4 tag families + native country_code)
funnel_stage:     TOFU | MOFU | BOFU
icp_segment:      asset-manager | bank | platform-operator | club-coop | financing-consultant   (one or more)
language:         de | en
branded:          branded | non-branded                                     (string values, NOT yes/no)
country_code:     DE | AT | CH | FR | NL | ES | IT | ... (ISO-3166 — native prompt field, not a tag)

# Optional (kept for drill-down reporting)
competitor_comparison:  yes | no   (only on comparison prompts)
instrument:       nachrangdarlehen | genussrechte | bonds | equity-tokens | spv | (empty)
asset_class:      real-estate | renewables | pe | member-shares | (empty)

# Deprecated as of 2026-04-18
region_target:    DACH | EU | Global   ← REDUNDANT with native country_code. Do not create new ones.
                                          Existing region_target:* tags are harmless; clean up in
                                          quarterly consolidation (workflows/prompt-set-consolidation.md).
```

**Why the change:** `create_prompt` (shipped 2026-04-18) has `country_code` as a required native field with the full 95+ ISO-3166 enum. There is no `EU` meta-code — prompts are strictly per-country. The old `region_target:*` tag family duplicated country info in a weaker way and is now redundant.

**Implications:**
- "DACH" as an analytical concept = filter on `country_code in ["DE", "AT", "CH"]` (see recipe R8 in `query-recipes.md`).
- "EU" as an analytical concept = filter on `country_code in ["DE", "AT", "CH", "FR", "NL", "ES", "IT", ...]` — maintain an explicit list in your recipe or skill.
- "Global" as an analytical concept = don't filter on `country_code` at all (or explicitly include `US`, `GB`, etc.).

**`branded` value correction:** the Peec-native convention is string values (`branded` / `non-branded`), not boolean `yes` / `no`. This was mis-labeled in earlier versions of the playbook; fix on any existing prompts during next consolidation pass.

**Maintenance rule:** if a new prompt is added that can't be mapped cleanly onto this schema, the prompt is probably misaligned to the ICP — reject or rework before tagging.

---

## When the prompt set needs to be rebuilt

A prompt-set rebuild is warranted if any of the following is true:

- More than 10% of prompts have retail-investor or crowdinvesting intent.
- An ICP segment has <3 prompts across MOFU/BOFU combined.
- A funnel stage has <10 prompts total.
- `list_topics` shows clusters that don't map to ONINO's category.
- Visibility on the 5 "top priority" MOFU prompts has been flat or declining for 4+ weeks.

Rebuild workflow: `workflows/prompt-set-rebuild.md` (follow-up doc — add when a rebuild is triggered).
