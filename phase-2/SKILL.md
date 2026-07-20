---
name: td-general-dashboarding-skill-phase-2
description: |
  INTERNAL — invoked by td-general-dashboarding-skill root skill only. Do not trigger directly.
  Phase 2: Deploy Dashboard Workflow (optional). Copy the locally embedded workflow templates, configure with project parameters, deploy to Treasure Data, and validate SINK tables.
  Use when: Routed here by the root td-general-dashboarding-skill SKILL.md when the Stage B path decision is Workflow (score 4-6, or skip_workflow = false/partial).
---

# Phase 2: Deploy Dashboard Workflow (Custom Dashboard Agent — Lite)

> **GUARDRAILS: Read `../references/guardrails-lite.md` before doing anything else in this session.**

**Phase Goal:** Convert approved Stage B queries into a scheduled Treasure Data Workflow that pre-aggregates metrics into workflow output tables (SINK tables) — making Phase 3 dashboard queries fast, fresh, and production-grade.

**Skip this phase if you don't need scheduled refresh** — go straight to Phase 3 (Build) and query source tables directly on demand.

**Skills used in this phase:** `workflow-skills:digdag` · `sql-skills:trino` · `sql-skills:trino-optimizer` · `sql-skills:time-filtering`

---

## ⚠️ Step 0: Extract Stage A/B Configuration (MANDATORY — 5 min)

**Do this FIRST, before any Phase 2 steps:**

Phase 2 needs 5 configuration values from Stage A/B. These were captured in Stage A Steps 1e+1f, 1g, 1q and stored in `state.md`.

**Action:**
1. Read `./<project_slug>/state.md` (local file)
2. Locate the "Session Setup" block (Stage A content — DO NOT EDIT)
3. Extract the fields listed in `./references/workflow-setup-configure.md` → **"Phase 2 Entry Requirements: Read From Stage A"** section
4. Proceed to Step 3a only after all 5 fields are extracted

**Why:** Phase 2 workflow optimization depends on knowing:
- **Historical window** — used in `td_time_range()` queries for partition pruning
- **Refresh schedule** — determines cron schedule in the `.dig` file
- **SINK database** — target for `create_table:` tasks
- **Workflow project name** — used in `.dig` file naming + `tdx wf` commands
- **Data volatility** — determines incremental processing pattern (append-only vs 1-day vs 7-day lookback)

**If any field is missing from Stage A/B:** Go back and complete Stage A/B first — do not proceed with Phase 2 on partial data.

---

→ **See `./deploy-workflow-guide.md`** for: pre-phase checklist, pre-deployment CRITICAL items, troubleshooting guide, deliverables, and next-phase routing.

**Steps 3a–3g:** See `./deploy-workflow-guide.md` step table. Key resources: `./references/workflow-setup-configure.md`, `workflow-deployment-validate.md`, `td-time-functions.md`, `testing-troubleshooting.md`.

---

## Custom Dashboard-Specific Phase 2 Considerations

### Copying the Workflow Template

The workflow template is embedded locally in this skill — no external repo access needed:

```bash
# Copy the workflow template into ./<project_slug>/workflows/
mkdir -p ./<project_slug>/workflows
cp -r ./references/workflow-templates/. ./<project_slug>/workflows/
```

Edit `./<project_slug>/workflows/input_params.yaml` with the confirmed values from `state.md` (source database/table, SINK database, date range, refresh mode). Then deploy directly:

```bash
tdx wf push ./<project_slug>/workflows --project <workflow_project_name>
tdx wf start <workflow_project_name> dashboard-workflow-launch
```

No branching, no PR review — a single push/start cycle per iteration.

### Step 3c: SINK Schema Design — Avoiding Multi-Dimension Double-Counting

⚠️ **When SINK tables grain by BOTH product/vehicle dimensions AND customer dimensions, customers with multiple products appear in multiple rows, causing SUM(customer_count) to over-count.**

**Pattern A: Separate SINK Tables (RECOMMENDED)**

If your dashboard needs both customer-level and vehicle-level analysis:
```sql
-- SINK Table 1: Customer KPIs (no vehicle dimension)
CREATE TABLE fact_customer_kpis AS
SELECT
  customer_id,
  segment,
  churn_risk,
  COUNT(DISTINCT vehicle_id) as num_vehicles,
  SUM(revenue) as total_revenue,
  COUNT(*) as total_transactions
FROM source_data
GROUP BY customer_id, segment, churn_risk;

-- SINK Table 2: Vehicle KPIs (vehicle dimension + customer)
CREATE TABLE fact_vehicle_kpis AS
SELECT
  DATE(date) as date_key,
  vehicle_make,
  vehicle_model,
  customer_segment,
  COUNT(DISTINCT customer_id) as unique_customers,  -- COUNT(DISTINCT), not SUM
  SUM(revenue) as total_revenue
FROM source_data
GROUP BY DATE(date), vehicle_make, vehicle_model, customer_segment;
```

Use fact_customer_kpis for "Total Unique Customers" KPI. Use fact_vehicle_kpis for "Revenue by Make/Model" chart.

**Pattern B: Pre-Computed Distinct Counts (In Same Table)**

If you must use a single SINK table with mixed dimensions:
```sql
CREATE TABLE fact_combined AS
SELECT
  DATE(date) as date_key,
  vehicle_make,
  customer_segment,
  COUNT(DISTINCT customer_id) as unique_customers,      -- NOT SUM(customer_count)!
  SUM(transaction_count) as total_transactions,
  SUM(revenue) as total_revenue
FROM source_data
GROUP BY DATE(date), vehicle_make, customer_segment;
```

**Phase 3 Impact:** The Phase 3 KPI queries will use `SELECT unique_customers FROM fact_combined` (safe) instead of `SELECT SUM(customer_count)` (would over-count).

---

→ **See `./references/workflow-deployment-validate.md`** for full deployment and SINK validation steps.

→ **See `./deploy-workflow-guide.md`** for: deliverables, end-of-phase checklist, quick reference, performance expectations, and next-phase routing.

→ **See `./CHECKLIST.md`** for a condensed Phase 2 decision checklist (setup, deploy, SINK validation gates).
---

**Version:** 1.0.0 (Lite)
**Last Updated:** 15 July 2026
**Author:** FDE Team
