---
name: td-general-dashboarding-skill-phase-4
description: INTERNAL — Phase 4 only. Optional: extract reusable skill (Track A) or deploy Foundry agent (Track B).
---

# Phase 4: Automate & Deploy (Custom Dashboard Agent — Lite)

> **GUARDRAILS: Read `../references/guardrails-lite.md` before doing anything else in this session.**

**Phase Goal:** Optionally turn the approved dashboard into a reusable asset — Track A extracts it as a parameterized skill, Track B deploys a companion Foundry agent for conversational analysis. Both are entirely optional; skip this phase to go straight to Phase 5 (Handoff) or close the engagement.

**Deliverable (Track A):** `./<project-slug>/skills/` — a self-contained, reusable dashboard skill (HTML Client only)
**Deliverable (Track B):** `./<project-slug>/agents/` — a deployed Foundry agent with knowledge bases

---

## Entry Condition

- ✅ Phase 3 complete: `dashboard.html` + `generate-data.js` approved by the user
- ✅ `state.md` accessible — read it for `project_slug`, confirmed metrics, SINK/source database names, and the Phase 1 compliance flag (if any)

**If the user doesn't need automation or an agent:** skip this phase entirely — go to Phase 5 (Handoff Documentation, optional) or close the engagement.

---

## How to Execute

**Quick start:** See `./CHECKLIST.md` for a fast gate-check before proceeding.

Ask the user which track(s) to run. Then follow the detailed guides:

- **Track A:** [`references/track-a-automation.md`](references/track-a-automation.md) — Steps 4a-0 through 4a-vii (skill extraction)
- **Track B:** [`references/track-b-ai-agent.md`](references/track-b-ai-agent.md) — Steps 4b-i through 4b-vi (Foundry agent deployment)

---

## Quality Gates

✅ Track A (if run): skill validated end-to-end against real data, packaged, and the full Step 4a-vi packaging instructions shown to the user (zip commands, install prompt) — never skipped, even if a zip already exists
✅ Track B (if run): `system_prompt.md` ≤ 9,000 characters, all 5 validation tests passing (or documented exceptions), agent deployed and verified in the Foundry UI
✅ `state.md` updated (append only) with track(s) run and their outputs

---

## Next Phase

### ➡ Route to Phase 5 (Optional)
**Condition:** User wants local handoff docs
**Next:** `../phase-5/handoff-documentation-guide.md`

### ➡ Close (if no Phase 5)
Mark engagement complete, share the final artifacts (`dashboard.html`, skill zip, and/or Foundry agent).

---

**Version:** 1.0.0 (Lite)
**Last Updated:** 15 July 2026
**Author:** FDE Team
