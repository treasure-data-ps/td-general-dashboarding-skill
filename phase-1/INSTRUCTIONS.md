---
name: phase-1-instructions
description: Phase 1 specific instructions for requirements gathering and data discovery
priority: HIGH
load_order: 1.1
---

# Phase 1 Instructions — Requirements & Data Discovery

**Read this BEFORE `./SKILL.md`.**

---

## Phase 1 Goal

Gather business requirements (Stage A), validate them against real data (Stage B), calculate promotion score (0-6), decide Workflow vs Non-Workflow path, and get user approval — **all in one session**.

**Deliverable:** `./<project-slug>/state.md` containing approved requirements, confirmed data findings, promotion score, and path decision.

---

## Quick Checklist (Quick Reference)

**Stage A: Session Setup + Requirements**
- [ ] Project slug chosen (short, kebab-case)
- [ ] Business goal / dashboard purpose (1 sentence)
- [ ] Platform target? (Treasure Work / Treasure AI Studio)
- [ ] Data source type? (Raw/Transactional / Pre-aggregated / Snapshot / Mixed)
- [ ] Use AskUserQuestion for every question (never plain text)
- [ ] **1a:** Dashboard purpose & success criteria
- [ ] **1b:** Metrics/KPIs (what to measure)
- [ ] **1c-1d:** Dimensions + Filters + Layout
- [ ] **⚠️ CRITICAL 1e-1f:** Date range, timezone, refresh/freshness
- [ ] **1g:** Historical depth
- [ ] **1h:** Sharing, access, target users (if customer-facing)
- [ ] **1j:** Alerts/thresholds (if applicable)
- [ ] **1o:** Exclusion rules & data quality

**Optional (only if relevant):**
- [ ] **1k:** Mobile/responsive design
- [ ] **1l:** Compliance/data sensitivity
- [ ] **1m:** Data source complexity / canonical ID
- [ ] **1n:** Drill-down depth

**Stage B: Data Discovery & Validation**
- [ ] Database selected, tables confirmed to exist (`tdx databases`, `tdx describe`)
- [ ] Time column identified (business-event datetime vs TD insert-time `time` column)
- [ ] Metrics inferred/validated against live queries
- [ ] Dimensions validated (cardinality, join fan-out checked)
- [ ] Data Quality Gate run (all checks passed)
- [ ] **HTML Client data size validated** (total rows, estimated file size, feasibility check)

**Scoring & Path Decision**
- [ ] Promotion score calculated (Q1 + Q2 + Q3 = ___ / 6)
- [ ] Path decision made (Score 0-2 → Phase 3, Score 3 → user chooses, Score 4-6 → Phase 2 then Phase 3)

**Completion**
- [ ] state.md created with all findings
- [ ] User approval obtained
- [ ] Ready to proceed to Phase 2 or Phase 3

---

---

## ✅ Before You Proceed: Required Reads

**Before executing Phase 1 Stage A, read these reference files:**
1. **`../references/guardrails-lite.md`** — Cross-phase guardrails (re-read on every engagement)
2. **`./phase-1/references/requirements-questionnaire-patterns.md`** — Requirement gathering patterns (if exists)

These files establish the baseline for all Stage A questions and ensure consistent scoping.

---

## Phase 1 Specific Rules (In Addition to Universal Rules)

### Rule P1-1: Requirements Must Be Validated Against Real Data

- [ ] **Stage A:** Gather business requirements (no data validation yet)
  - KPIs: What are they measuring?
  - Dimensions: How should metrics be sliced?
  - Filters: What does the audience need to filter on?
  - Audience: Who will use this?

- [ ] **Stage B:** Validate requirements against real data
  - Run `tdx describe <db>.<table>` for each source table
  - Verify columns exist (exact spelling, case-sensitive)
  - Test joins with sample queries
  - Verify time columns are present and non-nullable
  - Never accept "I assume the data exists" — verify with `tdx query`

**Why:** Many requirements fail at validation. Better to discover this in Phase 1 than Phase 3.

**Example validation query:**
```sql
-- Verify KPI column exists
SELECT SUM(revenue) FROM sales_table LIMIT 5;

-- Verify time column
SELECT MIN(created_at), MAX(created_at) FROM sales_table;

-- Verify dimension exists
SELECT DISTINCT country FROM sales_table LIMIT 10;
```

---

### Rule P1-2: Promotion Score Calculation Is Deterministic

**Score formula (0-6):**
- **0-2 (Low complexity):** Simple metrics, few tables, no joins, daily or less frequent refresh
  - **Path:** Phase 1 → Phase 3 (skip Phase 2)
  
- **3 (Optional):** Medium complexity, user's choice
  - **Path:** User decides Phase 2 (workflow) or Phase 3 (direct build)
  
- **4-6 (High complexity):** Many tables, complex joins, sub-hourly refresh, aggregation required
  - **Path:** Phase 1 → Phase 2 → Phase 3

**Scoring breakdown:**
- +1 if 2+ tables to join
- +1 if join is 1-to-many (fan-out risk)
- +1 if refresh frequency < daily
- +1 if aggregation required (GROUP BY)
- +1 if audience > 5 people (concurrency risk)

**Rule:** Calculate score before asking path decision. The score determines the path.

---

### Rule P1-2b: Dashboard Design Specification (ENFORCEMENT)

**⚠️ CRITICAL: Phase 1 Stage A must capture complete dashboard design before moving to Stage B.**

During Step 1c (Dimensions, Filters, Layout), capture:

1. **Tab Structure:**
   - How many tabs? (single or multiple)
   - What is each tab called?
   - Which metrics go on each tab?

2. **Global vs Tab Filters:**
   - Which filters apply to ALL tabs? (date, region, etc.)
   - Which filters apply to specific tabs only?

3. **Visualization Type per Metric:**
   - KPI card? Line chart? Bar chart? Pie chart? Table?
   - What dimensions slice each metric?

4. **Widget Details:**
   - Widget name
   - Chart type
   - Metric and calculation (SUM/COUNT/AVG/etc)
   - Dimensions (what to slice by)
   - Filters applied
   - Business definition

5. **Interactivity:**
   - Can users click to drill down?
   - Can users export data?
   - Which widgets update together?

**See:** `./references/dashboard-design-questions.md` for question sets and examples

**Capture in state.md:**
```yaml
### Dashboard Design Specification

**Tabs:** [number and names]

**Global Filters:**
  - [filter 1]: [type]
  - [filter 2]: [type]

**Widgets:**
  - [Tab Name] - [Widget Name]
    Chart Type: [KPI/Line/Bar/Pie/Table/...]
    Metric: [metric name]
    Calculation: [formula]
    Dimensions: [sliced by]
    Definition: [business meaning]
```

**Why:** This prevents rework in Phase 3. Users approve the complete design in Phase 1, so Phase 3 is execution, not design iteration.

---

### Rule P1-3: Join Key Validation (Phase 1→2 gate)

If requirements include 2+ tables:

- [ ] Identify join keys
- [ ] Test with sample queries:
  ```sql
  SELECT COUNT(DISTINCT customer_id) FROM customers;
  SELECT COUNT(DISTINCT customer_id) FROM orders;
  -- Counts should match (within 5%)
  ```
- [ ] If counts don't match, flag it for Phase 2 planning (pre-aggregation needed)

**Why:** Undetected join issues cause Phase 2 rework.

---

### Rule P1-4: Time Column Validation (Phase 1 discovery)

- [ ] Identify the business event or insert timestamp column
- [ ] Verify it's non-nullable: `tdx describe <db>.<table>`
- [ ] If nullable, flag as data quality issue — inform user
- [ ] Document the column name in state.md

**Example:**
```yaml
## Time Column Validation

Source Table: sales_events
- Column identified: `event_time` (business event time)
- Nullable: No ✓
- Format: TIMESTAMP
- Range: 2020-01-01 to 2026-07-22
```

---

### Rule P1-5: Special Case Paths (Sisense .dash or Treasure Insights API)

If user provides:
- **`.dash` file** → Follow `.dash` Special Case path (skip to dashboard prefilling)
- **Datamodel name/OID** → Follow Treasure Insights API Special Case path (fetch schema via API)
- **Both** → Follow Combined Resources path (validate consistency)
- **Neither** → Follow normal Stage A/B path

**Do NOT mix paths.** Once on a special path, stay on it.

**Why:** Special case paths have their own prefilling logic. Mixing causes duplicate questions.

---

### Rule P1-6: Negative Confirmation (Ask What NOT to Include)

At the end of requirements (Stage A), explicitly ask:

> "Is there anything you specifically do NOT want in this dashboard?"

Common answers:
- "Don't include revenue from EU (privacy)"
- "Don't show customer names (PII)"
- "Don't include last 24h (data quality lag)"

Document as "Exclusions" in state.md.

---

### Rule P1-7: Dimensions Validation

For each dimension user requests:

- [ ] Verify it exists in data: `SELECT DISTINCT <dimension> FROM <table> LIMIT 10`
- [ ] Check cardinality: `SELECT COUNT(DISTINCT <dimension>) FROM <table>`
- [ ] If cardinality > 1000, warn user (may be too granular for dashboard filters)

**Example:**
```yaml
## Dimensions Validation

- Dimension: country
  - Exists: ✓ (SELECT DISTINCT country ... returned 195 rows)
  - Cardinality: 195 (acceptable for filter)
  
- Dimension: customer_id
  - Exists: ✓
  - Cardinality: 1,200,000 (⚠️ WARNING: Very high cardinality, not suitable for filter)
  - Recommendation: Use name instead or provide search autocomplete
```

---

### Rule P1-7b: HTML Client Data Size Validation (ENFORCEMENT)

**⚠️ CRITICAL: Phase 1 must validate that dashboard design is feasible for HTML Client limits.**

HTML Client stores data INLINE in the HTML file (not fetched from API). This creates hard limits:

| Metric | Safe | Warning | Breaking |
|--------|------|---------|----------|
| HTML File Size | < 10 MB | 10-50 MB | > 50 MB |
| Data Rows (total) | < 100K | 100K-500K | > 500K |
| Load Time | < 2s | 2-5s | > 5s |

**During Stage B (data discovery), for EACH widget AND filter:**

1. **Analyze filters first (high-cardinality blocks):**
   ```bash
   # Check cardinality of each filter column
   tdx query "SELECT COUNT(DISTINCT region) FROM orders"
   tdx query "SELECT COUNT(DISTINCT status) FROM orders"
   tdx query "SELECT COUNT(DISTINCT customer_id) FROM orders"
   
   # Safe: < 50 values (use dropdown)
   # Warning: 50-500 values (consider search)
   # Danger: > 500 values (use search autocomplete, don't embed)
   # Breaks: > 10,000 values (definitely not in data.json)
   ```

2. **Estimate widget rows:**
   ```
   If aggregated (GROUP BY region, date, status):
     90 days × 6 regions × 4 statuses = 2,160 rows ✅
   
   If raw events (no aggregation):
     2.1M rows ❌ TOO LARGE
   ```

3. **Run actual query to test:**
   ```bash
   tdx query --output json < query.sql > data.json
   ls -lh data.json
   # If < 5 MB: ✅ Safe
   # If 5-20 MB: ⚠️ Warning
   # If > 20 MB: ❌ Likely breaks
   ```

4. **Identify problems:**
   - High-cardinality filters (customer_id, product_id, user_id)?
   - Too many dimensions (combinations explode)?
   - Raw events instead of aggregated?
   - Too much history (3+ years daily data)?
   - Too many dimensions in GROUP BY?
   - Too many tabs with independent data?

5. **Solutions:**
   - ✅ Remove high-cardinality filters (use search instead)
   - ✅ Aggregate more (weekly instead of daily, top 10 categories)
   - ✅ Filter scope (last 90 days instead of 3 years)
   - ✅ Reduce dimensions (remove least-used slicing dimension)
   - ✅ Paginate tables (show top 1,000 rows only)
   - ✅ Limit to 3-5 low-cardinality filters max
   - ✅ Limit to 3-5 tabs per dashboard
   - ❌ If none work: Recommend Phase 4 (backend API) instead

**Capture in state.md:**
```yaml
### HTML Client Data Validation

Widget | Type | Estimated Rows | Estimated Size | Status
--------|------|--------|--------|--------
[Widget Name] | [Type] | [N rows] | [X KB/MB] | [✅/⚠️/❌]

Total Estimated Size: [X MB]
Feasibility: [✅ Safe / ⚠️ Warning / ❌ Not Feasible]

If warning/failing, action taken:
  - [Solution applied: e.g., "Reduced to weekly instead of daily"]
  - [User approved: Yes/No]
```

**See:** `./references/html-client-data-limits.md` for detailed estimation and solutions

**Why:** Discovering data size issues in Phase 1 prevents Phase 3 build failures or slow dashboards.

---

### Rule P1-8: Metrics Validation

For each metric user requests:

- [ ] Verify column exists and is numeric
- [ ] Check for missing/null values
- [ ] Validate with a sample aggregation

**Example queries:**
```sql
-- Verify column exists and is numeric
SELECT SUM(revenue) FROM sales_table LIMIT 5;

-- Check for nulls
SELECT COUNT(*) total, COUNT(revenue) non_null, 
       COUNT(*) - COUNT(revenue) AS nulls 
FROM sales_table;

-- Quick aggregation test
SELECT SUM(revenue) FROM sales_table WHERE DATE(created_at) = '2026-07-22';
```

---

### Rule P1-9: Filter Edge Case Question (REQUIRED IN STAGE A)

**⚠️ CRITICAL: Ask explicitly: "What should KPIs show when a filter makes accurate computation impossible?"**

**Add this question to Stage A Step 1c (Filters & Layout):**

```
For each metric, decide edge case behavior:

Q: "Total Customers" filtered by "Product Category" 
   = Can't compute accurately (same customer buys multiple categories)
   Options:
     a) Show COUNT(DISTINCT customer) per category (separate count per category)
     b) Show "—" (dash, not available)
     c) Show estimate with "*" (estimate, confidence note)
   User decides: (a/b/c)

Q: "Revenue" filtered by "Date" 
   = Can compute accurately (revenue IS date-filterable)
   Options:
     a) Show filtered revenue (correct answer)
   User decides: (a)

Q: "Unique Visitors" filtered by "Page"
   = Can't compute accurately (same visitor visits multiple pages)
   Options:
     a) Show visitors per page (separate count per page, not unique across)
     b) Show "—"
     c) Show all-time visitors (unfiltered, maybe with warning)
   User decides: (a/b/c)
```

**Capture in state.md:**
```yaml
### Filter Edge Case Behaviors (if filter makes computation impossible)

Metric: Total Customers
- If filtered by Category: Show "—" (can't count distinct across categories)
- If filtered by Region: Show COUNT(DISTINCT customer_id) per region ✓

Metric: Unique Sessions
- If filtered by Page: Show sessions per page OR show "—"
- If filtered by Date: Show COUNT(DISTINCT session_id) per date ✓

Metric: Revenue
- All filters: Show filtered revenue (always computable)
```

**Why:** Prevents Phase 3 discovery of incomputable metrics. Better to decide in Phase 1.

---

### Rule P1-9a: Filter Interaction Rules (REQUIRED BEFORE PHASE 3)

**⚠️ CRITICAL: Specify what happens to metrics when filters are applied. Ask explicitly in Stage A Step 1c.**

For each **KPI/metric** that will have filters applied, specify:

1. **Date filter applied:** Does the metric recalculate? Show `—` if impossible to aggregate to single value?
2. **Category/dimension filter applied:** Does the metric update? Show adjusted value, `—`, or estimate?
3. **Custom segment filter applied:** How does this affect the KPI? Show for segment only, show total + segment, or exclude?

**Examples:**

| Metric | Date Filter | Category Filter | Segment Filter |
|--------|------------|-----------------|---|
| **Total Revenue** | ✅ Recalculate | ✅ Show by category | ✅ Show for segment |
| **Unique Customers** | ⚠️ Show `—` (can't aggregate) | ⚠️ Show by category | ✅ Show for segment |
| **Avg Order Value** | ✅ Recalculate | ✅ Show by category | ✅ Show for segment |

**Questions to ask the user:**

1. "For [Metric X], if someone filters to [specific category], should the value update or show `—`?"
2. "If a metric can't be accurately filtered (e.g., Unique Customers by date), should we show an estimate, hide it, or show a warning?"
3. "When multiple filters are applied together, should metrics show the intersection, or revert to unfiltered?"

**Capture in state.md:**
```yaml
### Filter Interaction Rules

**Metric: [Metric Name]**
- Date filter behavior: [Recalculate / Show — / Show estimate]
- Category filter behavior: [Show by category / Show — / Show estimate]
- Segment filter behavior: [Show for segment / Show total + segment]
- Multi-filter behavior: [Intersection / Revert]

**Metric: [Next Metric Name]**
- [Same structure]
```

**Why:** Prevents Phase 3 confusion about what filters should do to each metric. Discovered late = rework cycles.

---

### Rule P1-10: State.md Creation

At end of Phase 1, create `state.md` with this structure:

```yaml
# Dashboard Project: [project-slug]

## Phase 1 — Requirements & Data Discovery
**Date:** [date]
**Status:** ✅ Complete

### Business Requirements (Stage A)

**Primary Goal:** [1-2 sentence summary]

**KPIs:**
- [metric 1]: [definition]
- [metric 2]: [definition]

**Dimensions:**
- [dimension 1]: [cardinality]
- [dimension 2]: [cardinality]

**Filters:**
- [filter 1]: [options]
- [filter 2]: [options]

**Audience:** [names/roles]

**Exclusions:** [what NOT to include]

### Data Findings (Stage B)

**Source Tables:**
- [table 1]: [columns], [row count], [last updated]
- [table 2]: [columns], [row count], [last updated]

**Time Column:**
- Table: [table name]
- Column: [column name]
- Format: [TIMESTAMP/DATE/etc]
- Nullable: [YES/NO]

**Join Keys (if 2+ tables):**
- [table 1] ↔ [table 2]: [join key], [cardinality validated: YES/NO]

**Validation Results:**
- All columns verified: ✓
- All joins tested: ✓
- Time column present: ✓

### Promotion Score

**Final Score:** [X/6]

**Breakdown:**
- Multiple tables: [+1 if yes]
- 1-to-many joins: [+1 if yes]
- Sub-daily refresh: [+1 if yes]
- Aggregation required: [+1 if yes]
- High concurrency (5+ users): [+1 if yes]

### Dashboard Plan Summary (Displayed to User)

**Dashboard Filters (applied to all tabs):**
- [Filter 1]: [dimension/field], type: [dropdown/date-picker/search]
- [Filter 2]: [dimension/field], type: [dropdown/date-picker/search]

**Tabs:**
- Tab 1: [name] — Widgets: [count]
- Tab 2: [name] — Widgets: [count]

**Tab 1: [Tab Name]**

| Widget | Type | Metric | Dimensions | Filters | Calculation |
|--------|------|--------|-----------|---------|-------------|
| [Widget 1 Name] | [Chart Type: KPI/Bar/Line/Table] | [Metric] | [Sliced by] | [Filter 1, Filter 2] | [Formula/Aggregation] |
| [Widget 2 Name] | [Chart Type] | [Metric] | [Sliced by] | [Filters] | [Formula/Aggregation] |

**Tab 2: [Tab Name]**

| Widget | Type | Metric | Dimensions | Filters | Calculation |
|--------|------|--------|-----------|---------|-------------|
| [Widget Name] | [Chart Type] | [Metric] | [Sliced by] | [Filters] | [Formula/Aggregation] |

**Data Source:**
- Database: [database]
- Tables: [table1, table2, ...]
- Date Range: [range]
- Data Freshness: [last updated]
- Rendering: HTML Client (standalone, portable)

**HTML Client Data Size Validation:**

**Filters Analysis:**
- Global Filters: [list, e.g., "Date Range, Region (6), Status (4)"]
- Tab-Specific Filters: [list or "None"]
- Cardinality Check: [✅ All < 50 values OR ⚠️ [filter name] has [N] values OR ❌ [filter name] too high]
- Filter Data Size: [X KB]

**Widget Data:**
| Widget | Type | Rows | Size | Filters Applied | Status |
|--------|------|------|------|-----------------|--------|
| [Widget 1] | [Type] | [N] | [X KB] | [filter names] | ✅ |
| [Widget 2] | [Type] | [N] | [X KB] | [filter names] | ✅ |

**Total Size Breakdown:**
- Widget Data: [X KB]
- Filter Options: [X KB]
- HTML/JS/CSS: [~50 KB]
- JSON Overhead: [X KB]
- **Final HTML File:** [X MB]

- **Load Time (estimated):** [X seconds]
- **Tab Complexity:** [N tabs - low/medium/high]
- **Filter Complexity:** [N filters, max cardinality [N] - low/medium/high]
- **Feasibility:** ✅ Safe for HTML Client OR ⚠️ Warning OR ❌ Not feasible
- **Actions Taken (if any):** [e.g., "Removed high-cardinality customer filter", "Reduced to weekly aggregation"]

### Path Decision

**Recommended Path:** Phase 1 → [Phase 2 → Phase 3 OR Phase 3]

**Justification:** Score X indicates [workflow/non-workflow] path

**User Approval:**
- Plan reviewed: [YES / NO]
- Plan approved: [YES / NO]
- Path confirmed: [Phase 2 / Phase 3]
- Confirmation timestamp: [YYYY-MM-DD HH:MM:SS]

---

### Rule P1-11: Path Confirmation (ENFORCEMENT)

**⚠️ CRITICAL: CANNOT proceed to Phase 2 or Phase 3 without explicit user confirmation of the path.**

After calculating promotion score and recommending a path:

**If Score 0-2 (Non-Workflow recommended):**
- Present: "Recommended: Non-Workflow Path (Phase 3 only)"
- Wait for: `YES, BUILD DASHBOARD NOW (Non-Workflow)` OR `NO, SET UP WORKFLOW INSTEAD`
- If user confirms Non-Workflow: Proceed to Phase 3
- If user chooses Workflow instead: Proceed to Phase 2

**If Score 4-6 (Workflow recommended):**
- Present: "Recommended: Workflow Path (Phase 2 → Phase 3)"
- Wait for: `YES, PROCEED WITH WORKFLOW PATH` OR `NO, SKIP WORKFLOW AND BUILD NOW`
- If user confirms Workflow: Proceed to Phase 2
- If user declines Workflow: Proceed to Phase 3 instead

**If Score 3 (User choice):**
- Ask directly: "Which path? Phase 2 (Workflow) or Phase 3 (Non-Workflow)?"
- User answers: →  **Immediately proceed** (no second approval needed)
- If Phase 2 chosen: "Starting Phase 2 now..."
- If Phase 3 chosen: "Starting Phase 3 now..."

**CRITICAL: Auto-start after confirmation (Rule 0: Phase Auto-Advance)**
- Score 0-2 confirmed → "Starting Phase 3 now..."
- Score 4-6 confirmed → "Starting Phase 2 now..."
- User choice confirmed → Start chosen phase immediately

**See:** `./references/stage-b-path-routing.md` → "Approval Gates" section for full templates

**Why:** Path decisions have real cost/time/complexity implications. User must explicitly confirm they understand the tradeoffs. But once confirmed, start immediately (don't wait).

---

### Rule P1-12: Dashboard Plan Summary Display (ENFORCEMENT)

**⚠️ CRITICAL: Before asking for final approval, show the user a dashboard plan summary.**

**When:** End of Stage B (before path confirmation in Rule P1-10)

**What to show:**
```
📊 Dashboard Plan — <project-slug>

Source Database:   <database_name>
Tables:
  • <table_1> (<row_count>, updated <freshness>) — <FACT | DIMENSION>
  • <table_2> (<row_count>, updated <freshness>) — <FACT | DIMENSION>

Metrics confirmed:
  • <Metric 1>: <definition with sample value>
  • <Metric 2>: <definition with sample value>
  • <Metric 3>: <definition with sample value>

Dimensions confirmed:
  • <Dimension 1>: <N> values (<examples>)
  • <Dimension 2>: <N> values (<examples>)

Date range:        <default_range>
Data freshness:    <last updated>
Rendering:         HTML Client (standalone, portable)
Promotion score:   <X>/6

Conflicts detected: <None | list conflicts>
```

**Example:**
```
📊 Dashboard Plan — sales-performance

=== DASHBOARD FILTERS (Global - applies to all tabs) ===

Filters:
  • Date Range: date-picker (Last 90 days default)
  • Region: dropdown (North America, Europe, APAC, LATAM, EMEA, APJC)
  • Order Status: multi-select (Completed, Pending, Cancelled, Returned)

=== TABS ===

Tab 1: "Revenue Overview" — 4 widgets
Tab 2: "Orders & Customers" — 3 widgets
Tab 3: "Trends" — 2 widgets

=== TAB 1: REVENUE OVERVIEW ===

Widget 1: Total Revenue (KPI)
  • Chart Type: KPI card with trend
  • Metric: Total Revenue
  • Calculation: SUM(amount) WHERE status NOT IN ('cancelled', 'refunded')
  • Sample Value: $4,859,839
  • Filters Applied: Date Range, Region, Order Status
  • Definition: Total revenue from all completed orders (excludes cancelled/refunded)

Widget 2: Revenue by Region (Bar Chart)
  • Chart Type: Horizontal Bar Chart
  • Metric: Total Revenue
  • Dimensions: Region (x-axis shows region, y-axis shows revenue)
  • Calculation: SUM(amount) BY region WHERE status NOT IN ('cancelled', 'refunded')
  • Sample Data: North America ($2.1M), Europe ($1.5M), APAC ($900K), ...
  • Filters Applied: Date Range, Order Status (not Region — Region is displayed)
  • Definition: Revenue breakdown by geographic region

Widget 3: Revenue Trend (Line Chart)
  • Chart Type: Line Chart
  • Metric: Daily Revenue
  • Dimensions: Date (x-axis), Revenue (y-axis)
  • Calculation: SUM(amount) BY DATE(order_date) WHERE status NOT IN ('cancelled', 'refunded')
  • Period: Last 90 days (daily data points)
  • Filters Applied: Date Range, Region, Order Status
  • Definition: Daily revenue trend over time to spot seasonal patterns

Widget 4: Revenue by Order Status (Pie Chart)
  • Chart Type: Pie Chart
  • Metric: Revenue by Status
  • Dimensions: Order Status (slices)
  • Calculation: SUM(amount) BY status
  • Sample Data: Completed (85%, $4.1M), Cancelled (3%, $150K), Returned (2%, $80K), Pending (10%, $530K)
  • Filters Applied: Date Range, Region (not Status — Status is displayed)
  • Definition: Revenue distribution across order statuses

=== TAB 2: ORDERS & CUSTOMERS ===

Widget 1: Order Count (KPI)
  • Chart Type: KPI card
  • Metric: Order Count
  • Calculation: COUNT(*) FROM orders
  • Sample Value: 2,156,492
  • Filters Applied: Date Range, Region, Order Status
  • Definition: Total number of orders

Widget 2: Active Customers (KPI)
  • Chart Type: KPI card
  • Metric: Unique Customers
  • Calculation: COUNT(DISTINCT customer_id) WHERE purchase_date >= DATE_SUB(NOW(), 90 DAY)
  • Sample Value: 1,234,567
  • Filters Applied: Date Range, Region
  • Definition: Customers with at least 1 purchase in last 90 days

Widget 3: Orders by Status (Table)
  • Chart Type: Data Table
  • Metrics: Count, Revenue, Avg Order Value
  • Dimensions: Order Status (rows)
  • Calculation: COUNT(*), SUM(amount), AVG(amount) BY status
  • Sample Data:
    | Status     | Count   | Revenue  | Avg Value |
    | Completed  | 1.8M    | $4.1M    | $2,281    |
    | Pending    | 215K    | $530K    | $2,465    |
    | Cancelled  | 107K    | $150K    | $1,402    |
    | Returned   | 34K     | $80K     | $2,353    |
  • Filters Applied: Date Range, Region (not Status — Status is displayed)
  • Definition: Detailed breakdown of orders by status with metrics

=== TAB 3: TRENDS ===

Widget 1: Orders per Day (Line Chart)
  • Chart Type: Line Chart
  • Metric: Daily Order Count
  • Dimensions: Date (x-axis), Order Count (y-axis)
  • Calculation: COUNT(*) BY DATE(order_date)
  • Period: Last 90 days
  • Filters Applied: Date Range, Region, Order Status
  • Definition: Daily order volume trend

Widget 2: Customer Acquisition (Cumulative Line)
  • Chart Type: Cumulative Area Chart
  • Metric: New Customers
  • Dimensions: Date (x-axis), Cumulative Count (y-axis)
  • Calculation: COUNT(DISTINCT customer_id) CUMULATIVE BY DATE(first_purchase_date)
  • Period: Last 90 days
  • Filters Applied: Date Range, Region
  • Definition: Cumulative new customer count over time

=== DATA SOURCE ===

Source Database:   analytics_prod
Tables:
  • orders (2.1M rows, updated 2h ago) — FACT
  • customers (156K rows, updated 1d ago) — DIMENSION
  • regions (6 rows, static) — DIMENSION

Promotion Score:   5/6
Date Range:        Last 90 days
Data Freshness:    Updated 2 hours ago
Rendering:         HTML Client (standalone, portable)

=== HTML CLIENT DATA VALIDATION ===

**Filters Analysis:**
- Global Filters: Date Range, Region (6), Order Status (4)
- Filter data size: ~300 bytes
- Cardinality check: All filters < 50 values ✅

**Widget Data Size Estimates (Phase 1 Stage B):**

| Widget | Type | Rows | Size | Filter Impact | Status |
|--------|------|------|------|---------------|--------|
| Total Revenue (KPI) | KPI | 1 | < 1 KB | Uses all global filters | ✅ Safe |
| Revenue by Region (Bar) | Bar | 6 | < 5 KB | Region not in filter (shown in chart) | ✅ Safe |
| Revenue Trend (Line) | Line | 90 | < 30 KB | Uses all global filters | ✅ Safe |
| Revenue by Status (Pie) | Pie | 4 | < 2 KB | Status not in filter (shown in chart) | ✅ Safe |
| Order Count (KPI) | KPI | 1 | < 1 KB | Uses all global filters | ✅ Safe |
| Active Customers (KPI) | KPI | 1 | < 1 KB | Uses all global filters | ✅ Safe |
| Orders by Status (Table) | Table | 4 | < 5 KB | Status not in filter (shown in table) | ✅ Safe |
| Orders per Day (Line) | Line | 90 | < 30 KB | Uses all global filters | ✅ Safe |
| Customer Acq (Cumulative) | Area | 90 | < 30 KB | Uses global filters (Region, Date) | ✅ Safe |

**Total Size Breakdown:**
- Widget data: ~105 KB
- Filter dropdown options: ~300 bytes
- HTML/JS/CSS: ~50 KB
- JSON overhead (+20%): ~31 KB
- **Final HTML File:** ~186 KB

**Feasibility:** ✅ SAFE FOR HTML CLIENT
- Load time (estimated): < 1 second
- Browser compatibility: All modern browsers
- Performance: Smooth interactions, all filters responsive
- Filter complexity: Low (3 filters, max 6 values each)
- Tab complexity: Low (3 tabs)

**Actions Taken:** None needed — data and filters well within safe limits

Conflicts Detected: None
```

**Then ask:**
> "Does this dashboard plan look correct? If yes, we'll move to the next step. If no, what needs to change?"

**Why:** Users need to see a clear summary of what they're approving — metrics, dimensions, data freshness, and path — before committing to Phase 2 or Phase 3.

---

## Next Action

1. ✓ Phase 1 complete
2. User to review state.md and approve
3. If Score ≤2: Jump to Phase 3 (`../phase-3/INSTRUCTIONS.md`)
4. If Score 3: User decides Phase 2 or Phase 3
5. If Score ≥4: Jump to Phase 2 (`../phase-2/INSTRUCTIONS.md`)
```

---

## Phase 1 Pre-Flight Checklist

**Before starting Phase 1 Stage A, verify:**

### Engagement Setup
- [ ] Engagement type confirmed: new project or resuming existing?
- [ ] If resuming: project slug confirmed, state.md located
- [ ] User understands Phase 1 goal (1-2 hours, produces state.md)

### Data Access
- [ ] TD account accessible (`tdx auth show` works)
- [ ] User can access required databases (`tdx databases` list returned)
- [ ] At least one source table accessible via `tdx query`

### Requirements Clarity
- [ ] User has basic idea of KPIs (even if rough)
- [ ] User has basic idea of audience (who will use this?)
- [ ] User willing to validate requirements against real data (Phase 1 Stage B)

---

## Phase 1 Decision Tree

```
START Phase 1 Stage A: Gather Requirements
    ↓
Ask: KPIs? Dimensions? Filters? Audience?
    ↓
START Phase 1 Stage B: Validate Against Data
    ↓
For each source table:
    - Run: tdx describe <db>.<table>
    - Verify columns exist (exact spelling)
    - Test joins with sample queries
    ↓
Calculate Promotion Score (0-6)
    ↓
DECISION POINT:
    Score 0-2?  → Route to Phase 3 (skip Phase 2)
    Score 3?    → Ask user: Phase 2 (workflow) or Phase 3 (direct)?
    Score 4-6?  → Route to Phase 2 (workflow)
    ↓
Create state.md with requirements + findings
    ↓
Get user approval on state.md
    ↓
END Phase 1
```

---

## Common Phase 1 Blocks

### "I don't know my KPIs"
**Response:** 
> "That's OK. Let's start with a metric you want to track. Common examples: Revenue, Customer Count, Churn, Conversion Rate, NPS. What matters most to your business right now?"

### "The data might not exist"
**Response:**
> "Let's check! I'll run `tdx describe` on the table. If the column doesn't exist, we have two options: (1) find an alternative metric in the data, or (2) flag this as a data gap for your team to build."

### "I have 20 metrics I want"
**Response:**
> "Great. Let's prioritize by importance. Which 3-5 are must-haves? We can always add more later. (Phase 1 goal is to validate the core requirements, not build everything.)"

### "I'm not sure if the join will work"
**Response:**
> "Perfect. Let me test it. I'll run: `SELECT COUNT(DISTINCT customer_id) FROM customers` and `SELECT COUNT(DISTINCT customer_id) FROM orders`. If counts match (within 5%), the join is clean. If not, we have a fan-out issue that Phase 2 will handle."

---

## Phase Progression Gate (CANNOT Skip)

**⚠️ CRITICAL: Before proceeding to Phase 2 or Phase 3, verify:**

- [ ] All Stage A questions answered (1a-1o)
- [ ] All Stage B data validations complete (tables confirmed, columns verified, joins tested)
- [ ] **HTML Client data size validated** (all widgets checked, feasibility confirmed)
- [ ] Promotion Score calculated (0-6, with breakdown)
- [ ] Dashboard Plan Summary displayed to user (Rule P1-12) — includes data validation
- [ ] User reviewed and approved the plan (including data feasibility)
- [ ] Path Confirmation obtained (Rule P1-11)
- [ ] state.md created with ALL Phase 1 results:
  - Business requirements documented
  - Data findings documented
  - Time column identified
  - Join keys validated
  - Promotion score and justification
  - Dashboard plan summary captured
  - Path decision (Phase 2 or Phase 3) with user approval
  - "Next Action" pointer

**If ANY item is missing:**
> "Cannot proceed to next phase. Phase 1 incomplete.
> Missing: [specific item]
> Please complete before moving forward."

**Only after all items verified:**
- Append "Phase 1 COMPLETE" to state.md with timestamp
- Record: "Dashboard plan reviewed and approved by user"
- Move to next phase based on promotion score and user confirmation

---

## After Phase 1 Completes

**If Score 0-2:**
```
✓ Phase 1 Complete → Jump to Phase 3
  Read: ../phase-3/INSTRUCTIONS.md
  Then: ../phase-3/SKILL.md
```

**If Score 3:**
```
✓ Phase 1 Complete → User Decision
  Option A: Phase 2 (workflow for scheduled refresh)
    Read: ../phase-2/INSTRUCTIONS.md
  Option B: Phase 3 (direct build, no workflow)
    Read: ../phase-3/INSTRUCTIONS.md
```

**If Score 4-6:**
```
✓ Phase 1 Complete → Phase 2 Required
  Read: ../phase-2/INSTRUCTIONS.md
  Then: ../phase-2/SKILL.md
```

---

**Version:** 1.0.0
**Last Updated:** 22 July 2026
**Load Order:** 1.1 (read after master INSTRUCTIONS.md)
