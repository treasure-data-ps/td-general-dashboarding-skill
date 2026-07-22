---
name: phase-2-instructions
description: Phase 2 specific instructions for workflow deployment and SINK table creation
priority: HIGH
load_order: 1.2
---

# Phase 2 Instructions — Workflow Deployment

**Read this BEFORE `./SKILL.md`.**

**Phase 2 is OPTIONAL and only for Score 4-6 (or user override).**

---

## Phase 2 Goal

Deploy a scheduled Treasure Data workflow that:
- Runs on a defined schedule (daily/weekly/custom)
- Pre-aggregates metrics into SINK tables
- Provides optimized tables for Phase 3 dashboard queries

**Deliverable:** 
- SINK tables created and populated
- Workflow deployed and validated
- state.md appended with Phase 2 results

---

## Phase 2 Specific Rules (In Addition to Universal Rules)

### Rule P2-0: Dry-Run First (Before Approval)

**⚠️ CRITICAL: CANNOT request approval without showing dry-run output first.**

**Step 1: Run dry-run BEFORE presenting approval template**

```bash
# Show user what WILL happen (no actual creation)
tdx wf run [workflow_name] --dry-run
```

**Step 2: Present dry-run output to user**

Include in presentation:
- ✓ What tasks will run (task graph)
- ✓ What queries will execute (which SQL)
- ✓ What SINK tables will be created (names, schemas)
- ✓ Estimated duration per run
- ✓ Estimated query cost
- ✓ Schedule start time
- ✓ Frequency (daily/weekly/etc)

**Example:**
```
Dry-run shows:
  Task 1: aggregate_revenue.sql
    - Will read from: sales_events
    - Will write to: project_slug_sink_revenue
    - Estimated rows: 365 (one per day for 1 year)
  
  Task 2: aggregate_customers.sql
    - Will read from: customers
    - Will write to: project_slug_sink_customers
    - Estimated rows: 1,200,000
  
  Total cost per run: ~$2.50
  Total storage (incremental): ~500 MB/month
  Schedule: Daily at 02:00 UTC
```

**Step 3: Ask user: "Does this match your expectations?"**

- If NO → Adjust plan (different schedule, different metrics, etc.)
- If YES → Proceed to approval template (Rule P2-1)

**Why:** Dry-run lets user see exactly what will happen BEFORE committing. Prevents surprises.

---

### Rule P2-1: Approval Gate — SINK Table Creation (ENFORCEMENT)

**⚠️ CRITICAL: CANNOT create SINK tables without explicit user approval.**

**Step 1: Present approval template (COPY/PASTE exactly):**

```
📋 Ready to create SINK tables in Treasure Data:

Database: [database_name]

SINK Tables:
  • [project_slug]_sink_[metric_group_1]
  • [project_slug]_sink_[metric_group_2]
  • [project_slug]_sink_[metric_group_3]

Workflow:
  • Name: [workflow_name]
  • Schedule: [e.g., "Daily at 02:00 UTC"]
  • Duration: ~[Xm] per run

💰 Cost Impact:
  • Query cost: ~$X per run
  • Storage cost: ~$X per month (tables)
  • Total estimated: $X per month

⚠️ This CANNOT be easily undone. Tables will start populating immediately upon deployment.

To approve, please type exactly: YES, APPROVE SINK TABLE CREATION FOR [PROJECT_SLUG]
(not just "yes" — type the full phrase)
```

**Step 2: Wait for response**
- If user types: "YES, APPROVE SINK TABLE CREATION FOR [PROJECT_SLUG]" → Proceed ✓
- If user types: "NO" or "REVIEW" or anything else → **STOP immediately** ✗

**Step 3: If NO or REVIEW**
- Do not create anything
- Ask explicitly: "What would you like to change?"
- Gather feedback (e.g., different schedule, fewer tables, lower cost)
- Adjust plan and re-present approval template
- Repeat until explicit YES approval

**Step 4: If user says "just do it anyway"**
Respond:
> "I cannot proceed without explicit approval. SINK tables create real costs (storage + compute) and have real consequences (data starts flowing immediately). This cannot be easily undone.
>
> Please type exactly: YES, APPROVE SINK TABLE CREATION FOR [PROJECT_SLUG]
>
> Or tell me what to adjust so you can approve."

**Why:** SINK tables are real costs and real consequences. Accidental deployment = data flowing, compute charges, storage costs — hard to undo.

---

### Rule P2-2: Join Validation (MUST complete before Phase 2)

From Phase 1, you should have identified all join keys. **Validate before deployment:**

- [ ] For each join, test cardinality:
  ```sql
  SELECT COUNT(DISTINCT join_key) FROM table_a;
  SELECT COUNT(DISTINCT join_key) FROM table_b;
  -- Counts should match (within 5%)
  ```

- [ ] Check column types are identical:
  ```bash
  tdx describe <db>.table_a
  tdx describe <db>.table_b
  # join_key column type must match
  ```

- [ ] If cardinality mismatch, determine pre-aggregation strategy:
  ```sql
  -- Pre-aggregate many-side BEFORE join
  SELECT SUM(agg.total) FROM table_a a
  JOIN (
    SELECT join_key, SUM(amount) AS total FROM table_b GROUP BY join_key
  ) agg ON a.join_key = agg.join_key
  ```

**Why:** Invalid joins cause silent data inflation. Phase 2 is the right place to test and bake in the pre-aggregation.

---

### Rule P2-3: Time Column Must Be Present & Non-Nullable

- [ ] Confirmed from Phase 1: time column identified
- [ ] Time column must be non-nullable in source table
- [ ] If nullable, flag as data quality issue — may require fallback logic
- [ ] Workflow uses this column for incremental loading

**Example (Phase 2 workflow):**
```yaml
scheduling:
  frequency: daily
  time_column: event_time  # MUST be non-nullable
  backfill_start: 2026-01-01
```

---

### Rule P2-4: SINK Table Naming Convention

SINK table names MUST follow this pattern:

**`<project_slug>_sink_<metric_group>`**

Where:
- `project_slug` = lowercase, hyphens, derived from project name (e.g., "sales-dashboard" → "sales_dashboard")
- `metric_group` = lowercase, hyphens, logical grouping (e.g., "revenue", "pipeline", "customers")

**Examples:**
- `sales_dashboard_sink_revenue`
- `sales_dashboard_sink_pipeline`
- `sales_dashboard_sink_customers`

**Why:** Phase 3 dashboard queries reference these exact names. Misnamed tables = no data in dashboard.

---

### Rule P2-5: Workflow SQL Must Use Pre-Aggregation

Every Phase 2 workflow SQL query MUST pre-aggregate metrics:

```sql
-- ✅ RIGHT: GROUP BY dimensions that dashboard actually uses
SELECT 
  DATE(event_time) AS date,
  region,
  product_category,
  SUM(revenue) AS total_revenue,
  COUNT(DISTINCT customer_id) AS unique_customers
FROM source_events
GROUP BY DATE(event_time), region, product_category
ORDER BY date DESC, region, product_category
```

**NOT:**
```sql
-- ❌ WRONG: returns raw events (defeats Phase 2 purpose)
SELECT * FROM source_events
WHERE event_time >= CURRENT_DATE - INTERVAL 7 DAY
```

---

### Rule P2-6: Incremental Load Strategy

Phase 2 workflows should be **incremental** (not full refresh):

- [ ] Partition on time (date or day)
- [ ] Query only new/updated data since last run
- [ ] Append to SINK table (or replace day partition)

**Pattern:**
```yaml
# Daily incremental load
td:
  database: <db>

+aggregate_metrics:
  td>: queries/aggregate_metrics.sql
  parameters:
    start_time: ${session.last_attempt_time}
    end_time: ${moment().format('YYYY-MM-DD HH:mm:ss')}
  
  # Write to SINK table (incremental)
  insert_into: <project_slug>_sink_metrics
```

**Why:** Incremental loads are cheaper and faster than full refreshes.

---

### Rule P2-7: TD Account Access — Verify Before Deployment

- [ ] User can run `tdx auth show` (confirms valid auth)
- [ ] User has CREATE TABLE permission
- [ ] User has access to the target database

**Verification commands:**
```bash
# Test auth
tdx auth show

# Test database access
tdx databases

# Test CREATE TABLE permission
tdx query "CREATE TABLE IF NOT EXISTS test_table (id INT)" --database <target_db>
```

**If any of these fail, STOP and ask user to run `tdx auth setup`.**

---

### Rule P2-8: Workflow SQL Must Be Tested Before Deployment

Before deploying workflow, execute the aggregation query manually:

- [ ] Run: `tdx query < workflow_query.sql`
- [ ] Verify: Results are aggregated (not raw data)
- [ ] Verify: Row count is reasonable
- [ ] Spot-check: Numbers match expectations from Phase 1

**Example test:**
```sql
-- Test the aggregation query
SELECT 
  DATE(event_time) AS date,
  region,
  SUM(revenue) AS total_revenue,
  COUNT(DISTINCT customer_id) AS unique_customers
FROM source_events
WHERE event_time >= CURRENT_DATE - INTERVAL 7 DAY
GROUP BY DATE(event_time), region
ORDER BY date DESC, region
LIMIT 100;

-- Spot-check results:
-- - Dates are recent? ✓
-- - Regions are expected? ✓
-- - Revenue numbers look reasonable? ✓
```

---

### Rule P2-9: State.md Append

Append Phase 2 results to state.md (created in Phase 1):

```yaml
---

## Phase 2 — Workflow Deployment
**Date:** [date]
**Status:** ✅ Complete

### SINK Tables Created

- **Table:** [project_slug]_sink_[group_1]
  - Rows: [N]
  - Columns: [list]
  - First populated: [timestamp]
  
- **Table:** [project_slug]_sink_[group_2]
  - Rows: [N]
  - Columns: [list]
  - First populated: [timestamp]

### Workflow Deployed

- **Workflow Name:** [name]
- **Schedule:** [frequency, time, timezone]
- **Duration:** ~[Xm] per run
- **Last Run:** [timestamp]
- **Status:** ✓ Running successfully

### Validation Results

- [ ] Join cardinality validated
- [ ] Time column confirmed non-nullable
- [ ] SQL query tested manually (results verified)
- [ ] First run completed successfully
- [ ] SINK tables populated with expected data

---

## Next Action

1. ✓ Phase 2 complete
2. SINK tables ready for Phase 3 queries
3. Jump to Phase 3 (`../phase-3/INSTRUCTIONS.md`)
```

---

## Phase 2 Pre-Flight Checklist

**Before starting Phase 2, verify:**

### Prerequisites from Phase 1
- [ ] Promotion Score ≥ 4 (or Score 3 and user chose Phase 2)
- [ ] state.md exists from Phase 1
- [ ] All Phase 1 requirements + data findings documented in state.md

### TD Account Setup
- [ ] `tdx auth show` returns valid profile
- [ ] User has CREATE TABLE permission
- [ ] Target database is accessible

### Workflow Planning
- [ ] Aggregation SQL logic planned (pre-aggregation identified)
- [ ] Metric groups defined (dimensions for GROUP BY)
- [ ] Refresh schedule decided (daily/weekly/custom)
- [ ] Cost estimate provided

### User Approval
- [ ] User approved SINK table creation (approval template completed)
- [ ] User approved schedule and cost impact

---

## Phase 2 Decision Tree

```
START Phase 2: Workflow Deployment
    ↓
Is Promotion Score ≥ 4 (or user approved override)?
    YES ↓  → Continue
    NO  ↓  → ERROR: Should not be in Phase 2
    ↓
Verify prerequisites from Phase 1 in state.md
    ↓
Validate join cardinality (Rule P2-2)
    ↓
Plan aggregation logic (GROUP BY dimensions)
    ↓
Define SINK table names (Rule P2-4)
    ↓
Write and test aggregation SQL (Rule P2-8)
    ↓
Present cost/scope approval to user (Rule P2-1)
    ↓
User approves?
    YES ↓ → Deploy workflow
    NO  ↓ → Gather feedback, adjust, re-present
    ↓
Deploy workflow + create SINK tables
    ↓
Run first test cycle (manual trigger)
    ↓
Verify: SINK tables populated with expected data
    ↓
Append Phase 2 results to state.md
    ↓
END Phase 2 → Jump to Phase 3
```

---

## Common Phase 2 Blocks

### "The join has a cardinality mismatch"
**Response:**
> "This means one table has more unique keys than the other. We have two options: (1) Pre-aggregate the many-side table first (common fix), or (2) Use a different join key if available. Let me test the pre-aggregation approach."

### "The query is too slow"
**Response:**
> "Let's optimize: (1) Add partition pruning using td_time_range(), (2) Reduce the date range for testing, or (3) Pre-aggregate further before the join. Which would help?"

### "I'm not sure what the schedule should be"
**Response:**
> "How often do you need the dashboard to refresh? Common options: (1) Daily at 2 AM (most customers), (2) Weekly, (3) After business hours if data updates then. What fits your workflow?"

### "How much will this cost?"
**Response:**
> "Rough estimate based on query volume: ~$X per month. Workflow runs [frequency], and each run queries [N GB] of data. You can adjust the schedule to reduce cost if needed."

---

## Phase Progression Gate (CANNOT Skip)

**⚠️ CRITICAL: Before proceeding to Phase 3, verify:**

- [ ] Phase 1 marked COMPLETE in state.md (promotion score documented)
- [ ] Promotion score is 4-6 (or user explicitly chose Phase 2)
- [ ] All SINK tables created (listed in state.md)
- [ ] Workflow deployed and first run completed successfully
- [ ] Workflow runs without errors (check logs)
- [ ] SINK tables populated with expected data
- [ ] state.md appended with Phase 2 results:
  - SINK table names (all listed)
  - Workflow name and schedule
  - First run results
  - Validation checklist completed
  - "Next Action" pointer set to Phase 3

**If ANY item is missing or failed:**
> "Cannot proceed to Phase 3. Phase 2 incomplete or failed.
> Missing/Failed: [specific item]
> Please complete or debug before moving forward."

**Only after all items verified:**
- Append "Phase 2 COMPLETE" to state.md
- Proceed to Phase 3

---

## After Phase 2 Completes

**If workflow deploys successfully:**
```
✓ Phase 2 Complete → Jump to Phase 3
  Read: ../phase-3/INSTRUCTIONS.md
  Then: ../phase-3/SKILL.md
```

**If workflow fails:**
```
✗ Phase 2 Blocked → Troubleshoot
  Check:
  1. Is auth still valid? (tdx auth show)
  2. Do SINK tables exist? (tdx tables <db>)
  3. Did the first run complete? (tdx wf show <workflow_name>)
  
  If unresolved: Return to user with error details
```

---

**Version:** 1.0.0
**Last Updated:** 22 July 2026
**Load Order:** 1.2 (read after Phase 1 INSTRUCTIONS.md)
