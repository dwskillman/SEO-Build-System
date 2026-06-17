# modules/06-onpage.md — On-Page Brief

**Status:** Spec doc 4. Consumes `modules/05-competitor.md` (correlation artifacts), module 01 (page type), and modules 08/09/10 (coverage / EEAT / conversion). Produces the per-page brief a writer/builder actually uses.
**Role in pipeline:** Phase 2 (Analyze), the synthesis step. Per `pipeline.md` §7 row 06.
**Nature:** **This is where the model re-enters.** Module 05 measured and computed; 06 *interprets*. The model reads facts and stats — it never recomputes a number or re-measures a page. Its job is judgment within the rails below.

---

## 0. Mandate

The model's contributions here are: applying the page-type rubric, deciding per-finding whether to act / offload / route / ignore, writing recommendations in prose, and recording the *why* for every non-obvious call (`pipeline.md` §1.2). Everything numeric is inherited from 05's artifacts verbatim. If the brief contains a number that isn't traceable to a facts file or correlation artifact, it's fabricated — that's a defect.

**The central discipline of this module:** a statistically-real driver is an *input to a decision*, not an instruction. The interesting work is knowing when to **not** chase a real correlation because the page type forbids it. A brief that mechanically applies every driver is a failed brief.

---

## 1. Contract

**Inputs (all inherited, none re-measured):**
- `analysis/correlation/<page>.md` — reconciled drivers, target ranges, client gaps, **confidence stamp**, homogeneity verdict.
- Module 01 — assigned page type + the **page-role × SERP-intent verdict** (match or mismatch).
- Module 04 — the **"should this page rank for this term?"** gate result.
- Module 08 — coverage universe gap (present/absent subtopics, entities, questions).
- Module 09 — EEAT signal gaps, weighted by page type.
- Module 10 — conversion/LIFT findings + the conversion-path requirements.

**Output:** `reports/briefs/<page>.md` — a page-type-aware brief enforcing the progressive-disclosure rubric, every recommendation carrying an action + rationale + source artifact.

**Precondition gate:** if module 04 says the page **should not rank** for its target term (e.g. SERP wants a guide, page is a service page), that verdict is the **headline of the brief** and the engine does *not* proceed to term-level tuning. Optimizing factors on a page fighting the SERP's intent is wasted work; say so instead.

---

## 2. Page-type rubric [LOAD-BEARING]

Page type (from module 01) selects which goal the brief optimizes for. The same correlation artifact yields a *different* brief depending on type.

| Page type (funnel) | Optimizes for | Depth posture | Driver disposition default |
|---|---|---|---|
| Money / service / local landing (BOFU) | Conversion + just-enough relevance | **Lean.** Cap depth; offload surplus to cluster | Coverage/word-count drivers → offload, don't bloat |
| Informational / cluster (TOFU-MOFU) | Coverage + depth + links down to money pages | **Deep.** Comprehensiveness rewarded | Coverage/depth drivers → act here |
| Comparison / listicle | Format-match to what the SERP rewards | Structured (tables, multiple entities) | Format/structure drivers → act; format mismatch is the headline |
| Local landing (sub-case) | Local trust + proximity | Lean + trust-dense | NAP / reviews / map / local terms dominate |

---

## 3. Driver-disposition logic [LOAD-BEARING]

For each driver and gap in 05's artifact, the model picks one disposition. Rules constrain the choice; judgment operates inside them.

| Finding | On a money/BOFU page | On a cluster/TOFU page |
|---|---|---|
| Cheap on-page term/heading/schema driver (add a term, add FAQ schema, fix H1) | **Act** — low cost, no conversion harm | Act |
| Coverage / word-count / subtopic driver (SERP winners are comprehensive) | **Offload** the surplus to linked cluster content; keep this page lean enough only to establish relevance | **Act here** — this is where comprehensiveness lives |
| Off-page driver (referring domains, domain rank) | **Route** to the authority/link module (11); never write it as an on-page content task | Route to 11 |
| Driver conflicts with conversion rubric (e.g. "winners have more forms/links" vs distraction) | **Conversion wins;** note the tension and the trade made | Rare; case-by-case |
| Driver present but confidence = `low` / distribution-only | **Suggest** ("winners cluster here"), do not impose; no aggressive structural change off weak data | Suggest |
| Page-type × SERP-intent **mismatch** | **Headline finding** + the rank verdict; do not paper over with term tweaks | Headline finding |

The model records, per acted/ignored/offloaded driver, a one-line rationale. "Ignored: real driver but BOFU page — comprehensiveness routed to cluster page X" is a *good* brief line, not a gap.

---

## 4. The progressive-disclosure layout rubric [LOAD-BEARING]

Every money-page brief specifies layout in these bands (the resolution to coverage-vs-conversion):

1. **Above the fold:** intent-matched H1, value proposition, primary CTA, and the dominant trust signal for the type (reviews/aggregate-rating for local).
2. **Lead with the answer:** immediately below, the direct answer to the query's intent — satisfies the searcher and the intent signal before any depth.
3. **Supporting depth:** the coverage that *establishes relevance*, structured with headings/lists/schema — enough to clear "should this rank," no more.
4. **Offload band:** comprehensive subtopics live in cluster pages, linked from here (hub-and-spoke).
5. **Conversion path intact throughout:** CTA repeated without burying; form → Resend wired per module 10.

The brief must flag both failure modes explicitly when present: **CTA buried under SEO content**, and **too thin to establish relevance**. They are opposite failures against the same rubric, and a money page can exhibit either.

---

## 5. Brief structure (output format)

`reports/briefs/<page>.md`, in this order:

1. **Header** — URL, assigned page type, target keyword cluster, **confidence stamp + homogeneity verdict** (inherited from 05, surfaced so the reader knows how hard to lean on the rest).
2. **Rank verdict** — module 04's "should this rank?" result. If negative, this is the whole brief (plus the recommendation: which page/cluster *should* carry the term).
3. **Page-type × intent check** — match or mismatch, and if mismatch, what to do about it.
4. **Layout directives** — the §4 bands, with what belongs in each for this specific page.
5. **On-page actions** — the §3 dispositions as a prioritized list: factor, current value, target range, action, rationale, source artifact. Includes the deliberately-ignored drivers *with reasons*.
6. **Coverage actions** — from module 08: what to add here vs. offload to cluster.
7. **EEAT actions** — from module 09, weighted by type.
8. **Conversion actions** — from module 10 (LIFT), money pages only.
9. **Internal-link obligations** — hub-and-spoke: which cluster pages link in, which this page links to.
10. **Priority order** — ROI-ranked across all of the above, with confidence as a damping factor (don't rank a `low`-confidence structural change above a `high`-confidence cheap fix).

---

## 6. Confidence handling [LOAD-BEARING]

The confidence stamp from 05 is not decoration — it changes the *verb* of every recommendation:
- `high` → imperative ("set H1 to…", "add FAQ schema").
- `medium` → recommend with the caveat noted once.
- `low` / distribution-only → "winners cluster here; consider…" — and **suppress** aggressive structural directives (no "rewrite the page," no large word-count swings) off weak data.
- Where 05 flagged that **partial-correlation control was unavailable** (`05-competitor.md` §7), on-page drivers may be authority artifacts — the brief notes that the driver could be a domain-strength proxy, not a content lever.

A brief that issues `high`-confidence-style imperatives off `low`-confidence data is the failure this section exists to prevent.

---

## 7. Interfaces
**Consumes:** 05 (stats), 01 (type + intent verdict), 04 (rank gate), 08 (coverage), 09 (EEAT), 10 (conversion).
**Exposes:** the per-page brief to the rebuild phase (Phase 3 consumes briefs as the per-page target spec) and to the roadmap module (13) for site-level ROI ranking.

## 8. Deferred
- `rubrics/` — the detailed scoring sheets behind §2 (per-type relevance thresholds) and §4 (above-the-fold checklist).
- `modules/08-coverage.md`, `09-eeat.md`, `10-conversion.md` — the upstream finding-generators this module synthesizes.
- `modules/13-roadmap.md` — site-level synthesis and cross-page priority.
