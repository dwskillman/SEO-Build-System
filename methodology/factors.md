# factors.md — Factor Catalog & Correlation Methodology

**Status:** Spec doc 2 of the series. Referenced by the backbone (`pipeline.md` §8) and consumed by `modules/05-competitor.md`.
**Purpose:** This is the ground-truth document that keeps the measurement/interpretation split (`pipeline.md` §1.1) honest. It defines (a) every factor and the *exact deterministic method* used to measure it, and (b) the real-statistics methodology that turns those measurements into "which factors matter for this SERP, and what value should we target."

---

## 0. Mandate

The model never produces any number in this document. Scripts in `methodology/lib/` measure; scripts run the statistics; the model only *reads the resulting facts files and interprets them*. If implementing any part of this requires the model to count, estimate, or eyeball a value, that part is implemented wrong.

---

## 1. The two questions this engine answers

These are distinct and answered by different machinery. Do not conflate them.

1. **Does this factor matter for *this* SERP?** → answered by **correlation** (§2). Many factors will turn out not to matter for a given keyword; the engine must be able to say "no signal here."
2. **If it matters, what value should we target?** → answered by the **distribution of the top performers** (§2.5), *not* by the correlation. Correlation gives direction and strength; the top-cluster distribution gives the number to aim for.

A factor is only actionable when both are present: a significant correlation **and** a clear target range. Either alone is a half-finding.

---

## 2. Correlation methodology (real stats)

### 2.1 Sample
- Per target keyword, pull the top **30 organic results** via DataForSEO SERP API (exclude ads, local pack, and other non-organic SERP features from the correlation sample; analyze those separately in the SERP/local modules).
- **N = 30 is the floor for doing this honestly.** Top-10 is too small for stable rank correlation — critical Spearman ρ at N=10 is ≈0.65, so almost nothing clears significance and noise dominates. 30 gives usable power while staying inside the band where position is still meaningfully comparable.
- Drop results that fail to fetch/parse; record actual N used per factor (missing data shrinks N for some factors).

### 2.1.1 Source resilience & fallback chain [LOAD-BEARING]
The SERP pull is the single point of failure for the whole engine. Treat it as **two independent dependencies** with different backups.

**(a) The ranked list — which URLs rank where.** Real ranking data; never synthesize or guess an order. On DataForSEO failure, first separate transient from real: rate limits, timeouts, and 5xx get **exponential backoff + retry**, not fallback — most failures are transient and resolve on retry, and falling through on a blip needlessly degrades quality. Only on persistent failure, walk this chain (descending data quality):
1. **DataForSEO** — primary.
2. **Ahrefs SERP overview** — near-primary; already in the operator's stack. Returns ranked organic results with metrics. First real fallback.
3. **`web_search`-assembled set** — degraded. Recovers ranking URLs and approximate order; loses structured SERP metadata. Use to assemble the competitor list, then self-fetch each page.
4. **Headless-browser clean search** — last resort, genuinely lower quality: browser SERPs are personalized, feature-polluted, and IP-location-dependent. Force a clean, location-correct, non-personalized query and treat the result as suspect.

**(b) Page fetch/parse — measuring each result.** Fails independently of (a). If the ranking list is fine but DataForSEO's page parse fails, a **headless render or plain HTTP fetch is near-parity** — rendered-DOM extraction (§3.1) is already required, so the browser path substitutes cleanly with little quality loss.

**Governing rule — degrade transparently, never fabricate.** If the chain can only produce a dirty source *or* drops below the N=30 floor, the correlation engine does **not** run on it: fall back to **distribution-only** (top-cluster target ranges, no correlation/driver claims) and stamp the artifact low-confidence, the same way the homogeneity gate (§2.2) refuses a mixed SERP. *"Here's where winners cluster, but drivers couldn't be established reliably"* is the correct output, not confident correlations off garbage. **GSC** can sanity-anchor the operator's *own* page positions but cannot supply competitor ranks — it's a cross-check, not a SERP replacement.

### 2.2 Intent-homogeneity gate [LOAD-BEARING]
**Before correlating, verify the SERP is intent-homogeneous** — i.e. one page type dominates (mostly service pages, or mostly guides). A mixed SERP (some listicles, some service pages, some tools) makes factor-vs-position correlation meaningless, because you're correlating across pages that aren't trying to do the same thing. If the SERP is mixed (no page type ≥ ~60% of results), **do not report correlations** for that keyword — flag it as heterogeneous and fall back to top-cluster distributions only. This gate is computed from the page-type classifier (module 01) run across the SERP results. *Why:* this is the most common way correlation-based SEO analysis produces confident nonsense.

### 2.3 Coefficient: Spearman, with a defined sign convention
- Use **Spearman rank correlation** as the primary coefficient. Position is ordinal, relationships are typically monotonic but non-linear, and Spearman is robust to the outliers common in factor data. (Pearson assumes linearity and interval scale — wrong for this.)
- **Sign convention:** compute a `rank_score = (N + 1) − position` so higher = better-ranking, then correlate `factor_value` against `rank_score`. This makes **positive ρ = factor associates with better ranking**, which is the intuitive reading. Define it once, use it everywhere; the raw "factor vs position" sign is the opposite and will silently invert recommendations if used by accident.

### 2.4 Significance + multiple comparisons [LOAD-BEARING]
- For each factor compute ρ, its exact p-value, and the N actually used. Carry all three.
- Testing a few hundred factors against position per keyword guarantees false positives by chance. Apply **Benjamini-Hochberg FDR correction** across the full factor set per keyword. Only factors surviving FDR (default q = 0.10) are labeled **drivers**; the rest are **neutral / insufficient evidence** — *not* "do nothing," but "no reliable signal in this SERP."
- Never present an uncorrected significant correlation as a finding. The FDR step is what separates this from a tool that confidently reports 40 "ranking factors" that are noise.

### 2.5 Target-range derivation (independent of correlation)
- For each factor, compute the distribution among the **top 10** results: median and interquartile range (IQR, 25th–75th percentile).
- The **target range = the IQR of the top 10.** Median is the aim point. This is the POP "target range" output, but derived live from this SERP rather than a lookup table.
- This runs for every factor regardless of correlation result, because the operator still wants to see "where do winners cluster" even on non-driver factors.

### 2.6 Partial correlation — controlling for off-page strength [LOAD-BEARING]
On-page factors correlate with ranking partly because strong domains both rank well *and* tend to have more polished on-page. Raw correlation will over-credit on-page factors that are really proxying for authority.
- Pull an **off-page strength proxy** per result from DataForSEO Backlinks API: referring domains (primary) and domain rank.
- Compute **partial Spearman correlation** of each on-page factor against rank_score **while controlling for referring domains.** Report both the raw and the partial coefficient.
- The **partial coefficient is the one used for prioritization.** A factor whose signal collapses once you control for authority is not an on-page lever you can pull — say so. *Why:* this is the difference between "real correlation stats" and an r-value print-out; it's the single biggest rigor upgrade and the reason N=30 + backlink data is required.

### 2.7 Aggregation across a keyword cluster
A money page targets several keywords, not one. Run the engine per keyword, then reconcile to the page:
- A factor is a page-level driver if it's a driver for the majority of the page's target keywords (weighted by each keyword's value/volume).
- Target range at the page level = the union/overlap of per-keyword IQRs; where they conflict, widen to the envelope and flag the conflict for the model to resolve in the brief.

### 2.8 Honest limitations (state these in every report)
- **Correlation is not causation.** The engine identifies association in the current SERP; it cannot prove a change will move rankings. The partial-correlation step reduces but does not eliminate confounding.
- **Single-SERP snapshots are noisy.** Re-running on a different day can shift weak correlations. Only FDR-surviving drivers should be treated as stable.
- **Authority can't be optimized on-page.** When off-page strength dominates a SERP (partial correlations of on-page factors all weak), the honest finding is "winning here is a links/authority problem, not an on-page one" — and the system should say that rather than inventing on-page busywork.

---

## 3. Measurement layer (the deterministic recipes)

### 3.1 Page acquisition
- Measure each result on its **rendered DOM (post-JavaScript)**, not raw HTML, to match what Google evaluates. Use DataForSEO's parsed-page endpoint or a headless render; record which.
- Cache every fetched page in `data/serp/<keyword>/` so re-runs don't re-fetch (cost + reproducibility).

### 3.2 Main-content extraction [LOAD-BEARING]
All body/term factors are measured on the **main content region only** — not nav, header, footer, or sidebar. Use a readability-style content extractor to isolate the article/main region before any term counting. *Why:* counting terms across the whole DOM is the on-page equivalent of the Elementor "count from rendered HTML" failure — boilerplate nav/footer text pollutes every frequency measure and corrupts the correlation. Extract the content region first, deterministically, then measure.

### 3.3 Term-universe generation (how "important terms" are derived)
The important-terms list is not a fixed dictionary — it's generated from the SERP corpus, the way POP/Cora do it:
1. Extract main content from all N results.
2. Generate candidate 1–3-grams; drop stop words (use the configured stop-word list).
3. Score each candidate by **TF-IDF against the SERP corpus relative to a general-language baseline** — terms that are common across these results but rare in general language are the distinctive/topical terms.
4. Keep the top-ranked terms as the "important terms" set. Entities (via an NER pass) are added as a parallel set.
5. Each term then becomes a factor (its per-page usage is measured and correlated like any other).

### 3.4 Facts-file output
Every measurement run writes a facts file per keyword: a table of `result_url × factor → value`, plus the off-page proxy columns, plus the computed `rank_score`. This file is the sole input to the statistics step and the sole numeric input to the model. Format: machine-readable (CSV/JSON) in `data/facts/<keyword>.*`.

---

## 4. Factor catalog

Organized by category. Each factor: what it measures and the measurement method. Source is the rendered DOM main content unless noted. This catalog is the standard Cora/POP-equivalent set; it is **extensible** — add factors by appending here with a measurement recipe, never by having the model invent them at runtime.

### 4.1 Title (`<title>` / SERP title)
| Factor | Measurement |
|---|---|
| Title length (chars / pixels) | char count of `<title>`; pixel width via font metrics |
| Keyword in title | exact + partial match of target term in `<title>` (boolean + position index) |
| Important-term count in title | count of §3.3 terms present in `<title>` |

### 4.2 Meta description
| Factor | Measurement |
|---|---|
| Meta description length | char count of `meta[name=description]` |
| Keyword in meta description | match flags as above |

### 4.3 Headings (H1–H6)
| Factor | Measurement |
|---|---|
| H1 present / count | count of `<h1>` |
| Keyword in H1 | match flags |
| Heading count by level | counts of H2…H6 |
| Important-term count in headings | §3.3 terms across all headings |
| Question headings | headings matching the question-word list (PAA coverage proxy) |

### 4.4 URL
| Factor | Measurement |
|---|---|
| URL length | char count |
| Keyword in URL slug | match flag |
| Path depth | count of path segments |

### 4.5 Body content & keyword usage
| Factor | Measurement |
|---|---|
| Word count (main content) | token count of extracted main content (§3.2) |
| Keyword frequency | count of exact target term in main content |
| Keyword density | keyword frequency ÷ word count |
| Keyword in first 100 words | boolean |
| TF-IDF of keyword | TF-IDF vs SERP corpus |
| Important-term frequencies | per-term count for every §3.3 term (each is its own factor) |
| Entity coverage | count of corpus entities present (NER) |

### 4.6 Page structure (the POP "Page Structure" set)
| Factor | Measurement |
|---|---|
| Images | count of `<img>` in main content |
| Image alt coverage | % of images with non-empty alt |
| Internal links | count of links to same registrable domain |
| External links | count of links to other domains |
| Lists | count of `<ul>`/`<ol>` |
| Tables | count of `<table>` |
| Bold / strong terms | count of `<b>`/`<strong>` |
| Italic / em terms | count of `<i>`/`<em>` |
| Forms | count of `<form>` |
| FAQ present | FAQ schema OR question-heading cluster present (boolean) |
| Video present | `<video>` / known embed (boolean) |

### 4.7 Readability & content shape
| Factor | Measurement |
|---|---|
| Readability grade | Flesch-Kincaid (or equiv) on main content |
| Avg sentence length | tokens ÷ sentences |
| Avg paragraph length | tokens ÷ paragraphs |
| Passage/section count | count of heading-delimited sections |

### 4.8 Structured data / schema
| Factor | Measurement |
|---|---|
| Schema types present | parse JSON-LD/microdata; list `@type`s |
| Org / LocalBusiness schema | boolean |
| Author / Person schema | boolean (feeds EEAT module) |
| Review / AggregateRating schema | boolean + value (feeds EEAT + local) |
| FAQ / HowTo schema | boolean |

### 4.9 EEAT-detectable signals (present/absent; full rubric in module 09)
| Factor | Measurement |
|---|---|
| Visible author + credentials | author block / Person schema present |
| External citations | count of outbound links to authoritative domains |
| Last-updated date | date present in DOM/schema |
| About/contact depth | presence + word count of about/contact pages |
| NAP present | name/address/phone detected (local) |

### 4.10 Technical / page experience
| Factor | Measurement |
|---|---|
| HTTPS | scheme check |
| Core Web Vitals (LCP/CLS/INP) | DataForSEO/Lighthouse per result |
| Mobile-friendly | viewport + responsive checks |
| Page weight / requests | from crawl |

### 4.11 Off-page (controls, §2.6 — not on-page levers)
| Factor | Measurement |
|---|---|
| Referring domains | DataForSEO Backlinks API (primary control variable) |
| Domain rank | DataForSEO |
| Backlink count | DataForSEO |

---

## 5. Engine output artifact

`analysis/correlation/<keyword>.md` (+ machine-readable sibling), containing per factor:

| Column | Meaning |
|---|---|
| factor | name from §4 |
| raw_ρ, raw_p | Spearman vs rank_score |
| partial_ρ, partial_p | controlling for referring domains (§2.6) — **the prioritization column** |
| N | results actually used |
| driver? | survives FDR (§2.4) on partial_p |
| target_median, target_range | top-10 IQR (§2.5) |
| client_value | our page's measured value |
| gap, direction | client vs target, with sign |

Plus a header block: SERP intent class, **homogeneity verdict** (§2.2), dominant page type, and the §2.8 limitations stated explicitly. The page-level reconciliation (§2.7) is a second artifact, `analysis/correlation/<page>.md`.

---

## 6. Deferred to next docs
- **`modules/05-competitor.md`** — the orchestration of §2–§5 (pull → measure → stats → reconcile), error handling, and how it consumes module 01's page-type classifier for the homogeneity gate.
- **`modules/06-onpage.md` + `rubrics/`** — how the brief turns this artifact into page-type-aware, progressive-disclosure recommendations (and when it ignores a driver because the page type shouldn't chase that factor).
