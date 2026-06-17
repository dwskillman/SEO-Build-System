# modules/05-competitor.md — Competitor Correlation Engine

**Status:** Spec doc 3. Implements the methodology defined in `factors.md`. Feeds `modules/06-onpage.md`.
**Role in pipeline:** Phase 2 (Analyze), the heaviest module. Per `pipeline.md` §7 row 05.
**Nature:** Pure measurement + statistics. **The model does not run inside this module.** Scripts pull, measure, and compute; the only place a classifier is invoked is the page-type call to module 01 (§2.6). The model's first contact with this work is *reading the emitted artifact* in module 06.

---

## 0. Mandate

Everything here is deterministic and reproducible from cache. Given the same cached inputs, two runs must produce byte-identical facts files and identical statistics. If any stage's output depends on model judgment, it is in the wrong module — move it to 06. The one allowed external dependency is module 01's page-type label per URL, treated as a typed input (§2.6).

---

## 1. Contract

**Preconditions (provided by upstream phases):**
- A resolved **keyword → page mapping**: which target keywords belong to which page. Source: `config/site.md` `money_keywords` + the GSC/inventory modules (existing) or the seed-brief keyword research (greenfield).
- Per-keyword **search volume** (for cluster weighting, §2.9). Source: DataForSEO/Ahrefs keyword data.
- The client page URL per page (existing mode). In greenfield there is no client page; the engine runs in **target-only** mode (produces target ranges, no client gap).
- Integration credentials present by env reference (`pipeline.md` §1.4).

**Inputs:** keyword set, page mapping, client URLs (or none), config.
**Outputs:**
- `data/serp/<keyword>/` — cached ranked list + cached result pages.
- `data/facts/<keyword>.{csv,json}` — the facts file (`factors.md` §3.4).
- `analysis/correlation/<keyword>.md` (+ machine sibling) — per-keyword stats (`factors.md` §5).
- `analysis/correlation/<page>.md` — the reconciled page-level driver set (§2.9).
- Every artifact carries a **confidence stamp** (§3).

**Postcondition:** module 06 can produce a page-type-aware brief from `analysis/correlation/<page>.md` without re-touching any API.

---

## 2. Execution flow

Run per keyword (stages 1–8), then reconcile per page (stage 9). Stages are individually cached and resumable (§4).

### 2.1 Stage 1 — Resolve the run set
Expand the keyword→page mapping into a work queue. **Prioritize by the GSC opportunity signal** (`pipeline.md` §6.2): keywords attached to striking-distance pages (position 5–15, real impressions) run first. Record the queue so a re-run resumes mid-list.

### 2.2 Stage 2 — SERP acquisition (ranked list)
Pull top 30 organic results per keyword. **Invoke the fallback chain exactly as `factors.md` §2.1.1(a):** DataForSEO → (backoff+retry on transient) → Ahrefs SERP → web_search-assembled → headless-browser last resort. Record **which tier produced the list** — this drives the confidence stamp. Strip ads/local-pack/other SERP features from the correlation sample (they're handled by the SERP and local modules). Cache the raw and parsed ranked list in `data/serp/<keyword>/serp.*`.

### 2.3 Stage 3 — Page acquisition + main-content extraction
For each result URL, fetch the **rendered DOM** and extract the **main-content region only** (`factors.md` §3.1–3.2). This is an **independent dependency** from Stage 2: on page-parse failure, substitute headless render / HTTP fetch (`factors.md` §2.1.1(b)) without touching the ranked list. Drop unfetchable results and decrement N for that keyword; record dropped URLs. Cache each extracted page.

### 2.4 Stage 4 — Term-universe generation
From the corpus of extracted main content, generate the important-terms and entity sets per `factors.md` §3.3 (n-grams → stop-word removal → TF-IDF vs general-language baseline → top terms; NER for entities). Persist the term set with the keyword — module 06 needs it, and re-runs must reuse the same set for comparability.

### 2.5 Stage 5 — Factor measurement
Run the `methodology/lib/` measurement scripts over every result page for the full §4 factor catalog, including the term-set factors from Stage 4 and the off-page control columns (referring domains, domain rank) from the Backlinks API. Compute `rank_score = (N+1) − position`. Write the facts file (`factors.md` §3.4). **This is the only source of numbers for everything downstream.**

### 2.6 Stage 6 — Homogeneity gate [LOAD-BEARING]
Call **module 01's page-type classifier** on each result URL to get a coarse page-type label. Compute the dominant type's share. If no type ≥ 60% (`factors.md` §2.2), set `homogeneous = false`. The gate only needs coarse labels (service / guide / listicle / product / tool / local), so it can use module 01's fast/heuristic path rather than its full interpretive pass. Record the type distribution in the artifact header regardless of verdict.

### 2.7 Stage 7 — Statistics
Branch on the gate and N:
- **If `homogeneous` and N ≥ 30:** full path — Spearman raw ρ/p, **partial Spearman controlling for referring domains** (`factors.md` §2.6), Benjamini-Hochberg FDR (q=0.10) over the factor set, driver flags on `partial_p`. Plus top-10 IQR target ranges (§2.8).
- **Else (mixed SERP or N < floor):** **distribution-only** — compute top-cluster target ranges, emit **no** correlation/driver claims, set confidence = `low` (§3). Never compute correlations on a failed gate.

### 2.8 Stage 8 — Client measurement + gap
Existing mode only. Measure the client page with the identical scripts and term set, append `client_value`, `gap`, `direction` per factor. Greenfield skips this (target-only).

### 2.9 Stage 9 — Page-level reconciliation
A page targets multiple keywords; reconcile per `factors.md` §2.7:
- A factor is a **page-level driver** if it's a driver for the volume-weighted majority of the page's keywords.
- Page target range = envelope of per-keyword IQRs; flag conflicts for module 06 to resolve in the brief.
- Page confidence = the **lowest** confidence among its contributing keywords (a page is only as trustworthy as its weakest SERP pull).
Emit `analysis/correlation/<page>.md`.

---

## 3. Confidence stamp (every artifact)

Driven by data-source tier (Stage 2) × gate/N outcome (Stage 7). The model in module 06 must read this and temper recommendations accordingly.

| Confidence | Condition |
|---|---|
| `high` | DataForSEO or Ahrefs list, homogeneous, N ≥ 30, full stats |
| `medium` | full stats but list came from web_search tier, **or** N between floor and 30 after drops |
| `low` | distribution-only (mixed SERP or N < floor), **or** headless-browser-tier list |
| `none` | could not assemble a usable list at all → no artifact; surface to operator as a data failure |

`none` is a hard stop for that keyword, not a silent skip — it appears in the run report so the operator knows the keyword went unanalyzed.

---

## 4. Caching & idempotency
Per `pipeline.md` §1.5. Each stage keys its cache by `(keyword, stage, input-hash)`. A re-run:
- reuses cached SERP lists and pages unless `--refresh` or cache age exceeds a configured TTL (SERP data staleness matters; default TTL worth setting per operator);
- **never re-pays** for an unchanged keyword;
- recomputes stats cheaply from the cached facts file when only the statistical config (q, N floor, TTL) changed — no re-fetch.
The facts file is the reproducibility anchor: stats must be regenerable from it alone.

## 5. Concurrency, rate, and cost controls
- Parallelize page fetches within a keyword; **serialize/throttle paid API calls** (SERP, Backlinks) under the provider rate limit with the same backoff used in the fallback chain.
- Backlink pulls are per-result and costly — batch and cache them; reuse a domain's referring-domain count across keywords within a run.
- Emit a per-run **cost/coverage report**: keywords analyzed, tier used per keyword, N achieved, confidence distribution, and dropped URLs.

## 6. Interfaces
**Consumes:** module 01 (page-type labels, Stage 6); GSC/inventory (keyword→page mapping, priority); keyword-volume source (weighting).
**Exposes:** the per-keyword and per-page correlation artifacts + confidence stamps to module 06, and the cost/coverage report to the roadmap module (13).

## 7. Failure taxonomy (explicit handling)
| Failure | Handling |
|---|---|
| Transient API error (429/5xx/timeout) | backoff + retry before any fallback (§2.2) |
| DataForSEO persistent | walk fallback chain; stamp confidence by tier |
| Some result pages unfetchable | drop, decrement N, record; continue |
| N drops below floor after drops | distribution-only, confidence `low` |
| Mixed SERP (gate fails) | distribution-only, confidence `low` |
| No usable list from any tier | confidence `none`, hard-surface to operator |
| Backlink data unavailable | run raw correlation only, **flag that partial-correlation control was unavailable** (don't silently present raw as partial) |

The last row matters: if the off-page control can't be pulled, the partial-correlation rigor (`factors.md` §2.6) is gone — the artifact must say so rather than passing raw correlations off as authority-controlled.

---

## 8. Deferred
- `modules/06-onpage.md` — consumes these artifacts; defines how confidence stamps and the homogeneity verdict translate into tempered, page-type-aware recommendations, and when a statistically-real driver is *deliberately ignored* because the page type shouldn't chase it.
- `methodology/lib/` measurement scripts — the per-factor implementations from `factors.md` §4 (their own reference, since they're code, not spec).
