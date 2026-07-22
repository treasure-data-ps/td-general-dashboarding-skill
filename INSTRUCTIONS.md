---
name: master-instructions
description: Master instruction anchor read first in every session and after every context compaction
priority: CRITICAL
re_read_after: context_compaction
load_order: 0
---

# Master Instructions — READ THIS FIRST

**⚠️ CRITICAL INSTRUCTION PROTOCOL**

1. **If context was compacted/summarized:** STOP and re-read this entire file before proceeding
2. **Before ANY phase:** Read the phase's `INSTRUCTIONS.md` (see Phase Reference Index below)
3. **Before ANY action:** Verify the inline checklist for that action
4. **Before ANY creation (SINK/workflow/agent):** Get explicit user approval (Section 5)

---

## Load Order Reference

```
Load Order 0:     ./INSTRUCTIONS.md (master — read FIRST)
Load Order 0.5:   ./STATE-CHECKPOINT-SYSTEM.md (runtime validation)
Load Order 1:     ./references/INSTRUCTIONS.md (cross-phase)
Load Order 1.1-1.5: ./phase-N/INSTRUCTIONS.md (phase-specific)
```

---

## Phase Reference Index

| Phase | Read First | Then Read | Checklist Location |
|-------|-----------|-----------|-------------------|
| **Phase 1** | `./phase-1/INSTRUCTIONS.md` | `./phase-1/SKILL.md` | Inline in SKILL.md (Section: "Pre-Flight Checklist") |
| **Phase 2** | `./phase-2/INSTRUCTIONS.md` | `./phase-2/SKILL.md` | Inline in SKILL.md (Section: "Pre-Flight Checklist") |
| **Phase 3** | `./phase-3/INSTRUCTIONS.md` | `./phase-3/SKILL.md` | Inline in SKILL.md (Section: "Pre-Flight Checklist") |
| **Phase 4** | `./phase-4/INSTRUCTIONS.md` | `./phase-4/SKILL.md` | Inline in SKILL.md (Section: "Pre-Flight Checklist") |
| **Phase 5** | `./phase-5/INSTRUCTIONS.md` | `./phase-5/SKILL.md` | Inline in SKILL.md (Section: "Pre-Flight Checklist") |
| **Cross-Phase** | `./references/INSTRUCTIONS.md` | (guardrails + universal rules) | Reference throughout all phases |
| **Runtime Validation** | `./STATE-CHECKPOINT-SYSTEM.md` (load_order: 0.5) | (API checks + recovery) | Reference when resuming after break |

---

## Enforcement Gates (High-Risk Actions — CANNOT BYPASS)

These are **blocking gates**. If conditions are not met, Claude **CANNOT proceed** — not optional, not flexible, not negotiable.

### Gate 1: Approval Before Physical Objects (Phase 2, 4)

**CANNOT create:** SINK tables, workflows, agents, skills
**WITHOUT:** Explicit user approval using gate template

**Process:**
1. Present template: `📋 Ready to create: [details]`
2. **User MUST type:** "YES, APPROVE [ACTION]" (not just "yes")
3. If user types NO/REVIEW/anything else → **STOP completely**
4. Escalate: "What would you like to change?"
5. Adjust plan, re-present
6. ONLY after explicit YES → Proceed

**If user says "just do it anyway":**
> "I cannot proceed without explicit approval. This creates real costs (storage, compute) and real consequences (data modifications) that cannot be easily undone. Please type: YES, APPROVE [ACTION]"

---

### Gate 2: Spot-Checks Pass (Phase 3)

**CANNOT mark dashboard complete**
**WITHOUT:** 3+ KPI spot-checks verified

**Process:**
1. Render dashboard.html
2. Pick 3 KPIs from dashboard
3. For each KPI:
   - Run database query: `SELECT [metric] FROM [table]...`
   - Compare: DB value = Dashboard value?
   - If 1% off or more → **STOP and debug** (don't proceed)
4. **ALL 3 must match exactly**
5. Only then: Present for user approval

**If spot-checks fail:**
> "Dashboard shows [KPI]=[value] but database shows [value]. 1% mismatch. I cannot mark this complete until the numbers match exactly. Debugging..."

---

### Gate 3: State Validation (All Phases)

**CANNOT start or continue any phase**
**WITHOUT:** state.md passes integrity check

**Check before each phase:**
- [ ] state.md file exists (not lost/deleted)
- [ ] Previous phases are appended (not overwritten)
- [ ] "Next Action" pointer is present and clear
- [ ] If re-read after compaction: checkpoint proof present

**If state.md fails check:**
> "state.md integrity check failed:
> - File missing? Lost.
> - Previous phases overwritten? Corrupted.
> - Next action unclear? Can't determine where to resume.
>
> I cannot proceed without recovering state.md.
> User, please share state.md contents or confirm if we should restart Phase N."

---

## Universal Rules (Apply to Every Phase — NEVER VIOLATE)

### Rule 1: Data Integrity

- [ ] Every number comes from live SQL query result (never synthetic, never estimated)
- [ ] Never read raw query output files directly into AI context
- [ ] Always pipe query results through rendering scripts (Node.js)
- [ ] Spot-check at least 3 KPIs against the database after rendering
- [ ] Verify column names against schema BEFORE writing queries

**Why:** Customers spot-check dashboards. Wrong numbers destroy trust. Past incident: dashboard showed "New York #1 destination" — actual was Tokyo with 2× revenue.

---

### Rule 2: Physical Object Approval Gates (MANDATORY)

**Before creating ANY of these, you MUST get explicit user approval:**
- ✅ SINK tables (Phase 2)
- ✅ Workflows / scheduled jobs (Phase 2)
- ✅ Segments or parent segments (Phase 2)
- ✅ Foundry agents or skills (Phase 4)
- ✅ External activations (any phase)

**Approval format (COPY/PASTE and fill):**
```
📋 Ready to create the following:

- [OBJECT TYPE]: [name]
- [OBJECT TYPE]: [name]

📊 Scope:
- Database: [name]
- Schedule: [daily/weekly/custom]
- Cost: ~$X per month
- First run: ~Xm duration

✓ This CANNOT be undone easily.

Do you approve? (YES / NO / REVIEW DETAILS)
```

**If user says NO or REVIEW:**
- STOP immediately — do not proceed
- Gather feedback on what to change
- Return to planning phase, adjust, and re-present

**Why:** Physical objects are real costs (compute, storage), real consequences (data modifications), and hard to undo. Approval gates prevent accidental deployments, runaway costs, and unintended side effects.

---

### Rule 3: state.md Preservation (CRITICAL for phase continuity)

- [ ] NEVER overwrite, modify, or lose `state.md`
- [ ] Each phase APPENDS a new dated section — never replaces old sections
- [ ] Always include a "Next action" pointer at bottom so users can resume
- [ ] If state.md is missing or corrupted, STOP and ask user to recover it

**state.md structure:**
```yaml
# Dashboard Project: [project-slug]

## Phase 1 — Requirements & Data Discovery
**Date:** [date]
**Status:** ✅ Complete

**Findings:**
- Promotion Score: X/6
- Path Decision: [Workflow / Non-Workflow]
- KPIs: [list]
- Dimensions: [list]

---

## Phase 2 — Workflow Deployment (if applicable)
**Date:** [date]
**Status:** ✅ Complete

**Outputs:**
- SINK tables created: [list]
- Workflow deployed: [name]

---

## Next Action
Currently at end of Phase X. User should:
1. Read ./phase-N/INSTRUCTIONS.md
2. Verify Phase N pre-flight checklist
3. Continue with Phase N
```

**Why:** state.md is the single source of truth for phase continuity. Loss of state.md = complete project restart.

---

### Rule 4: Join Validation (Phase 2 critical)

- [ ] For multi-table dashboards, test ALL joins for cardinality BEFORE workflow deployment
- [ ] Query: `SELECT COUNT(DISTINCT join_key) FROM table_a` vs table_b
- [ ] Counts MUST match (or be within 5%)
- [ ] Check column types: `tdx describe <db>.table_a` — join columns must be same type
- [ ] If unmatched, STOP and debug before Phase 2 deployment

**Example fix (pre-aggregate many-side before joining):**
```sql
-- WRONG: aggregate after join → inflated totals
SELECT SUM(o.amount) FROM customers c JOIN orders o ON c.id = o.customer_id

-- RIGHT: pre-aggregate orders, then join
SELECT SUM(agg.total) FROM customers c
JOIN (
  SELECT customer_id, SUM(amount) AS total FROM orders GROUP BY customer_id
) agg ON c.id = agg.customer_id
```

**Why:** Invalid joins cause silent data duplication, inflated metrics, wrong segments. Past incident: revenue inflated from $4.3M to $6.9M due to undetected fan-out join.

---

### Rule 5: Phase Sequencing (STRICT — no arbitrary skipping)

- [ ] Score 0-2 → Phase 1 → Phase 3 (skip Phase 2) ✓
- [ ] Score 3 → Ask user: "Workflow or quick build?" ✓
- [ ] Score 4-6 → Phase 1 → Phase 2 → Phase 3 ✓
- [ ] NEVER allow Score 0-2 to enter Phase 2
- [ ] NEVER allow Score 4-6 to skip Phase 2 without explicit user override + documented reason

**Why:** Score routing is based on data complexity and refresh frequency. Skipping Phase 2 on a high-scoring project = performance problems downstream.

---

### Rule 6: Time Column Validation (Phase 2 critical)

- [ ] Every metric table MUST have exactly ONE valid business-event or insert-time column
- [ ] Run `tdx describe <db>.<table>` for each source table
- [ ] Identify the time column: `created_at`? `event_time`? `timestamp`? `updated_at`?
- [ ] Is it nullable? If yes, you have a data quality issue — raise with user
- [ ] Do NOT proceed to Phase 2 without this confirmed

**Why:** Phase 2 workflow uses time column for daily scheduling and incremental loads. Wrong column → meaningless schedules, missed data, duplicate loads.

---

### Rule 7: Special Case Path Enforcement (Flow critical)

- [ ] If user provides `.dash` file → Follow "`.dash` Special Case" path ONLY
- [ ] If user provides datamodel name/OID → Follow "Treasure Insights API Special Case" path ONLY
- [ ] If user provides both → Follow "Combined Resources Path" ONLY
- [ ] Do NOT ask normal Stage A questions (1a–1o) if on a special case path

**Why:** Special case paths have their own prefilling logic. Mixing paths causes duplicate requirements, conflicting decisions, wasted effort.

---

### Rule 8: Treasure Data Account Access (Phase 2 critical)

- [ ] Before Phase 2 starts, verify `tdx auth show` works
- [ ] Verify user has database CREATE TABLE permissions
- [ ] Ask user to run `tdx databases` to confirm access
- [ ] If auth fails or permissions missing, STOP and ask user to run `tdx auth setup`

**Why:** Phase 2 creates SINK tables in Treasure Data. Without valid auth, workflow deployment fails and blocks Phase 3.

---

### Rule 9: SINK Table Naming Convention (Phase 2→3 critical)

- [ ] SINK table names MUST follow pattern: `<project_slug>_sink_<metric_group>`
- [ ] Example: project_slug = "sales-dashboard", metric_groups = ["revenue", "pipeline"]
  - → SINK table names: `sales_dashboard_sink_revenue`, `sales_dashboard_sink_pipeline`
- [ ] Verify Phase 3 dashboard queries reference these exact names
- [ ] If names don't match, Phase 3 queries will find no data

**Why:** Misnamed SINK tables → dashboard shows empty tables, requires Phase 2 rework.

---

## Re-Read Protocol

**After EVERY context compaction or phase transition:**

1. **STOP and re-read this file** (`./INSTRUCTIONS.md`)
2. Identify current phase from `state.md`
3. Re-read `./phase-N/INSTRUCTIONS.md`
4. Re-read `./references/INSTRUCTIONS.md`
5. Verify the inline checklist in the current phase's SKILL.md
6. **Append re-read proof to state.md** (see template below)
7. Continue with next action

**Proof-of-read template (append to state.md):**

```yaml
## Re-Read Checkpoints — [date/time]

- ✓ Re-read ./INSTRUCTIONS.md (master)
- ✓ Re-read ./phase-N/INSTRUCTIONS.md (phase-specific)
- ✓ Re-read ./references/INSTRUCTIONS.md (guardrails)
- ✓ Verified inline checklist in phase-N/SKILL.md
- Ready to continue at: [next action]
```

---

## Quick Navigation

| I need to... | Do this |
|---|---|
| Start a new dashboard | 1. Read this file ✓ 2. Read `./phase-1/INSTRUCTIONS.md` 3. Read `./phase-1/SKILL.md` |
| Resume existing project | 1. Read this file ✓ 2. Read `./<project-slug>/state.md` 3. Jump to "Next Action" |
| Context was compacted | 1. **STOP** 2. Re-read this file ✓ 3. Re-read phase-N/INSTRUCTIONS.md 4. Re-read references/INSTRUCTIONS.md 5. Continue |
| Before Phase 2 deployment | 1. Re-read `./phase-2/INSTRUCTIONS.md` 2. Verify all 9 rules above 3. Get user approval 4. Proceed |
| Before Phase 4 agent creation | 1. Re-read `./phase-4/INSTRUCTIONS.md` 2. Get user approval 3. Proceed |

---

## 🔴 If You Cannot Verify an Instruction Item

**STOP immediately.** Do not proceed. Return to user with:
- Which instruction/rule cannot be verified
- Why it's blocking
- What user input is needed to proceed

**Example:**
> "Cannot verify Rule 4 (Join Validation). The query found 5,000 distinct keys in `customers` but 7,500 in `orders`. This indicates fan-out. Before Phase 2, I need to understand: (a) Is this expected? (b) Should I pre-aggregate? (c) Should we use a different join key?"

---

**Version:** 1.0.0
**Last Updated:** 22 July 2026
**Load Order:** 0 (read first)
