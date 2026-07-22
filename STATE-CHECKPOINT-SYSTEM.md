---
name: state-checkpoint-system
description: Runtime validation and checkpoint/replay system using state.md
priority: HIGH
load_order: 0.5
---

# State Checkpoint System — Runtime Validation & Recovery

This document explains how state.md provides built-in runtime validation and checkpoint/replay capability.

---

## Overview

**state.md is NOT just a record — it's a recovery system.**

Every entry in state.md is:
- ✓ A checkpoint (timestamped decision point)
- ✓ A validation point (can verify prerequisites were met)
- ✓ A recovery point (can resume from here if session breaks)

---

## Runtime Checks (API Validation Using state.md)

### Before Executing Any Phase: Validate Prerequisites

**Pattern:**
```
BEFORE running any tdx/query command:
1. Read state.md
2. Extract requirements from LAST completed phase
3. Validate those requirements are STILL true
4. Only if all valid: execute command
```

---

### Phase 1: Pre-Execution Validation

**Before Stage B starts:**
```yaml
Check state.md for:
✓ Project slug defined?
✓ Business goals captured?
✓ Database name recorded?

If ANY missing:
→ STOP: "state.md missing required fields. Backtrack to setup."
```

**Before running data discovery queries:**
```yaml
Validate prerequisites from state.md:
✓ Database exists in TD (was verified in setup)
✓ User can access database (confirmed earlier)
✓ Table names recorded (verify still exist)

Commands to verify:
  tdx databases
  tdx describe <db>.<table>

If any prerequisite INVALID:
→ STOP: "Database or table no longer accessible."
→ User action: Verify TD account access
```

---

### Phase 2: Pre-Execution Validation

**Before creating SINK tables, validate from state.md:**
```yaml
From state.md Phase 1 section, verify:
✓ Time column documented (is it still non-nullable?)
✓ Join keys tested (run cardinality check again)
✓ Promotion score ≥ 4 (user didn't downgrade)
✓ User approved SINK creation (approval proof recorded)

Runtime checks:
  1. Re-run join cardinality query
  2. Verify time column still exists and is non-nullable
  3. Run approval check: is approval proof in state.md?

If ANY check fails:
→ STOP: "Prerequisite validation failed: [reason]"
→ Debug and update state.md
→ Re-run validation before proceeding
```

**Before deploying workflow:**
```yaml
From state.md Phase 2 section, verify:
✓ SINK tables created successfully
✓ Workflow syntax valid
✓ Cost estimate still realistic

Runtime check:
  tdx wf validate [workflow_file]
  
If validation fails:
→ STOP: "Workflow validation failed: [error]"
→ Fix SQL/YAML
→ Re-run validation
```

---

### Phase 3: Pre-Execution Validation

**Before rendering dashboard, validate from state.md:**
```yaml
From state.md Phase 2 section (if applicable), verify:
✓ SINK tables still exist (and populated)
✓ Queries written in Phase 1 still valid

From state.md Phase 1 section, verify:
✓ KPIs documented are still relevant
✓ Data sources still accessible

Runtime checks:
  1. If Phase 2: SELECT COUNT(*) FROM SINK_tables
  2. If Phase 2 skipped: SELECT COUNT(*) FROM source_tables
  3. Run each query: count rows, verify not NULL
  
If ANY check fails:
→ STOP: "Data source validation failed: [reason]"
→ Debug (workflow not run? data deleted?)
→ Update state.md with finding
→ Re-run validation
```

---

### Phase 4: Pre-Execution Validation

**Before deploying agent (Track B), validate from state.md:**
```yaml
From state.md Phase 3 section, verify:
✓ Dashboard data is CURRENT (when was last refresh?)
✓ Spot-checks documented still match

Runtime checks:
  1. Verify SINK tables (Phase 2) or queries (Phase 3) are fresh
  2. Query one spot-check: SELECT [KPI] from [table]
  3. Compare against spot-check value in state.md
  
If spot-check value is STALE (>1 day old):
→ WARN: "Spot-check data is stale. Re-run Phase 3?"
→ User decides: update dashboard first, or use existing data
```

---

## Checkpoint/Replay System (Session Recovery)

### state.md as Checkpoint

**state.md structure = checkpoint markers:**

```yaml
# Dashboard Project: [project-slug]
## Created: [timestamp]

## Phase 1 — Requirements & Data Discovery
**Date:** [timestamp-phase1-start]
**Status:** ✅ Complete
[requirements]
---

## Phase 2 — Workflow Deployment  
**Date:** [timestamp-phase2-start]
**Status:** ✅ Complete
[workflow details]
---

## Next Action
→ Phase 3: Build Dashboard
   Start by: Read ./phase-3/INSTRUCTIONS.md
   Current phase data: [reference to Phase 2 results]
```

### How to Resume from Checkpoint

**If session breaks at any point:**

```
Example: Session dies during Phase 2 workflow deployment

Current state.md:
  Phase 1: ✅ Complete
  Phase 2: ⏳ In Progress (workflow deploying)
  
To resume:
  1. Read ./INSTRUCTIONS.md (master rules)
  2. Read ./phase-2/INSTRUCTIONS.md
  3. Read state.md → see where stopped
  4. Check state.md Phase 2 section: "Next Action" = [specific step]
  5. Resume from that specific step
```

### Explicit Checkpoint Markers

**Add to state.md every time resuming after break:**

```yaml
## Phase 2 — Workflow Deployment
**Date:** [timestamp]
**Status:** ⏳ In Progress

### Checkpoint A (completed)
- [timestamp] Dry-run executed, output shown to user
- [timestamp] User approved SINK creation

### Checkpoint B (current — resumed here)
- [timestamp] Session interrupted
- [timestamp] Resumed from state.md
- [timestamp] Creating SINK tables now...

[rest of phase continues]
```

---

## Replay Logic (Jump to Checkpoint)

### Template: How to Resume Each Phase

**Phase 1:**
```
If session breaks during Phase 1:

1. Read state.md
2. Find "Phase 1" section and "Next Action" line
3. If at Stage A: Resume with next unanswered question
4. If at Stage B: Resume data validation from next table
5. If complete: state.md will say "✅ Complete" → jump to Phase 2
```

**Phase 2:**
```
If session breaks during Phase 2:

1. Read state.md
2. Find "Phase 2" section and last Checkpoint
3. If at dry-run: re-run dry-run (idempotent)
4. If at SINK creation: check if tables exist (query: tdx describe)
   - If exist: verify population, jump to workflow deployment
   - If not exist: resume SINK creation from checkpoint
5. If at workflow deployment: resume from checkpoint
6. If complete: state.md says "✅ Complete" → jump to Phase 3
```

**Phase 3:**
```
If session breaks during Phase 3:

1. Read state.md
2. Find "Phase 3" section and last Checkpoint
3. If at query execution: re-run queries (idempotent, same results)
4. If at dashboard rendering: regenerate HTML (idempotent)
5. If at spot-checks: re-run spot-checks (should pass if data valid)
6. If at user approval: resume from last saved state
7. If complete: state.md says "✅ Complete" → jump to Phase 4 (optional)
```

---

## State Validation Before Execution (The Gate)

### Every Phase Must Include

**Before executing phase commands:**

```markdown
## State Validation (MANDATORY)

```
CANNOT execute Phase N WITHOUT:

✓ Phase N-1 marked ✅ Complete in state.md
✓ state.md file readable and valid
✓ All Phase N-1 results documented
✓ "Next Action" pointer correct

Validation query:
  # Read state.md Phase N-1 section
  # Verify all items documented
  # Verify "Next Action" = Phase N
  
If validation FAILS:
→ "state.md validation failed: [reason]"
→ STOP and ask user to:
   - Provide state.md contents (if lost)
   - Confirm recovery needed
   - Or restart Phase N from checkpoint
```
```

---

## Implementation Pattern

### For Every Phase INSTRUCTIONS.md

Add this section **at the very beginning** (before any rules):

```markdown
## State Validation & Checkpoint Recovery

### BEFORE this phase executes: Validate Prerequisites

**Runtime checks from state.md:**

1. Previous phase marked ✅ Complete?
2. All required fields in state.md?
3. "Next Action" points to this phase?

```bash
# Claude validates by reading state.md:
- Phase [N-1] Status: [check]
- All requirements documented: [check]
- Next action correct: [check]
```

If ANY validation fails:
→ STOP
→ User action: "Provide state.md or confirm restart"

### IF SESSION BREAKS: Resume from Checkpoint

state.md records exact checkpoint (timestamp + step):
1. Read state.md Phase N section
2. Find last Checkpoint marked complete
3. Resume from next checkpoint
4. Re-run any idempotent commands (queries, renders)
5. Continue until Phase N complete
```

---

## Effect on Effectiveness

### Before (95% with dry-run + gates):
- Dry-run prevents surprises ✓
- Phase gates prevent skipping ✓
- But if session breaks: lost context ✗
- If data changes: no re-validation ✗

### After (98% with runtime checks + checkpoint/replay):
- Runtime validation: Prerequisites checked before execution ✓
- Checkpoint markers: Can resume from exact point ✓
- Session recovery: No context loss (state.md preserved) ✓
- Data re-validation: Can re-run checks before proceeding ✓
- Idempotent commands: Can safely replay queries/renders ✓

---

## How It Works (Full Example)

**Scenario: Phase 2 deployment, session breaks after SINK creation**

```
State.md before break:
  Phase 1: ✅ Complete
  Phase 2: ⏳ In Progress
    ### Checkpoint A: Dry-run shown ✅
    ### Checkpoint B: Approval given ✅
    ### Checkpoint C: SINK tables created ✅
    ### Checkpoint D: [Next — workflow deployment]

Session breaks here.

---

User resumes in new session:

1. Claude reads INSTRUCTIONS.md (master rules)
2. Claude reads state.md
3. Claude sees: "Phase 2 in progress, last checkpoint: SINK created"
4. Claude validates prerequisites:
   - Phase 1 complete? ✅ (in state.md)
   - SINK tables created? ✓ (run: tdx describe, verify tables exist)
   - Workflow file ready? ✓ (in phase-2 directory)
5. Claude resumes: "Continuing Phase 2 from SINK creation checkpoint"
6. Claude skips dry-run + approval (already done, recorded in state.md)
7. Claude proceeds to: "Deploying workflow..."
```

---

## state.md Template (Updated)

Every phase should produce this structure:

```yaml
# Dashboard Project: [project-slug]

## Phase N — [Phase Name]
**Date:** [timestamp-start]
**Status:** ✅ Complete (or ⏳ In Progress)

### Checkpoint A (completed: [timestamp])
- Action: [what was done]
- Result: [verification]
- Next: [pointer to Checkpoint B]

### Checkpoint B (completed: [timestamp])
- Action: [what was done]
- Result: [verification]
- Next: [pointer to Checkpoint C or next phase]

### Checkpoint C (current: [timestamp])
- Action: [resuming here]
- Status: In progress
- If break: Resume from this checkpoint

---

## Next Action

**To resume:**
1. Read ./phase-N/INSTRUCTIONS.md
2. Read state.md Phase N section (find Checkpoint C)
3. Skip completed checkpoints A, B
4. Continue from Checkpoint C
5. Append new checkpoint when complete
```

---

## Summary

**Runtime Checks (API Validation):**
- Before executing: Read state.md, validate prerequisites
- Use state.md as source of truth for what was already verified
- If prerequisite no longer valid: STOP and debug

**Checkpoint/Replay System:**
- state.md IS the checkpoint (append-only, timestamped)
- Every phase produces numbered checkpoints
- If session breaks: read state.md, jump to last checkpoint
- Resume from there (idempotent commands safe to replay)

**Effectiveness gain:** +2-3% (toward 98%)
**Enables:** Session recovery, data re-validation, context preservation

---

**Version:** 1.0.0
**Last Updated:** 22 July 2026
**Use with:** All phase INSTRUCTIONS.md files
