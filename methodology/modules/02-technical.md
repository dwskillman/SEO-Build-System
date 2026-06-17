# modules/02-technical.md — Technical SEO

**Status:** Spec doc 13. Feeds 13 (roadmap, as foundational-priority items) and the Verify phase (build acceptance). Coordinates with 12 on crawl-safety (§6).
**Role in pipeline:** Phase 2 (Analyze) for the existing-site audit; Phase 4 (Verify) to confirm the built Astro site meets the technical bar. Per `pipeline.md` §7 row 02.
**Nature:** Almost entirely deterministic — measured scores and crawl facts checked against thresholds. Thin interpretation for issue prioritization.

---

## 0. Mandate

**The technical foundation must be sound before content work earns anything.** A page that's slow, noindexed, broken, or uncrawlable gains nothing from term tuning — which is why 13 sequences technical fixes as foundational, ahead of content polish (`13` §3). This module establishes that foundation and enforces it as a build acceptance criterion.

**Lighthouse 100 is the build standard** (§2) — realistic on a static Astro/Tailwind site in a way it never was on the WordPress/Elementor original, which is part of why the rebuild target was chosen. But **lab scores are not the ranking signal** — field Core Web Vitals are (§4). The module targets 100 in the lab *and* verifies real-user CWV, never one at the expense of the other. [LOAD-BEARING]

---

## 1. Contract

**Inputs:**
- **Lighthouse data, two paths** (§2): a DataForSEO OnPage/Lighthouse pull, **or** an operator-provided Lighthouse JSON report. Normalized to one internal schema regardless of source.
- DataForSEO OnPage crawl — status codes, redirects, duplicate content, indexability, canonical, broken links, structured-data validity, mobile, HTTPS, hreflang.
- Module 03 GSC — **field Core Web Vitals** (CrUX/Search Console), indexing/coverage status.
- The built `site/` (Verify phase).

**Outputs:**
- `analysis/technical/<scope>.md` — the technical issue list (prioritized) + the Lighthouse scorecard + field-CWV status.
- Verify **acceptance verdicts** per built page against the Lighthouse target (§5).

---

## 2. Lighthouse [LOAD-BEARING]

### Target
**100 across all four categories** — Performance, Accessibility, Best Practices, SEO — as the build standard for the Astro output.

### Two input paths (normalized)
- **Default: DataForSEO** Lighthouse/OnPage pull.
- **Operator-provided JSON:** accept a Lighthouse JSON report the operator supplies (e.g. run in a specific environment/throttling profile). When provided, it takes precedence for that page/run, since the operator chose the measurement conditions deliberately.
- Both are parsed into one internal schema: the four category scores + the individual audit results (so failing audits can be turned into specific fixes, not just a number).

### Hard vs. target, honestly
- **Accessibility, Best Practices, SEO = 100 is a hard requirement.** These are reliably achievable on a clean static build; anything under 100 is a concrete, fixable defect (missing alt text, contrast, no canonical, etc.) and blocks acceptance.
- **Performance = 100 is the target, with field CWV as the true bar.** Lab Performance 100 is achievable on static Astro but can be sensitive to measurement environment and to required third-party scripts (e.g. a booking widget, analytics). The module reports the score, lists exactly which audits cost points, and — where a specific 100 isn't reachable for a justified reason — documents the exception with its cause rather than silently passing or silently failing.

### Operator decision (surface, don't decide)
Whether **Performance 100 is a hard acceptance gate** (blocking deploy until met, accepting the constraints that implies for third-party scripts) **or** a target where **passing field CWV is the binding requirement** and lab 100 is best-effort. This changes Verify's behavior; it's the operator's call (added to the hand-off open-decisions list).

---

## 3. Crawl-level technical health

Deterministic issues from the OnPage crawl, each as present/severity facts:
- **Status & redirects** — 4xx/5xx, redirect chains/loops, mixed content.
- **Indexability** — noindex, canonical correctness, robots directives, orphaned/blocked pages.
- **Duplication** — duplicate titles/meta/content, parameter duplication.
- **Broken links** — internal and outbound.
- **Sitemap & robots.txt** — present, correct, current.
- **Structured-data validity** — schema parses and validates (complements 09's schema *presence* check with *validity*).
- **Mobile-friendliness, HTTPS, hreflang** (if multilingual).

The model prioritizes these by severity × affected-page-importance; it does not invent the findings.

---

## 4. Core Web Vitals — field data is the ranking signal

Pull **field CWV** (LCP, INP, CLS) from GSC/CrUX, not just lab:
- Field data reflects real users and is what Google's page-experience signals actually use.
- A page can score lab-100 while failing field CWV (real-network/device variance) — when lab and field disagree, **field is the bar that matters for ranking**; lab is the diagnostic that tells you why.
- On a fresh Astro build with no field history, monitor field CWV post-deploy (Phase 5) until CrUX data accumulates; until then lab is the only available signal, stated as such.

---

## 5. Two evaluation moments

1. **Analyze** — audit the existing site: Lighthouse baseline + crawl issues + field CWV. Establishes the before-picture and feeds 13 the foundational fix list.
2. **Verify (Phase 4)** — run Lighthouse on each **built, deployed Astro page** and check against the §2 targets. Accessibility/Best-Practices/SEO under 100, or Performance below the agreed bar, blocks the page from "done." This is where the 100 standard is actually enforced, page by page.

---

## 6. Relationship to module 12 (no overlap)

The launch-day crawl-safety catastrophe check — accidental site-wide `noindex` / `Disallow: /` on the production build — is **owned by 12** (`12` §5), as part of migration parity. 02 owns general technical health and the Lighthouse standard. To avoid duplication: 02 reports indexability *issues* as technical findings; 12 enforces the *hard pre-deploy gate* that no URL changes ship with an indexability catastrophe. They share the underlying crawl data, not the gate.

---

## 7. Interpretation layer / honesty
Bounded to prioritizing issues by severity and importance, and turning failing Lighthouse audits into specific fixes. The model never fabricates a score or a crawl finding. State in every artifact: **lab Lighthouse ≠ field ranking signal**; where they diverge, field CWV governs and lab explains.

## 8. Output artifact
`analysis/technical/<scope>.md`: the Lighthouse scorecard (four categories + failing audits, with source path noted), the prioritized crawl-issue list, field-CWV status, and — in Verify — the per-page acceptance verdict against the §2 target.

## 9. Interfaces
**Consumes:** DataForSEO OnPage/Lighthouse **or** operator Lighthouse JSON, 03 (field CWV, coverage), the built site (Verify).
**Exposes:** foundational technical fixes to **13** (sequenced ahead of content), the Lighthouse acceptance verdict to **Verify**, and shared crawl data to **12** (without owning its gate).

## 10. Deferred
- `modules/03-gsc.md` — supplies field CWV and coverage data this module reads.
- `modules/07-local.md` — the remaining focused module.
