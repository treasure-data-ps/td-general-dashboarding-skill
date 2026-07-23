# Phase 3: Dashboard Template — External data.json Pattern

**This template loads `data.json` at runtime, not inline data.**

---

## Files in This Directory

| File | Purpose |
|------|---------|
| **`dashboard.template.html`** | ⭐ **Main template** — Use this for all dashboards |
| `generate-data-with-caching.js` | Reference generate-data.js that writes external data.json |
| `dashboard.template.html.example` | Alternate example (deprecated — use main template instead) |

---

## How It Works

### Phase 3: Build Dashboard
1. **Create queries** → Write `generate-data-with-caching.js`
2. **Generate data** → Run: `SOURCE_DB=<db> node generate-data.js`
   - Writes: `data.json` (plain JSON with `_meta`)
   - Copies: `dashboard.template.html` → `dashboard.html`
3. **Test** → Open `dashboard.html` in browser

### Phase 4a: Extract Reusable Skill
1. **Copy template** → `skills/dashboard.template.html` (unchanged)
2. **Copy script** → `skills/generate-data.js` (with caching)
3. **Run** → `SOURCE_DB=<db> node generate-data.js`
   - Creates: `data.json` (cacheable)
   - Creates: `dashboard.html` (copy of template)

### Runtime: Browser
1. Load `dashboard.html`
2. Template fetches `data.json`
3. Renders: KPIs, charts, tables from `window.RAW`
4. Shows: "Data as of [time] ([age] ago)"

---

## Template Features

✅ **External data loading** — `fetch('data.json')` at startup  
✅ **Error handling** — Shows error if data.json missing  
✅ **Data freshness** — "Data as of [time] ([age] ago)" from `_meta.generated_at`  
✅ **Responsive design** — Mobile-friendly grid layout  
✅ **Multiple visualizations** — KPI cards, line chart, doughnut chart, table  
✅ **Chart.js integration** — Line & doughnut charts included  
✅ **Auto-render** — Renders whatever data exists in `RAW`  

---

## Expected data.json Structure

```json
{
  "_meta": {
    "generated_at": "2026-07-23T01:14:00Z",
    "source_db": "retail_large_dataset",
    "sink_db": "test_suraj",
    "skill": "retail-analytics-dashboard",
    "version": "1"
  },
  "kpis": [
    { "label": "Total Revenue", "value": 1234567, "format": "currency" },
    { "label": "Orders", "value": 42891, "format": "number" }
  ],
  "trend": [
    { "month": "2026-06", "revenue": 156789 },
    { "month": "2026-07", "revenue": 189234 }
  ],
  "distribution": [
    { "label": "Region A", "value": 500000 },
    { "label": "Region B", "value": 450000 }
  ],
  "rows": [
    { "date": "2026-07-23", "revenue": 5000, "orders": 15 },
    { "date": "2026-07-22", "revenue": 4200, "orders": 12 }
  ]
}
```

**Required:** `_meta` (at least `generated_at` timestamp)  
**Optional:** `kpis`, `trend`, `distribution`, `rows` (template renders whatever exists)

---

## Usage in Phase 3

### Step 1: Copy Template
```bash
mkdir -p ./<project-slug>/dashboards
cp dashboard.template.html ./<project-slug>/dashboards/
```

### Step 2: Customize generate-data.js
```bash
cp generate-data-with-caching.js ./<project-slug>/dashboards/
# Edit: Update queries, table names, database names
```

### Step 3: Run
```bash
cd ./<project-slug>/dashboards
SOURCE_DB=retail SINK_DB=retail node generate-data.js
```

### Step 4: Test
```bash
# Browser loads template → fetches data.json → renders dashboard
open dashboard.html
```

---

## Usage in Phase 4a (Reusable Skill)

### Step 1: Copy Template
```bash
cp dashboard.template.html ./<project-slug>/skills/
```

### Step 2: Copy Script with Caching
```bash
cp generate-data-with-caching.js ./<project-slug>/skills/generate-data.js
```

### Step 3: Run (first time)
```bash
cd ./<project-slug>/skills
SOURCE_DB=retail SINK_DB=retail node generate-data.js
# ~60s: queries + writes data.json + copies template
```

### Step 4: Run (subsequent times)
```bash
SOURCE_DB=retail SINK_DB=retail node generate-data.js
# ~0.1s: uses cached data.json + copies template (600× faster!)
```

### Step 5: Refresh Data
```bash
node generate-data.js --refresh
# Re-queries, updates data.json
```

### Step 6: Iterate HTML
```bash
node generate-data.js --html-only
# Rebuilds HTML from existing data.json (no queries, instant)
```

---

## Customization

### Change Chart Types
In the template, update `Chart(ctx, { type: ... })`:
- `'line'` — Line chart
- `'bar'` — Bar chart
- `'doughnut'` — Pie/doughnut
- `'scatter'` — Scatter plot

### Add Custom Rendering
Add logic before `initDashboard()` is called, or extend the rendering functions.

### Modify Styling
Edit CSS in `<style>` block. Template uses a 12-column grid system.

### Change Data Freshness Format
In the timestamp display section, modify the age formatting logic.

---

## Troubleshooting

**Q: "Error loading dashboard data" (404)**  
A: Run `generate-data.js` first to create `data.json`

**Q: Data shows "—" (dash)**  
A: Column not found in query. Check `generate-data.js` query column names match template expectations.

**Q: Charts not rendering**  
A: `window.RAW.trend` or `window.RAW.distribution` missing. Add them to your queries.

**Q: Template stuck on "Loading..."**  
A: Check browser console (F12) for errors. Usually: `fetch()` failed or JSON parse error.

---

## Standard Across Both Phases

✅ **Phase 3:** Use this template for building dashboards  
✅ **Phase 4a:** Use this template for reusable skills  
✅ Both load `data.json` externally at runtime  
✅ Both support caching & refresh workflows  
✅ Both show data freshness in UI

---

**Version:** 2.0.0 (External JSON Pattern)  
**Last Updated:** 2026-07-23  
**Status:** Standard for all Phase 3 & Phase 4a dashboards
