# Phase 3: Implementation Checklist & Common Pitfalls

**Synthesized from production code inspection (retail-dashboard-test-large).**

These 10 items (13-22) are specific code patterns that prevent silent failures, regressions, and user confusion.

---

## 13. Spot-Check Date Window Must Match Phase 1 Baseline (REQUIRED)

### The Problem

The spot-check used a rolling 30-day window starting from `dateEnd - 30 days`:
```javascript
const last30 = RAW.overview.filter(r => r.d >= dateEnd.slice(0,7) + '-01').slice(-30);
// dateEnd.slice(0,7) + '-01' → '2026-07-01' (first of month)
// NOT dateEnd - 30 days
```

But Phase 1 baseline was June 23 – July 19 (specific 30-day window from requirements gathering). The rolling window produces different data → "mismatch" that isn't actually a bug → eroded trust in spot-check.

### The Fix

Spot-checks must use the EXACT same date window as Phase 1 confirmed baseline.

```javascript
// CORRECT: Compute window correctly
const d = new Date(dateEnd);
d.setDate(d.getDate() - 30);
const cutoff = d.toISOString().slice(0, 10);
const last30Days = RAW.overview.filter(r => r.d >= cutoff && r.d <= dateEnd);

// Verify window matches Phase 1
console.log(`Spot-check window: ${cutoff} to ${dateEnd}`);
console.log(`Phase 1 baseline window (from state.md): June 23 – July 19`);
if (cutoff !== '2026-06-23' || dateEnd !== '2026-07-19') {
  console.warn('⚠️ Spot-check window differs from Phase 1 baseline — expected value may not match');
}
```

---

## 14. Spot-Check Expected Values Must Not Be Hardcoded (REQUIRED)

### The Problem

Expected values were hardcoded:
```javascript
console.log(`Expected (Phase 1/2 confirmed): $47,268,917.24`);
```

After the daily 02:00 UTC workflow runs and new data loads, this value becomes stale → spot-check perpetually shows "mismatch" → user ignores it.

### The Solution (Option A): Use Stable All-Time Metrics

Use metrics that are monotonically increasing (only go up, never down):

```javascript
// ✅ CORRECT: All-time total (stable, only increases)
const totalAllTimeRevenue = RAW.overview
  .map(r => r.net_revenue)
  .reduce((a, b) => a + b, 0);
const expectedAllTimeRevenue = 1247365824.92; // From Phase 2 final validation

console.log(`  Today's total all-time revenue: $${totalAllTimeRevenue.toLocaleString()}`);
console.log(`  Expected (Phase 2 final): $${expectedAllTimeRevenue.toLocaleString()}`);
const pct_diff = ((totalAllTimeRevenue - expectedAllTimeRevenue) / expectedAllTimeRevenue * 100).toFixed(2);
console.log(`  Difference: ${pct_diff}% (acceptable if < 1%, new data loaded since Phase 2)`);
```

### The Solution (Option B): Store Baseline in state.md

Store confirmed values in state.md at Phase 3 completion:

```yaml
## Spot-Check Baselines
Date range used for Phase 1/2 validation: 2026-06-23 to 2026-07-19 (30 days)

KPI Baselines (from Phase 2 final validation):
  - Net Revenue: $47,268,917.24
  - Order Count: 3,189,231
  - Unique Customers: 1,247,365
  - Email Sends: 42,891,472

At Phase 3 build time, read these values and compare to current data:
  - If match exactly: ✅ No data loaded since Phase 2
  - If higher: ✅ New data loaded (expected); difference recorded
  - If lower or missing: ❌ Data error; stop and investigate
```

Then in generate-dashboard.js:
```javascript
const state = readStatemd(); // Parse Phase 3 Spot-Check Baselines section
const expectedRevenue = parseFloat(state['Net Revenue'].split('$')[1]);
// Use expectedRevenue for comparison
```

---

## 15. Hidden Tab Charts Render with Zero Width (Chart.js display:none Bug) (REQUIRED FIX)

### The Problem

On page load, `renderAll()` renders all tabs, but only the active tab is `display:block`. The others are `display:none`. Chart.js measures canvas width via `offsetWidth`, which returns **0 for hidden elements**. Result: charts on hidden tabs look broken when first switched to.

```javascript
// renderAll() on page load
renderAll(); // Renders Sales, Customers, Web, Marketing

// At this point:
// Sales tab: display:block → canvas width measured correctly ✓
// Customers tab: display:none → canvas offsetWidth = 0 ✗
// Web tab: display:none → canvas offsetWidth = 0 ✗
// Marketing tab: display:none → canvas offsetWidth = 0 ✗
```

### The Fix (Option A): Re-Render on Tab Switch

Only render charts when the tab becomes visible:

```javascript
function switchTab(name, btn) {
  // Show/hide panels
  document.querySelectorAll('.tab-panel').forEach(p => p.style.display = 'none');
  document.querySelector(`#tab-${name}`).style.display = 'block';
  
  // Set active button
  document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
  btn.classList.add('active');
  
  // RE-RENDER CHARTS FOR ACTIVE TAB (new)
  const ov = filteredOverview();
  if (name === 'sales')       renderSales(ov);
  else if (name === 'web')    renderWeb(ov);
  else if (name === 'customers') renderCustomers(ov);
  else if (name === 'marketing') renderMarketing(ov);
  // Sales is default on load (already rendered)
}

// On page load, render only the default (Sales) tab
document.addEventListener('DOMContentLoaded', () => {
  const ov = filteredOverview();
  renderSales(ov); // Only this one initially
});
```

### The Fix (Option B): Use visibility:hidden Instead of display:none

Preserves canvas dimensions while keeping elements visually hidden:

```css
.tab-panel {
  visibility: hidden;    /* Invisible but takes space */
  position: absolute;    /* Stack off-screen or overlap */
}
.tab-panel.active {
  visibility: visible;
}
```

**Trade-off:** Option B uses more memory (all charts exist in DOM), but avoids re-render cost. Option A is leaner but requires re-rendering on switch.

---

## 16. Dead Data in Payload — Remove Unused Columns (REQUIRED)

### The Problem

`unique_openers` was loaded from `sink_overview_kpis` into every overview row (~1,827 rows) but never referenced by any render function. The dashboard used `RAW.email_rate_a.unique_openers` (from Q12 source query) instead because the SINK column overcounted.

This adds ~18 KB of dead data to the payload and is misleading to future readers.

### The Fix

```javascript
// WRONG: Including dead column
const ov_data = query_results.map(r => ({
  d: r.date,
  rev: num(r.net_revenue, 0),
  ord: num(r.order_count, 0),
  uo: num(r.unique_openers, 0), // ❌ NEVER USED
  sends: num(r.email_sends, 0)
}));

// CORRECT: Exclude and document why
const ov_data = query_results.map(r => ({
  d: r.date,
  rev: num(r.net_revenue, 0),
  ord: num(r.order_count, 0),
  // NOTE: unique_openers from SINK overcounts (APPROX_DISTINCT per date/campaign/type);
  // Phase 3 uses RAW.email_rate_a.unique_openers (queried from source, see Q12)
  sends: num(r.email_sends, 0)
}));
```

**Rule:** If a SINK column is not used in Phase 3 rendering, exclude it from the payload mapping and document why in a comment.

---

## 17. Tab Filters Must Be Symmetric Across All Tabs (REQUIRED)

### The Problem

The Web Engagement tab had three dimension filters (device, traffic source, event type). The Marketing tab had only two (email client, campaign name) — missing event type even though `mkt_monthly` contains an `evt` column.

A user wanting to see "opens by email client for a specific event type" had no way to filter it.

### The Fix

Inventory all dimensions in every tab's SINK table and add filter dropdowns for all:

```javascript
// For each tab, list all dimension columns
const tabDimensions = {
  sales: ['category', 'payment', 'status'],
  web: ['device', 'source', 'event_type'],
  marketing: ['email_client', 'campaign', 'event_type'],  // ← ADD event_type
  customers: ['tier', 'channel', 'income_tier', 'gender']
};

// Generate filter dropdowns for all
Object.entries(tabDimensions).forEach(([tab, dims]) => {
  dims.forEach(dim => {
    // Create dropdown: FS.<tab>.<dim>
    createDimensionFilter(`FS.${tab}.${dim}`, dim);
  });
});
```

**Rule:** If a dimension column exists in the SINK table, it must have a filter dropdown.

---

## 18. Filter-Active Indicator on Tab Buttons (REQUIRED)

### The Problem

When a user set a dimension filter on the Sales tab, switched to Customers, there was no visual indication that Sales had an active filter. They might forget, come back, and be confused by the numbers.

### The Fix

Add a dot or badge to tab buttons when that tab has active filters:

```javascript
function updateTabIndicators() {
  const filterState = {
    sales: FS.sales.category !== 'all' || FS.sales.payment !== 'all' || FS.sales.status !== 'all',
    web: FS.web.device !== 'all' || FS.web.source !== 'all' || FS.web.event_type !== 'all',
    customers: FS.customers.tier !== 'all' || FS.customers.channel !== 'all',
    marketing: FS.marketing.email_client !== 'all' || FS.marketing.event_type !== 'all'
  };
  
  Object.entries(filterState).forEach(([tab, hasFilter]) => {
    const btn = document.querySelector(`[onclick*="${tab}"]`);
    btn.classList.toggle('tab-filtered', hasFilter);
  });
}

// Call after every filter change:
function applyTabFilter(tab, dim, value) {
  FS[tab][dim] = value;
  updateTabIndicators();  // Add this
  renderAll();
}
```

```css
.tab-btn.tab-filtered::after {
  content: '●';
  color: var(--accent-1);
  margin-left: 4px;
  font-size: 0.6em;
}
```

---

## 19. Table Row Limits Must Be Transparent (REQUIRED)

### The Problem

Q8 fetched the top 20 campaigns by send volume (`LIMIT 20`). If there were more campaigns, users only saw a subset with no indication. The campaign count showed text-search-filtered count, not the cap from LIMIT.

### The Fix

**Rule:** Every table query must have:
1. A named constant for the limit
2. A note in the table header showing the limit
3. An indication that it's a ranked subset

```javascript
const CAMPAIGN_TABLE_LIMIT = 20;

// In generate-dashboard.js:
const campaigns = await query(`
  SELECT campaign_name, SUM(sends) as total_sends
  FROM mkt_monthly
  GROUP BY campaign_name
  ORDER BY total_sends DESC
  LIMIT ${CAMPAIGN_TABLE_LIMIT}
`);

// In HTML:
const header = `<h3>Top Campaigns by Send Volume (showing top ${CAMPAIGN_TABLE_LIMIT} of ${totalCampaigns})</h3>`;
// Renders as: "Top Campaigns by Send Volume (showing top 20 of 147)"
```

---

## 20. SINK Schema Validation at Build Time (REQUIRED)

### The Problem

If a SINK column was renamed between Phase 2 and Phase 3 (e.g., `net_revenue` → `revenue_net`), all queries returned undefined values. The `num()` helper coerced undefined to 0 → silent all-zeros dashboard with no error.

This is particularly risky for incremental builds where Phase 2 workflow changes are deployed independently of Phase 3 rebuilds.

### The Fix

Add schema validation after Q1 runs and before RAW assembly:

```javascript
const requiredCols = {
  overview: ['date', 'order_count', 'net_revenue', 'email_sends', 'unique_customers'],
  sales_daily: ['category', 'payment', 'status', 'revenue', 'order_count'],
  web_daily: ['device', 'source', 'event_type', 'pageviews', 'sessions'],
  marketing_daily: ['email_client', 'campaign', 'event_type', 'sends', 'opens'],
  customers_daily: ['tier', 'channel', 'revenue', 'order_count']
};

// Validate after queries run
function validateSchemas(results) {
  Object.entries(requiredCols).forEach(([queryKey, cols]) => {
    const row = results[queryKey]?.[0] || {};
    const missing = cols.filter(c => !(c in row));
    if (missing.length) {
      throw new Error(
        `❌ SCHEMA MISMATCH in query '${queryKey}':\n` +
        `Missing columns: [${missing.join(', ')}]\n` +
        `Available columns: [${Object.keys(row).join(', ')}]\n` +
        `This likely means Phase 2 SINK schema changed. Update Phase 3 column names and rebuild.`
      );
    }
  });
}

// Call this before RAW assembly:
validateSchemas(allQueryResults);
const RAW = assembleData(allQueryResults); // Only reached if validation passes
```

**Result:** Build fails fast and loudly with a clear error message.

---

## 21. Chart.js Library Must Be Inlined for True Offline Capability (RECOMMENDED)

### The Problem

dashboard.html loads Chart.js from a CDN:
```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.js"></script>
```

The skill markets the dashboard as "self-contained" and "opens offline", but this is only partially true. All data is inlined, but the charting library is not. On restricted networks, blocked CDNs, or jsDelivr outages, all charts fail to render.

### The Fix (Option A): Inline the Bundle

At build time, fetch and inline Chart.js:

```javascript
async function inlineDependencies() {
  const chartJsUrl = 'https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.js';
  const chartJs = await fetch(chartJsUrl).then(r => r.text());
  
  const html = fs.readFileSync('dashboard.html', 'utf8');
  const updated = html.replace(
    /<script src="https:\/\/cdn\.jsdelivr\.net\/npm\/chart\.js.*?"><\/script>/,
    `<script>${chartJs}</script>`
  );
  fs.writeFileSync('dashboard-inlined.html', updated);
  
  console.log(`Chart.js inlined (~200 KB); total payload now ~1.9 MB (before gzip)`);
}
```

**Trade-off:** Adds ~200 KB to the HTML (compresses to ~50 KB gzipped) but removes CDN dependency.

### The Fix (Option B): Document the Limitation

If inlining isn't feasible, clearly document the partial offline capability:

```html
<!-- In dashboard.html disclaimer: -->
<div class="disclaimer">
  <p>
    ⓘ <strong>Offline Capability:</strong> This dashboard is <strong>mostly</strong> self-contained.
    All data is inlined. However, the Chart.js charting library is loaded from a CDN on first load.
    <br>
    • With internet: Charts load from jsDelivr CDN (~1 second)
    • Without internet: Charts fail to render (library not available)
    <br>
    To make this truly offline: Download chart.js locally and serve from the same directory,
    or request the inlined version from your dashboard maintainer.
  </p>
</div>
```

---

## 22. Optimize Render Calls — Render Only Active Tab (REQUIRED PERFORMANCE)

### The Problem

`applyGlobalFilters()` (called on every date picker change) called `renderAll()`, which re-rendered all four tabs regardless of which was visible. This is 4× wasteful.

Additionally, rendering hidden canvases (see item 15) causes the zero-width sizing issue.

### The Fix

Track the active tab and only render that one:

```javascript
let activeTab = 'sales'; // Track which tab is active

function switchTab(name, btn) {
  activeTab = name;
  // ... show/hide logic ...
  // Re-render new active tab (see item 15)
  renderTabCharts(name);
}

function applyGlobalFilters() {
  // Only re-render the currently active tab
  renderTabCharts(activeTab);
  // Do NOT call renderAll()
}

function renderTabCharts(tab) {
  const ov = filteredOverview();
  if (tab === 'sales')       renderSales(ov);
  else if (tab === 'web')    renderWeb(ov);
  else if (tab === 'customers') renderCustomers(ov);
  else if (tab === 'marketing') renderMarketing(ov);
}

// On first page load:
document.addEventListener('DOMContentLoaded', () => {
  const ov = filteredOverview();
  renderSales(ov); // Only the default (active) tab
});
```

**Performance impact:** Date filter changes now only re-render 1 tab instead of 4 → ~3-4× faster filter updates.

---

## Complete Checklist: 10 Implementation Pitfalls

Before handing off Phase 3:

- [ ] **13. Spot-Check Window:** Date range matches Phase 1 baseline exactly (not rolling 30d)
- [ ] **14. Expected Values:** Baseline values read from state.md or use all-time stable metrics (not hardcoded)
- [ ] **15. Hidden Tab Charts:** Charts on hidden tabs re-rendered on switch (or use visibility:hidden)
- [ ] **16. Dead Payload:** No unused SINK columns in RAW mapping (documented why, if any)
- [ ] **17. Tab Filters:** All SINK dimension columns have filter dropdowns (symmetric across tabs)
- [ ] **18. Filter Indicator:** Tab buttons show visual indicator when filters active
- [ ] **19. Table Limits:** Every table shows "top N of M" and LIMIT is configurable
- [ ] **20. Schema Validation:** Build validates SINK schema at start; fails fast if mismatch
- [ ] **21. Offline Capability:** Chart.js inlined OR limitation clearly documented
- [ ] **22. Render Optimization:** Only active tab re-rendered on filter changes (not all 4)

---

**Version:** 1.0.0 (Lite, Code-Inspection-Validated)
**Last Updated:** 23 July 2026
**Source:** retail-dashboard-test-large code inspection
**Author:** FDE Team
