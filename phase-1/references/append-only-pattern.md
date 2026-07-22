# Append-Only Pattern (Critical for All Phases)

Every project has **one** local file: `./<project-slug>/state.md`

This file is the **single source of truth** for cross-phase context. It is **append-only** — never edited, only appended to.

## How It Works

`state.md` follows **append-only** — never rewrite or overwrite existing sections:

| Phase | Appends |
|---|---|
| Phase 1 (this phase) | Session Setup + Requirements block + Data Discovery block + Promotion Score + Path Decision |
| Phase 2 (Workflow, if run) | Workflow deployment details (SINK tables, schedule) |
| Phase 3 (Dashboard Build) | Rendering details, confirmed metrics/dimensions used |
| Phase 4 (Automate & Deploy, if run) | Skill/agent artifact locations |
| Phase 5 (Handoff, if run) | Doc file locations |

## The Rule

- **Never edit prior sections** — only append new dated blocks at the end
- **Every phase reads the entire file** — to understand prior decisions
- **Future sessions resume by reading** `state.md` → jumping to "Next action" at the bottom
- **No external context needed** — `state.md` is self-contained, git-tracked, and portable

This lets any future session read `state.md` top to bottom and understand what was decided and why.
