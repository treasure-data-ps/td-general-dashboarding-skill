# HTML Client Data Limits & Validation (Phase 1)

**Goal:** Validate that the dashboard design is feasible for HTML Client before Phase 3 build.

**Critical Constraint:** HTML Client stores all data INLINE in the HTML file (not fetched from API). This creates hard limits on:
- Total data volume
- Number of rows per dataset
- File size (affects load time)
- Browser performance

---

## Data Limits & Breaking Points

| Metric | Safe | Warning | Breaking |
|--------|------|---------|----------|
| **HTML File Size** | < 10 MB | 10-50 MB | > 50 MB (likely breaks) |
| **Total Data Rows** | < 100K | 100K-500K | > 500K (breaks on load) |
| **Datasets per Dashboard** | < 5 | 5-10 | > 10 (slows significantly) |
| **Load Time** | < 2s | 2-5s | > 5s (UX failure) |

---

## Step 1: Estimate Data Volume for Each Widget

For each widget in dashboard design, calculate:

```yaml
Widget: [Widget Name]
Chart Type: [KPI/Line/Bar/Table/...]
Data Source: [table name]
Query: [SQL]

Expected Rows:
  - Pre-aggregated? [yes/no]
  - Time period: [X days]
  - Group by: [dimensions]
  - Distinct combinations: [estimate]
  
Sample Calculation:
  • If metrics by (Date, Region, Status):
    • Dates: 90 days
    • Regions: 6 values
    • Status: 4 values
    • Expected rows: 90 × 6 × 4 = 2,160 rows
    
  • If raw events (non-aggregated):
    • Events in date range: 2.1M
    • ❌ TOO LARGE for HTML Client
```

---

## Step 2: Calculate Total HTML File Size

**Formula:**
```
Total Size ≈ (Sum of all data rows) × (Avg bytes per row) + JavaScript + HTML markup

Avg bytes per row:
  • Numeric columns (int, float): 8 bytes
  • Date columns: 10 bytes
  • Text columns: ~50 bytes average
  • JSON overhead: +20% for structure

Example:
  Widget 1 (KPI): 1 row × 5 columns (4 numeric, 1 text) ≈ 100 bytes
  Widget 2 (Bar Chart): 2,160 rows × 8 columns (6 numeric, 2 text) ≈ 2,160 × 120 = 259 KB
  Widget 3 (Trend Line): 90 rows × 3 columns (2 numeric, 1 date) ≈ 90 × 50 = 4.5 KB
  
  Subtotal (data): ~264 KB
  JS/HTML/CSS: ~50 KB
  JSON wrapper: +20% = +53 KB
  
  ➜ Total: ~367 KB ✅ SAFE
```

**Better way: Test with real data:**
```bash
# Run the Phase 3 query to get actual row count
tdx query --output json < query.sql > data.json

# Check file size
ls -lh data.json
# If < 5 MB: ✅ Safe
# If 5-20 MB: ⚠️ Warning (monitor performance)
# If > 20 MB: ❌ May break
```

---

## Step 3: Identify Data Size Issues & Solutions

### Issue 1: Too Many Rows (Raw Events)

**Problem:** User wants to display raw events (e.g., all 2.1M orders in a table).

**Solution:**
- ❌ Not feasible for HTML Client
- ✅ Options:
  1. **Aggregate first:** GROUP BY region/date (reduce 2.1M → 2,160 rows)
  2. **Filter scope:** Show last 7 days instead of 90 (reduce proportionally)
  3. **Use pagination:** Show top 1,000 rows only in table
  4. **Move to Phase 4:** Build a backend API + real dashboard server (beyond HTML Client scope)

### Issue 2: Too Many Dimensions

**Problem:** Dashboard slices by (Date, Region, Status, Category, Channel) = explosion of combinations.

**Example:**
```
365 days × 6 regions × 4 statuses × 42 categories × 5 channels 
= 3,075,600 rows ❌ BREAKS
```

**Solution:**
- Reduce dimensions:
  1. Remove least-used dimension (e.g., drop Channel)
  2. Keep only top 10 categories instead of all 42
  3. Aggregate by month instead of day
  4. Split into multiple dashboards (one per region)

### Issue 3: High Time Resolution

**Problem:** Widget shows daily data for 3 years = 1,095 rows per series.

**If dashboard has 10 time-series widgets:**
- 1,095 rows × 10 widgets × 3 columns = 32,850 rows
- ~4 MB data
- ✅ Safe, but monitor

**Solution:** Aggregate to weekly or monthly if not needed daily.

---

## Step 4: Update Dashboard Plan with Data Validation

**Add to Phase 1 state.md:**

```yaml
### HTML Client Data Validation

**Estimated Data Volume:**

Widget | Type | Rows | Size | Status
--------|------|------|------|--------
Total Revenue (KPI) | KPI | 1 | < 1 KB | ✅ Safe
Revenue by Region (Bar) | Bar | 6 | < 10 KB | ✅ Safe
Revenue Trend (Line) | Line | 90 | < 50 KB | ✅ Safe
Orders by Status (Table) | Table | 4 | < 5 KB | ✅ Safe

**Total Estimated Size:**
- Data only: ~65 KB
- With HTML/JS/CSS: ~200 KB
- Final HTML file: ~250 KB

**Status:** ✅ SAFE FOR HTML CLIENT

---

OR (if problems found):

**Total Estimated Size:**
- Data only: ~45 MB
- Final HTML file: ~50+ MB

**Status:** ⚠️ WARNING - May be slow to load
**Recommended Action:** Reduce scope (remove raw events table, aggregate by month instead of day)

---

OR (if critical):

**Total Estimated Size:**
- Data only: ~150 MB
- Final HTML file: > 100 MB

**Status:** ❌ NOT FEASIBLE FOR HTML CLIENT
**Required Action:** 
1. Aggregate data more (weekly instead of daily)
2. Filter to smaller time window (last 30 days instead of 90)
3. Remove raw data table
4. OR: Recommend alternative approach (Phase 4 backend + API)
```

---

## Question to Ask User During Phase 1

**Add to Step 1c (after layout preferences):**

```
AskUserQuestion:
  header: "Data Volume & Performance"
  question: "How much data detail do you need? (affects file size)"
  multiSelect: false
  options:
    - label: "Summary & Aggregates (Recommended for HTML Client)"
      description: "KPI cards + trend charts + dimension summaries. Fast load, small file."
    - label: "Summary + Some Detail"
      description: "Aggregates + a table with top 1,000 rows. Moderate file size (~5-10 MB)."
    - label: "All Raw Detail"
      description: "Full dataset with no aggregation. May be slow or fail. Not recommended for HTML."
    - label: "Not sure — Let's estimate first"
      description: "I'll estimate based on your data. Then you decide."
```

---

## Validation Script (For Phase 1 Agent)

**During Stage B, run this check:**

```python
# Pseudo-code for estimating HTML file size

def estimate_html_size(widgets):
    total_rows = 0
    total_bytes = 0
    
    for widget in widgets:
        query_result = run_query(widget['query'])
        rows = len(query_result)
        cols = len(query_result[0])
        avg_col_bytes = 100  # conservative estimate
        
        widget_bytes = rows * cols * avg_col_bytes
        total_rows += rows
        total_bytes += widget_bytes
        
        print(f"{widget['name']}: {rows} rows, {widget_bytes/1024:.0f} KB")
    
    # Add overhead
    js_html_css = 50_000  # ~50 KB
    json_overhead = total_bytes * 0.2
    
    final_size = total_bytes + js_html_css + json_overhead
    
    print(f"\nTotal: {total_rows} rows, {final_size/1024/1024:.1f} MB")
    
    if final_size < 10_000_000:  # 10 MB
        print("✅ SAFE for HTML Client")
    elif final_size < 50_000_000:  # 50 MB
        print("⚠️  WARNING - Monitor performance")
    else:
        print("❌ NOT FEASIBLE - Reduce scope")
    
    return final_size
```

---

## Recommendations by Use Case

| Dashboard | Feasibility | Recommendation |
|-----------|-------------|-----------------|
| KPI dashboard (5 metrics, no history) | ✅ Always OK | Build with HTML Client |
| Weekly trend dashboard (52 rows per chart) | ✅ Always OK | Build with HTML Client |
| Monthly trend dashboard (36 rows per chart) | ✅ Always OK | Build with HTML Client |
| Daily trend 2 years (730 rows per chart) | ⚠️ OK if < 5 charts | Aggregate to weekly or use 1 year history |
| Raw transaction table (1M+ rows) | ❌ Never OK | Aggregate, paginate, or use backend API |
| Real-time dashboard (updates every minute) | ❌ Never OK for HTML | Use backend + API in Phase 4 (out of scope) |

---

**Version:** 1.0.0
**Last Updated:** 22 July 2026
