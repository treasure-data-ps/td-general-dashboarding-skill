# Phase 3: Query & Filter Design Patterns

**Synthesized from production implementation (retail-dashboard-test-large).**

These 12 patterns should be applied mechanically in every Phase 3 build to avoid rework.

---

## 1. Filter Architecture — Two-Tier Pattern (REQUIRED)

All Phase 3 dashboards must implement **two tiers of filtering**:

### Tier 1: Global Date Range (F.start, F.end)
- **Source:** `sink_overview_kpis` (daily granularity, no dimension columns)
- **Behavior:** Filters all KPI cards and trend charts globally
- **State object:** `F.start`, `F.end` (date strings YYYY-MM-DD)

### Tier 2: Per-Tab Dimension Filters (FS.<tab>.<dimension>)
- **Source:** Distinct values from each tab's SINK table(s)
- **Behavior:** Filters only KPI cards/trend charts on the active tab
- **State objects:** `FS.<tab>.category`, `FS.<tab>.payment`, etc. (one per dimension)

**Implementation rule:**
- When NO dimension filter is active: Use data from overview (daily precision, all-time data)
- When ANY dimension filter is active: Switch to monthly+dimension data (see Data Shape Plan below)

**Why:** Date filtering requires daily or finer grain (impossible in pre-aggregated dimension data). Dimension filtering requires separation from date aggregation to avoid silent overpromising.

---

## 2. Data Shape Plan — Three Required Shapes (NOT TWO)

Every Phase 3 must generate exactly three data shapes. If you generate only two, the third will be discovered as a gap mid-project.

| Shape | Grain | Example | Use Case |
|-------|-------|---------|----------|
| **S1: Daily, no dims** | Daily, all dates | `sink_overview_kpis` | Date-filtered KPIs when no dim filter active |
| **S2: All-time, with dims** | Single aggregate row per dimension combo | `SELECT category, SUM(revenue) FROM sink GROUP BY category` | Breakdowns when no date filter needed |
| **S3: Monthly, with dims** ⭐ | Monthly grain per dimension combo | `SELECT DATE_TRUNC('month', date), category, SUM(revenue)... GROUP BY 1, 2` | **Respond to BOTH date + dimension filters** |

**Critical:** Shape S3 is what was missing in the original implementation. Without it, KPI cards couldn't respond to both filters simultaneously.

### Query Generation Rule

For each tab with dimensions:
- **Q1–Q3:** Overview + all-time dimension breakdowns (always generated)
- **Q4–Q6 (Shape S3):** Monthly+dimension queries (ONE PER TAB, MANDATORY)
  - Use same GROUP BY dimensions as that tab
  - Include `DATE_TRUNC('month', date)` or `SUBSTR(date, 1, 7)` as first column
  - Sort by date ASC to ensure latest months are complete
  - Set `LIMIT = 2 × expected_rows` (see Cardinality section below)

**Code pattern:**
```javascript
const queries = await Promise.all([
  // S1: Overview (all dates, no dims)
  query1_overview(),
  
  // S2: All-time breakdowns (with dims)
  query2_breakdown_all_time(),
  query3_breakdown_all_time(),
  
  // S3: MONTHLY + DIMS (REQUIRED) ⭐
  query4_monthly_with_dims(),  // Sales tab: monthly by category
  query5_monthly_with_dims(),  // Web tab: monthly by device
  query6_monthly_with_dims(),  // Customers tab: monthly by tier
]);
```

---

## 3. Monthly Query Cardinality — Pre-Calculate & Set LIMIT (REQUIRED)

### Cardinality Estimation Formula

For each monthly+dimension query:
```
Expected rows ≈ months × distinct_D1 × distinct_D2 × ... × distinct_DN

Example: Sales tab (category, payment, status)
  61 months × 12 categories × 4 payment types × 3 statuses = 8,784 rows (acceptable)

Example: Customers tab (tier, channel, income, gender)
  61 months × 5 tiers × 7 channels × 5 income × 3 gender = 32,175 rows (TOO HIGH)
  → Solution: Split into two tables (item 11 below)
```

### LIMIT Setting Rule

**Set `LIMIT` to at least `2 × expected_rows`** as a safety margin.

```sql
-- Correct: Conservative limit prevents silent truncation
SELECT SUBSTR(date, 1, 7) as month, category, SUM(revenue)
FROM sink_sales_monthly
WHERE td_time_range(...)
GROUP BY 1, 2
ORDER BY month ASC
LIMIT 20000  -- expected: ~8,784 rows; set to 2× = 17,568 (rounded up to 20K)
```

### Silent Truncation Warning

Add a post-query check in `generate-data.js`:

```javascript
const result = await query(sql, LIMIT);
if (result.rows.length === LIMIT) {
  console.warn(`⚠️  WARNING: Query returned exactly LIMIT (${LIMIT}) rows — may be truncated.`);
  console.warn(`⚠️  Most recent months may be missing. Increase LIMIT or simplify dimensions.`);
}
```

### Red Flags in state.md

When committing Phase 3, document:
- [ ] All monthly+dimension queries have calculated expected row count
- [ ] LIMIT set to ≥ 2× expected rows
- [ ] No query returned exactly LIMIT rows (no silent truncation)
- [ ] Actual row counts recorded for each query in Phase 3 section

---

## 4. Cross-Filter Pattern for Breakdown Charts (REQUIRED)

Breakdown charts must exclude their own dimension from active filters. Otherwise, selecting a value collapses the chart to a single bar.

### Wrong Pattern (Initial Implementation)
```javascript
// Category chart filtered by all active filters including category
function renderCategoryChart(data, filters) {
  const filtered = data.filter(row => 
    (!filters.category || row.category === filters.category) &&  // ❌ Collapses chart
    (!filters.payment || row.payment === filters.payment) &&
    (!filters.status || row.status === filters.status)
  );
  // Result: Selecting category=Electronics shows only 1 bar
}
```

### Correct Pattern (Cross-Filter)
```javascript
// Category chart filtered by payment + status, but NOT category
function renderCategoryChart(data, filters, chartDimension) {
  const filtered = data.filter(row => 
    (!filters.payment || row.payment === filters.payment) &&  // ✓ Active
    (!filters.status || row.status === filters.status) &&      // ✓ Active
    (!filters.category || row.category === filters.category)  // ✓ Optional (only if specifically requested)
  );
  // Result: Chart shows full category distribution given payment/status filters
}
```

### Implementation Rule

For every breakdown chart on every tab:
```
Chart dimension D → Filter by (date range + all active dims EXCEPT D)
```

This lets users see "how does D distribute given the other active filters?"

---

## 5. Breakdown Charts Must Always Respond to Date Filter (REQUIRED)

### The Problem

In the original implementation, when no dimension filter was active, breakdown charts used all-time aggregated data (e.g., `RAW.sales_dim`, pre-grouped by category only, no date column). These charts didn't respond to date range changes — only KPI cards did.

### The Fix

**Rule:** Breakdown charts should always query monthly+dimension data (Shape S3), filtered by the current date range.

```javascript
// WRONG: Different data source per filter state
function renderCategoryChart(data, filters) {
  if (filters.active_dims.length > 0) {
    // Use monthly data
    const filtered = monthlyData.filter(row => row.month >= filters.start && row.month <= filters.end);
  } else {
    // Use all-time all-category aggregate (no date column) ❌
    const filtered = allTimeData;
  }
}

// CORRECT: Consistent data source
function renderCategoryChart(data, filters) {
  // Always use monthly+dimension data (Shape S3)
  const filtered = monthlyData.filter(row => 
    row.month >= filters.start &&
    row.month <= filters.end
  );
  // Chart always responds to date filter
}
```

**Why:** Monthly grain is appropriate for breakdown charts regardless of filter state. There's no "precision requirement" that would justify switching to daily data.

---

## 6. Non-Additive Metrics — Handoff Rule (REQUIRED in Phase 2 → Phase 3)

### The Problem

Pre-aggregated SINK columns containing `COUNT DISTINCT` or `APPROX_DISTINCT` (e.g., `unique_sessions`, `unique_customers`) cannot be summed across rows without overcounting.

**Example:** `unique_sessions` aggregated per (date, device_type, traffic_source):
- A session viewing both mobile AND desktop appears in TWO rows
- `SUM(unique_sessions)` overcounts by ~2× (one session counted twice)

### The Solution (Phase 2 Responsibility)

For EVERY `COUNT DISTINCT` or `APPROX_DISTINCT` column in any SINK table, Phase 2 must explicitly document:

**Format for state.md (Phase 2 section):**
```yaml
## Non-Additive Metrics Handoff

sink_web_daily.unique_sessions:
  Type: APPROX_DISTINCT(session_id) per (date, device_type, source, event_type)
  Safe to SUM? NO — same session appears in multiple rows
  Phase 3 handling: Use from sink_overview_kpis (daily unique, no dims)
  
sink_marketing_daily.unique_customers:
  Type: APPROX_DISTINCT(customer_id) per (date, campaign, event_type, client)
  Safe to SUM? NO — same customer appears across campaigns
  Phase 3 handling: Query directly from source: 
    SELECT APPROX_DISTINCT(customer_id) FROM source
    WHERE date BETWEEN ? AND ?
```

### Phase 3 Implementation

When building KPI cards for non-additive metrics:

**Option A: Use Overview (if available)**
```javascript
// ✅ CORRECT: Use pre-deduplicated overview value
const uniqueCustomers = overviewData[0].unique_customers;  // Already correct, daily level
```

**Option B: Query Source Directly (if not in overview)**
```javascript
// ✅ CORRECT: Query distinct at build time
const result = await tdx_query(`
  SELECT APPROX_DISTINCT(customer_id) as unique_customers
  FROM retail_large_dataset.email_events
  WHERE date BETWEEN '${start}' AND '${end}'
`);
```

**Option C: Fallback to Additive Component**
```javascript
// ✅ CORRECT: Show additive metric instead with annotation
renderKPI({
  title: 'Email Opens',
  value: summedOpenCount,
  note: '(non-filterable by segment; based on event count, not unique customers)'
});
```

---

## 7. Payload Budget — Pre-Calculate Before Generating (REQUIRED)

### The Problem

In the retail implementation, monthly+dimension queries (S3) added ~1.2 MB to the payload (~524 KB → ~1,760 KB), discovered only AFTER build.

### Pre-Build Estimation

**Before generating Q4–Q6 (monthly+dimension queries), calculate:**

```
Estimated monthly data size = (expected_rows from item 3) × 40 bytes/row

Example:
  Sales: 8,784 rows × 40 = ~351 KB
  Web: 12,000 rows × 40 = ~480 KB
  Customers: 6,000 rows × 40 = ~240 KB
  Total new data: ~1,071 KB
  
  Current payload: 524 KB
  Projected total: 524 + 1,071 = 1,595 KB (acceptable < 2 MB)
```

### Budget Breakdown

Document this in state.md at Phase 3 start:

```yaml
## Payload Budget Breakdown

Projected:
  - sink_overview_kpis: ~180 KB
  - Monthly+dimension queries (S3): ~1,200 KB (calculate per tab)
  - All-time breakdowns (S2): ~50 KB
  - HTML/JS boilerplate: ~80 KB
  - Total: ~1,510 KB (acceptable < 2 MB)

Hard limits:
  - 2 MB: Browser load & memory ceiling
  - If > 1.5 MB: Consider splitting into separate dashboards
```

### Action Trigger

- **If projected > 1.5 MB:** Flag and require user acknowledgement
- **If projected > 2 MB:** STOP — split dashboard or reduce dimensions (item 11)

---

## 8. Known Limitation Annotation — .widget-note Pattern (REQUIRED)

### The Problem

When data is intentionally limited (all-time only, non-date-filterable, approximation), there was no standard way to surface this.

### The Solution

**HTML template includes `.widget-note` CSS class:**

```html
<style>
.widget-note {
  font-size: 0.85em;
  color: #666;
  margin-top: 8px;
  padding: 8px;
  background: #f5f5f5;
  border-left: 3px solid #ff9800;
}
</style>

<div id="income-chart"></div>
<div class="widget-note">
  ⓘ Income/gender data shown at all-time aggregate level
  (not date-filterable due to cardinality constraints).
</div>
```

### Rule: Annotate at Generation Time

For any widget where data is intentionally limited:
1. Generate the chart/KPI
2. Immediately render a `.widget-note` below it
3. Explain plainly: "what's limited?", "why?", "what data IS shown?"

**Examples:**
- "Monthly grain (not daily precision) due to dimension cardinality"
- "Date filter not applied; income brackets derived at all-time level"
- "Approximation; exact distinct count would require separate query"
- "Only last 90 days; full historical data available on request"

---

## 9. State.md Phase 3 Block — Append at Build Time (REQUIRED)

### The Problem

state.md accurately captured Phases 1 and 2 but was not updated to reflect Phase 3 iterations (new queries added, payload grew, limitations identified).

### The Solution

The Phase 3 rendering step should append a structured block to state.md, covering:

```yaml
## Phase 3 — Build Interactive Dashboard
**Date:** 2026-07-23
**Status:** ✅ Complete

### Query Plan (All Shapes)

**Shape S1: Daily, no dimensions (Overview)**
- Q1: sink_overview_kpis (1,827 rows, ~180 KB)

**Shape S2: All-time, with dimensions (Breakdowns)**
- Q2: sales by category (45 rows, ~2 KB)
- Q3: web by device (6 rows, ~0.5 KB)

**Shape S3: Monthly, with dimensions (Responsive to date + dim filters)**
- Q4: sales_monthly_dims (8,784 rows, ~351 KB)
- Q5: web_monthly_dims (12,000 rows, ~480 KB)
- Q6: customers_monthly_dims (6,000 rows, ~240 KB)

### Payload
- Total size: 1,512 KB (11 MB reduction from full history via SINK aggregation)
- Compression: ~60% when gzipped

### Filter Architecture

**Tier 1: Global date range** (F.start, F.end)
- Source: Q1 (sink_overview_kpis)
- Affects: All KPI cards, trend chart

**Tier 2: Per-tab dimension filters** (FS.<tab>.<dimension>)
- Tab Sales: FS.sales.category, FS.sales.payment, FS.sales.status
- Tab Web: FS.web.device, FS.web.source, FS.web.event_type
- Tab Customers: FS.customers.tier, FS.customers.channel
- Affects: KPI cards + breakdowns on active tab

**Cross-filter rule applied:** Each breakdown chart excludes its own dimension from active filters.

### Known Limitations
- Income/gender data: All-time aggregate only (monthly with 4 dims would exceed cardinality budget)
- Email funnel: Uses source query (not SINK) for accuracy on COUNT DISTINCT
- Mobile revenue: Last 90 days only (request histor separately if needed)

### Data Validation
- ✅ 5 KPIs spot-checked against source (±0.1% tolerance)
- ✅ All filters tested: date + dimension independence, cross-filter combinations
- ✅ Performance: Q4–Q6 queries < 3s; render < 2s

### Next Action
→ Ready for Phase 4 (automation/agent) or Phase 5 (handoff docs), or close and share dashboard.html
```

---

## 10. Special Characters — Use Unicode Escapes (REQUIRED in Templates)

### The Problem

The ÷ character in KPI sub-labels caused the Edit tool to fail silently (encoding mismatch).

### The Solution

**Rule:** Exclusively use Unicode escape sequences for non-ASCII in JavaScript strings.

```javascript
// ❌ WRONG: Direct non-ASCII character
sub: 'total opens ÷ sends'

// ✅ CORRECT: Unicode escape sequence
sub: 'total opens ÷ sends'
```

### Common Escapes

| Character | Escape | Example |
|-----------|--------|---------|
| ÷ | `÷` | `'revenue ÷ customers'` |
| → | `→` | `'Q1 → Q2 trend'` |
| — | `—` | `'period — all time'` |
| × | `×` | `'items × price'` |
| ≈ | `≈` | `'approx ≈ exact'` |
| & | `&` | `'sales & returns'` |

---

## 11. Dimension Combination Limit — Present Options When Over-Budget (REQUIRED)

### The Problem

The Customers tab needed monthly data for 4 dimensions (tier, channel, income, gender). All 4 would produce ~32,000 rows — exceeding payload budget. This trade-off was discovered too late and led to silent compromises.

### The Solution

**Before generating monthly+dimension queries, calculate cardinality (item 3). If > ~5,000 rows:**

Present explicit options to the user:

```
AskUserQuestion:
  header: "Dimension combination budget"
  question: |
    Customers monthly query would produce ~32,000 rows (61 months × 5 tiers × 7 channels × 5 income × 3 gender).
    
    This exceeds the ideal cardinality limit (~5,000 rows). Choose an approach:
    
    A) Drop lowest-priority dimensions (income + gender stay all-time, tier+channel get monthly)
    B) Separate tables: tier+channel → monthly (date-filterable); income+gender → all-time
    C) Collapse to fewer tiers (income: 5 tiers → 3; gender data removed)
    D) Accept the large table (test performance in Phase 3; may need optimization)
    
  options:
    - label: "Option A: Drop income/gender from monthly"
      description: "Tier+channel respond to date filter; income/gender always all-time"
    - label: "Option B: Split into 2 tables"
      description: "Highest-priority dims (tier+channel) get monthly; others stay all-time"
    - label: "Option C: Collapse tiers"
      description: "Simplify income/gender to reduce dimension cardinality"
    - label: "Option D: Include all (test later)"
      description: "Accept ~32K rows; optimize in Phase 3 if needed"
```

**Document chosen option in state.md:**
```yaml
## Dimension Combination Trade-off
Customers tab: Selected Option B (split tables)
  - cust_monthly (monthly): tier, channel → date-filterable
  - cust_demographic (all-time): income, gender → NOT date-filterable
  Reason: Stay within 1.5 MB payload budget
  Result: Income/gender charts annotated with .widget-note (see item 8)
```

---

## 12. sink_overview_kpis Limitation — State This Explicitly (REQUIRED)

### The Pattern

`sink_overview_kpis` was designed as the single source for date-filtered KPIs across all tabs. But this assumption breaks the moment a dimension filter is active.

### The Rule

**Explicitly document two separate code paths from the start:**

```yaml
## Data Sources for KPIs

**Path 1: No dimension filter active**
  - Source: sink_overview_kpis (daily grain, all-time data)
  - Data shape: One row per date, aggregated across all dimensions
  - Use case: Global date filtering only
  - KPI cards: Use columns from overview directly
  
**Path 2: Any dimension filter active**
  - Source: Monthly+dimension SINK table for active tab (Shape S3)
  - Data shape: One row per (month, dim1, dim2, ...)
  - Use case: Date + dimension filtering together
  - KPI cards: GROUP BY activated dimensions, aggregate across remaining dims
  
These are TWO SEPARATE QUERY PATHS designed to coexist.
Overview KPIs are NOT expected to cover all dashboard needs when filters are active.
```

### Implementation Pattern

```javascript
// CORRECT: Explicit code path selection
function renderTabKPIs(tab, filters) {
  if (Object.keys(filters.dimensions).length === 0) {
    // Path 1: Use overview (daily precision, all-time)
    const kpis = calculateFromOverview(overview_data);
  } else {
    // Path 2: Use monthly+dims table (monthly precision, current filter context)
    const kpis = calculateFromMonthlyDims(monthly_dims_data[tab], filters);
  }
  return kpis;
}
```

### Document in state.md

```yaml
## KPI Sources Summary
- Overview path: sink_overview_kpis (date-only queries)
- Tab path: Per-tab monthly+dimension query (when filters active)
- Non-additive metrics: Source tables queried at build time (email_events, etc.)
```

---

## Checklist: Apply All 12 Patterns

Before handing off Phase 3 to user approval:

- [ ] **1. Filter Architecture:** Two-tier pattern (global date + per-tab dims) fully wired
- [ ] **2. Data Shapes:** All 3 shapes generated (daily-nodim, all-time-dim, monthly-dim)
- [ ] **3. Monthly Cardinality:** Expected rows calculated; LIMIT set to 2×; no silent truncation warnings
- [ ] **4. Cross-Filter:** Breakdown charts exclude own dimension from active filters
- [ ] **5. Breakdown Date Filter:** All breakdown charts respond to date range (use Shape S3)
- [ ] **6. Non-Additive:** All COUNT DISTINCT columns documented + handled (overview/source query/fallback)
- [ ] **7. Payload Budget:** Pre-calculated and documented before build; no surprises
- [ ] **8. Widget Notes:** All limited widgets annotated with .widget-note pattern
- [ ] **9. State.md Phase 3 Block:** Query plan, payload, filter architecture, limitations appended
- [ ] **10. Unicode Escapes:** All non-ASCII in JS strings use `\uXXXX` format
- [ ] **11. Dimension Limits:** If cardinality > 5K rows, options presented + chosen approach documented
- [ ] **12. Overview Limitation:** Documented that overview covers date-only path; separate paths when filters active

---

**Version:** 1.0.0 (Lite, Production-Validated)
**Last Updated:** 23 July 2026
**Source:** retail-dashboard-test-large implementation review
**Author:** FDE Team
