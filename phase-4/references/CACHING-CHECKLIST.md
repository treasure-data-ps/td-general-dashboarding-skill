# Phase 4 Track A: Caching Implementation Checklist

**Use this checklist when implementing the data caching strategy in a dashboard skill.**

---

## Before You Start

- ✅ Read `data-caching-strategy.md` (conceptual overview)
- ✅ Review `generate-data-with-caching.js` template (reference implementation)
- ✅ Phase 3 dashboard is approved and working

---

## Step 1: Refactor `generate-data.js` (20 min)

### 1a. Add cache-check logic

```javascript
function checkCache() {
  if (refresh) return null;  // --refresh flag skips cache
  if (!fs.existsSync(DATA_PATH)) return null;
  var stat = fs.statSync(DATA_PATH);
  var ageMinutes = (Date.now() - stat.mtime) / (1000 * 60);
  if (ageMinutes > 60) return null;  // 60 min TTL
  return JSON.parse(fs.readFileSync(DATA_PATH, 'utf8'));
}
```

- [ ] `checkCache()` function added
- [ ] Honors `--refresh` flag (skip cache)
- [ ] Checks file existence
- [ ] Checks file age (default: 60 min)

### 1b. Parse CLI flags

```javascript
var args = process.argv.slice(2);
var refresh = args.includes('--refresh');
var htmlOnly = args.includes('--html-only');
var dataOnly = args.includes('--data-only');
```

- [ ] Flag parsing added to top of `main()`
- [ ] `--refresh` forces re-query
- [ ] `--html-only` skips queries, rebuilds HTML only
- [ ] `--data-only` generates data, skips HTML

### 1c. Extract queries into `fetchData()` function

```javascript
function fetchData() {
  // All queries here
  var kpiRows = query('SELECT ...');
  var trendRows = query('SELECT ...');
  
  return {
    _meta: { generated_at: new Date().toISOString(), ... },
    kpis: kpiRows,
    trend: trendRows
  };
}
```

- [ ] All query logic wrapped in `fetchData()`
- [ ] Returns `RAW` object with data
- [ ] `_meta` object includes: `generated_at`, `source_db`, `sink_db`, `skill`, `version`

### 1d. Add cache check to `main()`

```javascript
var RAW = checkCache();  // returns object or null
if (!RAW) {
  RAW = fetchData();  // run queries if no cache
}
```

- [ ] Cache check runs first (before queries)
- [ ] Falls back to `fetchData()` if cache miss

### 1e. Write `data.json`

```javascript
fs.writeFileSync(DATA_PATH, JSON.stringify(RAW, null, 2), 'utf8');
```

- [ ] `data.json` written with formatted JSON (indent 2)
- [ ] Includes `_meta` payload
- [ ] Skip write if `--html-only` flag

### 1f. Build HTML from `data.json`

```javascript
var template = fs.readFileSync(TEMPLATE_PATH, 'utf8');
var dataBlock = '<script>var RAW = ' + JSON.stringify(RAW) + ';<\/script>';
var html = template.replace('<!-- DATA_PLACEHOLDER -->', dataBlock);
fs.writeFileSync(OUTPUT_PATH, html, 'utf8');
```

- [ ] Template replaced with `<!-- DATA_PLACEHOLDER -->`
- [ ] `RAW` object injected as `<script>`
- [ ] Skip build if `--data-only` flag

---

## Step 2: Update Dashboard Template (10 min)

### 2a. Add data freshness display

In your `dashboard.template.html`, add this block:

```html
<div id="data-info" style="font-size: 12px; color: #999; padding: 8px; text-align: right;">
  Data as of: <span id="data-timestamp">—</span>
</div>

<script>
(function() {
  if (window.RAW && window.RAW._meta) {
    var dt = new Date(window.RAW._meta.generated_at);
    var ageMs = Date.now() - dt;
    var ageMin = Math.round(ageMs / (1000 * 60));
    var ageHr = Math.round(ageMs / (1000 * 60 * 60));
    var display = dt.toLocaleString() + ' (' + (ageMin < 60 ? ageMin + 'm' : ageHr + 'h') + ' ago)';
    document.getElementById('data-timestamp').textContent = display;
  }
})();
</script>
```

- [ ] Data freshness display added to template
- [ ] Shows `_meta.generated_at` timestamp
- [ ] Shows relative age (e.g., "2h ago")

### 2b. Verify data injection point

```html
<!-- DATA_PLACEHOLDER -->  ← generate-data.js will replace this
```

- [ ] Template contains `<!-- DATA_PLACEHOLDER -->` comment
- [ ] Comment is placed just before `</body>` or in `<head>`

---

## Step 3: Test the Implementation (15 min)

### 3a. Test 1: First run (queries + cache)

```bash
cd ./<project-slug>/skills
SOURCE_DB=your_db SINK_DB=your_db node generate-data.js
```

Expected output:
```
✅ Cache miss or expired. Running queries...
✅ Queries complete (24 trend rows)
✅ Wrote data.json (45.2 KB)
✅ Wrote dashboard.html (246 KB)
```

Verify:
- [ ] `data.json` written
- [ ] `data.json` contains `_meta` with `generated_at`
- [ ] `dashboard.html` generated
- [ ] No errors in console

### 3b. Test 2: Cache hit (second run)

```bash
node generate-data.js
```

Expected output:
```
✅ Cache hit: data.json is 2 minutes old
✅ Wrote dashboard.html (246 KB)
```

Timing check:
- [ ] **First run: ~60 seconds** (queries)
- [ ] **Second run: ~0.1 seconds** (cache)

### 3c. Test 3: Refresh flag

```bash
node generate-data.js --refresh
```

Expected output:
```
🔄 --refresh flag set, skipping cache.
✅ Queries complete (24 trend rows)
✅ Wrote data.json (45.2 KB)
✅ Wrote dashboard.html (246 KB)
```

Verify:
- [ ] Cache is bypassed
- [ ] Queries run
- [ ] `data.json` overwritten
- [ ] ~60 second execution

### 3d. Test 4: HTML-only flag

```bash
node generate-data.js --html-only
```

Expected output:
```
📄 HTML-only mode: rebuilding from existing data.json
✅ Loaded data.json
✅ Wrote dashboard.html (246 KB)
```

Timing check:
- [ ] **No queries run** (~0.1 seconds total)
- [ ] Uses existing `data.json`
- [ ] `dashboard.html` rebuilt

### 3e. Test 5: Data-only flag

```bash
node generate-data.js --data-only
```

Expected output:
```
✅ Queries complete (24 trend rows)
✅ Wrote data.json (45.2 KB)
✅ Data generated (--data-only: skipping HTML build).
```

Verify:
- [ ] `data.json` generated
- [ ] `dashboard.html` NOT modified
- [ ] ~60 second execution (queries only)

### 3f. Test 6: Open dashboard in browser

```bash
open dashboard.html
# or: npx serve .
```

Verify:
- [ ] Dashboard loads
- [ ] All KPIs show numbers (not "undefined")
- [ ] "Data as of: [timestamp] ([age] ago)" displayed
- [ ] No console errors (F12 → Console)

---

## Step 4: Validate Spot-Check Values (5 min)

Compare dashboard KPIs against `state.md` confirmed samples:

```javascript
// In browser console (F12)
console.log(RAW);  // view all data
console.log(RAW._meta);  // view metadata
console.log(RAW.kpis);  // view KPI values
```

- [ ] KPI values within ±5% of confirmed samples
- [ ] Dates/timestamps formatted correctly
- [ ] Trend row counts match expected granularity

---

## Step 5: Prepare for Reuse (5 min)

### 5a. Update `.gitignore`

```
# Cached dashboard data
data.json
dashboard.html
*.html.tmp
```

- [ ] `.gitignore` updated to exclude `data.json` and `dashboard.html`

### 5b. Add CLI flag documentation to SKILL.md

```yaml
---
name: fde-dashboard-retail-revenue
description: |
  Retail revenue dashboard. 
  
  Usage flags:
    --refresh     Re-query data and rebuild
    --html-only   Rebuild HTML from existing data.json (no queries)
    --data-only   Generate data.json without building HTML
---
```

- [ ] SKILL.md documents all CLI flags
- [ ] Environment variables documented (`SOURCE_DB`, `SINK_DB`)

---

## Final Checklist (Before Step 4a-vi)

- ✅ `generate-data.js` implements caching strategy
- ✅ `data.json` written with `_meta` payload
- ✅ CLI flags work: `--refresh`, `--html-only`, `--data-only`
- ✅ Cache timing: ~60s first run, ~0.1s cached runs
- ✅ Dashboard displays data freshness
- ✅ All tests pass (6 test scenarios above)
- ✅ Spot-check values within ±5%
- ✅ `.gitignore` updated
- ✅ SKILL.md documents usage

---

## When Caching Goes Wrong

| Issue | Cause | Fix |
|---|---|---|
| `data.json` not created | Cache check returns too early | Ensure `refresh` or cache-miss returns `null`, not `RAW` |
| `--html-only` fails ("data.json not found") | First run didn't generate cache | Always run without flags first to bootstrap cache |
| Dashboard shows stale data | Cache TTL too long | Reduce `CACHE_MAX_AGE_MINUTES` or run `--refresh` |
| "Data as of" timestamp wrong | `_meta.generated_at` not set | Verify `RAW._meta.generated_at = new Date().toISOString()` |
| Cache not being used (still ~60s) | Flag parsing broken | Verify `args.includes('--html-only')` works; debug with `console.log(args)` |

---

## Success Criteria (✅ = Ready for Step 4a-vi)

✅ **Performance:** First run ~60s, cached runs ~0.1s  
✅ **Portability:** Skills folder + `data.json` completely self-contained  
✅ **Reusability:** Change `SOURCE_DB`/`SINK_DB`, re-run, instant dashboard  
✅ **Observability:** Dashboard shows when data was generated  
✅ **Maintainability:** Separate data (JSON) from rendering (HTML)

---

**Version:** 1.0.0  
**Last Updated:** 2026-07-23  
**Scope:** Phase 4 Track A all dashboard skills (reference implementation)
