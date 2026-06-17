# modules/04-serp.md — SERP & Intent

**Status:** Spec doc 12 (closes the load-bearing set). Consumes 05's cached SERP + 01's coarse classifier. Feeds 01 (full assignment), 05 (homogeneity input), 06 (rank verdict), 07, 08, 09, 10, 13.
**Role in pipeline:** Phase 2 (Analyze), runs after the coarse classifier is available and before 01's full page-type assignment. Per `pipeline.md` §7 row 04.
**Nature:** Deterministic SERP measurement (features + result page-type distribution) feeding an interpretive read (intent, format-match, the rank verdict). Grounded in the distribution, not vibes.

---

## 0. Mandate

**Don't guess what the searcher wants — read it off the SERP.** The composition of the SERP is Google's *published opinion* on intent: which page types rank, which features appear. If `dentist near me` returns a local pack + service pages, Google has decided the intent is local-transactional; if a term returns listicles and guides, a service page won't rank there no matter how good it is. [LOAD-BEARING]

The module's highest-value output is the **rank verdict** (§5): should this page even chase this term? Answering "no" early saves every downstream module from optimizing a page that fights its SERP — the cheapest high-value finding in the system.

---

## 1. Contract

**Inputs:**
- Module 05's **cached SERP** per keyword (ranked results + the raw SERP). Reuse it — no new pull.
- Module 05's **SERP-feature data** (local pack, PAA, featured snippet, shopping, video, knowledge panel).
- Module 01's **coarse classifier** — to label each organic result's page type.
- The target keyword → page mapping + the client page's coarse type (01).
- `config/site.md` (locations, business — local + YMYL signals).

**Outputs:**
- `analysis/serp/<keyword>.md` — intent class, format-match, SERP-feature map, the **rank verdict**, and the **result page-type distribution**.
- The page-type distribution is the **shared input to 05's homogeneity gate** (§6) — computed once here, not twice.

---

## 2. Intent classification [LOAD-BEARING]

Read intent from the SERP composition (result types via 01's coarse classifier + the feature set), not from the keyword string alone. Classify into:
- **Informational** — guides/articles dominate; PAA, featured snippets present.
- **Commercial investigation** — comparisons/listicles ("best", "vs"), review content.
- **Transactional** — service/product pages, "buy/book" CTAs, shopping features.
- **Local** — local pack present, location-bound results, map.
- **Navigational** — a specific brand/entity dominates.

A keyword can carry mixed intent; record the dominant class and note secondary intent. The classification is grounded in the measured result distribution (§6), so it's defensible, not a guess.

---

## 3. Format-match detection

The **dominant result page type is the format Google rewards** for this term. Compare it to the client page's coarse type (01):
- **Match** — the client page is the right format; proceed to on-page tuning.
- **Mismatch** — the SERP wants format Y, the client page is format X. This is the raw material for the rank verdict (§5) and 01's mismatch flag (§5 there).

Format-match is about *kind of page*, not quality — a great service page is still the wrong artifact for a SERP full of comparison guides.

---

## 4. SERP-feature analysis

Map the features present, because each is both an intent signal and an opportunity:
- **Local pack** → local intent confirmed; **triggers module 07** (GBP/local) and weights local trust signals (09).
- **PAA** → the question set (handed to 08's coverage universe as Google's own questions).
- **Featured snippet** → a winnable position; note the format it rewards (paragraph/list/table).
- **Shopping / product** → transactional/product intent.
- **Video / image packs** → media-format expectations.
- **Knowledge panel** → entity/navigational signal.
Record which features are present and which are realistically winnable for the client.

---

## 5. The rank verdict [LOAD-BEARING]

Combine format-match (§3) with intent (§2) into a per-page-per-keyword verdict:
- **Should rank** — page type matches the rewarded format; on-page optimization is worthwhile. Proceed.
- **Should not rank (as-is)** — the page fights the SERP's intent/format. The verdict is **not** "optimize harder"; it's **re-target the page** or **build/route to the page type the SERP wants**. This is the precondition that gates 06 (a "no" verdict makes the mismatch the whole brief) and the input to 13's "don't optimize what shouldn't rank."

In **greenfield**, the verdict dictates *what type to build*: it tells 11/01 that the planned page targeting this keyword must be the format the SERP rewards, before a line of it is built.

---

## 6. The shared result page-type distribution

Compute **once**: run 01's coarse classifier across the keyword's organic results → the distribution of result page types. This single artifact serves three consumers, avoiding duplicate work:
- **04 itself** — intent (§2) and format-match (§3).
- **05's homogeneity gate** — reads this distribution for its ≥60%-dominant-type check (`05-competitor.md` §2.6 / `factors.md` §2.2), rather than recomputing.
- **01's full assignment** — the format half of the role × intent reconciliation.

State the dominant type and its share in the artifact header.

---

## 7. Existing vs greenfield
- **Existing** — intent/format/verdict computed against the client page's actual type; mismatches actionable now.
- **Greenfield** — identical SERP read; the verdict shapes what to build rather than what to fix. No client page to mismatch, but the intent + rewarded format are required *before* design.

---

## 8. Interpretation layer / honesty
The model reads the measured distribution + features and judges intent, format-match, and the verdict. It does not fabricate the distribution or feature set (those are deterministic). State:
- **Reading intent from the SERP is strong but not infallible** — a mixed SERP signals genuinely mixed/uncertain intent; flag it rather than forcing a single class. (This is the same condition that makes 05 fall to distribution-only — mixed SERP, lower confidence everywhere.)
- The verdict is a strategic call grounded in the current SERP; it can shift if the SERP shifts, so it carries the same snapshot caveat as the correlation engine.

## 9. Output artifact
`analysis/serp/<keyword>.md`: intent class (+ secondary), format-match verdict vs client type, SERP-feature map with winnability, the **rank verdict + recommended action**, and the **result page-type distribution** (dominant type + share). Header: keyword, local/YMYL flags, mixed-intent flag.

## 10. Interfaces
**Consumes:** 05 (cached SERP + features), 01 (coarse classifier), keyword/page mapping, config.
**Exposes:** intent verdict to **01** (full assignment) and **06**; the page-type distribution to **05** (homogeneity) and **01**; PAA to **08**; local trigger + features to **07**; intent/traffic to **10** (Relevance) and **09** (YMYL topic); verdicts to **13**.

## 11. The load-bearing set is complete
With 01 and 04, every module that other modules depend on is specified: the spine (`pipeline.md`), ground truth (`factors.md`), and modules 01, 04, 05, 06, 08, 09, 10, 11, 12, 13. What remains is genuinely peripheral or implementation: `03-gsc` and `02-technical` (standard data pulls, implementable from their interface contracts), `07-local` (a focused GBP/local module), the `methodology/lib/` scripts (code), the `rubrics/` sheets, and the orchestrator `SKILL.md` (contract already fully defined by `pipeline.md`).
