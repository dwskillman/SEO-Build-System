# modules/09-eeat.md — EEAT Signals

**Status:** Spec doc 7. Extends the EEAT-detectable factors in `factors.md` §4.8–4.9. Feeds `modules/10-conversion.md` (Anxiety), `modules/06-onpage.md`, and `modules/13-roadmap.md`.
**Role in pipeline:** Phase 2 (Analyze). Per `pipeline.md` §7 row 09.
**Nature:** Measurement-led inventory + thin interpretation. Scripts detect signals present/absent; the model judges *sufficiency* against page type and YMYL bar. The split holds — detection is fact, sufficiency is judgment.

---

## 0. Mandate

**EEAT is not a ranking number and cannot be measured directly.** It is the lens from Google's Quality Rater Guidelines (Experience, Expertise, Authoritativeness, Trust). There is no "EEAT score" in any ranking system, and this module must never present one as if there were. What the module *can* do is inventory the **detectable proxies** for each pillar and judge whether they're sufficient for the page's type and risk level. State this honestly in every artifact (§6). [LOAD-BEARING]

**Trust is the center.** Google has been explicit that Trustworthiness is the most important of the four; Experience, Expertise, and Authoritativeness exist to support it. Weight the inventory accordingly — a page strong on expertise signals but weak on trust signals is weak overall, not balanced.

**Reuse, don't re-measure.** The schema and EEAT-signal factors are already measured in `factors.md` §4.8–4.9 during the 05 measurement pass. This module *consumes and extends* those, adding the off-site reputation stream and the sufficiency judgment — it does not re-parse pages for signals already captured.

---

## 1. Contract

**Inputs:**
- Measured signals from `factors.md` §4.8–4.9 (author/Person schema, review/rating schema, citations, dates, NAP, about/contact presence).
- Module 01 — page type (drives weighting, §4).
- Off-site reputation sources — review platforms, brand mentions (DataForSEO / Ahrefs / business-data endpoints).
- `config/site.md` — business type, locations (YMYL detection, §2).
- Client page + site-level pages (about, author, contact).

**Output:** `analysis/eeat/<page>.md` and a site-level `analysis/eeat/_site.md` — the per-pillar signal inventory (present/partial/absent), the YMYL verdict, a sufficiency judgment per pillar weighted by page type, and prioritized signal-addition recommendations.

---

## 2. YMYL gate [LOAD-BEARING]

Before judging sufficiency, classify the topic's risk. **YMYL ("Your Money or Your Life") topics — health/medical, financial, legal, safety, major life decisions — raise the trust bar substantially**, because Google holds them to a higher quality standard. Detection is from business type + page topic (config + module 04 intent).

- **A dental/medical site is YMYL** (health). So are finance, legal, and safety niches. The example keyword set (`dentist near me`) is squarely YMYL.
- On YMYL pages, missing trust/expertise signals are **critical findings**, not nice-to-haves: real practitioner credentials, licensure, author identity, citations to authoritative medical sources, and verifiable business legitimacy move from "recommended" to "required."
- Non-YMYL pages keep a normal bar; over-demanding EEAT on a low-risk page is its own error.

The YMYL verdict is stamped on the artifact and changes the thresholds in §4.

---

## 3. The signal inventory (deterministic, by pillar)

Detection only — present / partial / absent, with the value where one exists. No judgment in this stage.

### Experience (first-hand)
| Signal | Detection |
|---|---|
| Original imagery (non-stock) | image analysis: stock-library fingerprints vs original photos |
| First-person practitioner content | "we treat / in our office" patterns, authored-by-practitioner |
| Case studies / before-after | section/schema detection |
| Real patient reviews (on-page) | review content + AggregateRating schema |

### Expertise
| Signal | Detection |
|---|---|
| Author identity + credentials | author block / Person schema with credentials |
| Practitioner bios with qualifications | bio pages, degrees/licenses named |
| Topical depth by a qualified author | author ↔ content-topic match |

### Authoritativeness
| Signal | Detection |
|---|---|
| External citations to authoritative sources | outbound links to .gov/.edu/medical bodies |
| Inbound authority | referring domains (reuse 05 backlink data) |
| Off-site reputation | review-platform ratings, brand mentions (off-site stream) |
| Recognized credentials / affiliations | association/board logos + verifiable references |

### Trustworthiness (the center — weight highest)
| Signal | Detection |
|---|---|
| NAP consistency | name/address/phone present + cross-source consistency |
| Verifiable business legitimacy | license #, registration, real address/map |
| Contact methods + about depth | contact channels, about-page substance |
| Reviews + aggregate rating | count, average, recency |
| Last-updated / published dates | DOM/schema dates |
| HTTPS, privacy, transparent pricing | security + disclosure signals |

### Off-site reputation stream
Pulled separately (not on-page): review-platform aggregate ratings and counts, unlinked brand mentions, and reputation consistency across sources. Lower-reliability than on-page signals — stamp accordingly — but a real EEAT input Google uses, so don't omit it.

---

## 4. Page-type & YMYL weighting [LOAD-BEARING]

The same inventory, judged against different thresholds:

| Page type | Pillars that dominate sufficiency |
|---|---|
| Local money / service (YMYL) | **Trust** (NAP, reviews, legitimacy, real bios) — these make or break it; then Experience (real photos, first-person), Expertise (practitioner credentials) |
| Informational / cluster | **Expertise + Authoritativeness** (qualified author, citations); Trust via accuracy + dates |
| Comparison / listicle | Authoritativeness (citations, methodology transparency) + Trust (disclosure, no hidden bias) |

YMYL lifts the trust threshold across all types. The model judges sufficiency *against the dominating pillars for this type*, not against the whole inventory equally — a local money page with great citations but no reviews or NAP is failing where it counts.

---

## 5. Interpretation layer (the model)

Bounded to:
1. **Sufficiency judgment** per pillar — given the detected signals (§3), the page type (§4), and the YMYL verdict (§2), is each dominating pillar adequately supported? Cite the detected facts.
2. **Recommendations** — what specific signals to add, prioritized by pillar weight and YMYL criticality (e.g. "add Person schema + visible DDS credential to author block" beats "add a second blog category").
3. **Rationale** — one line per non-obvious call.

The model never sets present/partial/absent (that's §3 detection) and never invents a numeric EEAT score (there isn't one, §0).

---

## 6. Honesty / limitations (state in every artifact)
- EEAT is a *rater framework*, not a measurable ranking signal; this module reports **proxies**, and proxy presence does not guarantee a ranking effect.
- Off-site reputation signals are lower-reliability than on-page detection — flagged as such.
- Detecting a signal's *presence* is not verifying its *truth* (a claimed credential isn't a verified one). The module reports what's present and verifiable-on-its-face; it does not adjudicate authenticity.

---

## 7. Output artifact format

`analysis/eeat/<page>.md` (+ site-level `_site.md`):

| Column | Meaning |
|---|---|
| pillar | Experience / Expertise / Authoritativeness / Trust |
| signal | from §3 |
| status | present / partial / absent (detection) |
| value | the detected value where applicable (rating, date, count) |
| weight_for_type | how much this signal matters for this page type (§4) |
| sufficiency | model judgment per pillar (§5) |
| recommendation, rationale | model |

Header: page type, **YMYL verdict**, the §6 limitations, and the dominating-pillar set for the type.

## 8. Interfaces
**Consumes:** `factors.md` §4.8–4.9 measured signals, 05 (backlink data, off-site), 01 (type), 04 (topic for YMYL), config, off-site reputation sources.
**Exposes:** the trust-signal inventory to **10** (Anxiety scoring — the dependency this module unblocks), EEAT actions to **06** (brief), and EEAT completeness to **13** (roadmap). The author/review/org schema gaps also inform the schema work shared with `factors.md` §4.8.

## 9. Deferred
- `modules/11-authority.md` — site-level topical + link authority (overlaps the Authoritativeness pillar at site scope; 09 is page+site signals, 11 is the cluster/link structure).
- `modules/12-migration-seo.md`, `13-roadmap.md`.
