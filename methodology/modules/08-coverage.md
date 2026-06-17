# modules/08-coverage.md — Coverage & Topical Completeness

**Status:** Spec doc 5. Consumes module 05's cached SERP corpus + DataForSEO SERP features. Feeds `modules/06-onpage.md` and `modules/11-authority.md`.
**Role in pipeline:** Phase 2 (Analyze). Per `pipeline.md` §7 row 08. This is the engine behind the operator's goals of "cover everything Google and users expect" and "be a topic authority."
**Nature:** Measurement-led, like 05. Scripts build the coverage universe and compute the gap; the model interprets *which* gaps matter and *where* they belong. The split holds: completeness is a measured quantity here, not a judgment call.

---

## 0. Mandate

The recurring failure this module avoids is "completeness as vibes." **Completeness is made measurable by one number: corpus coverage %** — the fraction of top-ranking results that address a given subtopic, entity, or question (§2.5). That number, not the model's intuition, defines what a complete answer contains. The model's job is to filter the measured universe for relevance and route each gap to the right page — never to decide from feel whether coverage is "enough."

---

## 1. Contract

**Inputs (mostly inherited — do not re-fetch what 05 cached):**
- Module 05's cached, main-content-extracted SERP corpus per keyword (`data/serp/<keyword>/`). **[LOAD-BEARING] reuse it** — re-pulling and re-extracting the same pages is wasted cost and risks drift between what 05 correlated and what 08 measures coverage against.
- DataForSEO SERP-feature data: **People Also Ask**, related searches, autocomplete. These are Google's own statements about the topic.
- Module 01 — page type (placement routing depends on it).
- Module 04 — intent class + the rank verdict (a coverage universe for a term the page shouldn't rank for is routed, not forced onto the page).
- Client page main content (existing mode) or none (greenfield → gap = the whole universe).

**Output:** `analysis/coverage/<page>.md` — the coverage universe with importance scores, the client gap (present / partial / absent per item), and a **suggested placement** (core-this-page vs offload-to-cluster vs FAQ) for every gap. Placement is a *suggestion*; module 06's page-type rubric makes the final call.

---

## 2. Building the coverage universe (deterministic)

### 2.1 Sources
Assemble from four streams, tagged by origin (origin affects trust):
- **Competitor main content** (reused 05 corpus) — subtopics and entities the ranking pages actually cover.
- **People Also Ask** — Google's explicit question set. Highest-trust question signal.
- **Related searches / autocomplete** — adjacent intents and phrasings.
- **The client's own content** — for the gap diff, not the universe.

### 2.2 Subtopic extraction + semantic dedupe [LOAD-BEARING]
- Aggregate headings (H2–H4) and section themes across the corpus; extract candidate subtopics.
- **Cluster semantically (embeddings) to merge near-duplicates** — "cost of treatment," "how much does it cost," "pricing" are one subtopic, not three. Without this the universe inflates into dozens of synonymous items and the gap analysis becomes noise. Each cluster gets one canonical label + its member phrasings.

### 2.3 Entity extraction
NER across the corpus → the entities a complete treatment references (procedures, conditions, brands-as-concepts, locations, credentials). Dedupe and canonicalize. Entities feed both coverage and the EEAT/schema modules.

### 2.4 Question extraction
- **PAA is the primary source** and is treated as authoritative — it's Google telling you the questions. Capture the PAA tree (questions expand into more questions).
- Supplement with question-form headings in the corpus and question patterns in client/competitor content.
- Each question carries its origin; PAA-origin questions outrank inferred ones.

### 2.5 Importance scoring [LOAD-BEARING]
Every universe item (subtopic / entity / question) gets:
- **corpus coverage %** = share of top-N results that address it. The core importance signal — high % = strong expectation, the thing a topic-authoritative page is *expected* to contain.
- **origin weight** — PAA / Google-feature origin lifts importance above content-only origin.
- **intent fit** — does the item match the SERP intent class (module 04), or is it adjacent? Adjacent-but-common items are cluster candidates, not core.
These three are carried numerically into the artifact; the model reads them, it does not invent them.

---

## 3. Gap analysis (deterministic)

Diff the client page against the universe. Per item, classify:
- **present** — adequately covered (the item's entities/terms appear in a dedicated section).
- **partial** — mentioned but thin (present without a real section / depth).
- **absent** — not covered.
Greenfield: every item is `absent` by definition; the universe *is* the build spec.

Output is a flagged table; no judgment yet.

---

## 4. Interpretation layer (the model)

The model reads the deterministic universe + gap and does four things only:
1. **Relevance filter** — drop universe items that are corpus-common but off-intent noise (boilerplate, competitor-brand-specific, legal footers that recur across sites). High corpus coverage % is a strong prior *for* keeping an item; overriding it requires a stated reason.
2. **Dedupe cleanup** — merge near-duplicate clusters the embedding step missed; split a cluster that conflated two real subtopics.
3. **Placement routing (suggestion)** — per gap, suggest core-this-page / offload-to-cluster / FAQ, using importance × intent-fit × page type:
   - high importance + on-intent → **core** (belongs on this page).
   - common but breadth/depth → **cluster** (offload; hub-and-spoke).
   - questions → **FAQ here** if few and on-intent, else a **dedicated cluster page** (matches the "answer on this page or another page to build relevancy" pattern).
4. **Rationale** — one line per non-obvious keep/drop/placement.

The model never assigns importance %, never decides present/partial/absent (those are §2–3 outputs), and never finalizes placement (06 does, per page type).

---

## 5. Confidence & homogeneity inheritance

Carry module 05's **homogeneity verdict** and **confidence stamp** for the keyword into the coverage artifact:
- On a **mixed SERP**, the subtopic universe is muddier (you're aggregating across page types), so subtopic/entity coverage is stamped lower confidence — but **PAA and related-search signals stay high-confidence regardless**, because they come straight from Google, not from correlating heterogeneous pages.
- Low confidence dampens the model's relevance-filtering aggressiveness: when the universe is noisy, prefer keeping items over pruning them.

---

## 6. Output artifact format

`analysis/coverage/<page>.md` (+ machine sibling):

| Column | Meaning |
|---|---|
| item | canonical subtopic / entity / question label |
| type | subtopic \| entity \| question |
| origin | content \| PAA \| related \| autocomplete |
| corpus_coverage_% | §2.5 importance core |
| origin_weight, intent_fit | §2.5 modifiers |
| client_status | present \| partial \| absent (§3) |
| suggested_placement | core \| cluster \| faq (model, §4) |
| rationale | model, non-obvious calls only |

Plus a header: keyword, intent class, homogeneity verdict, confidence, and the PAA tree captured in full (module 06 and the FAQ-schema work both draw on it).

---

## 7. Interfaces
**Consumes:** 05 (cached corpus, homogeneity, confidence), 01 (page type), 04 (intent + rank verdict), DataForSEO SERP features (PAA/related/autocomplete), client content.
**Exposes:** the coverage gap + placement suggestions to **06** (which finalizes act-here vs offload against the page-type rubric); the cross-page subtopic map to **11** (authority/hub-and-spoke); the PAA tree to the FAQ-schema work; coverage completeness scores to **13** (roadmap).

## 8. Deferred
- `modules/09-eeat.md`, `10-conversion.md` — the other two finding-generators 06 synthesizes.
- `modules/11-authority.md` — consumes the cross-page subtopic map to assess cluster completeness and internal linking at the site level (08 is page/universe scope; 11 is site scope).
- `modules/13-roadmap.md` — site-level synthesis.
