# Phase 3 Data Patterns — Non-Workflow vs Workflow Paths

## Data Pattern for Phase 3

### Non-Workflow Path (Skipped Phase 2)
Query source tables directly:

```sql
SELECT destination, COUNT(*) as bookings, SUM(amount) as revenue
FROM <source_db>.<source_table>
WHERE date >= DATE_ADD(CURRENT_DATE, -30)  -- Apply date filter first
  AND status != 'Cancelled'
GROUP BY destination
```

### Workflow Path (Ran Phase 2)
Query pre-aggregated SINK tables (workflow output tables, star schema):

```sql
SELECT destination, SUM(bookings) as bookings, SUM(revenue) as revenue
FROM <sink_db>.<sink_table>
WHERE date >= DATE_ADD(CURRENT_DATE, -30)
  AND loyalty_tier = '${filter_loyalty_tier}'  -- Filter at SQL WHERE, not client-side
GROUP BY destination
```

**Critical rule:** Apply ALL filters at SQL `WHERE` clause layer. Never post-filter client-side after fetching all rows — violates the Phase 2 SINK table contract, and breaks the row-level detail pattern described in `./filter-architecture.md`.

---

## Build Loop

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

**Each stage has a quality gate — cannot proceed until gate passes.** Step letters keep their original labels from `./steps.md` (Step 4a is a no-op here since the rendering engine is fixed to HTML Client, not a decision to make).

### Handling Discrepancies

**When dashboard values differ from Phase 1 spot-checks** (e.g., "email open rate 87.4% vs 30.5%"):

1. Recheck Phase 1 vs Phase 3 query definitions — are they identical?
2. Check aggregation method — SUM vs COUNT vs DISTINCT vs other?
3. Verify dimension filtering — are date ranges/filters applied consistently?
4. Confirm the SINK table has the required columns.
5. If unresolvable, document as an open item in `state.md` and flag it during the approval step (4i).

---

**Version:** 1.0.0 (Lite)
**Last Updated:** 15 July 2026
