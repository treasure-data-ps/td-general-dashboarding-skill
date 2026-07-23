# External data.json — Quick Reference

**TL;DR:** `generate-data.js` creates `data.json`, template loads it at runtime via `fetch()`. Data and HTML are decoupled.

---

## Before → After

### ❌ Old Pattern (Inline Data)
```javascript
// generate-data.js
var dataBlock = '<script>var RAW = ' + JSON.stringify(RAW) + ';</script>';
var html = template.replace('<!-- DATA_PLACEHOLDER -->', dataBlock);
fs.writeFileSync('dashboard.html', html);
```

**Result:** HTML file size = queries + template (~500 KB+). Every query changes HTML.

---

### ✅ New Pattern (External Data)
```javascript
// generate-data.js
fs.writeFileSync('data.json', JSON.stringify(RAW, null, 2));  // plain JSON
fs.writeFileSync('dashboard.html', template);                   // copy template as-is
```

**Result:** 
- `data.json` = just the data (~50 KB)
- `dashboard.html` = static template (~10 KB)
- Template loads data at runtime

---

## Files on Disk

### Skills Folder Structure
```
skills/
├── generate-data.js              ← Run this
├── dashboard.template.html       ← Master template (never edit)
├── dashboard.html                ← Output (copy of template)
├── data.json                     ← Generated data (gitignored)
└── knowledge/
    ├── business_context.md
    └── data_dictionary.md
```

**Key:** `dashboard.template.html` and `dashboard.html` are identical (no data injection).

---

## Template: Load data.json at Runtime

### Simplest: Using fetch()
```html
<!DOCTYPE html>
<html>
<body>
  <div id="dashboard">Loading...</div>

  <script>
    fetch('data.json')
      .then(r => r.json())
      .then(data => {
        window.RAW = data;
        
        // Show when data was generated
        var dt = new Date(window.RAW._meta.generated_at);
        var age = Math.round((Date.now() - dt) / (1000 * 60));
        console.log('Data from ' + dt.toLocaleString() + ' (' + age + 'm ago)');
        
        // Render your dashboard
        renderDashboard();
      });

    function renderDashboard() {
      // Use window.RAW here
      console.log(window.RAW.kpis);
    }
  </script>
</body>
</html>
```

### Better: With Error Handling
```html
<script>
  document.addEventListener('DOMContentLoaded', async function() {
    try {
      const response = await fetch('data.json');
      if (!response.ok) throw new Error('HTTP ' + response.status);
      
      window.RAW = await response.json();
      console.log('✅ Data loaded');
      
      renderDashboard();
    } catch (err) {
      console.error('❌ Failed to load data:', err);
      document.body.innerHTML = '<p>Error: ' + err.message + '</p>';
    }
  });

  function renderDashboard() {
    // Your render code here
  }
</script>
```

### Display Data Freshness
```html
<div style="font-size: 12px; color: #999;">
  Data as of: <span id="timestamp">—</span>
</div>

<script>
  // After window.RAW is loaded:
  if (window.RAW && window.RAW._meta) {
    var dt = new Date(window.RAW._meta.generated_at);
    var ageMs = Date.now() - dt;
    var minutes = Math.round(ageMs / 60000);
    var hours = Math.round(ageMs / 3600000);
    var age = minutes < 60 ? minutes + 'm ago' : hours + 'h ago';
    
    document.getElementById('timestamp').textContent = 
      dt.toLocaleString() + ' (' + age + ')';
  }
</script>
```

---

## Workflow

### Step 1: Generate Data
```bash
SOURCE_DB=retail SINK_DB=retail node generate-data.js
```

Output:
```
✅ Queries complete (24 trend rows)
✅ Wrote data.json (45.2 KB)
✅ Wrote dashboard.html (15 KB)
```

Files created:
- `data.json` — Plain JSON with `_meta` + data
- `dashboard.html` — Copy of template

### Step 2: Open Dashboard
```bash
open dashboard.html
# Browser loads template → fetches data.json → renders
```

### Step 3: Refresh Data
```bash
node generate-data.js --refresh
# Skips cache, re-queries, updates data.json
# dashboard.html doesn't change
```

### Step 4: Iterate HTML
```bash
node generate-data.js --html-only
# Rebuilds dashboard.html from existing data.json
# No queries, ~0.1s
```

---

## Key Differences

| Aspect | Inline | External |
|--------|--------|----------|
| **Data storage** | Inside HTML | `data.json` file |
| **HTML size** | Large (~500 KB) | Small (~15 KB) |
| **Data size** | Embedded | ~50 KB |
| **Cache benefits** | ❌ Can't cache | ✅ Cache data separately |
| **Refresh** | Rebuild HTML | Update data.json only |
| **Iterate CSS** | Re-query every time | Use `--html-only` |
| **Browser caching** | HTML only | JSON separately |
| **Share/ZIP** | Everything in 1 file | 2 files (HTML + JSON) |

---

## Common Questions

**Q: Why not embed data in HTML like before?**
A: External data enables caching, faster refresh, and separation of concerns. First run takes same time (~60s), but subsequent runs are instant (~0.1s).

**Q: How do I share the dashboard?**
A: ZIP `skills/` folder (includes template) + `data.json`. Recipients unzip and open `dashboard.html` in a browser.

**Q: What if data.json is missing?**
A: Template shows error ("data.json not found"). User must run `generate-data.js` first to generate it.

**Q: Can I deploy this to a web server?**
A: Yes! Upload `dashboard.html` + `data.json` to any web server. Browser loads template → fetches data.json → renders.

**Q: What if data is large (> 50 MB)?**
A: External JSON allows browser to cache it separately. Consider gzipping or splitting into multiple files.

---

## Checklist: External JSON Implementation

- [ ] `generate-data.js` writes plain JSON (not wrapped in `var RAW = ...`)
- [ ] Template loads data via `fetch('data.json')`
- [ ] Template assigns to `window.RAW` after fetch succeeds
- [ ] Error handling: catch 404 or network errors
- [ ] Data freshness displayed (from `_meta.generated_at`)
- [ ] `--refresh` works (re-queries, updates JSON)
- [ ] `--html-only` works (rebuilds HTML, no queries)
- [ ] First run: ~60s, second run: ~0.1s

---

## Performance

| Scenario | Time | Notes |
|----------|------|-------|
| First run (queries + write JSON) | ~60s | Standard |
| Second run (read cache + render) | ~0.1s | **600× faster** |
| `--refresh` (new queries) | ~60s | Atomic |
| `--html-only` (no queries) | ~0.1s | **Fastest** |
| Browser load (fetch JSON) | ~0.05-0.5s | Depends on file size |

---

**Version:** 1.0.0  
**Last Updated:** 2026-07-23  
**Status:** External data.json is the standard for all Phase 4 Track A dashboard skills
