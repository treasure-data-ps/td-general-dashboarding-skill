# Phase 4: Automate & Deploy (Track A/B) — Checklist Only

**Purpose:** Quick decision guide. Read this first; full guides (`automate-deploy-guide.md`, `references/track-a-automation.md`, `references/track-b-ai-agent.md`) are fallback when details needed.

---

## Pre-Phase-4 Gate

- [ ] Phase 3 complete: `dashboard.html` approved by the user
- [ ] Phase 2 workflow complete (SINK tables ready) OR Phase 2 skipped (querying source tables directly)
- [ ] `state.md` updated through Phase 3
- [ ] User has decided Track A, Track B, both, or neither (ask if not yet decided — see `automate-deploy-guide.md`)

---

## Track A: Extract Reusable Skill (Steps 4a-0 to 4a-vii)

- [ ] **4a-0:** Assemble knowledge package (`business_context.md`, `data_dictionary.md`, `sql_templates.md`, `metrics_catalog.md`) — copy from `phase-4/references/templates/`, fill placeholders
- [ ] **4a-i:** Extract dashboard skill definition — `./<project-slug>/skills/SKILL.md`, copy `dashboard.html` + `generate-data.js` from Phase 3 output into `./<project-slug>/skills/`
- [ ] **4a-ii:** Extract & parameterize query scripts — column-name audit against actual SINK/source schema, verify snapshot tables have no date filter, `--limit` on every query, parallel queries
- [ ] **4a-iii:** Create configuration templates (weekly/monthly/quarterly/annual patterns)
- [ ] **4a-iv:** Document deployment & replication checklist
- [ ] **4a-v:** Validate the extracted skill end-to-end — run `generate-data.js` against real data, spot-check against Phase 1 confirmed samples (±5%), reproduce dashboard, confirm no hardcoded values
- [ ] **4a-vi: Final Packaging** — ⚠️ Always show the full packaging instructions (zip commands, setup steps, ready-to-use install prompt) after `generate-data.js` validation passes. **Never skip or summarize this section, even if the zip was already created earlier in the session.**
- [ ] **4a-vii:** Generate Installation Guide (`skills/INSTALL.md`) — platform-specific install/run/troubleshooting instructions; include it in the packaged zip before sharing

---

## Track B: Deploy Companion Foundry Agent (Steps 4b-i to 4b-vi)

- [ ] **4b-i/ii:** Decide on agent + choose capability (Insights / Dashboard Generation / Reporting / Orchestration — composable via intent-routing)
- [ ] **4b-iii:** Pre-flight checks — Foundry/LLM access, output tables have data, audit for an existing stale Foundry project, compliance flags from `state.md` honored
- [ ] **4b-iv:** Configure knowledge bases — copy `agent-prompt-template.md` → `system_prompt.md`, create one `.yml` file per SINK table in `knowledge_bases/` (e.g. `sink_sales_revenue.yml`, using `{name, td_query}` format, not a plain string list — see `track-b-ai-agent.md` § Step 4b-iv); **check `system_prompt.md` ≤ 9,000 chars with `wc -c`** before pushing
- [ ] **4b-v:** Deploy agent — `tdx agent push -y`, verify in Foundry UI
- [ ] **4b-vi:** Test agent — `tdx agent test --run all`, iterate failures with `--reeval`

---

## Phase 4 → Phase 5 Handoff

- [ ] `state.md` updated with track(s) run, skill name (if Track A), agent name (if Track B)
- [ ] User informed of next step: Phase 5 (Handoff Documentation, optional) or close the engagement

---

**Full guides:** `../phase-4/automate-deploy-guide.md` and `../phase-4/references/`

**Version:** 1.0.0 (Lite)
**Last Updated:** 15 July 2026
**Author:** FDE Team
