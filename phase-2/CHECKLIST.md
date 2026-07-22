# Phase 2: Deploy TD Workflow — Checklist Only

**Purpose:** Quick decision guide. Read this first; full deploy guide (`deploy-workflow-guide.md`) is fallback when details needed.

**Skip this whole phase if you don't need scheduled refresh** — go straight to Phase 3 and query source tables directly.

---

## Pre-Phase-2 Gate

- [ ] Stage A/B complete — requirements + data discovery confirmed with the user
- [ ] Routing score 4–6 (Phase 2 required), or user explicitly chose Workflow at score 3
- [ ] `state.md` saved locally with all fields from Stage A/B filled in
- [ ] SINK database confirmed
- [ ] Workflow project name decided

---

## Steps 2a–2h: Workflow Deployment

### 2a: Copy Embedded Workflow Template
- [ ] Copy `./references/workflow-templates/` into `./<project_slug>/workflows/`

### 2b: Configure Workflow YAML
- [ ] Update `input_params.yaml`:
  - Source database
  - Source table names
  - SINK database
  - Schedule (frequency + time)
  - Date range config (lookback window, refresh mode)

### 2c: Build Queries from state.md
- [ ] Get Stage B validation queries from `state.md`
- [ ] Copy into workflow SQL files under `sql/`
- [ ] Verify all queries use confirmed metrics/dimensions (no inferences)

### 2d (optional): Configure Datamodel
- [ ] Skip unless feeding an analytics layer that needs an explicit schema config
- [ ] If needed: build `config.json`, validate with `jq '.'`

### 2e: Review Configuration
- [ ] Run through `pre-deployment-checklist.md`
- [ ] Confirm workflow file structure with the user
- [ ] Run `tdx wf push --dry-run`

### 2f: Deploy Workflow
- [ ] Deploy with `tdx wf upload` (new) or `tdx wf push` (existing)
- [ ] Verify SINK tables created with correct schema
- [ ] Trigger first run — backfill full historical window

### 2g: Validate SINK Output
- [ ] Run sample queries on SINK tables
- [ ] Verify metrics match Stage B confirmed values (no unexplained drift)
- [ ] Check row counts, date ranges
- [ ] Verify query performance on SINK (target: instant, < 1s)

### 2h: Incremental Strategy (if applicable)
- [ ] Assess SQL files for incremental eligibility
- [ ] Set `refresh_mode` and wire branching
- [ ] Push, run incremental test, confirm row counts don't double

---

## Quality Gate

Before Phase 3:
- [ ] All Stage B metrics reproduced in SINK tables (confirmed values match)
- [ ] Query performance acceptable (raw < 30s, SINK < 1s)
- [ ] SINK schema documented in `state.md`
- [ ] No data validation errors flagged

---

## Phase 2 → Phase 3 Handoff

- [ ] Update `state.md` status: "Phase 2 — Complete"
- [ ] SINK table info documented (names, columns, grain)
- [ ] Proceed to Phase 3: Build Dashboard from SINK tables
  - Phase 3 queries will read from SINK (not re-derive raw data)
  - Phase 3 validation will check against confirmed values (no re-queries)

---

**Full details:** See `deploy-workflow-guide.md` and `./references/workflow-templates/`

**Note:** This checklist reduces guide-reading time at the phase boundary — use `deploy-workflow-guide.md` only when a step needs more detail.
