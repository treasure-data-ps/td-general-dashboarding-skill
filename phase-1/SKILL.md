---
name: td-general-dashboarding-skill-phase-1
description: |
  INTERNAL — invoked by td-general-dashboarding-skill root skill only. Do not trigger directly.
  Phase 1: Requirements & Data Discovery (merged). Run session-setup questions, gather business requirements, validate against real data, score promotion (0-6), and produce a local state.md + approved dashboard plan.
  Use when: Routed here by root td-general-dashboarding-skill SKILL.md after Step 0 (new engagement path).
---

# Phase 1: Requirements & Data Discovery (Custom Dashboard Agent — Lite)

> **GUARDRAILS: Read `../references/guardrails-lite.md` before doing anything else in this session.**

**Phase Goal:** Gather business requirements (Stage A), validate them against real data (Stage B), calculate the promotion score (0–6), decide Workflow vs Non-Workflow path, and get user approval — all in one session, with no Confluence and no git branching.
**Deliverable:** A local `state.md` file at `./<project-slug>/state.md` containing the approved requirements, confirmed data findings, promotion score, and path decision.

---

## 🔑 Append-Only Pattern (Critical for All Phases)

`state.md` follows **append-only** — never rewrite or overwrite existing sections:

| Phase | Appends |
|---|---|
| Phase 1 (this phase) | Session Setup + Requirements block + Data Discovery block + Promotion Score + Path Decision |
| Phase 2 (Workflow, if run) | Workflow deployment details (SINK tables, schedule) |
| Phase 3 (Dashboard Build) | Rendering details, confirmed metrics/dimensions used |
| Phase 4 (Automate & Deploy, if run) | Skill/agent artifact locations |
| Phase 5 (Handoff, if run) | Doc file locations |

**The Rule:** Never edit prior sections. Each phase appends a new dated section at the bottom. This lets any future session read `state.md` top to bottom and understand what was decided and why.

---

## Pre-Phase 1 Checklist

Before starting, confirm:

- ✅ Engagement type confirmed (new project or resuming an existing `./<project-slug>/`)
- ✅ TD account is accessible (for data discovery via `tdx`)

**If resuming (not starting fresh):** Ask for the project slug, read `./<project-slug>/state.md`, and resume at the step listed under "Next action" at the bottom of the file — don't re-ask answered questions.

---

## Phase 1 Workflow: 2 Stages

### Stage A: Requirements Gathering

→ **See `./requirements-gathering-guide.md`** for the full step reference table and batching plan.

Run the session-setup questions (project slug, business goal, platform, data source type) in one batch, then gather core requirements (purpose, metrics, dimensions, filters, date range, sharing) in a small number of batched `AskUserQuestion` calls.

→ **See `./references/steps-1a-1o.md`** for detailed guidance on core requirement steps
→ **See `./references/steps-1k-1n-optional.md`** for optional step details (only ask if relevant)

### Stage B: Data Discovery & Validation

Validate Stage A's requirements against real data: confirm the database/tables exist, discover the correct time column, infer/validate metrics and dimensions from live queries, classify filter scope, and run the Data Quality Gate before finalizing.

→ **See `./references/stage-b-database-discovery.md`** for the full data-discovery walkthrough (time column discovery, metric/dimension inference, filter scope classification, tab grouping)
→ **See `./references/validation-queries.md`** for the Data Quality Gate (Checks 1–12) and reusable SQL patterns
→ **See `./references/confirmed-values-checkpoint.md`** for how to record confirmed metric/dimension values once so later phases don't re-query them
→ **See `./references/workflow-notes.md`** for conflict-resolution and large-table performance handling

### Scoring & Path Decision (Steps 1p + 2f)

Combine the promotion score (0–6) with the data-source constraint (pre-aggregated → can skip Phase 2/Workflow) to decide Workflow vs Non-Workflow.

→ **See `./references/steps-1p-1t.md`** for the scoring questions and field-capture tables
→ **See `./references/stage-b-path-routing.md`** for the canonical routing table (single source of truth for path decisions)

**Canonical routing (score 0–6):**

| Score | Path |
|---|---|
| 0–2 | Non-Workflow → skip straight to Phase 3 (Build Dashboard) |
| 3 | Ask user: "Quick build (Phase 3 directly) or scheduled refresh (Phase 2 Workflow first)?" |
| 4–6 | Workflow recommended → Phase 2, then Phase 3 |

If the data source is pre-aggregated (`skip_phase_3` in the old numbering, i.e. Phase 2/Workflow not needed here), route straight to Phase 3 regardless of score.

---

## Finalization: Write `state.md` and Get Approval

1. **Ask for a project slug** if not already collected (short, kebab-case — e.g. `acme-campaign-perf`). Use `./<project-slug>/` as the working directory for everything this skill produces.

2. **Create the working directory and write `state.md`:**

   ```bash
   mkdir -p ./<project-slug>
   cat > ./<project-slug>/state.md << 'EOF'
   # <Project Slug> — Dashboard State

   ## Phase 1 — Session Setup (<YYYY-MM-DD>)
   - Business goal: <one sentence>
   - Platform: <Treasure Work / Treasure AI Studio>
   - Data source type: <Raw/Transactional / Pre-aggregated / Snapshot / Mixed>

   ## Phase 1 — Requirements
   - Metrics: <list>
   - Dimensions: <list>
   - Filters: <list, with scope classification>
   - Date range / timezone: <...>
   - Exclusion rules: <SQL or description>

   ## Phase 1 — Data Discovery
   - Database.Table(s): <...>
   - Time column: <column, business-event or insert-time>
   - Confirmed metric definitions: <...>
   - Confirmed dimension values: <...>
   - Data Quality Gate: <Pass/Fail summary>

   ## Phase 1 — Promotion Score & Path
   - Q1 (frequency): <0-2>
   - Q2 (history): <0-2>
   - Q3 (audience): <0-2>
   - Total: <0-6>
   - Path decision: <Workflow / Non-Workflow> — <reasoning>

   ## Next action
   - <Phase 2 (Workflow) or Phase 3 (Build Dashboard) — be specific about what to run next>
   EOF
   ```

3. **Quality gate before approval:**
   - All core requirements (1a–1j, 1o) captured
   - Data Discovery findings don't contradict Stage A requirements (if they do, resolve and note the trade-off in `state.md`)
   - Promotion score calculated and path decision made
   - Data Quality Gate checks pass (or failures are explicitly acknowledged)

4. **Get user approval** via `AskUserQuestion`: "Does this capture your requirements and the confirmed data findings?"

→ See `./references/step-1u-finalization.md` and `./references/exit-checklist.md` for the full validation checklist and error-handling guidance.

---

## Phase 1 Deliverables (End of Phase 1)

- ✅ Project slug recorded — used for the working directory `./<project-slug>/`
- ✅ Platform + data source type confirmed
- ✅ Core requirements gathered (metrics, dimensions, filters, date range, sharing, exclusions)
- ✅ Database/tables confirmed to exist; time column identified
- ✅ Metrics and dimensions validated against live data
- ✅ Filter scope classified (dashboard-level / tab-level / widget-level)
- ✅ Data Quality Gate run (Checks 1–12)
- ✅ Promotion score calculated (0–6) and path decision made
- ✅ `./<project-slug>/state.md` created with all of the above
- ✅ User approved the plan
- ✅ Ready for Phase 2 (Workflow, if chosen) or Phase 3 (Build Dashboard)

---

## Quick Reference: Which File?

| Question | Reference |
|---|---|
| Session setup + core requirements questions | `./references/steps-1a-1o.md` |
| Optional requirements (mobile, compliance, drill-down)? | `./references/steps-1k-1n-optional.md` |
| Promotion scoring + path decision logic | `./references/steps-1p-1t.md`, `./references/stage-b-path-routing.md` |
| Database/table/time-column/metric/dimension discovery | `./references/stage-b-database-discovery.md` |
| Filter scope classification rules | `./references/stage-b-database-discovery.md` (Step 2d-filter) |
| Data Quality Gate (12 checks) + SQL patterns | `./references/validation-queries.md` |
| Avoiding redundant re-querying in later phases | `./references/confirmed-values-checkpoint.md` |
| Conflict resolution, large-table performance | `./references/workflow-notes.md` |
| Final validation + approval flow | `./references/step-1u-finalization.md`, `./references/exit-checklist.md` |
| Directory index of all Phase 1 reference files | `./references/INDEX.md` |

---

## Next Phase

### ➡ Move to Phase 2 (Workflow — optional) or Phase 3 (Build Dashboard)

**Condition:** Phase 1 requirements + data discovery approved, `state.md` created.

**What happens next:**
- If path decision = Workflow → go to Phase 2 to deploy a scheduled `.dig` workflow, then Phase 3
- If path decision = Non-Workflow (or pre-aggregated data source) → skip straight to Phase 3
- Phase 3 reads `./<project-slug>/state.md` for confirmed metrics, dimensions, and data source details — no re-asking needed

---

**Version:** 1.0.0 (Lite)
**Last Updated:** 15 July 2026
**Author:** FDE Team
