# modules/10-conversion.md — Conversion (CRO)

**Status:** Spec doc 6. Engine is the **`conversion-optimizer` skill** (LIFT/ICE/Fogg). Consumes modules 01/04/09; feeds `modules/06-onpage.md` and the Verify phase.
**Role in pipeline:** Phase 2 (Analyze) for the audit; re-applied in Phase 4 (Verify) as a regression check. Per `pipeline.md` §7 row 10.
**Nature:** Judgment-led (unlike the measurement-led 05/08), but **evidence-grounded** — scores must cite actual page elements and use deterministic facts wherever they exist (§4). This is the module the operator's 20% target runs through.

---

## 0. Mandate

**Do not reinvent the conversion methodology.** The `conversion-optimizer` skill *is* the engine: its LIFT six-factor scorecard, /100 readiness roll-up, Fogg (B = MAT) CTA cross-check, ICE prioritization, and A/B hypothesis format are adopted verbatim. This module specifies only the *pipeline wiring* around it — how it gets its context, where it grounds in facts, what it emits, and how it owns the Resend path. [LOAD-BEARING]

**Money pages only.** Page type (module 01) gates this module. BOFU money / service / local-landing pages get the full LIFT audit. Informational / cluster pages do **not** — their goal is coverage (08), not conversion, and scoring them for conversion-readiness is a category error. Run 10 only where module 01 assigns a conversion-bearing type.

---

## 1. Contract

**Inputs:**
- The **`conversion-optimizer` skill** (the engine).
- Module 01 — page type (the gate) + value-prop/offer context.
- Module 04 — intent class + traffic source (the skill's "Relevance / message-match" input).
- Module 09 — detected trust signals (the skill's "Anxiety-reduction" input, as facts not guesses).
- Module 05 — competitor/SERP context (the skill's optional "competitive context" — already pulled, no new fetch).
- The page: rendered money-page copy (existing) **or** the planned page brief from 06 (greenfield / pre-build).
- `config/site.md` — business, offer, conversion goal, locations.

**Output:** `analysis/conversion/<page>.md` — the LIFT scorecard, ICE-ranked recommendations, Fogg read on the CTA/form, A/B hypotheses, **and the conversion-path requirements** (§5). Consumed by 06 (folds into the brief) and re-checked in Verify.

**Adaptation of the skill's Step 6:** inside the pipeline the skill's report goes to the artifact above for 06 to consume — it is **not** separately saved-and-presented to the operator as a standalone audit. Same template, different destination. (Standalone presentation would duplicate what 06's brief already surfaces.)

---

## 2. Engine — the conversion-optimizer skill

Run the skill's workflow (Steps 1–5) on the money page. The skill supplies: the LIFT factor definitions and 1–5 scoring (5 = handled well), the weighting (Value Prop 30% / Clarity 20% / Relevance 15% / Anxiety 15% / Distraction 10% / Urgency 10%), the /100 bands, the Fogg cross-check, and ICE. Do not alter these — the pipeline's job is to feed them good inputs and capture the output.

---

## 3. Auto-populate the skill's context (don't ask the operator)

The skill's Step 1 demands conversion goal, audience, and traffic source as critical context. In the pipeline these already exist upstream — populate them rather than blocking on a question:
- **Conversion goal** ← `config/site.md` (e.g. "book appointment", "submit lead form"). If genuinely absent, this is the one thing worth asking; never invent it silently (per the skill).
- **Audience** ← business brief / config.
- **Traffic source & intent** ← module 04's intent class (organic search intent for the target keyword) — this is what drives the Relevance/message-match score.
- **Competitive context** ← module 05 (already pulled).
Flag any inferred item in the artifact's assumptions block, exactly as the skill requires.

---

## 4. Ground scores in deterministic facts [LOAD-BEARING]

LIFT scoring is judgment, but three of the six factors have measurable substrates that must be used rather than eyeballed:
- **Distraction** ← count of competing nav links, outbound links, and CTAs on the page (deterministic, from the same DOM parse the measurement layer already does). A page with 40 nav links scores low on Distraction as a *fact*, not an impression.
- **Anxiety** ← module 09's trust-signal inventory (reviews/rating present, NAP, guarantees, privacy, real bios). Score Anxiety-reduction against what 09 actually detected.
- **Relevance** ← module 04's intent match (does the page's promise match the keyword's intent?).
The model still assigns the 1–5, but it must justify each with the cited fact. Value Proposition, Clarity, and Urgency remain primarily judgment (the skill's domain).

---

## 5. The conversion path — form → Resend [LOAD-BEARING]

The persuasion audit is half the module; the other half is the **mechanism the 20% is measured on.** This module owns its requirements; the rebuild phase implements them:
- **Form = minimal fields** (Fogg Ability): capture only what the lead requires. Every extra field lowers Ability and conversion. The module specifies the field set per conversion goal.
- **Form → Resend → lead notification.** The path is the conversion event. Define the notification recipient and the lead payload.
- **Anxiety-reducing confirmation** (Fogg + LIFT Anxiety): explicit success state and a response-time promise ("we'll reply within X"). A silent or ambiguous submit kills trust.
- **Spam protection without friction** — honeypot / token over CAPTCHA where possible (CAPTCHA taxes Ability; and per the safety boundary the system never solves CAPTCHAs anyway).
- **Persistence (Neon) only if needed** — lead storage / booking history. Many local sites need none.
- **Secrets boundary:** Resend keys and Neon strings are env vars the operator sets in the dashboards; the system references, never enters or stores them (`pipeline.md` §1.4).

---

## 6. Two evaluation moments

1. **Analyze (this phase)** — audit the existing money page (existing) or the planned page from 06 (greenfield/pre-build). The skill is explicitly built for the planning phase, so pre-build auditing of a brief is a first-class use, not a stretch. Output feeds 06.
2. **Verify (Phase 4)** — re-run the LIFT read on the *rendered, built* page to catch regressions: did the build bury the CTA, bloat the form, or drop a trust signal? A money page that scored well in planning but regressed in build is exactly what Verify exists to catch.

---

## 7. The Distraction factor *is* the coverage-vs-conversion check

LIFT's **Distraction** score is the quantified form of the failure mode module 06's layout rubric guards against (`06-onpage.md` §4: "CTA buried under SEO content"). When 06 offloads comprehensive coverage to cluster pages to protect conversion, **module 10's Distraction score is the validation of that trade** — they are two views of the same decision. If Distraction scores low *because* SEO depth crowds the page, that is direct evidence the coverage offload (08 → 06) didn't go far enough. Surface this linkage in the artifact so the two modules reconcile rather than pull against each other.

---

## 8. The 20% target — readiness vs. outcome [LOAD-BEARING]

The skill produces a **/100 conversion-readiness score, not a predicted conversion rate.** Be honest about the difference in every artifact:
- Readiness is the lever the page controls. The 20% is an *outcome* that also depends on traffic-intent match — which is why the intent module (04) feeding Relevance matters as much as the page craft.
- Frame 20% as the ceiling the system engineers toward on tight high-intent (often local) terms, not a uniform pass/fail line. A high readiness score on well-matched traffic is the honest path to it; a readiness score presented *as* a predicted rate is a misrepresentation.

---

## 9. Output artifact format

`analysis/conversion/<page>.md` — the skill's report template (LIFT scorecard, detailed findings, Fogg check, ICE-ranked recommendations, A/B hypotheses, quick-wins/bigger-bets), plus a **Conversion-path requirements** section (§5), plus the §7 Distraction-reconciliation note, plus the assumptions block. Header carries the page type and conversion goal.

## 10. Interfaces
**Consumes:** the `conversion-optimizer` skill; 01 (type/offer), 04 (intent/traffic), 09 (trust signals), 05 (competitive context), config.
**Exposes:** LIFT findings + path requirements to **06** (brief), the regression read to **Verify**, and the readiness score to **13** (roadmap, money-page ROI).

## 11. Deferred
- `modules/09-eeat.md` — supplies the trust-signal inventory this module scores Anxiety against (worth drafting before or alongside, given the dependency).
- `modules/11-authority.md`, `12-migration-seo.md`, `13-roadmap.md`.
