# modules/01-inventory.md — Inventory & Page-Type

**Status:** Spec doc 11. The first analysis module to run. Feeds essentially every downstream module (05, 06, 08, 09, 10, 11, 12, 13) plus 04.
**Role in pipeline:** start of Phase 2 (Analyze). Per `pipeline.md` §7 row 01.
**Nature:** Deterministic inventory + classification, with an interpretive page-type assignment that combines content role with SERP intent (§5).

---

## 0. Mandate

**Page type is the single most-read input in the system.** 06 selects its rubric by it, 08 routes coverage by it, 09 weights EEAT by it, 10 gates the conversion audit on it, 11 assigns hubs/spokes by it. If page type is wrong, every downstream module is wrong in the same direction. So this module gets it right by combining **two inputs, not one** (`pipeline.md` Phase 2): what the page *is* (its content/role) **and** what its keyword's SERP *wants* (module 04's intent verdict). When those disagree, the mismatch is a top-level finding (§5). [LOAD-BEARING]

To avoid a circular dependency (04 needs page-type labels for SERP results; full page-type assignment needs 04's verdict), the classifier has **two paths** run at different times:
- a **coarse classifier** (content-only, no intent) that 04 and 05 call on SERP-result URLs, run first;
- a **full assignment** (coarse role × 04 intent) for the operator's *own* pages, run after 04.

Execution order: coarse classifier available → 04 runs (intent/format using coarse labels) → 01 full assignment runs (using 04's verdict) → everything else.

---

## 1. Contract

**Inputs:**
- Phase 1 ingest crawl + URL inventory (existing), **or** the planned page set from 11 + seed brief (greenfield).
- Module 03 GSC opportunity data (for prioritization, §6).
- Module 04 intent verdict per page's target keyword (for full assignment, §5) — consumed *after* 04 runs.

**Outputs:**
- `analysis/inventory.md` (+ machine sibling) — the page list with assigned type, target-keyword mapping, mismatch flags, and **priority rank**.
- The **coarse classifier** exposed as a callable service (used by 04 and 05's homogeneity gate).

---

## 2. Inventory
- **Existing:** reuse the Phase 1 crawl — full URL set, with each page's extracted content, metadata, and structural signals already captured. No re-crawl.
- **Greenfield:** there are no live URLs; the "inventory" is the planned page set from 11's cluster architecture + the seed brief. Page types are assigned by design intent rather than detected (§7).

Attach the target-keyword mapping (from GSC/config existing; from seed brief greenfield) — a page is classified partly by what it's trying to rank for.

---

## 3. Page-type taxonomy

AI-assigned from analysis (per the operator's direction that the system determines type rather than being told it), confirmable by the operator. Default taxonomy:
- **Money / service** — sells/converts a service (BOFU).
- **Local landing** — service + location (BOFU, local intent).
- **Product / transactional** — sells a product.
- **Informational / cluster** — educational, supporting content (TOFU-MOFU).
- **Comparison / listicle** — "best X", "X vs Y".
- **Hub / pillar** — category page anchoring a cluster.
- **Homepage**.
- **Utility** — contact, about, policy (not ranking targets).

The taxonomy is extensible; add a type here with its signal pattern (§4), never let the model invent one ad hoc at runtime.

---

## 4. The coarse classifier [LOAD-BEARING]

Content-only, fast, **no intent input** — so it can run on any URL standalone (this is the service 04's format analysis and 05's homogeneity gate call on SERP results). Classifies from deterministic signals:
- **URL pattern** (`/services/`, `/blog/`, `/products/`, `/locations/`).
- **Schema** (LocalBusiness/Service vs Article vs Product).
- **Structural signals** — presence of price, booking/contact form, CTA density, product markup vs long-form article structure, comparison tables.
- **Content shape** — depth, heading patterns, question density.

Outputs a coarse type label + confidence. Where signals are ambiguous, the model adjudicates, but the common cases resolve on signals alone. This path never waits on 04 — that's what keeps the execution order acyclic.

---

## 5. Full page-type assignment [LOAD-BEARING]

For the operator's own pages, combine:
- the **coarse content role** (§4 — what the page is), and
- module 04's **SERP-intent verdict** for the page's target keyword (what Google rewards for that term).

Resolution:
- **Agree** → assign that type with high confidence.
- **Disagree (mismatch)** → assign the content role as the *current* type, but flag a **page-type × intent mismatch** as a top finding: the page is built as type X, but its target keyword's SERP wants type Y. This is frequently the single most valuable finding for a money page that won't rank, and it sets up 06's rank-verdict precondition and 13's "don't optimize what shouldn't rank" handling.

The mismatch is not silently reconciled — it's surfaced for the operator and routed (re-target the page, or build the page type the SERP wants).

---

## 6. Prioritization [LOAD-BEARING]

Not all pages get equal analytic depth (`pipeline.md` §6.2). Rank the work queue by GSC opportunity:
- **Striking-distance pages** (position 5–15 with real impressions) first — the highest-ROI targets, small moves for large gains.
- High-impression / low-CTR pages next.
- Money pages weighted up within tiers.
- Pages with no impressions and no strategic role (utility) deprioritized.

This priority rank drives 05's run set (its Stage 1) and 13's sequencing. Greenfield prioritizes by money-keyword value instead (no GSC history).

---

## 7. Existing vs greenfield
- **Existing** — detect types from live pages (§4–5), flag mismatches, prioritize by GSC.
- **Greenfield** — assign types by design from 11's architecture + seed brief; "mismatch" can't occur (no built page yet), but 04's intent verdict still dictates *what type each planned page must be* to win its keyword. Prioritize by keyword value.

---

## 8. Interpretation layer / honesty
The model adjudicates ambiguous coarse cases (§4) and performs the role × intent reconciliation (§5). It does not fabricate the deterministic signals (URL/schema/structure) or the GSC priority data. State that page-type assignment on ambiguous pages carries lower confidence, and that this confidence propagates (a low-confidence type assignment makes every downstream module's read of that page correspondingly tentative).

## 9. Output artifact
`analysis/inventory.md`: per page — URL, coarse type, assigned type, target keywords, **mismatch flag + verdict**, priority rank, assignment confidence. Header notes the taxonomy version and the existing/greenfield mode.

## 10. Interfaces
**Consumes:** Phase 1 crawl (existing) / 11 + seed brief (greenfield), 03 (GSC priority), 04 (intent verdict, for §5).
**Exposes:** the coarse classifier service to **04** and **05**; page type + priority + mismatch flags to **06, 08, 09, 10, 11, 12, 13**.

## 11. Deferred
- `modules/04-serp.md` — supplies the intent verdict §5 consumes (drafted next; together 01 + 04 close the load-bearing set).
- `modules/03-gsc.md` — supplies the prioritization data §6 consumes.
