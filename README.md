# SEO Build System

A specification for an **agnostic WordPress→Astro/Tailwind rebuild + full SEO pipeline**. Point it at a domain; it rebuilds the site (or builds one from a bare domain) and runs a complete SEO program — technical, on-page, competitor correlation, coverage, EEAT, conversion, authority, migration — deploying to Vercel.

This repository currently contains the **methodology specification**, not the implementation. The specs are the source of truth an implementer (Claude Code) builds against.

## Architecture in brief

- **Static methodology bundle** (this `methodology/` folder, reusable) + a **generated per-site workspace** (created when aimed at a domain).
- Two state machines: `input_mode` (existing | greenfield) and `deploy_target` (local | preview | production).
- Five phases: Ingest → Analyze → Rebuild → Verify → Deploy/Monitor.
- Four gates: **A** target-spec approval · **B** migration parity · **C** production/URL changes · **D** design-system present.
- Core principle throughout: **measurement and interpretation are separate layers** — scripts measure, the model interprets.

## Structure

```
methodology/
  pipeline.md            # the spine: state machines, phases, gates, contracts
  factors.md             # ground truth: factor catalog + real-correlation methodology
  modules/
    01-inventory.md      # inventory + page-type classification + prioritization
    02-technical.md      # technical SEO + Lighthouse-100 build standard
    04-serp.md           # SERP analysis, intent, the rank verdict
    05-competitor.md     # the competitor correlation engine
    06-onpage.md         # page-type-aware on-page brief
    08-coverage.md       # topical coverage / completeness
    09-eeat.md           # EEAT signals + YMYL
    10-conversion.md     # CRO / LIFT (conversion path → Resend)
    11-authority.md      # topic clusters + internal linking + cannibalization
    12-migration-seo.md  # the rankings guard (redirects, parity)
    13-roadmap.md        # capstone: synthesis + ROI-ranked plan
claude-code-handoff-prompt.md   # entry instruction: how to ingest the set before building
docs/
  elementor-to-astro-prompt.md  # earlier design note (Elementor extraction approach)
```

## Read order

`pipeline.md` → `factors.md` → `01` → `04` → `05` → `06` → `08` → `09` → `10` → `11` → `12` → `13`. The documents are densely cross-referenced; read all before reasoning about any one.

## Status

**Specified (load-bearing core):** `pipeline.md`, `factors.md`, and modules 01, 02, 04, 05, 06, 08, 09, 10, 11, 12, 13.

**Not yet written:** modules `03-gsc` and `07-local`; the `methodology/lib/` measurement scripts (code); the `rubrics/` detail sheets; and the orchestrator `SKILL.md` (its contract is fully defined by `pipeline.md`).

## How to use

Start with `claude-code-handoff-prompt.md`. It instructs an implementer to read the whole set, verify the environment, and produce a planning document (dependency-ordered build plan, contradiction/seam report, missing-pieces assessment, load-bearing inventory, measurement/interpretation boundary map, and open operator decisions) **before writing any code**.
