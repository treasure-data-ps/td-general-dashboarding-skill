---
name: phase-3-instructions
description: Phase 3 specific instructions for interactive dashboard building
priority: HIGH
load_order: 1.3
---

# Phase 3 Instructions — Build Interactive Dashboard

**Read this BEFORE `./SKILL.md`.**

**Phase 3 is MANDATORY for all projects (after Phase 1, optionally Phase 2).**

---

## Phase 3 Goal

Build a single portable `dashboard.html` file that:
- Queries either SINK tables (Phase 2 output) or source tables directly
- Renders interactive filters, charts, and KPI cards
- Opens standalone in a browser (no server required)
- Contains all data inlined (no separate API calls)

**Deliverable:** 
- User-approved interactive `dashboard.html`
- state.md appended with Phase 3 results

---

## Phase 3 Specific Rules (In Addition to Universal Rules)

### Rule P3-1: Query Source Decision

Before writing any Phase 3 query, decide:

**Option A: Query SINK Tables (Phase 2 completed)**
```sql
-- Phase 2 created pre-aggregated SINK tables
SELECT * FROM <db>.<project_slug>_sink_revenue
WHERE date >= '2026-07-01'
ORDER BY date DESC
```
✅ **Advantages:** Fast, pre-aggregated, optimized
❌ **Constraints:** Depends on Phase 2 completing successfully

**Option B: Query Source Tables Directly (Phase 2 skipped)**
```sql
-- Phase 3 does the aggregation on-the-fly
SELECT 
  DATE(event_time) AS date,
  region,
  SUM(revenue) AS total_revenue
FROM source_events
WHERE event_time >= CURRENT_DATE - INTERVAL 30 DAY
GROUP BY DATE(event_time), region
ORDER BY date DESC, region
```
✅ **Advantages:** No Phase 2 setup needed
❌ **Constraints:** Slower if high-volume data, may need query optimization

**Decision tree:**
```
Phase 2 completed?
  YES → Use SINK tables (faster)
  NO  → Query source tables directly (slower, but works)
```

---

### Rule P3-2: Query Execution & Data Validation

For every query used in dashboard:

- [ ] Execute manually first: `tdx query < query.sql`
- [ ] Verify results:
  - Are there rows returned? (not empty)
  - Do columns match what template expects?
  - Are numbers reasonable (not 0 or extreme)?
  - Do dates/timestamps look correct?

**Test queries before rendering:**
```bash
# Test the KPI query
tdx query --output json << 'EOF'
SELECT SUM(revenue) AS total_revenue FROM sales_table
WHERE date >= '2026-07-01'
EOF

# Expected output: {"total_revenue": 1234567.89}
# If output is null or 0, debug before rendering
```

---

### Rule P3-3: Dashboard Data Flows Through Node, Never AI

**Pattern (MANDATORY):**
```bash
# ✅ RIGHT: data → node script → dashboard.html
tdx query --output json --limit 1000 < query.sql | node render.js

# ❌ WRONG: data → AI context
# I would read the query output directly from /tmp/results.json
```

**Why:** Query results contain customer data (emails, IDs, financial info). Data must never pass through AI context. The rendering script processes it safely.

---

### Rule P3-4: Spot-Check at Least 3 KPIs After Rendering

After the dashboard.html is built, verify the numbers:

```
Dashboard shows:
  • Total Revenue: $4.5M
  • Unique Customers: 15,240
  • Conversion Rate: 3.2%

Spot-check each:
  SELECT SUM(amount) FROM revenue_table WHERE date >= '2026-07-01'
  → Result: $4,500,000 ✓ (matches dashboard)
  
  SELECT COUNT(DISTINCT customer_id) FROM customers WHERE active = true
  → Result: 15,240 ✓ (matches dashboard)
  
  SELECT COUNT(converted) / COUNT(*) FROM events WHERE date >= '2026-07-01'
  → Result: 0.032 (3.2%) ✓ (matches dashboard)
```

**If any KPI is 1% off → STOP. Debug before approving.**

---

### Rule P3-5: Dashboard Must Open Standalone in Browser

**Test before calling complete:**

```bash
# ✅ RIGHT: opens without server
open dashboard.html  # Browser opens it directly

# ❌ WRONG: requires server
# "You need to run 'npm start' to view this"
```

**Why:** The entire value of HTML Client is portability. If it needs a server, the build is wrong.

---

### Rule P3-6: Interactive Elements Must Be Tested

After rendering, manually test:

- [ ] Filters work (e.g., clicking "region" dropdown changes data)
- [ ] Charts respond to filter changes
- [ ] Page loads without errors (browser console)
- [ ] Numbers update correctly when filters change

**Test sequence:**
```
1. Open dashboard.html in browser
2. Filter: region = "US" → verify table updates
3. Filter: date = "last 7 days" → verify chart updates
4. Clear all filters → verify data returns to full view
5. Check browser console (F12) → no red errors
```

---

### Rule P3-7: Join Validation Results (From Phase 2)

If Phase 2 was completed, join validation was done. If Phase 2 was skipped:

- [ ] Verify join does NOT fan-out (row count after join ≤ 2× before)
- [ ] If fan-out detected, pre-aggregate many-side BEFORE querying

**Check (Phase 3 only if Phase 2 skipped):**
```sql
-- Count rows before join
SELECT COUNT(*) FROM table_a;  -- 1,000 rows

-- Count rows after join
SELECT COUNT(*) FROM table_a a
JOIN table_b b ON a.id = b.a_id;  -- Should be ≤ 2,000 rows

-- If result > 2,000, you have fan-out → pre-aggregate table_b first
```

---

### Rule P3-8: Rendering Must Use Node.js Scripts

All file I/O and template rendering must use Node.js:

**Pattern:**
```javascript
// generate-data.js — fetch query results
const results = await tdxQuery(querySQL);
fs.writeFileSync('data.json', JSON.stringify(results));

// render.js — inject data into HTML template
const data = JSON.parse(fs.readFileSync('data.json'));
const html = template.replace('{{DATA}}', JSON.stringify(data));
fs.writeFileSync('dashboard.html', html);
```

**Never:**
- Use Python for file I/O (not the standard)
- Let AI read raw query results (data flows script-only)

---

### Rule P3-9: Theme & Branding (Treasure AI 2026)

Dashboard must use official Treasure AI 2026 branding:

- **Primary colors:** Dark 2 (#2D40AA), Accent 1 (#847BF2), Light 1 (#FFFFFF)
- **Accessibility:** WCAG AA minimum (AAA preferred)
- **Typography:** Poppins (headers), Manrope (body)

See: `../references/treasure-data-theme.md`

---

### Rule P3-10: State.md Append

Append Phase 3 results to state.md (updated in Phase 2 or created in Phase 1):

```yaml
---

## Phase 3 — Build Interactive Dashboard
**Date:** [date]
**Status:** ✅ Complete

### Dashboard Details

- **File:** dashboard.html
- **Size:** [Xmb]
- **Data Inlined:** ✓ (self-contained)
- **Filters:** [list, e.g., "region", "date_range"]
- **Charts:** [list of viz types]
- **KPIs:** [list]

### Data Sources

- Query 1: Revenue by region
  - Source: [SINK table or source table]
  - Rows returned: [N]
  - Last updated: [timestamp]
  
- Query 2: Customer trends
  - Source: [table]
  - Rows returned: [N]
  - Last updated: [timestamp]

### Spot-Check Validation

- [ ] KPI 1: [metric name] = [value] ✓ (verified against DB)
- [ ] KPI 2: [metric name] = [value] ✓ (verified against DB)
- [ ] KPI 3: [metric name] = [value] ✓ (verified against DB)

### Testing Results

- [ ] HTML opens standalone (no server needed)
- [ ] Filters work correctly
- [ ] Charts update when filters change
- [ ] Browser console shows no errors
- [ ] Page loads in <3s on typical connection

### User Approval

- Dashboard approved: YES
- Feedback: [list any requested changes]

---

## Next Action

1. ✓ Phase 3 complete (dashboard ready)
2. Optional: Phase 4 (automation + agents)
3. Optional: Phase 5 (handoff documentation)
4. Or: Go live with dashboard
```

---

## Phase 3 Pre-Flight Checklist

**Before starting Phase 3, verify:**

### Prerequisites
- [ ] Promotion Score available from Phase 1
- [ ] state.md exists from Phase 1 (+ Phase 2 if applicable)
- [ ] All requirements + data findings documented

### Query Readiness
- [ ] Decision made: use SINK tables or source tables directly?
- [ ] Queries written for each dashboard element (KPIs, charts, filters)
- [ ] Queries tested manually (`tdx query < query.sql`)
- [ ] Results verified (not empty, numbers reasonable)

### Dashboard Scope
- [ ] KPIs defined (what metrics to show)
- [ ] Filters defined (what dimensions to slice by)
- [ ] Charts/visualizations planned
- [ ] Layout planned (tabs, sections, etc.)

### Rendering Setup
- [ ] Node.js scripts ready (generate-data.js, render.js)
- [ ] HTML template ready
- [ ] Treasure AI 2026 theme applied
- [ ] Data will not pass through AI context (confirmed)

---

## Phase 3 Decision Tree

```
START Phase 3: Build Interactive Dashboard
    ↓
Verify prerequisites from Phase 1/2
    ↓
Decide: SINK tables (Phase 2 done) or source tables (Phase 2 skipped)?
    ↓
Write queries for each dashboard element
    ↓
Execute each query manually (tdx query)
    ↓
Verify: results not empty, numbers reasonable
    ↓
Execute Node.js pipeline: query → render.js → dashboard.html
    ↓
Open dashboard.html in browser (standalone, no server)
    ↓
Test: filters work, charts update, no errors
    ↓
Spot-check 3+ KPIs against database
    ↓
All checks pass?
    YES ↓ → User approval
    NO  ↓ → Debug (data issue? rendering issue? query issue?)
    ↓
User approves?
    YES ↓ → Append Phase 3 results to state.md
    NO  ↓ → Gather feedback, adjust, re-test
    ↓
END Phase 3
```

---

## Common Phase 3 Blocks

### "The dashboard is slow to load"
**Response:**
> "Let's check: (1) Are queries already fast? (tdx query < query.sql — time it), (2) Is the HTML too large? (check file size), (3) Are there too many data rows? (limit to most recent 1000, or use SINK tables from Phase 2 for pre-aggregation)"

### "The chart shows the wrong data"
**Response:**
> "Let me verify: (1) Run the query manually to see what it returns, (2) Check if filters are applying correctly, (3) Verify the template is reading the right column from the query result. Common issue: column name mismatch between query and template."

### "I want the dashboard to update in real-time"
**Response:**
> "Phase 3 builds a static dashboard (data at build time). For live data, we'd need: (1) Phase 2 workflow refreshing SINK tables on schedule, or (2) Phase 4 Foundry agent for real-time queries. Which makes sense for your use case?"

### "Can I share this dashboard with others?"
**Response:**
> "Yes! Since dashboard.html is self-contained (all data inlined), you can: (1) Email the file directly, (2) Host it on any web server, (3) Upload to a shared drive. No setup needed on recipient's end — just open in browser."

---

## After Phase 3 Completes

**If dashboard is approved:**
```
✓ Phase 3 Complete → Dashboard Ready for Use

Optional next steps:
  Phase 4: Automation + Agents (../phase-4/INSTRUCTIONS.md)
    - Track A: Extract a reusable skill
    - Track B: Deploy a Foundry agent for conversational access
  
  Phase 5: Handoff Docs (../phase-5/INSTRUCTIONS.md)
    - Create Architecture, Usage Guide, Runbook, Access guide
  
  Or: Go live and iterate based on user feedback
```

**If dashboard needs rework:**
```
✗ Phase 3 Not Approved → Gather Feedback

User feedback:
  - [requested change 1]
  - [requested change 2]

Next steps:
  1. Adjust queries and/or templates
  2. Re-test and re-spot-check
  3. Re-present to user
```

---

**Version:** 1.0.0
**Last Updated:** 22 July 2026
**Load Order:** 1.3 (read after Phase 1-2 INSTRUCTIONS.md)
