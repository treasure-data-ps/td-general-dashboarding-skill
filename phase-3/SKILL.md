---
name: td-general-dashboarding-skill-phase-3
description: |
  INTERNAL — invoked by td-general-dashboarding-skill root skill only. Do not trigger directly.
  Phase 3: Build + Validate Interactive Dashboard (HTML Client only). Execute the build+validate loop — generate queries, build structure, connect data via generate-data.js, render, validate accuracy, test filters, and get approval.
  Use when: Routed here by the root SKILL.md after Phase 1 (Requirements + Data Discovery) or Phase 2 (Workflow deployment, if used).
---

# Phase 3: Build + Validate Interactive Dashboard (Custom Dashboard Agent — Lite)

> **GUARDRAILS: Read `../references/guardrails-lite.md` before doing anything else in this session.**

**Phase Goal:** Build an interactive HTML Client dashboard from validated queries, run it through a build-validate loop, and get user approval — producing a live, accurate, performant `dashboard.html` ready for optional Phase 4 (automation) or Phase 5 (handoff).

**Deliverable:** User-approved `dashboard.html` (self-contained) + `generate-data.js`

**Rendering engine:** HTML Client only — a single portable `dashboard.html` file with data inlined at generation time. No server, no engine-choice question to ask.

---

→ **See `./build-interactive-dashboard-guide.md`** for: pre-phase checklist, step table (4a–4l), build+validate loop, quality gates, deliverables, quick reference, and next phase routing.

---

## Custom Dashboard-Specific Phase 3 Considerations

### Data Pattern for Phase 3

**Non-Workflow path (skipped Phase 2):** Query source tables directly
```sql
SELECT destination, COUNT(*) as bookings, SUM(amount) as revenue
FROM <source_db>.<source_table>
WHERE date >= DATE_ADD(CURRENT_DATE, -30)  -- Apply date filter first
  AND status != 'Cancelled'
GROUP BY destination
```

**Workflow path (ran Phase 2):** Query pre-aggregated SINK tables (workflow output tables, star schema)
```sql
SELECT destination, SUM(bookings) as bookings, SUM(revenue) as revenue
FROM <sink_db>.<sink_table>
WHERE date >= DATE_ADD(CURRENT_DATE, -30)
  AND loyalty_tier = '${filter_loyalty_tier}'  -- Filter at SQL WHERE, not client-side
GROUP BY destination
```

**Critical rule:** Apply ALL filters at SQL `WHERE` clause layer. Never post-filter client-side after fetching all rows — violates the Phase 2 SINK table contract, and breaks the row-level detail pattern described in `references/filter-architecture.md`.

### Build Loop

```
STAGE 1: BUILD (Steps 4b-4d)
+- 4b: Generate dashboard query scaffolding
+- 4c: Build dashboard structure (HTML Client)
+- 4d: Write generate-data.js, connect data, run it

STAGE 2: VALIDATE (Steps 4e-4h)
+- 4e: Render dashboard.html + jsdom headless validation
+- 4f: Validate data accuracy against manual spot-checks
+- 4g: Test filter interactions
+- 4h: Performance check

STAGE 3: APPROVE & DOCUMENT (Steps 4i-4l)
+- 4i: Get user feedback + approval
+- 4j: Document dashboard parameters (update state.md)
+- 4k: Test mobile & responsive design (conditional)
+- 4l: Initial load performance & UX
```

**Each stage has a quality gate — cannot proceed until gate passes.** Step letters keep their original labels from `references/steps.md` (Step 4a is a no-op here since the rendering engine is fixed to HTML Client, not a decision to make).

**When dashboard values differ from Phase 1 spot-checks** (e.g., "email open rate 87.4% vs 30.5%"):
1. Recheck Phase 1 vs Phase 3 query definitions — are they identical?
2. Check aggregation method — SUM vs COUNT vs DISTINCT vs other?
3. Verify dimension filtering — are date ranges/filters applied consistently?
4. Confirm the SINK table has the required columns.
5. If unresolvable, document as an open item in `state.md` and flag it during the approval step (4i).

---

→ **See `./build-interactive-dashboard-guide.md`** for: build+validate loop, quality gates, deliverables, quick reference, and next phase routing.

→ **See `./CHECKLIST.md`** for a condensed Phase 3 decision checklist (build, validate, filter/performance gates).

---

**Version:** 1.0.0 (Lite)
**Last Updated:** 15 July 2026
**Author:** FDE Team
