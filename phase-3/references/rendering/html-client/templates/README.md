---
name: html-client-templates
description: |
  Copy-paste starter templates for HTML Client-Side Dashboard. Self-contained .html files with embedded CSS/JavaScript. Chart.js, responsive design, no backend needed. Download and share via email/Confluence.
---

# HTML Client-Side Dashboard Templates

Self-contained HTML files you can download and share. No backend required - all charts, filtering, and sorting happen in the browser.

**Stack:** HTML5, CSS3, JavaScript (ES6+), Chart.js, Responsive Design

---

## Template 1: Basic KPI Dashboard

**Best for:** Simple metrics overview, shareable reports  
**Build time:** 10 minutes  
**File size:** ~207KB (self-contained, includes inlined Chart.js)  
**Features:**
- 4 KPI cards with values
- Summary section
- Last updated timestamp
- Responsive on mobile/desktop
- Can be emailed or embedded in Confluence

---

## Template 2: Table Dashboard

**Best for:** Data exploration, sorting, searching  
**Build time:** 15 minutes  
**File size:** ~209KB (includes inlined Chart.js)  
**Features:**
- Sortable columns (click to sort)
- Search/filter rows
- Status badges with color coding
- CSV export button
- Summary metrics above table
- Responsive table scrolling on mobile

---

## Template 3: Multi-Chart Dashboard

**Best for:** Comprehensive analysis with visualizations  
**Build time:** 20 minutes  
**File size:** ~208KB (includes inlined Chart.js)  
**Features:**
- Line chart (trends)
- Bar chart (comparisons)
- Pie chart (distribution)
- KPI cards
- Legend and tooltips
- Responsive charts (resize with window)

---

## HTML Client-Side Advantages

✅ **No backend needed** — All HTML in one file  
✅ **Email/Confluence shareable** — Download and send  
✅ **Works offline** — No internet required after loading  
✅ **Fast loading** — No API calls (data embedded)  
✅ **Browser-based** — Works anywhere (Chrome, Safari, Firefox, Edge)  
✅ **No build process** — Open in browser, works immediately  
✅ **Print-friendly** — Generate PDF from browser  

---

## HTML Client-Side Limitations

❌ **Static snapshot** — Data is generated at `generate-data.js` run time; re-run to refresh  
❌ **Requires local server** — `fetch('data.json')` needs `npx serve .` or `preview_document`; bare `file://` won't work  
❌ **Large datasets slow** — Client-side rendering; limit to ~10K rows  
❌ **No persistence** — Filters don't save between sessions  
❌ **No authentication** — All data visible to anyone with the files  

---

## Critical Distinction: generate-data.js (Build Time) vs render.js (Runtime)

**This is the most common source of confusion.**

### generate-data.js — BUILD TIME (on your machine)
- **Has:** Database/tdx access, FDE's credentials, full query power
- **Does:** Runs queries, transforms data, produces dashboard.html
- **Never sent to:** Customer (only dashboard.html is delivered)
- **Runs:** Once, when you build the dashboard
- **Critical rule:** ALWAYS add `--limit <N>` flag to tdx queries or results silently truncate to 40 rows

### render.js — RUNTIME (in customer's browser)
- **Has:** NO database access, ONLY the RAW data object embedded in the page
- **Does:** Takes embedded data, formats it, renders charts/tables/filters
- **Sent to:** Customer (embedded in dashboard.html or loaded via data.json)
- **Runs:** Every time customer opens dashboard.html
- **Can do:** Format numbers, apply client-side filters, sort tables — but CANNOT query the database

**Example:** If a tab fails to initialize in the customer's browser (e.g., bad column reference in render.js), the error is caught by `safeInit()` error boundary and logged. Other tabs continue rendering. Before this wrapper, one tab's error would crash the entire script.

---

## Two Patterns: Pattern A (Inline) vs Pattern B (Fetch)

The templates support two architecturally different patterns:

### Pattern A: Inline Data Injection (< 2MB) ✅ RECOMMENDED
- `generate-data.js` produces `dashboard.html` with data embedded inline via `<!-- DATA_PLACEHOLDER -->`
- One self-contained file; no separate data file
- `render.js` reads the global `RAW` object already in the page
- **Best for:** Most dashboards (< 2MB), shareable single files, email/Confluence delivery
- **Structure:** HTML file has embedded `<script>var RAW = {...}</script>` before `</body>`

### Pattern B: Separate Data File (> 2MB)
- `generate-data.js` produces `dashboard.html` AND `data.json` separately
- `dashboard.html` uses `fetch('data.json')` at runtime to load data
- **Requires:** A local server (`npx serve .` or `preview_document`)
- **Best for:** Large dashboards (> 2MB), when file size matters
- **Caveats:** Must deliver both files together (zip them); `file://` URLs don't work with fetch

**These checked-in templates follow Pattern A.** If you need Pattern B (> 2MB payload), modify `generate-data.js` to detect size and write `data.json` instead — see the size-check logic in generate-data.js for the logic. Then modify `render.js` to `fetch('data.json')` instead of reading inline `RAW`.

**Do NOT mix patterns.** Stick to one per dashboard. Pattern A is the default and works for 95% of dashboards.

---

## Optional Walkthrough: Manual `render.js` + `fetch()` Workflow (Pattern B style — NOT the default)

**⚠️ Reference only.** The 3 checked-in templates (`kpi-dashboard.html`, `table-dashboard.html`, `multi-chart-dashboard.html`) already contain their own inline rendering `<script>` — they do NOT need a separate `render.js` file or `fetch('data.json')` call for the default Pattern A workflow. The steps below only apply if you deliberately opt into Pattern B (payload > 2MB) and need to wire up a standalone `render.js` that fetches a separate `data.json`.

### Step 1: Choose a Template
Copy the HTML file into your engagement's dashboard directory.

### Step 2: Start from `generate-data.js`
A starter `generate-data.js` is included in this directory. It covers Pattern A (inline injection) and Pattern B (separate data.json). Copy and customize the SQL queries and column names for your SINK table:

```bash
cp generate-data.js /path/to/engagement/dashboards/html-client/generate-data.js
# Edit: DB, table names, column names in the Queries section
SOURCE_DB=your_database node generate-data.js
```

**⚠️ CRITICAL: The `query()` helper includes `--limit 10000` by default.** Without this flag, results silently truncate to the default 40 rows. Always verify queries return expected row counts before delivering to customer. The correct tdx flag is `tdx --json query -d "<db>" "<sql>" --limit <N> 2>/dev/null` — not `tdx query --format json`.

### Step 3: Run `generate-data.js`
```bash
SOURCE_DB=your_db node generate-data.js
# or for Workflow path (SINK tables):
SINK_DB=your_sink_db node generate-data.js
```
Outputs `dashboard.html` (Pattern A) or `data.json` (Pattern B).

### Step 4: Write `render.js`
Create a JS file that reads `data.json` via `fetch()` and renders charts/tables.
See `render.js` in this same `templates/` folder for the full pattern.

### Step 5: Link `render.js` in `dashboard.html`
Add at the bottom of `<body>`:
```html
<script src="render.js"></script>
```

### Step 6: Serve and Test
```bash
npx serve .   # open http://localhost:3000
# or: preview_document in Treasure Work
```

### Step 7: Share
Zip all three files together:
```bash
zip dashboard.zip dashboard.html render.js data.json
```
Recipients unzip and open in any browser.

### Step 8: Alternative Sharing Methods
- Download .html file
- Email to stakeholders
- Embed in Confluence
- Print to PDF

---

## Data Format

All templates use this standard data structure:

```javascript
{
  metrics: {
    metric1: 1500000,
    metric2: 4500,
    metric3: 333.33,
    metric4: 12.5
  },
  rows: [
    { id: 1, name: 'Item 1', value1: 50000, value2: 100, status: 'Active' },
    { id: 2, name: 'Item 2', value1: 45000, value2: 95, status: 'Inactive' }
  ],
  chartData: [
    { label: 'Jan', value: 50000 },
    { label: 'Feb', value: 48000 }
  ]
}
```

---

## Common Customizations

### Change Colors (CSS Variables)

```css
:root {
  --primary-color: #3b82f6;    /* Blue */
  --success-color: #10b981;    /* Green */
  --warning-color: #f59e0b;    /* Orange */
  --danger-color: #ef4444;     /* Red */
}
```

### Add More KPI Cards

Copy-paste KPI card HTML:

```html
<div class="kpi-card">
  <div class="kpi-label">Metric Name</div>
  <div class="kpi-value">$1,234,567</div>
  <div class="kpi-change">↑ +12.5%</div>
</div>
```

### Add More Filters

```html
<div class="filter-group">
  <label>Status:</label>
  <select id="statusFilter">
    <option value="all">All</option>
    <option value="active">Active</option>
    <option value="inactive">Inactive</option>
  </select>
</div>
```

### Change Chart Type

```javascript
// Bar chart
new Chart(ctx, {
  type: 'bar',
  data: chartData,
  options: chartOptions
});

// Line chart
new Chart(ctx, {
  type: 'line',
  data: chartData,
  options: chartOptions
});

// Pie chart
new Chart(ctx, {
  type: 'pie',
  data: chartData,
  options: chartOptions
});
```

---

## Browser Compatibility

| Browser | Support |
|---------|---------|
| Chrome | ✅ Full |
| Firefox | ✅ Full |
| Safari | ✅ Full |
| Edge | ✅ Full |
| IE 11 | ⚠️ Partial (no ES6 features) |

---

## File Size Optimization

If HTML file is too large:

1. **Reduce data rows** — Limit to 100-500 rows instead of 10K
2. **Remove unused charts** — Delete chart divs and code
3. **Minify CSS/JS** — Use online minifiers
4. **Compress images** — If adding logos/images
5. **Chart.js is inlined directly** — not loaded via CDN, so the file works fully offline; this adds a fixed ~200KB and is not itself reducible without breaking offline support

---

## Testing Before Sharing

- [ ] Charts render without errors
- [ ] Filters work (click, select, search)
- [ ] Sorting works (click column headers)
- [ ] CSV export downloads file
- [ ] Page prints to PDF
- [ ] Mobile layout responsive
- [ ] No console errors (F12 → Console)
- [ ] Works offline (disconnect internet, reload)

---

## Deployment Options

### Email
```
Attach: dashboard.html
Recipient: stakeholder@company.com
Size limit: Usually OK (< 10MB)
```

### Confluence
1. Create page in Confluence
2. Upload HTML as attachment
3. Link to dashboard.html
4. Share Confluence page URL

### Web Server
```
Upload: /dashboards/sales-dashboard.html
Access: https://yourserver.com/dashboards/sales-dashboard.html
Share: Email the link
```

### Cloud Storage
```
Upload: Google Drive, OneDrive, Dropbox
Share: Share link
Access: Open in browser
```

---

## Security Considerations

⚠️ **XSS Protection:**
- Don't inject user input without sanitizing
- Use `textContent` instead of `innerHTML` for dynamic content

⚠️ **Data Privacy:**
- HTML files can be saved locally
- Don't include highly sensitive data
- Remind recipients to delete after use

⚠️ **CORS:**
- If fetching data from API, API must allow CORS
- Or embed data directly in HTML

---

## Next Steps

1. Choose a template (1-3)
2. Copy the HTML file
3. Replace sample data with your real data
4. Customize colors/title
5. Test in browser
6. Save as `.html` file
7. Share via email/Confluence/link
---

**Version:** 1.0.0
**Last Updated:** 23 June 2026
**Author:** FDE Team
