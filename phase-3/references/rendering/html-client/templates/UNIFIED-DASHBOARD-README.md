# Unified Treasure Data Dashboard

A single, flexible HTML template that combines **KPI cards**, **charts**, and **data tables** with tabs, all branded with Treasure Data 2026 colors.

**File:** `unified-dashboard.html`  
**Size:** ~800 KB (HTML + CSS + JS inlined)  
**Data:** Inline (< 2MB) or separate `data.json` (> 2MB)  
**Browser:** Chrome, Firefox, Safari, Edge (responsive)  

---

## Why Unified? Combining All Individual Templates

Instead of choosing between 3 separate templates:

```
❌ KPI Dashboard (4 cards only)
❌ Table Dashboard (sortable rows only)
❌ Multi-Chart Dashboard (multiple charts only)
```

**✅ Unified Dashboard provides ALL features in one file with tabs:**

### Feature Mapping from Individual Templates

| Original Template | Now in Unified Tab | Features |
|-------------------|-------------------|----------|
| **KPI Dashboard** | 📊 **KPI Overview** | 4+ KPI cards, formatted numbers, auto-populated from metrics |
| **Multi-Chart Dashboard** | 📈 **Analysis** | Line chart (trends), Bar chart (comparisons), Pie chart (distribution) |
| **Table Dashboard** | 📋 **Data Explorer** | Sortable columns, full-text search, status badges, responsive |
| *(New bonus feature)* | ⬇️ **Export** | CSV export, JSON export, PDF print |

### Complete Tab Feature List

| Tab | Purpose | Features |
|-----|---------|----------|
| **📊 KPI Overview** | Executive summary | 4+ KPI cards with auto-formatted numbers, hover effects, custom borders |
| **📈 Analysis** | Trend & analysis | 3 charts (line/bar/pie), Chart.js, Treasure Data colors, responsive resize |
| **📋 Data Explorer** | Data exploration | Sortable table (click headers), full-text search, status badges, CSV/JSON export |
| **⬇️ Export** | Data download | Download as CSV, Download as JSON, Print to PDF |

**All in ONE file. No more choosing. All features together.**

---

## Quick Start

### 1. Using Example Data

```bash
# Copy template and example data
cp unified-dashboard.html dashboard.html
cp example-data.json data.json

# Open in browser (with local server)
npx serve .
# or: python3 -m http.server 8000

# Visit: http://localhost:3000 (or :8000)
```

### 2. Using Your Own Data (Pattern A: Inline)

```bash
# Edit generate-data-unified.js with your queries
# Change the queries section:
#   - metrics: SELECT your KPI metrics
#   - chartData: SELECT your trend/category data
#   - rows: SELECT your table data

# Set database and run
SOURCE_DB=your_database node generate-data-unified.js

# Output: dashboard.html (with data embedded)
open dashboard.html
```

### 3. Using Large Datasets (Pattern B: Separate data.json)

```bash
# If data.json > 2MB, script auto-creates separate file
SOURCE_DB=your_database LIMIT=50000 node generate-data-unified.js

# Edit dashboard.html to fetch data.json:
# Change line ~350 from:
#   if (typeof RAW !== 'undefined')
# To:
#   fetch('data.json').then(r => r.json()).then(d => { window.RAW = d; init(); })

# Run with server
npx serve .
```

---

## Data Format

The dashboard expects this JSON structure:

```json
{
  "metrics": {
    "Total Revenue": 4859839,
    "Order Count": 2156492,
    "Active Customers": 1234567,
    "Avg Order Value": 2254.18
  },
  "chartData": [
    {
      "label": "Jan",
      "date": "2026-01-01",
      "value": 425000,
      "category": "North America",
      "amount": 425000
    }
  ],
  "rows": [
    {
      "id": 1001,
      "order_date": "2026-08-15",
      "customer_name": "Acme Corp",
      "region": "North America",
      "status": "Active",
      "revenue": 125000,
      "items": 42,
      "rating": 4.8
    }
  ]
}
```

**Mapping:**
- **metrics** → KPI card values (display in Tab 1)
- **chartData** → Trend line, bar chart, pie chart (display in Tab 2)
- **rows** → Table rows for data explorer (display in Tab 3)

---

## Customization

### Change Colors (Treasure Data Theme)

Edit the `:root` CSS variables:

```css
:root {
  --td-primary: #2D40AA;      /* Dark Blue */
  --td-accent: #847BF2;       /* Purple */
  --td-success: #10B981;      /* Green */
  --td-warning: #F59E0B;      /* Orange */
  --td-danger: #EF4444;       /* Red */
}
```

### Add More KPI Cards

The script auto-populates from `metrics` object. Just add more keys:

```json
"metrics": {
  "Total Revenue": 4859839,
  "Order Count": 2156492,
  "Active Customers": 1234567,
  "Avg Order Value": 2254.18,
  "Conversion Rate": 0.032,           // ← New
  "Churn Rate": 0.015                 // ← New
}
```

### Add Filter Fields

In the Data Explorer tab, search already works across all fields. To add dropdowns:

```html
<!-- In table-header div, add: -->
<select id="regionFilter" class="filter-select">
  <option value="">All Regions</option>
  <option value="North America">North America</option>
  <option value="Europe">Europe</option>
</select>

<!-- Then filter in JavaScript -->
<script>
  document.getElementById('regionFilter').addEventListener('change', (e) => {
    if (e.target.value) {
      filteredRows = rawData.rows.filter(r => r.region === e.target.value);
    }
  });
</script>
```

### Change Chart Types

Edit the `initLineChart()`, `initBarChart()`, `initPieChart()` functions:

```javascript
// Change bar chart to horizontal:
new Chart(ctx, {
  type: 'bar',
  data: { ... },
  options: {
    indexAxis: 'y'  // ← Horizontal
  }
});

// Use different chart types:
type: 'radar'     // Radar chart
type: 'scatter'   // Scatter plot
type: 'bubble'    // Bubble chart
```

---

## Features

✅ **Responsive Design**
- Desktop: Multi-column layout
- Tablet: 2-column layout
- Mobile: Single-column, stacked

✅ **Tab Navigation**
- Tab buttons at top
- Smooth fade-in transitions
- Mobile-friendly scrolling

✅ **KPI Cards (Tab 1)**
- Formatted numbers (1.2M instead of 1200000)
- Hover effects (lift card)
- Custom border colors

✅ **Charts (Tab 2)**
- Line chart: Trends over time
- Bar chart: Category comparison
- Pie/Doughnut chart: Distribution
- Responsive resize
- Chart.js with Treasure Data colors

✅ **Data Explorer (Tab 3)**
- Full-text search across all fields
- Click column headers to sort
- Status badges with color coding
- Responsive table scrolling on mobile

✅ **Export (Tab 4)**
- Export as CSV
- Export as JSON
- Print to PDF
- Responsive table scrolling

✅ **Treasure Data Branding**
- Header with gradient background
- Treasure Data colors throughout
- Professional styling
- Print-friendly CSS

---

## Limitations

❌ **Static snapshot** — Re-run generate-data-unified.js to refresh  
❌ **No real-time updates** — Data captured at build time  
❌ **Large datasets** — Limit data.json to < 50MB (browser memory)  
❌ **Authentication** — All data visible (no user auth)  
❌ **Offline fetch** — Pattern B requires local server (file:// doesn't work)

---

## Best Practices

### Query Optimization
- Always add `--limit <N>` to queries (prevents silent truncation)
- Use `td_time_range()` for fast date filtering
- Pre-aggregate data (GROUP BY) when possible
- Limit chartData to 90 rows (one per day for 3 months)

### Data Size
- Keep total JSON < 2MB for inline (Pattern A)
- If > 2MB, use separate data.json (Pattern B)
- Limit table rows to 1,000-10,000 rows
- For large datasets, aggregate or paginate

### Mobile Testing
- Test on iPhone, iPad, Android tablets
- Verify charts resize correctly
- Test search and table scrolling
- Check that export buttons work

---

## Deployment Options

### Email
Attach `dashboard.html` to email (single file, no dependencies)

### Confluence
Upload `dashboard.html` as attachment, link from page

### Web Server
```bash
# nginx/Apache
cp dashboard.html /var/www/dashboards/
# Access: https://yourserver.com/dashboards/dashboard.html

# Or cloud storage
gsutil cp dashboard.html gs://your-bucket/dashboards/
# Share via public link
```

### Zip (for large datasets)
```bash
zip dashboard.zip dashboard.html data.json
# Recipients unzip and open locally with: npx serve .
```

---

## Security Considerations

⚠️ **Data Privacy**
- HTML files can be saved locally
- Don't include highly sensitive data
- Remind recipients to delete after use

⚠️ **XSS Protection**
- Script uses `textContent` (not `innerHTML`)
- User input from search is safely escaped
- Export functions are safe (local blob downloads)

⚠️ **CORS** (if using external API)
- data.json must be served from same origin
- Or API must allow CORS headers
- Or embed data inline (Pattern A)

---

## Troubleshooting

### "No data found" error
Check that `data.json` is in same directory as `dashboard.html` (Pattern B)  
Or verify that data is embedded inline (Pattern A)

### Charts not rendering
- Check browser console (F12 → Console)
- Verify Chart.js loaded (check <script> tag)
- Make sure `chartData` array has valid numbers

### Table shows "No data available"
- Check that `rows` array is not empty
- Verify column names in data match template
- Check `status` field for badge formatting

### Search not finding rows
- Search is case-insensitive full-text
- Check that row data is populated
- Verify column names are strings

### Large file slow to load
- Reduce LIMIT in generate-data-unified.js
- Use Pattern B (separate data.json)
- Compress data before inline embedding

---

## Examples

### Sales Dashboard
```
Metrics: Total Revenue, Order Count, Avg Order Value, Customer Count
Charts: Revenue by Region (bar), Revenue Trend (line), Order Status (pie)
Table: All orders with region, status, amount
```

### Operations Dashboard
```
Metrics: API Response Time, Error Rate, Uptime, Active Sessions
Charts: Response time trend, Error rate by service, Session distribution
Table: Server logs with timestamps, errors, recovery status
```

### Customer Dashboard
```
Metrics: Total Customers, Churn Rate, Lifetime Value, Retention
Charts: Customer growth (line), Churn by cohort (bar), Segment distribution (pie)
Table: Customer list with segment, churn risk, LTV
```

---

**Version:** 1.0.0  
**Last Updated:** 22 July 2026  
**Treasure Data 2026 Branding**  
**Pattern A (Inline) & Pattern B (Separate data.json) Support**
