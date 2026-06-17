# modules/11-authority.md — Topical Authority & Internal Linking

**Status:** Spec doc 9. Consumes module 08 (cross-page subtopic map), 03 (GSC cannibalization), 05 (backlink concentration), and the Phase 1 internal-link graph. Feeds 06, 12, and 13.
**Role in pipeline:** Phase 2 (Analyze), the **site-scope** module (08/09 are page-scope; this is the structure that connects them). Per `pipeline.md` §7 row 11.
**Nature:** Deterministic graph/crawl analysis + interpretive architecture design. Scripts map the link graph, click depth, orphans, anchors, and cannibalization; the model designs the cluster and the link plan.

---

## 0. Mandate

**Topical authority is a site-level property, not a page-level one.** A single page can't own a topic; the *cluster collectively* does. This module is where the operator's "be a topic authority for the keywords we need" goal becomes a concrete structure: a **hub-and-spoke topic-cluster architecture** with money/pillar pages as hubs, supporting content as spokes, and deliberate internal links binding them. [LOAD-BEARING]

This is also the structural resolution to the coverage-vs-conversion tension that runs through the whole system: **hubs stay lean and convert; spokes carry the comprehensive depth; spokes link up to hubs.** Modules 08 and 06 decide what offloads where per page — 11 designs the cluster they offload *into* and the links that make it cohere.

---

## 1. Contract

**Inputs:**
- Phase 1 **internal-link graph** + URL inventory (existing) — the current structure.
- Module 08 — the **cross-page subtopic map** (what subtopics the topic demands, and which exist on-site): the demand signal for spokes.
- Module 03 (GSC) — query→URL data for **cannibalization** detection.
- Module 05 — **backlink concentration** per page (where external authority actually lands).
- Target keyword → page mapping + page types (01).

**Outputs:**
- `analysis/authority/cluster-map.*` — the hub-and-spoke architecture (existing audited, or greenfield designed).
- `analysis/authority/link-plan.*` — per-page internal-link obligations (which page links to which, with anchor guidance).
- `analysis/authority/cannibalization.*` — competing-page findings + resolutions.
- `analysis/authority/orphans.*` — pages with no internal inlinks.

---

## 2. Topic-cluster architecture [LOAD-BEARING]

- **Identify hubs** — the money/pillar pages that should own each target topic (from the keyword→page mapping + page types).
- **Map spokes** — supporting/cluster content that covers the breadth around each hub.
- **Find the gap** — overlay module 08's subtopic demand on the existing content. Subtopics with strong corpus-coverage % (08 §2.5) that have **no spoke page** are the missing cluster content the site needs to build to establish authority. This is where "cover everything Google expects" becomes a list of pages to create, not just sections to add.
- **Bind the cluster** — every spoke links up to its hub; the hub links down to its key spokes. A cluster without internal links is just unconnected pages, not an authority structure.

In **greenfield**, this section *is* the information architecture — there's no existing structure to audit, so the module designs the full hub-and-spoke sitemap from the keyword set + 08 demand.

---

## 3. Cannibalization detection & resolution [LOAD-BEARING]

When multiple pages target the same query, they split signals and compete with each other — diluting the authority the operator is trying to concentrate.
- **Detect** from GSC: a single query whose impressions/clicks are spread across multiple URLs, or multiple pages ranking in the same SERP for the same term.
- **Resolve** (model judgment): either **consolidate** (merge into one stronger page — which hands a consolidation redirect to module 12) or **differentiate** (re-target the pages to distinct intents/keywords so they stop competing).
- This is a high-value site-level finding: fixing cannibalization often lifts the surviving page without any new content.

---

## 4. Internal-link structure

**Deterministic crawl (facts):**
- **Click depth** — clicks from homepage to each page. Money pages buried deep lose equity and discoverability.
- **Orphans** — pages with zero internal inlinks (lost to crawlers and equity flow).
- **Anchor inventory** — existing internal anchor text per target, to spot over-optimization (spammy exact-match repetition) or uselessness ("click here").
- **Equity flow** — how link weight moves through the graph toward (or away from) money pages.

**The link plan (model):**
- **Contextual spoke→hub links** — relevant cluster content links to the money page it supports, passing both relevance and equity.
- **Reduce money-page click depth** — surface hubs in primary nav / shallow structure.
- **Anchor guidance** — descriptive, varied, relevant; never exact-match-stuffed. Recommend specific anchors per link.
- **Fix orphans** — assign each a logical inlink.

---

## 5. Link-equity routing

Combine module 05's backlink concentration with the internal graph: **does external authority actually reach the money pages?** External links often land on blog posts or the homepage, not money pages. The plan routes that equity inward — internal links from the high-authority pages (where backlinks concentrate) to the money pages that need to rank. This is one of the cheapest ranking levers available and is invisible without joining the two data sources.

---

## 6. Existing vs greenfield
- **Existing** — audit the current cluster + links, produce the gap (missing spokes, cannibalization, orphans, equity misrouting) and the remediation plan.
- **Greenfield** — design the cluster architecture and internal-link structure from scratch; the cluster-map *is* the sitemap the rebuild builds to.

---

## 7. Interpretation layer (the model)
Bounded to: designing the cluster architecture (hub/spoke assignment, missing-spoke list from 08 demand), resolving cannibalization (consolidate vs differentiate), writing the internal-link plan (which links, which anchors), and routing equity. Each non-obvious call carries a rationale. The model never fabricates the graph metrics (click depth, orphans, anchor inventory, cannibalization data) — those are §3–4 deterministic outputs.

## 8. Output artifacts
Cluster map, link plan (per-page obligations), cannibalization report (with consolidate/differentiate verdicts), orphan list. All under `analysis/authority/`.

## 9. Interfaces
**Consumes:** Phase 1 link graph, 08 (subtopic demand), 03 (cannibalization data), 05 (backlink concentration), 01 (page types), keyword mapping.
**Exposes:** per-page internal-link obligations to **06** (folded into the brief's internal-link section); the intended cluster + link structure to **12** (the structure §5 there preserves against; consolidation verdicts become 12 redirects); the authority gap + cannibalization fixes to **13** (roadmap).

## 10. Deferred
- `modules/13-roadmap.md` — the capstone synthesis that ROI-ranks the missing-spoke builds, cannibalization fixes, and link-plan work against everything else.
