# modules/13-roadmap.md — Roadmap (synthesis & prioritization)

**Status:** Spec doc 10, the capstone. Consumes **every** analysis module (02–12). Produces the single plan the operator acts on.
**Role in pipeline:** end of Phase 2 (Analyze); together with the 06 briefs it constitutes what **Gate A** approves. Re-runs each Monitor cycle (Phase 5). Per `pipeline.md` §7 row 13.
**Nature:** Deterministic aggregation + scoring; interpretive sequencing and conflict resolution. The numbers are computed; the *order and coherence* are judged.

---

## 0. Mandate

The system's failure mode without this module is **twelve disconnected reports the operator has to reconcile.** 13 exists to prevent that: one plan, every item traceable to its source module, ranked by ROI, ordered by dependency, weighted toward the money pages. [LOAD-BEARING]

Three principles govern it:
1. **One common currency.** Every finding from every module is scored on the *same* scale so a redirect fix, a missing cluster page, a title tweak, and a CRO change can be ranked against each other. The scale is **ICE** (already native to the system via the conversion skill), with **Impact grounded in GSC opportunity data**, not vibes (§2).
2. **Sequencing is a graph, not a sorted list.** [LOAD-BEARING] ROI rank alone produces an unbuildable plan. Real dependencies constrain order (§3).
3. **Confidence propagates to the end.** A finding built on a low-confidence correlation (`05` stamp) or a low-reliability EEAT proxy cannot rank as if it were certain. The stamps that started at the SERP pull reach all the way into the roadmap's priority.

---

## 1. Contract

**Inputs (all module outputs):**
- 02 technical, 03 GSC opportunity, 04 intent + rank verdicts, 05 correlation (+ confidence stamps), 06 briefs, 07 local, 08 coverage, 09 EEAT, 10 conversion, 11 authority + cannibalization, 12 migration risk.
- GSC traffic/position data + keyword volume (for Impact grounding).
- Page types + money-page designation (01).

**Output:** `reports/roadmap.md` (+ machine sibling) — the ROI-ranked, dependency-ordered, phased plan, every item carrying source module, rationale, ICE, and confidence.

---

## 2. Common-currency scoring [LOAD-BEARING]

Every finding becomes a roadmap item scored with ICE (1–10 each, averaged):
- **Impact — grounded in GSC, not guessed.** Estimated traffic/conversion value if the fix lands: derived from the page's current GSC impressions/position (striking-distance pages at position 5–15 with real impressions = high impact), keyword volume, **and money-page weighting** (a money-page conversion fix outweighs an equal-effort informational tweak). Impact is a *modeled estimate*, stated as such (§8).
- **Confidence — inherits the stamps.** Built from the source finding's reliability: 05's correlation confidence, 08's corpus-coverage strength, 09's signal reliability, 10's evidence grounding. A `low`-confidence input caps the item's Confidence score. This is where the data-quality thread that began at the SERP fallback chain terminates.
- **Ease — implementation effort.** A meta-tag edit is high ease; building a 12-page cluster is low ease.

ICE-rank descending gives the within-constraint ordering. But constraints come first (§3).

---

## 3. Dependency sequencing [LOAD-BEARING]

The plan is a DAG. Known ordering constraints the roadmap must respect regardless of ICE:
- **Cluster spoke before offload.** 06 can't offload coverage to a cluster page (and 12 won't pass it as "preserved") until that spoke exists (11). Build the spoke first.
- **Resolve cannibalization before optimizing the survivor.** Optimizing a page that's competing with itself wastes effort (11 → then on-page work).
- **Redirects before URL changes.** 12's redirect map must be complete before any URL-changing work ships (Gate C).
- **Technical foundation before content polish.** A page that's noindexed, slow, or broken (02) gains nothing from term tuning until the foundation is fixed.
- **Rank-verdict gate first.** If 04 says a page shouldn't chase its term, the item is "re-target / route to correct page," not "optimize" — and that re-targeting precedes any on-page work.

The model resolves non-structural ordering; the structural constraints above are hard.

---

## 4. Cross-module conflict resolution

Modules can recommend opposing actions (coverage wants depth; conversion wants focus). The per-page hierarchy was already set (06 §3: conversion wins on money pages, coverage offloads to cluster; 10 §7: Distraction validates the offload). 13's job is to ensure the final plan reflects the **resolved** trade as a single coherent item — never two contradictory items the operator has to adjudicate. Where a genuine unresolved tension remains, 13 surfaces it explicitly as a decision for the operator, with the trade-off stated, rather than silently picking.

---

## 5. Phasing

Group the ranked, sequenced items into bands the operator can execute:
- **Quick wins** — high ICE, high ease, no blocking dependencies (cheap on-page fixes, schema additions, obvious cannibalization merges).
- **Foundational** — must-precede work (technical fixes, redirect map, core cluster spokes, equity routing) even where ease is low, because downstream items depend on it.
- **Bigger bets** — high impact, low ease, longer horizon (full cluster build-out, major content/CRO overhauls).
Money-page items float up within each band.

---

## 6. Relationship to Gate A and the target-spec

The 06 briefs are the **per-page** target spec (what each page becomes). This roadmap is the **site-level sequenced plan** (in what order, weighted by ROI, respecting dependencies). Together they are what the operator approves at **Gate A** before any rebuild. Approving the briefs without the roadmap would leave the build unordered; approving the roadmap without the briefs would leave it unspecified. Both, together, gate Rebuild.

---

## 7. Living roadmap

In Monitor (Phase 5), the roadmap re-runs: re-pull, re-analyze, **diff against the prior roadmap and the migration baseline** (12 §7). Completed items drop off; new opportunities (a page that moved into striking distance, a new cannibalization, a ranking drop) appear. The roadmap is the system's ongoing output, not a one-time artifact — which is what makes the whole pipeline a monitoring loop rather than a single audit.

---

## 8. Honesty (state in the roadmap)
- **Impact is modeled, not promised.** GSC-grounded estimates project potential; they are not guarantees a fix will rank or convert. Frame ranges, not certainties.
- **Confidence-damped items say so.** An item capped by low input confidence is labeled — "worth testing, not yet certain" — so the operator doesn't read every line as equally sure.
- **Correlation-not-causation carries through.** Items derived from 05 drivers inherit that caveat; the roadmap doesn't launder a correlation into a promise by the time it reaches the top of the list.

---

## 9. Output artifact format

`reports/roadmap.md`:
1. **Executive summary** — the state of the site, the single highest-leverage move, the headline risks (esp. migration, 12).
2. **Phased plan** — Quick wins / Foundational / Bigger bets, each item: action, target page(s), source module, ICE, confidence, dependencies, rationale.
3. **Sequenced build order** — the DAG flattened into an executable order respecting §3 constraints.
4. **Open decisions** — unresolved cross-module tensions (§4) for the operator.
5. **Money-page focus** — a dedicated cut of the plan for the revenue pages.
6. **Monitoring deltas** (on re-runs) — what changed since last roadmap.

## 10. Interfaces
**Consumes:** every analysis module (02–12), GSC + volume, page types.
**Exposes:** the roadmap to the operator and to **Gate A** (with the 06 briefs); the sequenced build order to **Phase 3 (Rebuild)**; the living deltas to **Phase 5 (Monitor)**.

---

## 11. Series status

This completes the analysis-and-synthesis core: `pipeline.md`, `factors.md`, and modules 05, 06, 08, 09, 10, 11, 12, 13. **Still unwritten** and required before implementation:
- **Input/data modules 01 (inventory + page-type), 02 (technical), 03 (GSC), 04 (SERP/intent), 07 (local)** — referenced throughout as interfaces but not yet specced. 01 and 04 are the most load-bearing of these (page type and the rank verdict feed nearly every module).
- **`methodology/lib/`** — the deterministic measurement scripts behind `factors.md` §4 (code, not spec).
- **`rubrics/`** — the detailed scoring sheets behind 06 §2/§4, 09 §4, 10.
- **`SKILL.md`** — the orchestrator wiring the phases, branches, and gates (its contract is `pipeline.md`; it needs writing as the actual entry point).
