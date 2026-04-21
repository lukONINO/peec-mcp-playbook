# Workflow — Content-Gap Hunt

Target call budget: **2-5 MCP calls.**

**Schema-verified 2026-04-18.** Uses flat `start_date`/`end_date`, no `order_by` (sort client-side), `classification` is a returned column (not a filter field — filter client-side after the response). Native `gap` operator does the "competitors cited, ONINO not cited" expression in one filter entry.

## When to use

- Monthly content-planning session.
- SEO team asks "where should we be writing?"
- An ICP segment is under-performing in weekly reports and you need to find the specific URLs driving the gap.

## Inputs

- Scope: filter by `funnel_stage` (default MOFU+BOFU) and/or `icp_segment`
- Date range: default past 28 days

## Steps

### Step 1 — Gap scan over URLs — 1 call

Use recipe R3, filtered to the scope:

```json
get_url_report({
  project_id: "<onino_project>",
  start_date: "<today - 28d>",
  end_date: "<today>",
  dimensions: ["tag_id"],
  filters: [
    { field: "gap", operator: "gt", value: 0 },
    { field: "tag_id", operator: "in", values: ["funnel_stage:MOFU", "funnel_stage:BOFU"] }
  ],
  limit: 100
})
```

Each row includes a returned `classification` column (values: `LISTICLE`, `COMPARISON`, `CATEGORY_PAGE`, `PRODUCT_PAGE`, `HOMEPAGE`, `PROFILE`, `ALTERNATIVE`, `DISCUSSION`, `HOW_TO_GUIDE`, `ARTICLE`, `OTHER`). **Filter client-side** to `["LISTICLE", "COMPARISON", "CATEGORY_PAGE", "HOW_TO_GUIDE"]` and sort by `citation_count` desc.

### Step 2 — Gap scan over domains — 1 call

Recipe R7-adjacent:

```json
get_domain_report({
  project_id: "<onino_project>",
  start_date: "<today - 28d>",
  end_date: "<today>",
  dimensions: ["domain"],
  filters: [
    { field: "gap", operator: "gt", value: 0 }
  ],
  limit: 30
})
```

Each row has a `classification` column (values: `CORPORATE`, `EDITORIAL`, `INSTITUTIONAL`, `UGC`, `REFERENCE`, `COMPETITOR`, `OWN`, `OTHER`). **Filter client-side** to `["EDITORIAL", "REFERENCE"]` — editorial and reference domains with high gap scores are outreach targets. Sort by `citation_count` desc client-side.

### Step 3 — Read top 3 gap URLs — 3 calls

```json
get_url_content({ project_id: "<onino_project>", url: "<exact url from Step 1>" })
```

Run for each of the top 3 URLs.

For each: identify the specific angle the article takes, which competitors it recommends, and what ONINO-specific strength is missing from the article.

## Prioritization rules

Rank the gap URLs by commercial value:

1. **LISTICLE + MOFU** = top priority (buyers shortlist from these).
2. **COMPARISON + BOFU** = second priority (buyers decide from these).
3. **CATEGORY_PAGE + MOFU** = third (harder to get added but worth the outreach).
4. **HOW_TO_GUIDE** = fourth (lower commercial intent).
5. Skip TOFU gaps unless they're in a deeply authoritative publication (asset manager trade press, etc.).

## Output

Two artifacts:

**1. URL action list** — CSV or Notion table with:
```
url | domain | classification | citation_count | competitors_mentioned | recommended_action
```

Where `recommended_action` is one of:
- "Pitch editor for inclusion" (EDITORIAL domains)
- "Create comparable internal content" (if it's a listicle we can replicate on our own domain)
- "Add ONINO to X free listing service" (if it's a directory-style CATEGORY_PAGE)
- "Outreach via LinkedIn" (if it's a profile-style article by an individual author)

**2. One-page brief to Alex** — the top 5 prioritized gaps, with the one-line action for each.

## Budget audit

| Step | Calls |
|---|---|
| 1. URL gap | 1 |
| 2. Domain gap | 1 |
| 3. Read top 3 | 3 |
| **Total** | **5** |
