---
name: phase-1-instructions
description: Phase 1 specific instructions for requirements gathering and data discovery
priority: HIGH
load_order: 1.1
---

# Phase 1 Instructions — Requirements & Data Discovery

**Read this BEFORE `./SKILL.md`.**

---

## Phase 1 Goal

Gather business requirements (Stage A), validate them against real data (Stage B), calculate promotion score (0-6), decide Workflow vs Non-Workflow path, and get user approval — **all in one session**.

**Deliverable:** `./<project-slug>/state.md` containing approved requirements, confirmed data findings, promotion score, and path decision.

---

## Phase 1 Specific Rules (In Addition to Universal Rules)

### Rule P1-1: Requirements Must Be Validated Against Real Data

- [ ] **Stage A:** Gather business requirements (no data validation yet)
  - KPIs: What are they measuring?
  - Dimensions: How should metrics be sliced?
  - Filters: What does the audience need to filter on?
  - Audience: Who will use this?

- [ ] **Stage B:** Validate requirements against real data
  - Run `tdx describe <db>.<table>` for each source table
  - Verify columns exist (exact spelling, case-sensitive)
  - Test joins with sample queries
  - Verify time columns are present and non-nullable
  - Never accept "I assume the data exists" — verify with `tdx query`

**Why:** Many requirements fail at validation. Better to discover this in Phase 1 than Phase 3.

**Example validation query:**
```sql
-- Verify KPI column exists
SELECT SUM(revenue) FROM sales_table LIMIT 5;

-- Verify time column
SELECT MIN(created_at), MAX(created_at) FROM sales_table;

-- Verify dimension exists
SELECT DISTINCT country FROM sales_table LIMIT 10;
```

---

### Rule P1-2: Promotion Score Calculation Is Deterministic

**Score formula (0-6):**
- **0-2 (Low complexity):** Simple metrics, few tables, no joins, daily or less frequent refresh
  - **Path:** Phase 1 → Phase 3 (skip Phase 2)
  
- **3 (Optional):** Medium complexity, user's choice
  - **Path:** User decides Phase 2 (workflow) or Phase 3 (direct build)
  
- **4-6 (High complexity):** Many tables, complex joins, sub-hourly refresh, aggregation required
  - **Path:** Phase 1 → Phase 2 → Phase 3

**Scoring breakdown:**
- +1 if 2+ tables to join
- +1 if join is 1-to-many (fan-out risk)
- +1 if refresh frequency < daily
- +1 if aggregation required (GROUP BY)
- +1 if audience > 5 people (concurrency risk)

**Rule:** Calculate score before asking path decision. The score determines the path.

---

### Rule P1-3: Join Key Validation (Phase 1→2 gate)

If requirements include 2+ tables:

- [ ] Identify join keys
- [ ] Test with sample queries:
  ```sql
  SELECT COUNT(DISTINCT customer_id) FROM customers;
  SELECT COUNT(DISTINCT customer_id) FROM orders;
  -- Counts should match (within 5%)
  ```
- [ ] If counts don't match, flag it for Phase 2 planning (pre-aggregation needed)

**Why:** Undetected join issues cause Phase 2 rework.

---

### Rule P1-4: Time Column Validation (Phase 1 discovery)

- [ ] Identify the business event or insert timestamp column
- [ ] Verify it's non-nullable: `tdx describe <db>.<table>`
- [ ] If nullable, flag as data quality issue — inform user
- [ ] Document the column name in state.md

**Example:**
```yaml
## Time Column Validation

Source Table: sales_events
- Column identified: `event_time` (business event time)
- Nullable: No ✓
- Format: TIMESTAMP
- Range: 2020-01-01 to 2026-07-22
```

---

### Rule P1-5: Special Case Paths (Sisense .dash or Treasure Insights API)

If user provides:
- **`.dash` file** → Follow `.dash` Special Case path (skip to dashboard prefilling)
- **Datamodel name/OID** → Follow Treasure Insights API Special Case path (fetch schema via API)
- **Both** → Follow Combined Resources path (validate consistency)
- **Neither** → Follow normal Stage A/B path

**Do NOT mix paths.** Once on a special path, stay on it.

**Why:** Special case paths have their own prefilling logic. Mixing causes duplicate questions.

---

### Rule P1-6: Negative Confirmation (Ask What NOT to Include)

At the end of requirements (Stage A), explicitly ask:

> "Is there anything you specifically do NOT want in this dashboard?"

Common answers:
- "Don't include revenue from EU (privacy)"
- "Don't show customer names (PII)"
- "Don't include last 24h (data quality lag)"

Document as "Exclusions" in state.md.

---

### Rule P1-7: Dimensions Validation

For each dimension user requests:

- [ ] Verify it exists in data: `SELECT DISTINCT <dimension> FROM <table> LIMIT 10`
- [ ] Check cardinality: `SELECT COUNT(DISTINCT <dimension>) FROM <table>`
- [ ] If cardinality > 1000, warn user (may be too granular for dashboard filters)

**Example:**
```yaml
## Dimensions Validation

- Dimension: country
  - Exists: ✓ (SELECT DISTINCT country ... returned 195 rows)
  - Cardinality: 195 (acceptable for filter)
  
- Dimension: customer_id
  - Exists: ✓
  - Cardinality: 1,200,000 (⚠️ WARNING: Very high cardinality, not suitable for filter)
  - Recommendation: Use name instead or provide search autocomplete
```

---

### Rule P1-8: Metrics Validation

For each metric user requests:

- [ ] Verify column exists and is numeric
- [ ] Check for missing/null values
- [ ] Validate with a sample aggregation

**Example queries:**
```sql
-- Verify column exists and is numeric
SELECT SUM(revenue) FROM sales_table LIMIT 5;

-- Check for nulls
SELECT COUNT(*) total, COUNT(revenue) non_null, 
       COUNT(*) - COUNT(revenue) AS nulls 
FROM sales_table;

-- Quick aggregation test
SELECT SUM(revenue) FROM sales_table WHERE DATE(created_at) = '2026-07-22';
```

---

### Rule P1-9: State.md Creation

At end of Phase 1, create `state.md` with this structure:

```yaml
# Dashboard Project: [project-slug]

## Phase 1 — Requirements & Data Discovery
**Date:** [date]
**Status:** ✅ Complete

### Business Requirements (Stage A)

**Primary Goal:** [1-2 sentence summary]

**KPIs:**
- [metric 1]: [definition]
- [metric 2]: [definition]

**Dimensions:**
- [dimension 1]: [cardinality]
- [dimension 2]: [cardinality]

**Filters:**
- [filter 1]: [options]
- [filter 2]: [options]

**Audience:** [names/roles]

**Exclusions:** [what NOT to include]

### Data Findings (Stage B)

**Source Tables:**
- [table 1]: [columns], [row count], [last updated]
- [table 2]: [columns], [row count], [last updated]

**Time Column:**
- Table: [table name]
- Column: [column name]
- Format: [TIMESTAMP/DATE/etc]
- Nullable: [YES/NO]

**Join Keys (if 2+ tables):**
- [table 1] ↔ [table 2]: [join key], [cardinality validated: YES/NO]

**Validation Results:**
- All columns verified: ✓
- All joins tested: ✓
- Time column present: ✓

### Promotion Score

**Final Score:** [X/6]

**Breakdown:**
- Multiple tables: [+1 if yes]
- 1-to-many joins: [+1 if yes]
- Sub-daily refresh: [+1 if yes]
- Aggregation required: [+1 if yes]
- High concurrency (5+ users): [+1 if yes]

### Path Decision

**Recommended Path:** Phase 1 → [Phase 2 → Phase 3 OR Phase 3]

**Justification:** Score X indicates [workflow/non-workflow] path

**User Approval:** [YES / PENDING]

---

## Next Action

1. ✓ Phase 1 complete
2. User to review state.md and approve
3. If Score ≤2: Jump to Phase 3 (`../phase-3/INSTRUCTIONS.md`)
4. If Score 3: User decides Phase 2 or Phase 3
5. If Score ≥4: Jump to Phase 2 (`../phase-2/INSTRUCTIONS.md`)
```

---

## Phase 1 Pre-Flight Checklist

**Before starting Phase 1 Stage A, verify:**

### Engagement Setup
- [ ] Engagement type confirmed: new project or resuming existing?
- [ ] If resuming: project slug confirmed, state.md located
- [ ] User understands Phase 1 goal (1-2 hours, produces state.md)

### Data Access
- [ ] TD account accessible (`tdx auth show` works)
- [ ] User can access required databases (`tdx databases` list returned)
- [ ] At least one source table accessible via `tdx query`

### Requirements Clarity
- [ ] User has basic idea of KPIs (even if rough)
- [ ] User has basic idea of audience (who will use this?)
- [ ] User willing to validate requirements against real data (Phase 1 Stage B)

---

## Phase 1 Decision Tree

```
START Phase 1 Stage A: Gather Requirements
    ↓
Ask: KPIs? Dimensions? Filters? Audience?
    ↓
START Phase 1 Stage B: Validate Against Data
    ↓
For each source table:
    - Run: tdx describe <db>.<table>
    - Verify columns exist (exact spelling)
    - Test joins with sample queries
    ↓
Calculate Promotion Score (0-6)
    ↓
DECISION POINT:
    Score 0-2?  → Route to Phase 3 (skip Phase 2)
    Score 3?    → Ask user: Phase 2 (workflow) or Phase 3 (direct)?
    Score 4-6?  → Route to Phase 2 (workflow)
    ↓
Create state.md with requirements + findings
    ↓
Get user approval on state.md
    ↓
END Phase 1
```

---

## Common Phase 1 Blocks

### "I don't know my KPIs"
**Response:** 
> "That's OK. Let's start with a metric you want to track. Common examples: Revenue, Customer Count, Churn, Conversion Rate, NPS. What matters most to your business right now?"

### "The data might not exist"
**Response:**
> "Let's check! I'll run `tdx describe` on the table. If the column doesn't exist, we have two options: (1) find an alternative metric in the data, or (2) flag this as a data gap for your team to build."

### "I have 20 metrics I want"
**Response:**
> "Great. Let's prioritize by importance. Which 3-5 are must-haves? We can always add more later. (Phase 1 goal is to validate the core requirements, not build everything.)"

### "I'm not sure if the join will work"
**Response:**
> "Perfect. Let me test it. I'll run: `SELECT COUNT(DISTINCT customer_id) FROM customers` and `SELECT COUNT(DISTINCT customer_id) FROM orders`. If counts match (within 5%), the join is clean. If not, we have a fan-out issue that Phase 2 will handle."

---

## After Phase 1 Completes

**If Score 0-2:**
```
✓ Phase 1 Complete → Jump to Phase 3
  Read: ../phase-3/INSTRUCTIONS.md
  Then: ../phase-3/SKILL.md
```

**If Score 3:**
```
✓ Phase 1 Complete → User Decision
  Option A: Phase 2 (workflow for scheduled refresh)
    Read: ../phase-2/INSTRUCTIONS.md
  Option B: Phase 3 (direct build, no workflow)
    Read: ../phase-3/INSTRUCTIONS.md
```

**If Score 4-6:**
```
✓ Phase 1 Complete → Phase 2 Required
  Read: ../phase-2/INSTRUCTIONS.md
  Then: ../phase-2/SKILL.md
```

---

**Version:** 1.0.0
**Last Updated:** 22 July 2026
**Load Order:** 1.1 (read after master INSTRUCTIONS.md)
