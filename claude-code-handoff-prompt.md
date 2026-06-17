# Hand-off: implement the SEO-rebuild pipeline from its spec set

**Run this in planning mode. Do not write code, scaffold files, or call any API in this pass.** Your only job here is to ingest a complete spec set, understand it as a whole, and produce a planning document. Building comes after that document is approved.

---

## What this is

A specification for an **agnostic WordPress→Astro/Tailwind rebuild + full SEO pipeline**: point it at a domain, it rebuilds the site (or builds one from a bare domain) and runs a complete SEO program — technical, on-page, competitor correlation, coverage, EEAT, conversion, authority, migration — deploying to Vercel.

The architecture: a **static methodology bundle** (reusable, ships once) plus a **generated per-site workspace** (created when aimed at a domain). It branches on two state machines — `input_mode` (existing | greenfield) and `deploy_target` (local | preview | production) — runs five phases (Ingest → Analyze → Rebuild → Verify → Deploy/Monitor), and stops at four gates (A: target-spec approval, B: migration parity, C: production/URL changes, D: design-system present).

The spec set (thirteen documents):
- `pipeline.md` — the spine: state machines, phases, gates, inter-phase contracts.
- `factors.md` — ground truth: the factor catalog + the real-correlation methodology.
- `modules/01-inventory.md`, `04-serp.md` — the upstream load-bearing modules (page type, intent, the rank verdict).
- `modules/02-technical.md` — technical SEO + the Lighthouse-100 build standard.
- `modules/05-competitor.md`, `06-onpage.md`, `08-coverage.md`, `09-eeat.md`, `10-conversion.md`, `11-authority.md`, `12-migration-seo.md` — the analysis modules.
- `modules/13-roadmap.md` — the capstone synthesis.

---

## Step 1 — Ingest the whole set before reasoning about any part

Read all thirteen, in this order: `pipeline.md` → `factors.md` → `01` → `04` → `02` → `05` → `06` → `08` → `09` → `10` → `11` → `12` → `13`.

These documents are densely cross-referenced — modules cite each other's sections by number and inherit each other's outputs. **Reading any one in isolation will mislead you.** Do not begin planning until you have read all thirteen and can hold the dependency graph in your head.

---

## Step 2 — Discover the environment (verify, don't assume)

Before planning the build, confirm what's actually available and report it:
- **Integrations:** DataForSEO, Google Search Console, Ahrefs, Vercel, Resend, Neon — which are reachable, and are credentials present by env reference? (You never enter credentials yourself.)
- **The measurement stack** the `lib/` scripts will need: a stats library for Spearman / partial correlation / Benjamini-Hochberg FDR (e.g. scipy/statsmodels), an NER capability, a readability-style main-content extractor, and an embeddings path for semantic dedupe.
- **The render/extract path:** headless rendering for post-JS DOM extraction (needed both for the rebuild's content extraction and for measuring competitor pages).

Report what's present and what's missing. Missing capabilities are planning inputs, not things to silently work around.

---

## Step 3 — Produce the planning document (the deliverable — no code)

One document containing these six artifacts:

1. **Dependency-ordered implementation plan.** The build order, respecting (a) the acyclic execution order the specs define — coarse classifier (01) → SERP/intent (04) → full page-type assignment (01) → downstream — and (b) the measurement-before-interpretation layering. Note that the `methodology/lib/` measurement layer and the facts-file format (`factors.md` §3.4) are the foundation almost everything sits on, and should be built and validated first.

2. **Contradiction & seam report.** Every inconsistency, overlapping responsibility, undefined interface, or gap you find across the twelve docs. One known seam to confirm is reconciled: the SERP result page-type distribution is computed in `04` §6 and consumed by `05`'s homogeneity gate and `01`'s full assignment — verify that's coherent end to end, and surface any others like it. Be thorough; twelve docs written in sequence will have seams, and finding them now is the point of this pass.

3. **Missing-pieces assessment.** These are referenced but not yet specced: `03-gsc`, `07-local`, the `methodology/lib/` scripts, the `rubrics/` detail sheets, and the orchestrator `SKILL.md` (whose contract is `pipeline.md`). For each, state whether implementation can proceed from existing interface contracts or whether it needs a spec written first — and recommend which to spec before building.

4. **Load-bearing inventory.** Extract every decision marked **[LOAD-BEARING]** across all docs into a single list, each with its stated rationale. These are deliberate and must not be simplified, merged, or "optimized away" during the build. Confirm you understand each. If you believe any is wrong, say so here — do not act on that belief silently.

5. **Measurement / interpretation boundary map.** Per module, what is deterministic script (in `lib/`) versus model judgment. This split is the system's core principle (`pipeline.md` §1.1); getting it wrong reintroduces the exact detail-drift this whole spec set exists to prevent. Flag any place a doc is ambiguous about which side of the line a step falls on.

6. **Open operator decisions.** The dials the specs left to me, surfaced as explicit questions: business/lead value for ROI weighting (`13` §4), the FDR threshold q and the homogeneity dominant-type cutoff (`factors.md` §2.2/§2.4), the SERP cache TTL (`05` §4), the correlation N floor (`factors.md` §2.1), whether **Lighthouse Performance 100 is a hard deploy gate or a target with field Core Web Vitals as the binding bar** (`02` §2), and whether each project scaffolds a fresh Astro repo from template. Do not pick these for me — list them for decision.

---

## Rules that hold throughout

- **No code, no scaffolding, no API calls until the planning document is approved.**
- **The specs are the source of truth.** If disk reality — an unavailable API, a missing library, a platform constraint — contradicts a spec, flag it and propose an alternative. Never silently deviate from a spec.
- **Never weaken a [LOAD-BEARING] decision.** Flag and ask; do not reinterpret. Prefer pausing at a gate or emitting an artifact over improvising past an ambiguity.
- **Carry the why, not just the what.** When you eventually build, the rationale behind non-obvious decisions travels with the code, so a later pass doesn't reverse a deliberate choice.
- **Respect the gates and the secrets boundary.** Credentials are env references the operator sets; you reference, never enter or store them.

---

## Why this pass exists

Earlier attempts on this project failed because building started before the methodology was pinned down — the agent improvised the hard decisions and drifted. This spec set exists precisely so that doesn't happen again. A planning pass that surfaces the seams, the missing pieces, and the open decisions *now* is far cheaper than discovering them mid-build. Understand the whole, plan the order, flag the gaps — then stop and wait for approval before writing anything.
