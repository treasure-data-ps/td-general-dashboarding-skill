---
name: cross-phase-instructions
description: Cross-phase guardrails, data integrity rules, and rendering constraints
priority: HIGH
load_order: 1
---

# Cross-Phase Instructions (Guardrails & Universal Rules)

**Read this SECOND (after `../INSTRUCTIONS.md`).**

These rules apply **across all phases** and must never be violated. Every rule has a reason derived from production incidents.

---

## ⚠️ BEFORE EVERY ENGAGEMENT — Required Reads

**On every new engagement (not resumed), re-read these guardrails files in order:**
1. **`./guardrails-lite.md`** (293 lines) — Core cross-phase guardrails, scoping, and data integrity rules
2. **`./treasure-data-theme.md`** (396 lines) — Brand colors, fonts, component styles (establishes dashboard aesthetics)
3. **`../STATE-CHECKPOINT-SYSTEM.md`** (453 lines) — Runtime validation and recovery protocol (establishes state.md discipline)

**Why:** These three files establish the baseline for ALL downstream decisions (brand compliance, data integrity, scoping accuracy). Skipping them leads to rework (wrong colors, incorrect queries, over-scoped dashboards).

---

## 0. Phase Auto-Advance (READ THIS FIRST BEFORE EVERY PHASE TRANSITION)

**⚠️ CRITICAL: NEVER stop at phase completion. IMMEDIATELY auto-advance or ask the next step.**

**Root cause of past stalls:** Phase completion sections said "read phase-N INSTRUCTIONS.md next" but didn't tell the skill **what to say or when to say it**. Result: engagements paused waiting for user to ask "what's next?"

**Fix: Explicit scripts at each phase completion**

### Phase 1 Completion → Auto-Route
```
Score 0-2: "Phase 3 is optimal for you. Starting Phase 3..."
Score 3: "Your project could go either way. Phase 2 (workflow) or Phase 3 (direct)?"
        [ONE immediate yes/no question, no "let me know"]
Score 4-6: "Phase 2 (workflow) is recommended. Proceeding to Phase 2..."
```

### Phase 2 Completion → Auto-Advance
```
"Phase 2 complete. Now building the dashboard (Phase 3)..."
[Start Phase 3 immediately, no approval needed]
```

### Phase 3 Completion → Present Phase 4
```
"Dashboard approved. What's next?

Track A: Extract this as a reusable skill (1 hour)
Track B: Deploy conversational agent (1.5 hours)
Track C: Close engagement

→ Which? (A/B/C)"
[Only ask once, get immediate answer]
```

### Phase 4 Completion → Present Phase 5
```
"[Track A/B] complete. Final step: documentation & handoff?

Phase 5: Create runbooks, access guides, handoff docs (1 hour)
Or: Close engagement

→ Phase 5? (YES/NO)"
[Only ask once]
```

---

## 1. Data Integrity — NO Exceptions

### Never use synthetic, mock, or hardcoded data
Every number in a dashboard must come from a live SQL query result. No estimated values, no "probably around this much", no hardcoded sample arrays. If you don't have a query result yet, don't start building.

**Why:** Customers spot-check dashboards. Wrong numbers destroy trust and require complete rebuilds.

**Past incident:** Dashboard showed "New York is #1 destination" — actual top destination was Tokyo with 2× revenue. This required rebuilding the entire dashboard.

---

### Never read raw query output files into AI context
Query results in `/tmp/` contain customer emails, IDs, counts, and financial data. Always pipe through node scripts (e.g., `render.js`) so data flows through the script only, never through AI context.

**Pattern:**
```bash
# ✅ RIGHT: pipe through script
tdx query --output json <query> | node render.js

# ❌ WRONG: read directly
cat /tmp/query_results.json  # (then I would read this)
```

---

### Always verify column names against actual table schema
Run `tdx describe <db>.<table>` first. Copy exact column names (case-sensitive). 

**Why:** Past incident: dashboard showed all zeros because `customers` ≠ `total_customers` in the SINK schema. This wasted 2 days of debugging.

---

### Always spot-check dashboard numbers against the database
After rendering, pick at least 3 KPIs and verify them with a direct SQL query. Numbers must match to the exact value — not "approximately". If 1% off, something is wrong.

```sql
-- After rendering dashboard showing "Revenue: $4.5M"
-- Spot-check with:
SELECT SUM(amount) FROM revenue_table WHERE date >= 2026-01-01
-- Result must match exactly
```

---

### Always check for join fan-out before writing aggregation queries
If joining 1-to-many tables, count rows before and after. If `after_join > 2 × before_join`, you have fan-out.

**Fix = pre-aggregate the many-side table into a subquery keyed on the join key BEFORE joining. Never aggregate after a fan-out join.**

```sql
-- WRONG: aggregate after join → inflated totals
SELECT SUM(o.amount) FROM customers c JOIN orders o ON c.id = o.customer_id

-- RIGHT: pre-aggregate orders, then join
SELECT SUM(agg.total) FROM customers c
JOIN (
  SELECT customer_id, SUM(amount) AS total FROM orders GROUP BY customer_id
) agg ON c.id = agg.customer_id
```

**Past incident:** Revenue inflated from $4.32M to $6.86M due to undetected fan-out on a 1-to-many orders join. Required complete Phase 2 rework.

---

### Always use GROUP BY on only the dimensions the dashboard actually filters on
Source tables are often built at a finer grain than the dashboard needs. Identify which dimensions the filters use, then GROUP BY only those.

**Past incident:** 7-dimension table caused 60% row inflation when only 4 dimensions were filtered. Dashboard metrics were systematically high.

---

## 2. Database & Query Execution

### Never assume a database name — always AskUserQuestion
Use `tdx databases` to discover available databases, then present them as options. Never guess or infer the database name.

```javascript
// ✅ RIGHT: discover and ask
tdx databases
// → present as AskUserQuestion options

// ❌ WRONG: assume
// "probably the database is called 'analytics'"
```

---

### Always run queries in parallel
Every data-generation script must use `Promise.all()` for all independent queries. Sequential queries multiply latency.

**Past incident:** 5 sequential queries took 5s total; switching to `Promise.all()` brought it to 1.5s — 70% faster.

```javascript
// ✅ RIGHT
const [a, b, c] = await Promise.all([queryA(), queryB(), queryC()]);

// ❌ WRONG
const a = await queryA();  // wait
const b = await queryB();  // wait again
const c = await queryC();  // wait again
```

---

### Always add --limit for tables that may return more than 40 rows
`tdx query` defaults to 40 rows. Dimension tables and histogram buckets regularly have 60+ rows. Missing rows cause silent gaps in charts.

```bash
# ✅ RIGHT: explicit limit
tdx query --limit 100 "SELECT * FROM dimensions"

# ❌ WRONG: relies on default
tdx query "SELECT * FROM dimensions"  # returns only 40 rows, silent truncation
```

---

### SELECT only the columns the template actually reads
Before writing any query, map what the template displays, then fetch only those columns.

**Past incident:** Unused columns added 445 KB to every response, slowing rendering and burning context.

---

## 3. state.md Preservation & Validation (All Phases Critical)

**RULE: state.md integrity must be verified before starting or resuming any phase.**

### Validation Checklist (CANNOT BYPASS)

Before starting any phase, run this validation:

```
✓ state.md file exists (not deleted/lost)?
✓ Previous phases are APPENDED (not overwritten)?
✓ Each phase section has a date and status?
✓ "Next Action" pointer is present at bottom?
✓ If re-read after compaction: checkpoint proof present?

If ANY fails:
  → STOP immediately
  → CANNOT proceed to next phase
  → Ask user to recover or confirm restart
```

### What Happens if state.md Fails Validation

```
CANNOT proceed. State is corrupted or lost.

Options:
1. User provides state.md contents (paste it back)
2. User confirms recovery is possible (contact support)
3. User wants to restart phase from beginning
4. User wants to restart entire project (Phase 1)

I will not guess or proceed without confirmed state.
```

**Why:** state.md is the single source of truth across phases. Without it, context is lost, decisions can't be traced, and phase continuity breaks.

---

## 4. Rendering (HTML Client only)

This skill produces a single portable `dashboard.html` file with data inlined at build time — no server, no separate API calls at runtime. This is the only rendering pattern in scope.

### Never use Python for file I/O — always use Node.js
`python3` is not the standard for this pipeline. All file reading, template injection, and output writing must use `node` (`generate-data.js` / `render.js`).

---

### Never let AI read raw query results directly — always pipe through script
Same rule as Data Integrity above: query results flow `tdx query → node script → dashboard.html`, never through AI context.

---

### Know the rendering floor before optimizing queries

- `tdx query` per query → ~0.5s floor
- `node` script startup → ~0.3s floor

If queries are already faster than this floor, stop optimizing SQL — you're limited by script startup, not the queries.

---

### Always confirm the final HTML file opens standalone
Since the whole point of HTML Client is portability, open `dashboard.html` directly in a browser (no dev server) before calling it done. If it needs a running server to render, the build is wrong.

```bash
# ✅ RIGHT: opens standalone
open dashboard.html  # browser opens it directly

# ❌ WRONG: requires server
# "You need to run 'npm start' to view this"
```

---

## 4. Requirements & Definitions

### Always resolve metric definition ambiguity explicitly — offer "show both"
When two valid definitions exist (e.g., unique openers vs total opens), don't force a binary choice. Propose showing both.

**Past incident:** Email open rate was 30.5% (unique) vs 87.4% (total) — showing both resolved the ambiguity and gave more insight.

```markdown
# Open Rate

There are two valid definitions:
1. **Unique opens:** X% (each user counted once)
2. **Total opens:** Y% (all opens counted)

Which would you like? Or both?
```

---

## 5. Queries & Performance

### Always test cardinality before aggregating
For any multi-table query, verify that the join doesn't fan-out:

```sql
-- Step 1: Count unique keys on both sides
SELECT COUNT(DISTINCT customer_id) FROM customers;           -- 1,000
SELECT COUNT(DISTINCT customer_id) FROM orders;              -- 950

-- Step 2: Counts should match (or be within 5%)
-- If orders has MORE distinct keys than customers, you have an orphaned record problem
-- If customers has more, that's OK (not all customers have orders)

-- Step 3: Join and check row count
SELECT COUNT(*) FROM customers c JOIN orders o ON c.id = o.customer_id;
-- Should be close to orders row count
```

---

### Always include partition pruning in Trino/Hive queries
Use `td_time_range()` or `td_interval()` to filter on date partitions:

```sql
-- ✅ RIGHT: partition pruning
SELECT * FROM events
WHERE TD_TIME_RANGE(time, '2026-01-01', '2026-07-31')

-- ❌ WRONG: scans all partitions
SELECT * FROM events WHERE DATE(time) >= '2026-01-01'
```

---

## 6. Phase-Specific Constraints

### Phase 1: Requirements must be validated against real data
- ✅ Ask for business requirements (Stage A)
- ✅ Validate against real data (Stage B: `tdx describe`, sample queries)
- ✅ Never accept "I assume the data exists" — verify it exists
- ✅ Calculate promotion score (0-6) based on data complexity + refresh frequency

---

### Phase 2: SINK tables must be created ONLY after user approval
- ✅ Present cost/scope/naming plan
- ✅ Get explicit YES/NO approval
- ✅ If NO → gather feedback, adjust, re-present
- ✅ Never skip this approval gate

---

### Phase 3: Dashboard rendering must be tested against real data
- ✅ Query SINK tables (Phase 2 output) or source tables directly
- ✅ Verify query results before rendering
- ✅ Spot-check at least 3 KPIs
- ✅ Test filters and interactivity manually
- ✅ Confirm HTML opens standalone

---

### Phase 4: Agent knowledge bases must include spot-checked values
- ✅ Schema alone is not enough
- ✅ KBs must include confirmed values (e.g., "Total customers (confirmed): 1,000")
- ✅ Use these as ground truth anchors so agents don't compute from fan-out data

**Past incident:** Stale KB referenced deprecated table name. Agent returned wrong numbers. Debugging took 4 hours.

---

### Phase 5: Handoff docs must be testable
- ✅ Access & Ownership guide must have actual names (not "contact your admin")
- ✅ Runbook must include actual table/workflow names
- ✅ Architecture doc must diagram the real flow

---

## 7. Tooling & Implementation Best Practices

### Python Zipfile as Default (Phase 4 packaging)

When packaging extracted skills or agent projects, use **Python's `zipfile` module** as the default, not shell `zip` command.

**Why:** `zip` command is not guaranteed to exist on all systems (especially Windows). Python is always available in this environment.

**Pattern:**

```python
# ✅ RIGHT: Python zipfile (always available)
import zipfile
import os

def create_skill_zip(skill_dir, output_path):
    with zipfile.ZipFile(output_path, 'w', zipfile.ZIP_DEFLATED) as zf:
        for root, dirs, files in os.walk(skill_dir):
            for file in files:
                file_path = os.path.join(root, file)
                arcname = os.path.relpath(file_path, skill_dir)
                zf.write(file_path, arcname)

create_skill_zip('./fde-dashboard-skill', './fde-dashboard-skill.zip')
```

**Not:**
```bash
# ❌ WRONG: shell zip (may not exist)
zip -r fde-dashboard-skill.zip fde-dashboard-skill/
```

---

### State.md Size Discipline (All phases)

**Each phase block in state.md must not exceed 50 lines.** Large artifacts go in referenced files.

**Structure:**
```yaml
## Phase 2 — Workflow Deployment
**Date:** 2026-07-22
**Status:** ✅ Complete
**Duration:** 1.5 hours

### SINK Tables Created
- project_sink_revenue (365 rows, date-only aggregation)
- project_sink_customers (1.2M rows, pre-deduplicated)

### Performance
- Query latency: 0.8s avg
- Incremental load: 2min per day

### SINK → Dashboard Mapping
See: `./phase-2/sink-dashboard-mapping.md`

### Data Validation
- ✅ 3+ KPIs verified against source
- ✅ Join fan-out checked
- ✅ COUNT DISTINCT pattern validated

### Next Action
→ Proceed to Phase 3: Build Interactive Dashboard
```

**Large artifacts go elsewhere:**
- Full SQL queries → `./phase-2/queries/`
- SINK schema → `./phase-2/references/sink-schema.md`
- Join validation analysis → `./phase-2/references/join-analysis.md`

**Why:** Keeps state.md navigable and reviewable. By Phase 5, a large state.md becomes unusable.

**Guidelines:**
- Phase 1 block: ≤ 40 lines (requirements + data findings summary)
- Phase 2 block: ≤ 50 lines (SINK summary + link to detailed schema)
- Phase 3 block: ≤ 45 lines (dashboard summary + link to full plan)
- Phase 4 block: ≤ 35 lines (skill/agent summary + link to full config)
- Phase 5 block: ≤ 30 lines (handoff summary + link to docs)

**"Next Action" must always be at the bottom of each phase block and stay ≤ 2 lines.**

---

### Phase Completion: Always Proactively Present Next Step (ENFORCEMENT)

**⚠️ CRITICAL: After EVERY phase completes, immediately present the next step with a default action. NEVER wait for user to ask "what's next?"**

**Pattern (apply after each phase completes):**

```
✅ Phase [N] Complete

### Summary
[2-3 line summary of what was accomplished]

### Next Step
Ready for Phase [N+1]? Here's what we'll do:

**Option A (Recommended):** Phase [N+1] — [Description]
→ We'll [action 1], [action 2], [action 3]
→ Estimated time: X hours
→ Requires: [prerequisites]

**Option B:** Skip to Phase [M]
→ Reason: [when this makes sense]

**Option C:** Close engagement
→ Your dashboard is complete and ready to share

**→ Which would you like? (A/B/C or describe something different)**
```

**Examples:**

**After Phase 1:**
```
✅ Phase 1 Complete — Requirements validated against live data

### Summary
- 4 KPIs identified: Revenue, Customers, Order Value, Churn
- 3 source tables confirmed: sales_events, customers, returns
- Promotion Score: 5 (workflow recommended)

### Next Step
Ready for Phase 2? Here's what we'll do:

**Option A (Recommended):** Phase 2 — Build Pre-Aggregation Workflow
→ We'll create SINK tables for fast dashboard queries
→ Estimated time: 2-3 hours
→ Requires: Treasure Data account access + write permissions

**Option B:** Skip to Phase 3 (Direct Build)
→ We'll build the dashboard directly from source tables
→ Reason: Simpler but slower queries

**Option C:** Close engagement
→ Your requirements are documented in state.md

**→ Which would you like? (A/B/C or describe something different)**
```

**After Phase 3:**
```
✅ Phase 3 Complete — Dashboard approved and ready

### Summary
- 5 tabs rendered: KPI Overview, Analysis, Data Explorer, Insights, Export
- All 3 KPIs spot-checked (Revenue ±0%, Customers ±0.2%, AOV ±0.1%)
- 2.3 MB HTML file, loads in 3 seconds

### Next Step
What's next for your dashboard?

**Option A (Recommended):** Phase 4A — Extract Reusable Skill
→ We'll package this dashboard as a reusable Claude skill
→ Estimated time: 1 hour
→ Benefit: Future dashboards for similar projects build 10× faster

**Option B:** Phase 4B — Deploy Conversational Agent
→ We'll create a Foundry agent so users can ask questions in natural language
→ Estimated time: 1.5 hours
→ Benefit: Users ask "What's revenue by region?" instead of opening filters

**Option C:** Phase 5 — Documentation & Handoff
→ We'll create runbooks, architecture docs, and access guides
→ Estimated time: 1 hour
→ Benefit: CS/ops teams can troubleshoot without escalating

**Option D:** Close engagement
→ Your dashboard is ready to share with stakeholders

**→ Which would you like? (A/B/C/D or describe something different)**
```

**Why:**
- Users don't know the phase structure (they don't read INSTRUCTIONS.md)
- "What's next?" questions waste time and context (they pull conversation off-track)
- Default recommendations steer projects toward value (A is always the most common next step)
- Options give agency (user chooses, not forced forward)
- Prevents stalls ("do I continue or close?")

**Timing:**
- **Immediate:** Present within the same message that announces completion
- **Not:** Wait for user to say "thanks, what's next?"
- **Not:** Say "Phase 3 is complete" and then wait
- **Not:** Offer generic "Do you want to continue?" — be specific about what Phase N+1 contains

**Example bad pattern (❌ DO NOT DO THIS):**
```
✅ Phase 3 complete. Ready for Phase 4?
```

**Example good pattern (✅ DO THIS):**
```
✅ Phase 3 Complete

### Summary
[...]

### Next Step
Ready for Phase 4A (Extract Skill)?
→ We'll create a reusable skill from this dashboard
→ Time: 1 hour
→ Or prefer Phase 4B (Agent) or Phase 5 (Docs)?

→ Your choice: 4A / 4B / 5 / or close engagement
```

**In state.md:**
Always end each phase with a "Next Action" pointer that references this completed phase:
```yaml
## Next Action
✅ Phase 3 Complete. Ready for Phase 4A (Extract Skill)?
→ Choose: 4A (extract skill) / 4B (agent) / 5 (docs) / close
```

---

## 8. If an Instruction Cannot Be Followed

**STOP immediately.** Return to user with:

1. **Which instruction is blocking**
   > "Rule 4 (Join Validation): Cannot validate join cardinality"

2. **Why it's blocking**
   > "The query to count distinct keys in `orders` timed out after 60s"

3. **What's needed to proceed**
   > "I need either: (a) you to run the query manually and share the count, (b) a smaller date range to query, or (c) confirmation that the join is correct"

**Never skip an instruction because it's "taking too long" or "seems optional". Stop and ask.**

---

**Version:** 1.0.0
**Last Updated:** 22 July 2026
**Load Order:** 1 (read second)
