# Phase 4: Data Caching Strategy (Reference Implementation)

**Goal:** Separate data generation (queries → JSON) from rendering (JSON → HTML), enabling local caching, instant re-renders, and visible data freshness.

---

## The Problem with Current Approach

| Issue | Current (Pattern A/B) | With Caching |
|---|---|---|
| Re-open dashboard | Re-queries TD (~60s) | Reads `data.json` instantly (~0.1s) |
| Iterate on HTML/CSS | Re-queries TD | Use `--html-only`, no queries |
| Show data age | Hidden (embedded timestamp only) | Visible: "Data as of Jul 23, 1:14 AM (2h ago)" |
| Refresh on demand | Manual rebuild | `--refresh` flag, atomic swap |
| Cache locally | ❌ Not possible | ✅ `data.json` is portable, versionable |

---

## Architecture

```
OFFLINE (generate-data.js):
  ├─ Check cache: Does data.json exist & is it recent?
  │   ├─ Yes + no --refresh → Skip queries, use cached data
  │   └─ No or --refresh → Run queries against TD
  │
  ├─ Fetch data from TD (if needed)
  │
  ├─ Write data.json to disk
  │   {
  │     "_meta": {
  │       "generated_at": "2026-07-23T01:14:00Z",
  │       "source_db": "retail_large_dataset",
  │       "sink_db": "test_suraj",
  │       "skill": "retail-dashboard",
  │       "version": "1"
  │     },
  │     "kpis": [...],
  │     "trend": [...]
  │   }
  │
  └─ Copy dashboard.template.html → dashboard.html (no data injection)

RUNTIME (Browser loads dashboard.html):
  ├─ Load dashboard.html (static template)
  │
  ├─ Template loads data.json via:
  │   <script src="data.json"></script>
  │   OR: fetch('data.json').then(r => r.json()).then(d => { window.RAW = d; })
  │
  └─ Display: "Data as of Jul 23, 1:14 AM (2h ago)"
     (computed from RAW._meta.generated_at)
```

---

## File Structure

```
skills/
├── generate-data.js          ← Orchestrator (queries → data.json)
├── dashboard.template.html   ← Static template (never changes between runs)
├── dashboard.html            ← Output HTML (copy of template)
├── data.json                 ← External cache (generated, gitignored)
└── data-old.json             ← Optional: backup before refresh
```

**Key principle:** 
- `dashboard.template.html` is **static** — never edited, copied to `dashboard.html` as-is
- Template loads `data.json` at runtime via `<script src="data.json">` or `fetch('data.json')`
- Data and HTML are **decoupled** — can refresh either independently

---

## Step-by-Step Implementation

### 1. Extract & Refactor `generate-data.js`

```javascript
'use strict';

var execSync = require('child_process').execSync;
var fs = require('fs');
var path = require('path');

var DB_SOURCE = process.env.SOURCE_DB || 'your_database';
var DB_SINK = process.env.SINK_DB || DB_SOURCE;
var TEMPLATE_PATH = path.join(__dirname, 'dashboard.template.html');
var DATA_PATH = path.join(__dirname, 'data.json');
var OUTPUT_PATH = path.join(__dirname, 'dashboard.html');

// ─── Cache helper ───────────────────────────────────────────────────────────
function checkCache(maxAgeMinutes) {
  if (!fs.existsSync(DATA_PATH)) return null;
  var stat = fs.statSync(DATA_PATH);
  var ageMinutes = (Date.now() - stat.mtime) / (1000 * 60);
  if (ageMinutes > maxAgeMinutes) return null;
  return JSON.parse(fs.readFileSync(DATA_PATH, 'utf8'));
}

// ─── Query helper ──────────────────────────────────────────────────────────
function query(sql, limitRows) {
  var limit = process.env.DASHBOARD_ROW_LIMIT || limitRows || 10000;
  if (sql.toUpperCase().indexOf('LIMIT') === -1) {
    sql = sql + ' LIMIT ' + limit;
  }
  var result = execSync(
    'tdx query -d "' + DB_SINK + '" "' + sql + '" --format json --limit ' + limit,
    { encoding: 'utf8', timeout: 60000 }
  );
  // Parse JSON: tdx prepends status lines, so find first '['
  var match = result.match(/\[\s*\{/);
  var jsonStr = match ? result.slice(match.index) : '[]';
  return JSON.parse(jsonStr);
}

// ─── Main logic ─────────────────────────────────────────────────────────────
function main() {
  var args = process.argv.slice(2);
  var htmlOnly = args.includes('--html-only');
  var dataOnly = args.includes('--data-only');
  var refresh = args.includes('--refresh');

  console.log('');
  console.log('═══════════════════════════════════════════════════════════');
  console.log('Dashboard Data Generator (with Caching)');
  console.log('═══════════════════════════════════════════════════════════');
  console.log('');

  var RAW = null;

  // ─── Option 1: --html-only (build from existing cache) ────────────────────
  if (htmlOnly) {
    if (!fs.existsSync(DATA_PATH)) {
      console.error('❌ ERROR: --html-only but data.json not found. Run without flags first.');
      process.exit(1);
    }
    console.log('📄 Loading cached data...');
    RAW = JSON.parse(fs.readFileSync(DATA_PATH, 'utf8'));
    console.log('✅ Loaded data.json (' + (fs.statSync(DATA_PATH).size / 1024).toFixed(1) + ' KB)');
  }
  // ─── Option 2: Check cache first (unless --refresh) ────────────────────────
  else if (!refresh && !dataOnly) {
    var cached = checkCache(60); // 60 min cache
    if (cached) {
      console.log('✅ Cache hit: data.json is recent');
      RAW = cached;
    } else {
      console.log('🔄 Cache miss or expired. Running queries...');
    }
  }

  // ─── If no cache, run queries ───────────────────────────────────────────────
  if (!RAW) {
    console.log('🔄 Fetching data from ' + DB_SINK + '...');
    
    var kpiRows = query('SELECT SUM(revenue) AS revenue, COUNT(*) AS count FROM table LIMIT 1');
    var trendRows = query(
      'SELECT SUBSTR(date, 1, 7) AS month, SUM(revenue) AS revenue FROM table GROUP BY 1 LIMIT 24'
    );

    RAW = {
      _meta: {
        generated_at: new Date().toISOString(),
        source_db: DB_SOURCE,
        sink_db: DB_SINK,
        skill: 'retail-analytics-dashboard',
        version: '1'
      },
      kpis: kpiRows,
      trend: trendRows.map(r => ({ month: r.month, revenue: parseFloat(r.revenue) }))
    };

    console.log('✅ Queries complete (' + trendRows.length + ' trend rows)');
  }

  // ─── Write data.json (unless --html-only) ──────────────────────────────────
  if (!htmlOnly) {
    fs.writeFileSync(DATA_PATH, JSON.stringify(RAW, null, 2), 'utf8');
    console.log('✅ Wrote ' + DATA_PATH + ' (' + (fs.statSync(DATA_PATH).size / 1024).toFixed(1) + ' KB)');
  }

  // ─── Exit if --data-only ───────────────────────────────────────────────────
  if (dataOnly) {
    console.log('');
    console.log('✅ Data generated. Skipping HTML build (--data-only).');
    return;
  }

  // ─── Build dashboard.html ──────────────────────────────────────────────────
  console.log('');
  console.log('🏗️  Building dashboard.html...');
  
  var template = fs.readFileSync(TEMPLATE_PATH, 'utf8');
  var dataBlock = '<script>var RAW = ' + JSON.stringify(RAW) + ';<\/script>';
  var html = template.replace('<!-- DATA_PLACEHOLDER -->', dataBlock);

  fs.writeFileSync(OUTPUT_PATH, html, 'utf8');
  console.log('✅ Wrote ' + OUTPUT_PATH + ' (' + (Buffer.byteLength(html, 'utf8') / 1024).toFixed(0) + ' KB)');

  console.log('');
  console.log('═══════════════════════════════════════════════════════════');
  console.log('✅ Dashboard ready!');
  console.log('');
  console.log('Next: open dashboard.html in your browser');
  console.log('');
  console.log('Refresh data:');
  console.log('  node generate-data.js --refresh');
  console.log('');
  console.log('Iterate HTML only (no queries):');
  console.log('  node generate-data.js --html-only');
  console.log('');
  console.log('Generate data without building HTML:');
  console.log('  node generate-data.js --data-only');
  console.log('');
}

main();
```

### 2. Update Template to Load `data.json` at Runtime

In `dashboard.template.html`, add code to load data.json before initializing the dashboard:

**Option A: Using fetch (Recommended)**
```html
<!-- At end of <body>, before other scripts that use window.RAW -->
<script>
(function() {
  // Load data.json at runtime
  fetch('data.json')
    .then(response => response.json())
    .then(data => {
      window.RAW = data;
      
      // Show data freshness
      if (window.RAW && window.RAW._meta) {
        var dt = new Date(window.RAW._meta.generated_at);
        var ageMs = Date.now() - dt;
        var ageMin = Math.round(ageMs / (1000 * 60));
        var ageHr = Math.round(ageMs / (1000 * 60 * 60));
        var age = ageMin < 60 ? ageMin + 'm ago' : ageHr + 'h ago';
        var elem = document.getElementById('data-timestamp');
        if (elem) elem.textContent = dt.toLocaleString() + ' (' + age + ')';
      }
      
      // Call your render function
      if (typeof initDashboard === 'function') {
        initDashboard();  // or your init function name
      }
    })
    .catch(err => {
      console.error('Failed to load data.json:', err);
      document.body.innerHTML = '<p>Error loading data. data.json not found.</p>';
    });
})();
</script>
```

**Option B: Using script tag (simpler)**
```html
<!-- Add to <head> or before other scripts that reference window.RAW -->
<script src="data.json"></script>
<!-- data.json must export: var RAW = {...}; -->

<!-- Display freshness and init dashboard -->
<script>
(function() {
  if (window.RAW && window.RAW._meta) {
    var dt = new Date(window.RAW._meta.generated_at);
    var ageMs = Date.now() - dt;
    var ageMin = Math.round(ageMs / (1000 * 60));
    var elem = document.getElementById('data-timestamp');
    if (elem) elem.textContent = dt.toLocaleString() + ' (' + ageMin + 'm ago)';
  }
  if (typeof initDashboard === 'function') initDashboard();
})();
</script>
```

**Option C: Using ES Module import**
```html
<script type="module">
  import RAW from './data.json' assert { type: 'json' };
  window.RAW = RAW;
  
  // Show freshness
  if (RAW._meta) {
    var dt = new Date(RAW._meta.generated_at);
    var elem = document.getElementById('data-timestamp');
    if (elem) elem.textContent = dt.toLocaleString();
  }
  
  // Init dashboard
  if (typeof initDashboard === 'function') initDashboard();
</script>
```

**Add freshness display anywhere in HTML:**
```html
<div id="data-info" style="font-size: 11px; color: #999; padding: 4px 8px;">
  Data as of: <span id="data-timestamp">—</span>
</div>
```

### 3. Usage Examples

**First run (queries + builds HTML):**
```bash
SOURCE_DB=retail SINK_DB=retail node generate-data.js
# → Runs queries → writes data.json → builds dashboard.html
```

**Subsequent run (from cache):**
```bash
SOURCE_DB=retail SINK_DB=retail node generate-data.js
# → Skips queries → uses cached data.json → builds dashboard.html instantly
```

**Refresh data:**
```bash
SOURCE_DB=retail SINK_DB=retail node generate-data.js --refresh
# → Re-runs all queries → overwrites data.json → rebuilds dashboard.html
```

**Iterate HTML only:**
```bash
node generate-data.js --html-only
# → Skips queries → uses existing data.json → rebuilds dashboard.html
# (no env vars needed; uses same data.json)
```

**Generate data without building HTML:**
```bash
SOURCE_DB=retail SINK_DB=retail node generate-data.js --data-only
# → Runs queries → writes data.json → exits (no HTML build)
```

---

## Benefits

| Scenario | Time | Impact |
|---|---|---|
| **First build** | ~60s (queries only) | Standard |
| **Re-open dashboard** | ~0.1s (read JSON) | **60× faster** |
| **Iterate HTML/CSS** | ~0.1s (--html-only) | **No queries** |
| **Refresh data** | ~60s + 0.1s HTML | Atomic, no stale HTML |
| **Share dashboard** | 1 file + 1 JSON | Portable, cacheable |

---

## Template Example

See `phase-3/references/rendering/html-client/templates/dashboard.template.html.example` for a complete working example that:
- ✅ Loads `data.json` at runtime via `fetch()`
- ✅ Handles errors gracefully (if data.json not found)
- ✅ Displays data freshness ("Data as of...")
- ✅ Initializes dashboard charts and KPIs from `window.RAW`

**Copy this template and customize the chart/KPI rendering logic for your dashboard.**

---

## Integration with Phase 4 Track A Workflow

**Step 4a-ii: Extract & Parameterize Query Scripts**

When extracting `generate-data.js` from Phase 3:

1. ✅ Keep the same query logic
2. ✅ Wrap queries in `function fetchData()` that returns `RAW`
3. ✅ Add cache-check logic at top of `main()`
4. ✅ Parse CLI flags: `--refresh`, `--html-only`, `--data-only`
5. ✅ Write `data.json` with `_meta` payload
6. ✅ Build HTML that references `data.json` (or loads via `<script>` tag)

**Step 4a-v: Validate the Extracted Skill**

1. Run: `SOURCE_DB=<db> SINK_DB=<db> node generate-data.js`
2. Verify: `data.json` written with `_meta.generated_at`
3. Test cache: `node generate-data.js --html-only` (should be instant)
4. Test refresh: `node generate-data.js --refresh` (should re-query)

---

## Schema: `data.json`

```json
{
  "_meta": {
    "generated_at": "2026-07-23T01:14:00.000Z",
    "source_db": "retail_large_dataset",
    "sink_db": "test_suraj",
    "skill": "retail-analytics-dashboard",
    "version": "1"
  },
  "kpis": [
    { "label": "Total Revenue", "value": 1234567.89, "format": "currency" },
    { "label": "Orders", "value": 42891, "format": "number" }
  ],
  "trend": [
    { "month": "2026-06", "revenue": 156789.12 },
    { "month": "2026-07", "revenue": 189234.56 }
  ],
  "summary": {
    "source": "retail_large_dataset.daily_sales",
    "period": "Last 90 days",
    "records": 42891,
    "updated": "2026-07-23T01:14:00Z"
  }
}
```

---

## Gitignore

Add to `.gitignore`:
```
# Cached dashboard data
data.json
dashboard.html
*.html.tmp
```

---

## Deployment Checklist

- ✅ `generate-data.js` includes cache-check logic
- ✅ `data.json` written with `_meta` payload
- ✅ Dashboard template reads `data.json` (not inline)
- ✅ Freshness timestamp displayed in UI ("Data as of…")
- ✅ `--refresh` flag works (re-queries + overwrites `data.json`)
- ✅ `--html-only` flag works (uses existing `data.json`, no queries)
- ✅ First run: ~60s (queries); subsequent runs: ~0.1s (cache)

---

**Version:** 1.0.0  
**Last Updated:** 2026-07-23  
**Scope:** Reference implementation for Phase 4 Track A all dashboard skills
