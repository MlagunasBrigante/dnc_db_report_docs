---
status: static — source of truth (2026-10-29)
type: prioritization-risk
project: dnc-cron-scrub
phase: "scrum-scoping Phase 4 — Prioritize & De-risk"
created: 2026-10-29
review: at the start of every sprint (S2–S6), monthly on the 5th, quarterly in full
---
# Prioritization and Risk Register — dnc-cron-scrub

Backlog: [[Backlog - dnc-cron-scrub]] · Scope: [[DNC Update - Scope]]

> [!important] Re-run this phase at the start of S3, S4, S5 and S6
> Those sprints are high-uncertainty, so their stories carry `moscow: tbd`. At each sprint start: MoSCoW the stories, re-run the pre-mortem, re-score this register.

## 1. Prioritization: MoSCoW per sprint (accepted 2026-10-29)
- **Framework:** MoSCoW inside each sprint. Must items are ordered by risk and dependencies (`s1_order` property). Sprint order itself is fixed.
- **Why:** the sprint sequence is locked and the team is small, so the open question is what's essential inside each sprint, not numeric scoring. RICE's "reach" means little for an internal pipeline.
- **Alternative:** Cost of Delay, if a hard compliance deadline appears (e.g. Legal sets a date).
- **Reassess when:** a sprint starts, a decision item (D-01…D-07) lands, or a monthly run reveals a new gap.
- Michelle's judgment overrides the labels: they drive the conversation, they don't decide it.
- Labels live in each story's `moscow` property.

### S1 — Gap-closing
| Order | Must (12) |
|---|---|
| 1 | [[Story - Rotate exposed credentials and retire the old repo]] (exposure is live; dependent flows → reminders) |
| 2 | [[Story - INFO logs with secret masking]] · [[Story - Remove the hardcoded LawRuler session cookie]] |
| 3 | [[Story - One retry policy for every API call]] |
| 4 | [[Story - Lead download never truncates silently]] · [[Story - Failed scrub batches are retried and reported]] |
| 5 | [[Story - Config validated before any API call]] · [[Story - Report labels and unmatched warning fixed]] |
| 6 | [[Story - Warning and error emails always arrive]] · [[Story - Monthly report email as a summary table]] · [[Story - Soft-deletion request to the LawRuler team]] · [[Story - Alert delivery failure raises an alert]] |

**Should (7, all approved):**
- [[Story - Spike Teams posting through Graph]] → [[Story - Teams alert channel]]
- [[Story - Runbook for failed or degraded runs]]
- [[Story - Scrub secrets from historic run logs]] (after order 1 confirms the inventory)
- [[Story - Per-stage duration and memory in every run]]
- [[Story - Per-stage inputs and outputs reference]] (**must be done before S2 starts**)
- [[Story - README, Jira and vault references re-scoped]]

**Could (4, to be attempted in S1):**
- [[Story - Stage 4 stops reloading the full lead history]] → [[Story - Memory log with quarterly review]] → [[Story - Stage 2 mean memory reduced]]
- [[Story - Mac run-night safeguards until cutover]]

### S2 — Technical migration
**All Must**, including [[Story - Backfill compatible legacy runs]] (schemas barely changed, so very doable) and the new [[Story - Backups and restore drill for the dnc database]].

### S3–S6
`moscow: tbd`, prioritized at each sprint start.

## 2. Michelle's resource checklist (nudge list)
Resources the Should items need from Michelle. Claude nudges for these at S1 kickoff and before each dependent story starts.
- [ ] Create the **dedicated Teams channel** and share its team and channel identifiers → [[Story - Teams alert channel]]
- [ ] Brief **e.cervera** that they're the backup and will receive error alerts → [[Story - Runbook for failed or degraded runs]]
- [ ] Confirm who can rotate each credential (LawRuler admin, dncscrub, Entra app) → [[Story - Rotate exposed credentials and retire the old repo]]
- [ ] Approve the historic-log summary report before originals are deleted → [[Story - Scrub secrets from historic run logs]]
- [ ] Approve the Jira DNCUP re-scope and the Jira↔vault sync mapping fix → [[Story - README, Jira and vault references re-scoped]]
- [ ] Identify the **LawRuler account manager** → [[Story - Soft-deletion request to the LawRuler team]]

## 3. Risk register (pre-mortem, 2026-10-29)
*These scores are relative estimates to help prioritize; they are not actuarial predictions.* P = probability, I = impact (1–5).

| # | Risk | Category | P | I | Score | Owner | Trigger | Status |
|---|---|---|---|---|---|---|---|---|
| R16 | **Michelle gets stuck or overwhelmed in S3–S4** (statistics plus dashboard on her own) | People | 4 | 4 | **16** | Michelle | A milestone has no progress for 2 weeks, or a soft date slides twice | Active |
| R1 | One person owns pipeline, development and statistics across 6 sprints | People | 4 | 4 | **16** | Michelle | S1 not done by the Jan 1 run, or Michelle out more than a week | Active |
| R2 | Exposed credentials are misused before rotation | Technical | 3 | 5 | **15** | Michelle | Unknown LawRuler/Graph activity; rotation not done by mid-Nov | Active |
| R3 | LawRuler API flakiness or changes break a stage | External | 4 | 3 | **12** | Michelle | Unresolved items rise; new error types | Monitoring |
| R5 | The "removed by the 7th" promise isn't kept | External | 4 | 3 | **12** | Michelle + LawRuler account manager [TBD] | Request not acknowledged by the 5th; requested leads present in the next snapshot | Active |
| R6 | Legal record has no off-machine backup | Technical | 2 | 5 | 10 | Michelle | No backup before cutover | Mitigated by story (S2 Must) |
| R7 | Backfill or migration corrupts the legal record | Technical | 2 | 5 | 10 | Michelle | Row-count or checksum mismatch | Monitoring |
| R8 | Lead-level data reaches an external recipient | Process | 2 | 5 | 10 | Michelle | Allowlist error logged; external recipient added | Monitoring |
| R9 | Database setup undecided or no admin rights, so S2 is blocked | Process | 3 | 3 | 9 | Michelle | Open questions unanswered at S2 start | Monitoring |
| R10 | Legacy runs incompatible, so S3 gets less history | Technical | 2 | 3 | 6 | Michelle | Profiling marks most runs "not loadable" | Monitoring (P lowered: schemas barely changed) |
| R11 | Scope creep across 6 sprints | Process | 3 | 3 | 9 | Michelle | Stories added mid-sprint without the gate | Monitoring |
| R12 | Late n8n decision (D-04) flips the S2 design | Process | 2 | 4 | 8 | Michelle | D-04 open at S2 start | Monitoring |
| R13 | Deleted logs later needed as legal evidence | Process | 2 | 4 | 8 | Michelle | Legal request references a pre-2026-11 run | Monitoring |
| R4 | Mac runs miss a month before cutover (sleep, folder rename) | Technical | 2 | 2 | 4 | Michelle | No run folder by the 2nd | Downgraded: about a month of buffer; handled by a Could story |
| R14 | Graph won't allow app-only posting to Teams | External | 3 | 2 | 6 | j.fernandez | Spike result | Monitoring |
| R15 | Memory work destabilizes the run | Technical | 2 | 3 | 6 | Michelle | Restart test fails; results differ before vs after | Monitoring |

### Mitigation cards (score ≥ 12), accepted 2026-10-29: we'll try not to need them
**R16 — Overwhelm in S3–S4 | 16**
- **Owner:** Michelle (Claude nudges)
- **Trigger:** no milestone progress for 2 weeks, or a soft date slides twice.
- **Mitigation:**
  - **soft, flexible deadlines** per milestone, set at sprint start, which can slide by agreement;
  - a **weekly check-in nudge** ("what's the next small step?") and a nudge 3 days before each soft date;
  - milestones kept small (M1–M5 are already go/no-go steps); time-box exploration.
- **Contingency:** split the stalled milestone into smaller steps; pull e.cervera in for the mechanical parts; temporarily drop S4 scope to the first dashboard page.
- **Residual:** 3×3 = 9.

**R1 — Key-person dependency | 16**
- **Owner:** Michelle
- **Trigger:** S1 not done by the Jan 1 run, or Michelle out more than a week.
- **Mitigation:** e.cervera shadows one S1 story end to end; the runbook and per-stage documentation are written early; stories stay small enough to hand over.
- **Contingency:** pause S3–S6 and keep only the S1 Must items moving.
- **Residual:** 3×3 = 9.

**R2 — Exposed credentials | 15**
- **Owner:** Michelle
- **Trigger:** unknown LawRuler or Graph activity; rotation not done by mid-November.
- **Mitigation:** the rotation story goes first; count secret matches in the historic logs to confirm the inventory; credentials that other flows depend on get recurring reminders until rotated.
- **Contingency:** revoke access immediately and review provider audit logs.
- **Residual:** 2×5 = 10, while some credentials are still pending because other flows depend on them.

**R3 — LawRuler flakiness | 12**
- **Owner:** Michelle
- **Trigger:** unresolved items rise; new error types.
- **Mitigation:** shared retry policy, failed-item events, patch pass (S2), unresolved items in alerts.
- **Contingency:** manual `once --patch`.
- **Residual:** 4×2 = 8.

**R5 — "Removed by the 7th" | 12**
- **Owner:** Michelle + LawRuler account manager [TBD]
- **Trigger:** request not acknowledged by the 5th.
- **Mitigation:** identify the account manager before the first automated request; the watchdog checks the request was sent; Michelle forwards it until then.
- **Contingency:** escalate through Matt or Brad; soften the wording to "Removal requested" until delivery is reliable.
- **Residual:** 2×3 = 6.

## 4. Monitoring schedule
| Frequency | Activity | Owner | Output |
|---|---|---|---|
| Monthly, on the 5th after the watchdog | Quick register review: did any trigger fire? | Michelle | Status column updated |
| Weekly during S3–S4 | Check-in nudge (R16) | Claude → Michelle | Next small step |
| Each sprint start (S2–S6) | Re-run Phase 4: MoSCoW + pre-mortem | Michelle | This file updated |
| Quarterly (with the memory review) | Full reassessment | Michelle + e.cervera | Revised register |
| On trigger | Activate contingency; note it in the run report | Risk owner | Incident note |
