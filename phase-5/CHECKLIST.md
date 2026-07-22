# Phase 5: Handoff Documentation — Checklist Only

**Purpose:** Quick decision guide. Read this first; full SKILL.md is fallback when details needed.

---

## Prerequisites

- [ ] Phase 3 dashboard is approved and working
- [ ] Phase 2 workflow completed (if applicable)
- [ ] state.md up-to-date from Phase 1-4
- [ ] Project owner identified (name, email, Slack handle)
- [ ] Backup owner identified
- [ ] Escalation contacts finalized

---

## 4 Documentation Files to Create

### 1. Architecture.md — Data Flow & Components

**What to document:**
- [ ] Data flow diagram (source tables → workflow? → SINK → queries → dashboard.html)
- [ ] Component list: Source tables, Workflow (if Phase 2), SINK tables, Queries, Dashboard
- [ ] For each component: name, row count, refresh frequency, last updated
- [ ] Dependencies (TD account access, source data exists, SINK populated)
- [ ] Refresh frequency (when is data current?)
- [ ] Troubleshooting section (common issues + debugging steps)

**Real information required:**
- [ ] Actual table names (not placeholders)
- [ ] Actual row counts (not estimates)
- [ ] Actual workflow name, schedule, duration
- [ ] Actual SINK table names following pattern: `<project_slug>_sink_<metric_group>`

---

### 2. Usage Guide.md — How to Use Dashboard

**What to document:**
- [ ] 1-2 sentence overview (what dashboard measures, who uses it)
- [ ] How to open (download, email, web server)
- [ ] KPIs section: List actual KPIs with real values + interpretation
- [ ] Filters section: For each filter, what options are available, what happens when applied
- [ ] Charts section: Describe each visualization (line chart, bar, table, etc.)
- [ ] Tips & tricks (export CSV, share, print to PDF, spot-check)
- [ ] Common questions & answers

**Real information required:**
- [ ] Actual KPI names and current values (e.g., "Total Revenue: $4.5M")
- [ ] Actual filter options (e.g., "Region: US, EU, APAC")
- [ ] Actual chart types and descriptions
- [ ] Real use cases and tips

---

### 3. Runbook.md — Operations & Troubleshooting

**What to document:**
- [ ] Owner & backup owner (names, emails, Slack)
- [ ] Daily operations: Health checks, what to monitor
- [ ] Weekly maintenance: Workflow history, SINK table growth, user feedback
- [ ] Troubleshooting section: Common issues with diagnosis + resolution steps
- [ ] Emergency procedures: Critical issues, escalation, stakeholder communication
- [ ] Scheduled maintenance windows (if any)
- [ ] Contacts & escalation path (real people, real numbers)

**Real information required:**
- [ ] Owner full name + email (not "contact your admin")
- [ ] Backup owner full name + email
- [ ] Real workflow name, table names
- [ ] Real diagnostic commands (tdx queries to run)
- [ ] Real escalation contacts + phone numbers

---

### 4. Access & Ownership.md — Permissions & Change Management

**What to document:**
- [ ] Project owner (full name, email, Slack, timezone, responsibilities)
- [ ] Backup owner (full name, email, Slack)
- [ ] Access tiers:
  - Tier 1 (View only): Who? What can they do?
  - Tier 2 (Operator): Who? What can they do?
  - Tier 3 (Admin): Who? Full control
- [ ] How to request access (email, approval process, timeline)
- [ ] Change management (minor vs major, approval process, timeline)
- [ ] Permissions in Treasure Data (CREATE TABLE scope, SELECT scope, etc.)
- [ ] Disaster recovery procedures (backup, restore, recovery time)
- [ ] Support contact & response times

**Real information required:**
- [ ] Actual names, emails, Slack handles (tier 1, 2, 3)
- [ ] Actual permission scopes for each role
- [ ] Actual escalation contacts
- [ ] Actual support SLA and response times

---

## Content Validation Checklist

**Before finalizing each document:**

### Architecture.md
- [ ] Data flow is specific to this project (not template)
- [ ] All table names are actual (not `[table_name]`)
- [ ] All costs are actual estimates (not `$X`)
- [ ] Troubleshooting section has real commands
- [ ] Refresh frequency is documented
- [ ] Dependencies are clear (what needs to be running)

### Usage Guide.md
- [ ] KPIs have real values (not placeholders)
- [ ] Filters are actual options from data (not examples)
- [ ] Chart descriptions match actual visualizations
- [ ] Tips are real workflows users would do
- [ ] Common questions are from actual users (not generic)

### Runbook.md
- [ ] Owner/backup owner are real people (with Slack handles)
- [ ] Daily checks reference actual workflow/table names
- [ ] Diagnostic queries are copy-paste ready
- [ ] Troubleshooting steps are tested
- [ ] Emergency contacts are real phone numbers (not "[manager]")

### Access & Ownership.md
- [ ] All names are real people (no "contact admin")
- [ ] Tier descriptions match actual roles in team
- [ ] Permission scopes are verified against Treasure Data auth
- [ ] Escalation contacts are real phone numbers
- [ ] SLA/response time is realistic

---

## Quality Gate Before Handoff

**Review each document with owner:**

- [ ] Architecture.md approved by owner (data flow correct, costs accurate)
- [ ] Usage Guide.md approved by audience (instructions clear, KPI descriptions helpful)
- [ ] Runbook.md approved by operator (commands work, troubleshooting steps valid)
- [ ] Access & Ownership.md approved by owner (permissions correct, contacts current)

**If approved:** Finalize and save to local folder

**If feedback:** Update documents and re-present

---

## File Locations

All 4 files saved in `./<project-slug>/` directory:

```
[project-slug]/
├── state.md
├── dashboard.html
├── Architecture.md ← Create
├── Usage Guide.md ← Create
├── Runbook.md ← Create
└── Access & Ownership.md ← Create
```

---

## Next Steps After Phase 5

**All 4 documents complete:**
- [ ] Append Phase 5 results to state.md
- [ ] Save documents to shared location (email, Drive, wiki, etc.)
- [ ] Share with stakeholders
- [ ] Training/onboarding using documents

**Project is now production-ready:**
- ✅ Phase 1: Requirements validated
- ✅ Phase 2: Workflow deployed (optional)
- ✅ Phase 3: Dashboard built & approved
- ✅ Phase 4: Automation deployed (optional)
- ✅ Phase 5: Handoff documentation complete

**No further Claude involvement needed** (unless changes requested)

---

**Version:** 1.0.0
**Last Updated:** 22 July 2026
**Purpose:** Quick checklist for Phase 5 handoff documentation
