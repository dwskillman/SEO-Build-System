# SEO Rebuild Pipeline — System Spec (Backbone)

**Status:** Spine / pipeline contract. This is the first of several spec documents.
**Audience:** Claude Code (implementer) + operator (you).
**Scope of this doc:** the two state machines, the pipeline phases, the gates between them, the artifact each phase hands the next, the integrations and their boundaries, and the static-bundle-vs-generated-workspace architecture. It deliberately does **not** contain the factor definitions, module rubrics, or the correlation-engine internals — those are separate follow-on docs (see §8).

---

## 0. Mandate to the implementer

Implement to the **contracts** below, not to your own idea of a better shape. Several decisions here are marked **[LOAD-BEARING]**. Those were made deliberately, for reasons stated inline. Do not "simplify," merge, or optimize away a load-bearing decision without flagging it and getting a yes — silently improving them is the single most likely way this build goes wrong.

When a phase's behavior is under-specified, prefer: (a) emit an explicit artifact, (b) stop at a gate and ask, over (c) improvising and continuing. A wrong guess that flows downstream is far more expensive than a pause.

---

## 1. Operating principles

**1.1 Measurement and interpretation are separate layers. [LOAD-BEARING]**
Scripts and APIs (DataForSEO, GSC, parsers) *measure*. They produce deterministic **facts files** — tables of counts, positions, distributions, present/absent flags — as ground truth. The model *interprets* those facts: correlation, gap analysis, prioritization, recommendation writing. The model must never count term frequencies, estimate word counts, or eyeball competitor structure. *Why:* every "it keeps messing up the details" failure traces back to a model deriving numbers it should have computed. Counting from scraped HTML is lossy; a script is not.

**1.2 Carry the *why*, not just the *what*.**
Each generated artifact records the reasoning behind non-obvious calls (page-type assignment, the "should this rank" verdict, coverage trade-offs). A fresh agent reading only conclusions will quietly reverse deliberate decisions (e.g. keeping money pages lean). Rationale travels with the artifact.

**1.3 Approval gates over autonomy on live sites. [LOAD-BEARING]**
The pipeline runs ~90% unattended, but stops at hard gates before anything irreversible or rankings-affecting. Fully autonomous "point and walk away" against a live, money-making site is where the expensive mistakes live. Gates are cheap insurance. Enumerated in §5.

**1.4 Secrets stay with the operator. [LOAD-BEARING]**
API keys and connection strings (Resend, Neon, DataForSEO, etc.) live in environment variables the operator sets in the relevant dashboards. The system *references* them; it never enters, stores, prints, or commits them.

**1.5 Phases are idempotent and resumable.**
Each phase reads prior artifacts and writes its own. Re-running a phase re-reads inputs and regenerates outputs without redoing upstream phases. Artifacts in `data/` double as a cache and as checkpoints; re-runs diff against the last run rather than re-pulling paid API data.

---

## 2. The two state machines

The pipeline branches on two independent flags set in config when you point it at a target.

### 2.1 `input_mode` — what we're starting from
Detected at phase 1 by checking for a live site at the domain.

| Value | Meaning | Front-half behavior |
|---|---|---|
| `existing` | A live site responds at the domain | **Diagnostic.** Ingest extracts and preserves content; analysis diffs against current pages; migration-SEO (redirects, parity) runs. |
| `greenfield` | Bare domain, no live site | **Generative.** Nothing to ingest or preserve; migration-SEO does not run; analysis *defines* pages from scratch. Requires a seed brief (§3.3). |

**[LOAD-BEARING] — The analysis layer is shared across both modes.** Intent, SERP shape, competitor correlation, coverage, EEAT, conversion rubric run identically. Only the *front* (ingest) and the *consumer of the analysis* differ: in `existing`, analysis produces a gap-and-fix list against real pages; in `greenfield`, it produces the page set itself. Do not fork these modules per mode.

**Greenfield guardrails:**
- A domain name alone is insufficient intent signal — greenfield **requires** a seed brief (business, services, location, target keywords) and must confirm it with the operator before building. Do not invent a business.
- Even with no site, run a quick **history check** (backlinks / prior index presence). An expired domain with link equity changes strategy.

### 2.2 `deploy_target` — where the build lives
Set by operator; transitions only on explicit instruction toward `production`.

| Value | Meaning | Deploy behavior |
|---|---|---|
| `local` | Astro dev server only | Nothing deployed. |
| `preview` | Temp Vercel domain | **Autodeploy on each completed unit of work**; surface the preview URL each cycle (§6.3). |
| `production` | Live | Only on explicit "go live." **Hard gate** (§5). |

---

## 3. Architecture: static bundle + generated workspace

**[LOAD-BEARING]** Everything reusable is static and ships once. Everything site-specific is generated when you point the system at a target. This is what makes it agnostic-until-aimed rather than a one-off.

### 3.1 Static methodology bundle (ships once, never edited per-site)
```
/methodology/
  SKILL.md                 # orchestrator: phase order, branch logic, gate handling
  pipeline.md              # this document
  factors.md               # factor definitions + exact measurement method   [DEFERRED §8]
  modules/                 # one contract file per analysis module           [DEFERRED §8]
  rubrics/                 # page-type rubrics, LIFT/conversion, EEAT signals [DEFERRED §8]
  templates/
    astro-base/            # the fresh-Astro scaffold every project clones
  lib/                     # deterministic measurement scripts (the "measure" layer)
```

### 3.2 Generated per-site workspace (created on aim)
```
/projects/<domain>/
  config/
    site.md                # generated config (schema below)
  data/                    # raw API pulls + parsed facts files (cache + checkpoints)
  analysis/                # module outputs (gap lists, correlation, coverage, etc.)
  target-spec/             # the approved per-page blueprint (output of Analyze)
  site/                    # the Astro/Tailwind build
  reports/                 # roadmap, parity reports, baselines
```

### 3.3 Config schema (`config/site.md`)
Generated at aim; the seed brief block is required only for `greenfield`.
```yaml
domain: example.com
input_mode: existing | greenfield        # detected, confirmable
deploy_target: local | preview | production
competitors: []                          # discovered + operator-confirmed
locations: []                            # for local intent
money_keywords: []                       # the terms that must win
design_system: design.md                 # REQUIRED; operator supplies each run. Checked at Rebuild start (Gate D).
seed_brief:                              # REQUIRED if greenfield
  business:
  services: []
  location:
  target_keywords: []
integrations:
  dataforseo: env            # all secrets by env reference only
  gsc: env
  vercel: env
  resend: env | none
  neon: env | none
```

---

## 4. Pipeline phases & contracts

Linear spine with two branch points (`input_mode` at Ingest; `deploy_target` at Deploy). Each phase: **trigger → inputs → output artifact (the contract) → gate**.

### Phase 1 — Ingest
- **Trigger:** operator aims the system at a domain.
- **Process:** detect `input_mode`. If `existing`: crawl full URL inventory, extract content via **WP REST API** as the universal extractor (works across Gutenberg/Divi/Elementor/classic); apply a **builder-specific enhancer** only when that builder is detected (e.g. the Elementor `_elementor_data` extractor); capture current metadata, schema, internal-link graph; snapshot a **rankings/GSC baseline**. If `greenfield`: confirm seed brief, run history/backlink check.
- **Output artifact:** `data/ingest/` — page inventory, extracted content (verbatim), current-state metadata/schema, link graph, **baseline snapshot** (the before-picture and safety net).
- **Gate:** none, but `existing` must not proceed without a baseline snapshot.

> **[LOAD-BEARING] — universal extractor first, builder enhancers second.** "Agnostic WordPress" ≠ "agnostic Elementor." REST API is the floor that works regardless of builder; `_elementor_data` and friends are enhancements applied on detection, never the foundation.

### Phase 2 — Analyze
- **Trigger:** ingest artifacts present (or seed brief confirmed).
- **Process:** run the shared analysis modules (§7) — intent + SERP shape + **"should this page rank for this term?" gate**, competitor correlation, coverage universe, EEAT signals, conversion rubric, technical, page-type assignment.
- **Page type is assigned from two inputs:** what the page *is* (its role/content) **and** what the keyword's SERP *wants* (Google's opinion). When they disagree, surface the mismatch — it's often the most valuable finding for a money page that won't rank.
- **Output artifact:** `target-spec/` — a per-page blueprint: page type, target keywords, the "should this rank" verdict, coverage requirements, conversion/layout requirements, and (in `existing` mode) the gap-and-fix list vs. the current page; (in `greenfield`) the page definition from scratch.
- **Gate: GATE A — approve the target spec.** Nothing gets rebuilt until the operator approves the blueprint.

### Phase 3 — Rebuild
- **Trigger:** target-spec approved (Gate A passed).
- **Process:** scaffold Astro/Tailwind from `templates/astro-base/`; build the component library and design tokens once; compose pages from preserved content (existing) or generated content (greenfield) **driven by the target-spec**; enforce the **progressive-disclosure layout rubric** on money pages (answer + value prop + CTA above the fold; supporting depth below; deepest coverage offloaded to linked cluster pages). Wire conversion path (form → Resend) and persistence (Neon) only where the target-spec calls for them.
- **Output artifact:** `site/` — the Astro build, committed incrementally (per completed unit of work).
- **Gate: GATE D — design-system present.** The very first action of Rebuild is to confirm the operator-provided `design.md` exists. If missing, **stop and remind the operator**; do not scaffold or generate anything against a default/assumed design. *Why:* the design system is supplied fresh each run and design starts here — building even one component without it means rework, and it's an easy thing to forget to hand over.

### Phase 4 — Verify
- **Trigger:** a page/section reaches "done."
- **Process:** **visual parity** (screenshot-diff vs. original, `existing` mode); **SEO parity** (redirects present, metadata/schema preserved or improved, no indexable content lost); **conversion/UX rubric** check; **Lighthouse/CWV**.
- **Output artifact:** `reports/parity/` per page.
- **Gate: GATE B — migration-SEO hard gate (`existing` mode).** A page is not "done" if it lost indexable content or lacks a redirect from its old URL. *Why:* missed redirects and dropped metadata are the most common way a rebuild tanks existing rankings.

### Phase 5 — Deploy + Monitor
- **Trigger:** operator says "go live" (for production); continuous for preview.
- **Process:** `preview` autodeploys per unit of work and surfaces the URL; `production` deploys only on explicit instruction.
- **Output artifact:** live/preview URLs; ongoing audit loop that re-runs analysis on a cadence and diffs against baseline.
- **Gate: GATE C — production deploy + any URL change going live.**

---

## 5. Gates (the complete list)

- **GATE A — Target spec approval.** After Analyze, before any rebuild. Operator approves the per-page blueprint.
- **GATE D — Design-system present.** First action of Rebuild. Blocks scaffolding/generation if operator-provided `design.md` is missing; reminds the operator.
- **GATE B — Migration-SEO parity (existing mode).** Per page, in Verify. Blocks "done" without content preservation + redirect.
- **GATE C — Production / URL changes.** Before anything goes live or any URL changes ship.

All other transitions run unattended.

---

## 6. Integrations & boundaries

**6.1 DataForSEO** — SERP pulls, competitor page data, on-page/technical crawl. The "measure" layer for everything SERP- and competitor-side. Cache pulls in `data/`; diff on re-run to avoid re-paying.

**6.2 Google Search Console** — baseline snapshot + opportunity analysis (striking-distance keywords, high-impression/low-CTR pages, cannibalization). **GSC drives prioritization:** pages at position 5–15 with real impressions feed the front of the per-page work queue; the system does not audit all pages equally.

**6.3 Vercel** — deploy control surface for `preview` and `production`. **[LOAD-BEARING] — debounce deploys to one per completed unit of work** (section/page done → commit → deploy → surface preview URL), not per file write. Git-connected preview deploys (commit → preview URL) are the mechanism. *Why:* per-write deploys mean dozens of deploys per page and a flood of URLs.

**6.4 Resend** — the conversion path, not a side feature. The form → Resend → lead-notification flow **is** the money-page conversion mechanism the 20% target is measured on, so it lives inside the conversion module. Wired only where the target-spec defines a conversion action.

**6.5 Neon** — opt-in persistence (lead storage, bookings, audit history). Many brochure/local sites won't need it; include only when a detected need calls for it.

**Boundary (all of the above):** secrets by env reference only, set by the operator in the relevant dashboard. Per §1.4.

---

## 7. Module inventory (analysis layer)

One-line contracts only; full factor definitions and rubrics are deferred (§8). All run in both input modes (§2.1).

| # | Module | Emits |
|---|---|---|
| 01 | Inventory + page-type | page list with type assigned from page-role × SERP-intent |
| 02 | Technical | crawl issues: status, dupes, indexability, CWV |
| 03 | GSC opportunity | striking-distance, low-CTR, cannibalization; the work queue |
| 04 | SERP / intent | intent class, format Google rewards, **"should this rank?" verdict** |
| 05 | Competitor correlation | per-keyword factor distributions; client gap with direction + magnitude (the Cora-style engine) |
| 06 | On-page brief | per-page, page-type-aware brief enforcing progressive-disclosure layout |
| 07 | Local | GBP / local-pack / NAP / reviews (local intent) |
| 08 | Coverage | entity/question universe across top results; present/absent gap vs. our page |
| 09 | EEAT | detectable trust/expertise signals as present/absent, weighted by page type |
| 10 | Conversion | LIFT eval on money pages; conversion-path + layout requirements |
| 11 | Authority | site-level cluster, hub-and-spoke, internal-link adequacy |
| 12 | Migration-SEO | redirect map, pre/post parity (existing mode only) |
| 13 | Roadmap / target-spec | synthesis: the approved per-page blueprint, ROI-ranked by page type |

---

## 8. Deferred to follow-on spec docs

This backbone intentionally stops at the contracts. Next, in order of how badly Claude Code needs them pinned down:

1. **`factors.md`** — the full factor list with the *exact measurement method* for each (this is what keeps §1.1 honest).
2. **`modules/05-competitor.md`** — the correlation engine: SERP pull → per-page measurement → distribution → gap ranking, and the rigor choice (lightweight clustering vs. real correlation stats).
3. **`modules/06-onpage.md` + `rubrics/`** — the page-type rubrics and the progressive-disclosure layout rules the brief enforces.
4. **`rubrics/conversion.md`** — LIFT scoring + the Resend conversion-path requirements.
5. **`modules/12-migration-seo.md`** — redirect mapping + parity-check detail (the Gate B logic).

Recommended drafting order next: **`factors.md` first** (everything measures against it), then the correlation engine.
