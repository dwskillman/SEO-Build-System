# modules/12-migration-seo.md — Migration SEO (the rankings guard)

**Status:** Spec doc 8. Guards **Gate B** (`pipeline.md` §5) and contributes to **Gate C**. Consumes Phase 1 ingest baseline + the approved target-spec.
**Role in pipeline:** redirect map drafted in Analyze; parity enforced per-page in Phase 4 (Verify); baseline diff in Phase 5 (Monitor). Per `pipeline.md` §7 row 12.
**Nature:** Mostly deterministic parity-checking; thin interpretation for redirect-target selection and the deliberate-vs-accidental judgment (§3, §8).

---

## 0. Mandate

**This module is the brake, not the gas.** Its single purpose is to ensure the rebuild does not lose the rankings the live site already has — the most common way a site rebuild fails. It is **conservative by default: when in doubt, preserve.** [LOAD-BEARING]

**Existing mode only.** Greenfield has no live site to preserve, so this module does not run (it produces a no-op artifact noting greenfield). Gate on `input_mode`.

A page is **not "done"** until this module clears it. That is the contract of Gate B, and it is non-negotiable: missed redirects and silently dropped metadata are exactly the failures that send live traffic off a cliff weeks after launch.

---

## 1. Contract

**Inputs:**
- **Phase 1 ingest baseline** — full URL inventory, extracted content (verbatim), current metadata/schema, internal-link graph, and the **rankings/GSC baseline snapshot** (the before-picture).
- The approved **target-spec** (`target-spec/`) — what each page was *intended* to become, including deliberate content moves (offloads to cluster pages, consolidations).
- The built `site/`.
- Module 06 briefs (for the offload/consolidation decisions to reconcile against).

**Outputs:**
- `reports/migration/redirect-map.*` — old URL → new URL, complete.
- `reports/migration/parity/<page>.md` — per-page parity verdict (content, metadata, schema, links).
- Gate B verdict per page; pre-deploy site-gate verdict.
- Post-deploy `reports/migration/baseline-diff.*` — rankings vs the Phase 1 snapshot.

---

## 2. Redirect mapping [LOAD-BEARING]

Every old URL must resolve to a deliberate destination. No page goes live leaving an old URL to 404.
- **1:1 where the page persists** — old URL → its new URL, **301 (permanent)**, never 302 (302 doesn't pass signals as a permanent move).
- **Consolidation (many → one)** — when coverage/06 merged thin pages, each old URL → the consolidated page.
- **Removal** — a removed page redirects to the **most relevant surviving page** (parent, category, or closest topical match), **not** a blanket redirect to the homepage. Mass-redirecting to the homepage is treated by search engines as a soft 404 and loses the signal entirely.
- **No chains or loops** — old → new directly; never old → interim → new. Detect and flatten chains; detect and break loops.
- **New cluster pages** (created by 06's offload) have no old URL and need no redirect — but must appear in the sitemap and be internally linked (§5), or they're orphans.

The redirect map is drafted in Analyze (so the operator can review URL changes before any build) and finalized before Gate C.

---

## 3. Content parity — judged against intent, not naive diff [LOAD-BEARING]

This is what separates a smart guard from a dumb one. The rebuild **deliberately** changes content (06 offloads comprehensive coverage to cluster pages, consolidates thin pages, tightens money pages). A naive old==new diff would flag every intended improvement as a "loss" and the gate would be useless.

So content parity is reconciled against the **target-spec and the cluster**:
- **Deliberate offload ≠ loss.** A subtopic removed from a money page is *preserved at site level* if it now lives on the cluster page the target-spec routed it to, **with a link from the money page to that cluster page**. Verify the destination actually exists and is linked. If the subtopic was removed but the cluster page doesn't exist or isn't linked, *that* is a loss.
- **Vanished content = loss.** Indexable content present in the ingest baseline, not in the new page, and not accounted for by an offload/consolidation in the target-spec → flagged as accidental loss. Hard fail for Gate B.
- **Measured, not eyeballed.** Compare baseline vs built on word count, headings, key entities/terms (reuse the measurement layer), internal links, and images-with-alt. The model only adjudicates the *reconciliation* against the target-spec (§8) — the diff itself is deterministic.

---

## 4. Metadata & schema parity

Preserved-or-improved, never silently dropped:
- **Title, meta description, H1, canonical** — present on the new page; changes must be intentional (per 06's brief), not accidental omissions. A money page that lost its title tag is a hard fail.
- **Structured data** — schema types present on the old page (LocalBusiness, Organization, Review/AggregateRating, FAQ, author) are preserved or extended. Dropping review or LocalBusiness schema on a local money page is a critical migration loss.
- **Canonicalization** — every new page has a correct self-canonical; no accidental cross-canonical that de-indexes a page.

---

## 5. Internal links, sitemap & crawl preservation

- **Internal-link graph** — the link relationships from the baseline are preserved or *intentionally* improved (hub-and-spoke from 11). Flag **orphaned pages** (no internal links in) — they lose discoverability and equity.
- **XML sitemap** — regenerated to the new URL set, submitted; old removed URLs excluded.
- **robots.txt / meta robots** — sanity gate: the build must not ship an accidental site-wide `Disallow: /` or `noindex` (a shockingly common launch-day catastrophe). Explicitly verify the production build is indexable.
- **Redirect-after-crawl** — confirm 301s actually resolve on the deployed target before Gate C.

---

## 6. The gates

**Gate B (per page, in Verify) — hard.** A page clears only if: (a) its old URL(s) have a valid 301 to it, (b) no accidental content loss (§3), (c) title/H1/canonical present, (d) old schema preserved-or-improved. Any failure blocks "done."

**Pre-deploy site gate (feeds Gate C).** Before any URL change goes live: redirect map complete (every old URL mapped), no chains/loops, no orphans, sitemap regenerated, production build verified indexable. The system refuses to ship URL changes with an incomplete redirect map.

---

## 7. Baseline & post-deploy monitoring

The Phase 1 rankings/GSC snapshot is the safety net. After deploy:
- diff current GSC positions/impressions/clicks against the baseline per page;
- **alert on drops** beyond a threshold (especially money pages), surfacing the likely cause (lost redirect, dropped content, deindexation);
- keep the baseline so the operator can prove recovery or trigger rollback.
This makes ranking regressions *detectable within days*, not discovered months later.

---

## 8. Interpretation layer (the model)
Bounded to:
1. **Redirect-target selection** for removed/consolidated pages — pick the most topically relevant surviving destination (not homepage).
2. **Deliberate-vs-accidental adjudication** (§3) — reconcile each content delta against the target-spec: intended move (with verified linked destination) vs accidental loss.
3. **Rationale** per non-obvious redirect or reconciliation.
The model never overrides a deterministic hard fail (missing redirect, dropped title, site-wide noindex) — those are non-negotiable regardless of judgment.

---

## 9. Output artifacts
Redirect map; per-page parity reports with Gate B verdict; pre-deploy site-gate report; post-deploy baseline-diff with alerts. All under `reports/migration/`.

## 10. Interfaces
**Consumes:** Phase 1 ingest baseline (content, metadata, schema, link graph, rankings snapshot), target-spec, 06 briefs, 11 (intended link structure), the measurement layer (parity diffs), GSC (baseline + monitoring).
**Exposes:** Gate B/C verdicts to the pipeline; the redirect map to the rebuild/deploy phases; baseline-diff alerts to the operator and 13.

## 11. Deferred
- `modules/11-authority.md` — the intended internal-link/cluster structure §5 preserves against.
- `modules/13-roadmap.md` — folds migration risk into the launch plan.
