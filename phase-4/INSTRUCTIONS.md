---
name: phase-4-instructions
description: Phase 4 specific instructions for automation and agent deployment
priority: MEDIUM
load_order: 1.4
---

# Phase 4 Instructions — Automate & Deploy Agents

**Read this BEFORE `./SKILL.md`.**

**Phase 4 is OPTIONAL (after Phase 3 approved).**

---

## Phase 4 Goal

Extend the dashboard with two optional automation tracks:

**Track A: Extract Reusable Skill**
- Convert Phase 3 dashboard to a reusable Claude Code skill
- Enables faster builds of similar dashboards in the future

**Track B: Deploy Conversational Agent**
- Create a Foundry agent with access to dashboard data
- Users can ask questions in natural language
- Agent queries data on-the-fly or uses precomputed tables

**Deliverable:**
- Track A: A reusable skill (SKILL.md + references/)
- Track B: A Foundry agent project (agent.yml + knowledge base)
- state.md appended with Phase 4 results

---

## Phase 4 Specific Rules (In Addition to Universal Rules)

### Rule P4-0: Dry-Run First (Track B Agent Only)

**⚠️ CRITICAL (Track B only): Test agent with sample queries BEFORE deployment.**

**Step 1: Deploy agent to staging/test environment**

```bash
# Create agent in test project (not production)
tdx agent push --project [project_slug]-test
```

**Step 2: Run sample queries to verify KB accuracy**

Test the knowledge base:
```
Query 1: "What is the total revenue for last month?"
  KB says: Revenue table = [table_name]
  Agent queries: SELECT SUM(revenue) FROM [table_name]...
  Result: Returns value ✓

Query 2: "Show me customers by region"
  KB says: Regions available = US, EU, APAC
  Agent queries: SELECT * FROM [table] WHERE region IN (...)
  Result: Returns expected regions ✓

Query 3: "What's the churn rate?"
  KB says: Churn rate = (churned / total) customers
  Agent queries: SELECT COUNT(*) ... correctly calculates ✓
```

**Step 3: Verify KB spot-checks**

Before production deployment:
- [ ] All tables referenced in KB actually exist in TD
- [ ] All sample values in KB are current (not stale)
- [ ] All query patterns in KB run without errors
- [ ] Agent responds with expected data (no hallucinations)

**If any test fails:**
- Fix KB (update table names, refresh spot-check values)
- Redeploy to test
- Re-run tests
- Repeat until all pass

**Step 4: Only after all tests pass**
- Deploy to production
- Proceed to approval gate

**Why:** Agent KB accuracy = trust. Testing in staging prevents production failures.

---

### Rule P4-1: Track A Approval Gate — Skill Creation (ENFORCEMENT)

**⚠️ CRITICAL: CANNOT extract skill without explicit user approval.**

**Step 1: Present approval template (COPY/PASTE exactly):**

```
📋 Ready to create reusable skill:

Skill Name: [name]
Description: [1-2 sentences]

Skill will:
  • Accept similar data structures (same source tables/columns)
  • Generate dashboard.html in <30 minutes (typical)
  • Be shareable with other teams
  • Be stored at: [path]

Skill components:
  • SKILL.md (instructions)
  • INSTRUCTIONS.md (rules)
  • CHECKLIST.md (quick reference)
  • references/ (query templates, theme)
  • tests/ (end-to-end verification)

To approve, please type exactly: YES, APPROVE SKILL EXTRACTION FOR [SKILL_NAME]
```

**Step 2: Wait for response**
- If user types: "YES, APPROVE SKILL EXTRACTION FOR [SKILL_NAME]" → Proceed ✓
- If user types: "NO" or anything else → **STOP** ✗

**Step 3: If NO**
- Do not create skill
- Ask: "Would you like to skip Track A?"
- If yes: Move to Track B or Phase 5

**If user says "just extract it":**
> "I cannot proceed without explicit approval. Skill extraction creates a new asset that will be reused across projects. I need your confirmation.
>
> Please type exactly: YES, APPROVE SKILL EXTRACTION FOR [SKILL_NAME]"

---

### Rule P4-2: Track B Approval Gate — Agent Creation (ENFORCEMENT)

**⚠️ CRITICAL: CANNOT deploy agent without explicit user approval.**

**Step 1: Present approval template (COPY/PASTE exactly):**

```
📋 Ready to deploy Foundry agent:

Agent Name: [name]
Description: [1-2 sentences]

Agent capabilities:
  • Answer natural language questions about dashboard metrics
  • Query live or precomputed tables
  • Support conversational analysis
  • Audit trail (all queries logged, read-only access)

Cost impact:
  • Agent queries cost: ~$X per query
  • Monthly estimate: ~$X (depending on usage)
  • Monthly budget alert: [set if applicable]

Access control:
  • Who can access: [internal / shared / public]
  • Permissions: Read-only (SELECT only, no INSERT/UPDATE/DELETE)
  • Can be disabled: Yes (by revoking permissions)

To approve, please type exactly: YES, APPROVE AGENT DEPLOYMENT FOR [AGENT_NAME]
```

**Step 2: Wait for response**
- If user types: "YES, APPROVE AGENT DEPLOYMENT FOR [AGENT_NAME]" → Proceed ✓
- If user types: "NO" or anything else → **STOP** ✗

**Step 3: If NO**
- Do not deploy agent
- Ask: "Would you like to skip Track B?"
- If yes: Move to Phase 5

**If user says "just deploy it":**
> "I cannot proceed without explicit approval. Agent deployment creates a new conversational interface to your data. I need your confirmation.
>
> Please type exactly: YES, APPROVE AGENT DEPLOYMENT FOR [AGENT_NAME]"

---

### Rule P4-3: Track A — Skill Structure

Extracted skill must follow standard structure:

```
skill-name/
├── SKILL.md (orchestrator + instructions)
├── INSTRUCTIONS.md (skill-specific rules)
├── references/
│   ├── queries/
│   │   ├── kpi_1.sql
│   │   ├── kpi_2.sql
│   │   └── ...
│   ├── templates/
│   │   └── dashboard.html.template
│   ├── treasure-data-theme.md
│   └── [other references]
├── tests/
│   ├── test-query-execution.js
│   ├── test-html-rendering.js
│   └── test-spot-check.js
└── .claude-plugin/
    └── marketplace.json (if shareable)
```

---

### Rule P4-4: Track A — Query Templates Must Be Parameterized

All extracted queries must accept parameters:

```sql
-- ✅ RIGHT: parameterized for reuse
SELECT 
  DATE(event_time) AS date,
  region,
  SUM(amount) AS revenue
FROM {{SOURCE_TABLE}}
WHERE event_time >= '{{START_DATE}}'
  AND event_time < '{{END_DATE}}'
GROUP BY DATE(event_time), region
ORDER BY date DESC, region
```

**Not:**
```sql
-- ❌ WRONG: hardcoded to this project
SELECT ... FROM sales_events WHERE event_time >= '2026-07-01'
```

**Why:** Reusable skill accepts different source tables + date ranges.

---

### Rule P4-5: Track B — Agent Knowledge Base

Foundry agent knowledge base MUST include:

1. **Schema documentation** (table names, column names, types)
2. **Spot-checked values** (e.g., "Total customers: 1,200,000")
3. **Dimension options** (e.g., "Regions: US, EU, APAC")
4. **Query patterns** (how to filter, aggregate, join)

**Example KB section:**
```yaml
tables:
  - name: sales_revenue
    description: "Daily revenue by region"
    columns:
      - date: "Date (YYYY-MM-DD)"
      - region: "Region code (US, EU, APAC)"
      - revenue: "Revenue amount (USD)"
    verified_values:
      - "Total revenue (all time): $45.2M"
      - "Average daily: $125K"
    sample_query: |
      SELECT SUM(revenue) FROM sales_revenue 
      WHERE date >= '2026-01-01'

dimensions:
  - region: ["US", "EU", "APAC"]
  - date_range: ["Last 7 days", "Last 30 days", "YTD"]
```

---

### Rule P4-6: Track B — Agent Approval Before Tools Access

Foundry agent MUST NOT access production systems without explicit approval:

- [ ] Agent queries are read-only (SELECT only, no INSERT/UPDATE/DELETE)
- [ ] Agent has access to SINK tables (Phase 2) or limited source tables
- [ ] Agent queries are audited (all user queries logged)
- [ ] User can disable agent at any time (revoke permissions)

---

### Rule P4-7: Track A — Skill Must Be Tested End-to-End

Before finalizing extracted skill:

- [ ] Test 1: Use skill to build a dashboard from scratch
- [ ] Test 2: Verify output matches Phase 3 dashboard structure
- [ ] Test 3: Spot-check numbers from Test 1 dashboard against DB

```javascript
// test-skill-end-to-end.js
async function testSkill() {
  // 1. Load skill
  const skill = loadSkill('skill-name');
  
  // 2. Run with test inputs
  const result = await skill.build({
    source_table: 'test_events',
    start_date: '2026-07-01',
    metrics: ['revenue', 'customers']
  });
  
  // 3. Verify output
  assert(result.html_file_exists, 'dashboard.html created');
  assert(result.contains('Total Revenue'), 'KPI present');
  
  // 4. Spot-check
  const dashValue = extractFromHTML(result, 'Total Revenue');
  const dbValue = await queryDB('SELECT SUM(amount) FROM test_events');
  assert(dashValue === dbValue, 'numbers match');
}
```

---

### Rule P4-8: Track B — Agent Knowledge Base Must Be Accurate

Before deploying agent:

- [ ] All table names verified (match actual DB schema)
- [ ] All column names verified (exact spelling, case-sensitive)
- [ ] Spot-checked values confirmed current (not stale)
- [ ] Query patterns tested (run sample queries, verify results)

**Validation checklist:**
```bash
# Verify table exists
tdx describe <db>.sales_revenue

# Verify columns
tdx query "SELECT * FROM sales_revenue LIMIT 1"

# Verify sample aggregate
tdx query "SELECT COUNT(*), SUM(revenue) FROM sales_revenue"
# Compare against KB documented values
```

---

### Rule P4-9: State.md Append (Phase 4)

Append Phase 4 results to state.md:

```yaml
---

## Phase 4 — Automate & Deploy Agents
**Date:** [date]
**Status:** ✅ Complete

### Track A: Reusable Skill

- **Skill Name:** [name]
- **Repository:** [path or URL]
- **Status:** ✓ Extracted and tested
- **Tests:** [N] end-to-end tests passing

**Queryable Parameters:**
  - source_table: [default or options]
  - start_date: [format]
  - end_date: [format]
  - metrics: [list]
  - dimensions: [list]

---

### Track B: Foundry Agent

- **Agent Name:** [name]
- **Visibility:** [internal / shared / public]
- **Status:** ✓ Deployed and tested
- **Knowledge Base:** [tables], [dimensions], [verified values]

**Agent Capabilities:**
  - Query 1: [description]
  - Query 2: [description]
  - Query 3: [description]

**Access Control:**
  - Read-only access: ✓
  - Audit logging: ✓
  - Can be disabled: ✓

---

### Validation Results

- [ ] Track A: End-to-end skill test passed
- [ ] Track A: Output matches Phase 3 structure
- [ ] Track B: KB tables verified against DB schema
- [ ] Track B: KB spot-checked values confirmed current
- [ ] Track B: Sample agent queries return expected results

---

## Next Action

1. ✓ Phase 4 complete (optional automation done)
2. Optional: Phase 5 (handoff documentation)
3. Or: Project complete and ready for production
```

---

## Phase 4 Pre-Flight Checklist

**Before starting Phase 4, verify:**

### Prerequisites
- [ ] Phase 3 dashboard is approved and working
- [ ] state.md exists and up-to-date
- [ ] Decision made: Track A (skill), Track B (agent), or both?

### Track A Setup (if chosen)
- [ ] Phase 3 dashboard queries are parameterized
- [ ] Reusable skill structure designed
- [ ] Template variables identified
- [ ] Test plan created

### Track B Setup (if chosen)
- [ ] Foundry agent is new or existing project available
- [ ] Knowledge base schema documented
- [ ] Query patterns identified
- [ ] Access control planned (read-only verified)

---

## Phase 4 Decision Tree

```
START Phase 4: Automate & Deploy Agents
    ↓
Is Phase 3 dashboard approved?
    NO  ↓ → ERROR: Cannot proceed without Phase 3 complete
    YES ↓ → Continue
    ↓
Decide: Track A (skill), Track B (agent), or both?
    ↓
TRACK A FLOW (if chosen):
    ├─ Get user approval (Rule P4-1)
    ├─ Extract queries to templates
    ├─ Create SKILL.md + INSTRUCTIONS.md
    ├─ Create test suite
    ├─ Run end-to-end tests
    ├─ Verify output matches Phase 3
    └─ Finalize + document
    ↓
TRACK B FLOW (if chosen):
    ├─ Get user approval (Rule P4-2)
    ├─ Create/load Foundry agent project
    ├─ Document knowledge base (tables + schema)
    ├─ Add spot-checked values
    ├─ Verify KB against actual DB schema
    ├─ Test sample queries
    ├─ Configure access control (read-only)
    └─ Deploy agent
    ↓
Append Phase 4 results to state.md
    ↓
END Phase 4
```

---

## Common Phase 4 Blocks

### Track A: "How do I make this reusable?"
**Response:**
> "Parameterize the queries: replace hardcoded values with {{PLACEHOLDER}}. For example, {{SOURCE_TABLE}}, {{START_DATE}}, {{METRICS}}. Then the skill can accept these as inputs for any similar project."

### Track B: "My agent is giving wrong numbers"
**Response:**
> "Common causes: (1) KB has stale spot-checked values, (2) KB has wrong table/column names, (3) Sample queries in KB aren't matching actual data. Let me verify KB against current schema and update it."

### Track A: "What if the next project has different columns?"
**Response:**
> "Good question. The skill accepts column names as parameters. So for Project 2 with different columns, you provide a mapping: {original_column} → {project_2_column}. The SKILL.md documents how to do this."

### Track B: "Can the agent write to the database?"
**Response:**
> "No — agent has read-only access (SELECT only). This is a security gate to prevent accidental modifications. If you need write access, that would require separate approval + audit logging."

---

## Phase Progression Gate (CANNOT Skip)

**⚠️ CRITICAL: Before proceeding to Phase 5 (optional), verify:**

- [ ] Phase 3 marked COMPLETE in state.md
- [ ] If Track A chosen: skill extracted and end-to-end tests PASSED
- [ ] If Track B chosen: agent deployed and sample queries tested
- [ ] state.md appended with Phase 4 results:
  - Track A: Skill name, location, test results
  - Track B: Agent name, KB accuracy, sample queries verified
  - Both: Completion status
  - "Next Action" pointer (Phase 5 optional or project complete)

**If ANY item is missing or tests failed:**
> "Cannot proceed. Phase 4 incomplete or tests failed.
> Missing/Failed: [specific item]
> Please complete before moving forward."

**Only after all items verified:**
- Append "Phase 4 COMPLETE" to state.md
- If user wants Phase 5: proceed to phase-5/INSTRUCTIONS.md
- Or project can close (automation done, or skipped)

---

## After Phase 4 Completes

**If Track A created:**
```
✓ Reusable skill ready
  - Can be shared with other teams
  - Speeds up future similar dashboards
  - Documented with parameters
```

**If Track B created:**
```
✓ Foundry agent deployed
  - Users can ask questions in natural language
  - Agent queries live or precomputed data
  - All queries logged for audit
```

**Both or neither:**
```
✓ Phase 4 complete (or skipped)
  Optional: Phase 5 (handoff documentation)
  Or: Project complete and production-ready
```

---

**Version:** 1.0.0
**Last Updated:** 22 July 2026
**Load Order:** 1.4 (read after Phase 1-3 INSTRUCTIONS.md)
