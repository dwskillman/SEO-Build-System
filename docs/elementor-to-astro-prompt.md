# Migration task: Elementor WordPress → Astro + Tailwind

You are migrating our WordPress/Elementor site (synced locally via WP Studio) into an Astro + Tailwind project. Previous attempts kept producing wrong page titles, missing sections, and incorrect layouts. **The root cause is that we were scraping rendered HTML for structure and copy — but Elementor's real layout and content do not live in the rendered DOM.** We are changing the approach. Do not write any code yet. Investigate, then produce a plan.

## Core principle: change the source of truth

Elementor stores the canonical structure of every page as JSON in `wp_postmeta` under the key `_elementor_data`. It is a clean tree of sections/containers → columns → widgets, each with `elType`, `widgetType`, `settings`, and nested `elements`. This is the editor's actual source — exact copy, exact structure, exact widget types, with none of the wrapper-div noise of rendered HTML.

**From now on:**
- **Copy and structure** come from `_elementor_data`. Never re-type text from rendered HTML or transcribe it off a screenshot.
- **Layout** comes from Elementor's generated CSS (`wp-content/uploads/elementor/css/post-{id}.css` and global CSS) and from `settings` in the JSON — not from inferring flex/grid out of the DOM.
- **Design tokens** (colors, fonts, spacing) come from Elementor's Global settings, mapped into `tailwind.config`. These are the single source; do not eyeball hex values from screenshots.
- Screenshots are for **verification only**, never as the primary source for copy or structure.

## Phase 0 — Investigate (do this before planning anything)

1. Confirm how to access the local site: is `wp-cli` available against the WP Studio instance? If not, find the alternative (direct DB query, REST API, or reading the SQL dump).
2. List all pages and their post IDs.
3. Dump `_elementor_data` for ONE representative page (e.g. `wp post meta get <id> _elementor_data --format=json`) and actually read its structure. Determine the Elementor version and whether it uses the legacy section/column model or the newer flex-container model — the JSON tree differs between them.
4. Locate the per-post generated CSS and the global Elementor CSS. Confirm where global colors/typography are defined.
5. Read our existing `design.md` and reconcile it against the real Elementor global settings. Note any conflicts.
6. Report what you found before proposing the plan.

## Phase 1 — Design tokens
Extract Elementor's global colors, typography, and spacing into `tailwind.config`. This is the foundation every component depends on. Verify the mapped values against the originals.

## Phase 2 — Content manifests
Write a small deterministic parser that flattens each page's `_elementor_data` into a per-page manifest: an ordered list of sections, each with its widget types and verbatim text content. This is mechanical extraction — the model must not paraphrase, summarize, or "clean up" any copy. The manifest is the ground truth for every page.

## Phase 3 — Component library (build before pages)
Elementor sites are repetitive. From the manifests, identify the recurring section patterns (hero variants, CTA bands, card grids, testimonial rows, etc.). Build a small set of reusable Astro components for those patterns **once**, and verify each against the original before reusing it. Do not re-derive layout per page.

## Phase 4 — Compose pages
Each page becomes: compose the existing components, fed with content from that page's manifest. Work **one page at a time, and one section at a time within a page.** Check off each section in the manifest as it is built so none are dropped. Commit after each page.

## Phase 5 — Visual verification loop (required)
After building each page, render the Astro output, screenshot it with Playwright, and diff it against the original page screenshot. Iterate until it matches. Without this closed loop there is no way to catch drift — treat it as a mandatory step per page, not optional polish.

## Rules
- Never re-derive copy from HTML or screenshots; copy comes only from the manifest.
- Layout comes from generated CSS and JSON settings, not DOM inference.
- One page at a time, one section at a time, commit incrementally.
- If something in the source is ambiguous, stop and ask rather than guessing.

## Your deliverable right now
A concrete, ordered plan that follows the phases above, grounded in what you actually found in Phase 0 (real page IDs, real Elementor version, real token locations). Flag anything in this approach that doesn't fit what you discover on disk, and propose the fix. Do not start writing migration code until the plan is approved.
