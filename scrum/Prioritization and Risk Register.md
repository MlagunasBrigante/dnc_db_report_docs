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

> [!important] Re-run this phase at the start of S2, S3, S4, S5 and S6
> At each sprint start: MoSCoW the stories, set the sprint's order property (`s<N>_order`) on its Musts, re-run the pre-mortem, re-score this register. S3–S6 are high-uncertainty, so their stories carry `moscow: tbd` until then.

## 1. Prioritization: MoSCoW per sprint (accepted 2026-10-29)
- **Framework:** MoSCoW inside each sprint. Must items are ordered by risk and dependencies (`s1_order` property). Sprint order itself is fixed.
- **Why:** the sprint sequence is locked and the team is small, so the open question is what's essential inside each sprint, not numeric scoring. RICE's "reach" means little for an internal pipeline.
- **Alternative:** Cost of Delay, if a hard compliance deadline appears (e.g. Legal sets a date).
- **Reassess when:** a sprint starts, a decision item (D-01…D-07) lands, or a monthly run reveals a new gap.
- `<owner>`'s judgment overrides the labels: they drive the conversation, they don't decide it.
- Labels live in each story's `moscow` property.

### S1 — Gap-closing
| Order | Must (12) |
|---|---|
| 0 | [[Story - Dry-run never calls paid APIs or touches real data]] (dry-runs currently pay for scrubs and can poison real run data) |
| 1 | [[Story - Rotate exposed credentials and retire the old repo]] (exposure is live; dependent flows → reminders) — **deferred 2026-11-02 (D-08)**, revisit at the close of *No silent data loss* |
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
**Re-run:** 2026-11-11

> [!info] Where S2 stands (updated 2026-11-23)
> Order 1 ([[Story - Raw snapshot and run record land in Postgres]]) done 2026-11-20 (AC4 passed on the ITServer, guide § 12c). Order 0 ([[Story - Parallel run and early host move to the ITServer]]) done 2026-11-23: comparison identical, the ITServer is the production host. Next by order = order 2 ([[Story - All stages write to Postgres with lineage]]), then order 3. Orders 2–3 decided 2026-11-23 (D-36): copy pattern, 2026-11-21 baseline in `dnc_test`, read-only via permissions. 2026-11-23 (D-35, path B): pulling systemd + backups ahead of order 2 was considered and rejected; the in-container scheduler runs 2026-12-01.
> Open owner items: (b) systemd unit install not yet exercised on the ITServer (R20; sudo and Docker confirmed). (c) Backup target decided 2026-11-23 (D-35: external SSD mounted only during the backup; key in Vaultwarden + sealed offline copy); storing the key is still to do.
> Settled 2026-11-20: (a) backups moved before the backfills (D-28); (e) LawRuler API username/password kept until the last sprint (D-29); (f) Outlook reminder for the PostgreSQL 17 upgrade set.
> Housekeeping (owner's call): local branches `verify/tmp`, `verify/y`, `verify/s1-merge`, the merged `s2/feat/raw-snapshot-postgres`, `s2/feat/raw-snapshot-pg17` and the merged `s1/*` branches can be deleted. Mac run 2026-12-01 02:00 UTC untouched.

**MoSCoW confirmation:** All 17 stories are `moscow: must` (no changes proposed). 2026-11-20: +1 Must, order 0 (D-32).

**Order table:**
| Order | Must (18) |
|---|---|
| 0 | [[Story - Parallel run and early host move to the ITServer]] (added 2026-11-20, D-32; before 2026-12-01, ahead of every other S2 story) |
| 1 | [[Story - Raw snapshot and run record land in Postgres]] (schema + DDL guide; `<owner>` creates DB + roles) |
| 2 | [[Story - All stages write to Postgres with lineage]] (foundation for failures & backfill) |
| 3 | [[Story - Closed months become read-only]] · [[Story - Report generated from Postgres]] (parallel after 2) |
| 4 | [[Story - Failed items captured per stage]] (after all stages in PG) |
| 5 | [[Story - Patch pass reprocesses a run's failed items]] (enables scheduling) |
| 6 | [[Story - Unresolved items reach the alerts]] (completes failure epic) |
| 7 | [[Story - Monthly run from versioned systemd units]] (systemd foundation + install script) |
| 8 | [[Story - Scheduled patch pass on the 3rd]] · [[Story - Watchdog on the 5th]] · [[Story - Backups and restore drill for the dnc database]] (parallel after 7; backups before any backfill, D-28; R6/R7 mitigation) |
| 9 | [[Story - Backfill the container-era runs]] (after backups) |
| 10 | [[Story - Profile legacy runs against the stage contract]] (learning story) |
| 11 | [[Story - Backfill compatible legacy runs]] (after profiling) |
| 12 | [[Story - Define cutover acceptance criteria]] (decision before cutover prep) — ✅ done 2026-11-20 (D-31, D-32) |
| 13 | [[Story - Cut over and complete a scheduled run on the ITServer]] (last; physical move) |
| 14 | [[Story - Rotate exposed credentials and retire the old repo]] (unblocked: GitHub deletion, archive redaction; LR API username/password kept until the last sprint, D-29) |

### S3–S6
`moscow: tbd`, prioritized at each sprint start.

## 2. `<owner>`'s resource checklist (nudge list)
Resources the Should items need from `<owner>`. Claude nudges for these at S1 kickoff and before each dependent story starts.
- [x] Create the **dedicated Teams channel** and share its team and channel identifiers → [[Story - Teams alert channel]]
- [x] Brief **e.cervera** that they're the backup and will receive error alerts. ✅ 2026-11-11: done (`<owner>`). → [[Story - Runbook for failed or degraded runs]]
- [x] Confirm who can rotate each credential (LawRuler admin, dncscrub, Entra app): `<owner>` can (2026-11-02); rotation deferred, D-08 → [[Story - Rotate exposed credentials and retire the old repo]]
- [x] Approve the historic-log summary report before originals are deleted: approved 2026-11-09 (D-15) → [[Story - Scrub secrets from historic run logs]]
- [x] Review the README rewrite: approved 2026-11-10 → [[Story - README, Jira and vault references re-scoped]] (Jira deprecated 2026-10-29)
- [x] Identify the **LawRuler account manager**: Lindsey Humphreys (2026-11-05) → [[Story - Soft-deletion request to the LawRuler team]]

### S2
- [x] Create the `dnc` database and `dnc_*` roles on the ITServer using the DDL guide → [[Story - Raw snapshot and run record land in Postgres]] ✅ 2026-11-20: guide Steps 1–11 executed end to end (both schemas, dry-run + verify MATCH)
- [x] Confirm ITServer access for Docker operations (pull, run, volume mounts) → [[Story - Monthly run from versioned systemd units]] ✅ 2026-11-13: Docker 29.3.0 + Compose v5.1.0, daemon reachable as it.admin
- [ ] Confirm ITServer access for systemd unit installation (requires sudo/root) → [[Story - Monthly run from versioned systemd units]]
- [x] Choose the backup target (cloud storage and/or external SSD) → [[Story - Backups and restore drill for the dnc database]] ✅ 2026-11-23 (D-35): external SSD on the ITServer, mounted only during the backup
- [ ] Store the backup encryption key in the team password manager (with second copy) → [[Story - Backups and restore drill for the dnc database]] (2026-11-23, D-35: Vaultwarden on autobot + sealed offline copy; key created when the story runs)
- [x] Define the cutover acceptance criteria (parallel run, alert test, data diff) → [[Story - Define cutover acceptance criteria]] ✅ 2026-11-20 (D-31, D-32)
- [x] Decide LawRuler API username & password rotation → [[Story - Rotate exposed credentials and retire the old repo]] ✅ 2026-11-20 (D-29): keep the current user for now; reviewed in the last sprint (S6), reminder in [[Epic - Automated soft-deletion]]

## 3. Risk register (pre-mortem, 2026-10-29)
*These scores are relative estimates to help prioritize; they are not actuarial predictions.* P = probability, I = impact (1–5).

| # | Risk | Category | P | I | Score | Owner | Trigger | Status |
|---|---|---|---|---|---|---|---|---|
| R16 | **`<owner>` gets stuck or overwhelmed in S3–S4** (statistics plus dashboard on her own) | People | 4 | 4 | **16** | `<owner>` | A milestone has no progress for 2 weeks, or a soft date slides twice | Active |
| R1 | One person owns pipeline, development and statistics across 6 sprints | People | 4 | 4 | **16** | `<owner>` | S1 not done by the Jan 1 run, or `<owner>` out more than a week | Active |
| R3 | LawRuler API flakiness or changes break a stage | External | 4 | 3 | **12** | `<owner>` | Unresolved items rise; new error types | Monitoring. Re-score when [[Story - Patch pass reprocesses a run's failed items]] and [[Story - Unresolved items reach the alerts]] close (`<owner>`, 2026-11-11: planned mitigations don't lower a score until they ship). |
| R5 | The "removed by the 7th" promise isn't kept | External | 4 | 3 | **12** | `<owner>` + LawRuler account manager (Lindsey Humphreys) | Request not acknowledged by the 5th; requested leads present in the next snapshot | Active |
| R2 | Exposed credentials are misused before rotation | Technical | 2 | 5 | 10 | `<owner>` | Unknown LawRuler/Graph activity; rotation not done by mid-Nov | Graph secret rotated 2026-11-11. LawRuler OAuth client secret not rotatable (global/static), accepted. LawRuler API username/password: kept as is until the last sprint, reviewed there (D-29, 2026-11-20). 2026-11-11: P reduced 3→2 (most credentials addressed). 2026-11-19 (D-26): same credentials also on the ITServer `.env` (chmod 600) for setup; risk stays open |
| R6 | Legal record has no off-machine backup | Technical | 2 | 5 | 10 | `<owner>` | No backup before cutover | 2026-11-20 (D-32): the host move comes before the backups story, so ITServer runs need an interim off-machine copy ([[Story - Parallel run and early host move to the ITServer]]). 2026-11-23 (D-34): no interim copy; data never leaves the ITServer by design, so the legal record sits on one host until the backups story ships (order 8). `<owner>` accepts the window and aims to ship backups sooner. 2026-11-23 (D-35, path B): backups stay at order 8 (not pulled ahead), so this window lasts until order 8 ships; target external SSD mounted only during the backup, so loss of the whole server stays a residual risk (re-score on ship, not close). Mitigated by story (S2 Must, not built yet). Re-score when [[Story - Backups and restore drill for the dnc database]] closes (`<owner>`, 2026-11-11). |
| R7 | Backfill or migration corrupts the legal record | Technical | 2 | 5 | 10 | `<owner>` | Row-count or checksum mismatch | Monitoring. 2026-11-20 (D-28): backups + restore drill now ordered before every backfill; score unchanged until it ships. |
| R8 | Lead-level data reaches an external recipient | Process | 2 | 5 | 10 | `<owner>` | Allowlist error logged; external recipient added | Monitoring |
| R9 | Database setup undecided or no admin rights, so S2 is blocked | Process | 3 | 3 | 9 | `<owner>` | Open questions unanswered at S2 start | Closed 2026-11-10 (D-18: dedicated DB; `<owner>` creates DB + roles) |
| R11 | Scope creep across 6 sprints | Process | 3 | 3 | 9 | `<owner>` | Stories added mid-sprint without the gate | Monitoring |
| R19 | Mac and ITServer diverge during S2 migration (dual maintenance) | Process | 3 | 3 | 9 | `<owner>` | Mac Dec 1 run uses old code; ITServer tests use new schema | Closed 2026-11-23: the ITServer took over the monthly job (order 0, D-32); the Mac container is stopped (rollback only). |
| R21 | `<owner>` sole DB admin; no backup person for DB operations | People | 3 | 3 | 9 | `<owner>` | `<owner>` unavailable during migration or post-cutover issue | Active |
| R12 | Late n8n decision (D-04) flips the S2 design | Process | 2 | 4 | 8 | `<owner>` | D-04 open at S2 start | Closed 2026-11-10 (D-04 = no) |
| R13 | Deleted logs later needed as legal evidence | Process | 2 | 4 | 8 | `<owner>` | Legal request references a pre-2026-11 run | Monitoring |
| R18 | Testing on production `dnc` DB via test table risks data corruption | Process | 2 | 4 | 8 | `<owner>` | Test code touches real partitions; bad transaction rollback | Active |
| R20 | `<owner>` lacks Docker/systemd access on ITServer | Process | 2 | 4 | 8 | `<owner>` | Install script fails with permission errors | Active. 2026-11-13: Docker confirmed (29.3.0, Compose v5.1.0); systemd/sudo still unconfirmed. 2026-11-19: sudo confirmed (`<owner>` installed git with `dnf`); likelihood lowered in practice, but systemd unit install not yet exercised — keep open until the systemd story runs. |
| R22 | PostgreSQL 17 reaches end of life (2029-11-08) without a planned major upgrade | Technical | 2 | 4 | 8 | `<owner>` | 2029-05-01 with no upgrade plan | Open. Reminder: plan the upgrade to the newest supported major (expected 22, released ~Sep 2029) in 2029; major upgrades need pg_upgrade or dump/restore (D-23). ✅ 2026-11-20: Outlook reminder set (`<owner>`). |
| R4 | Mac runs miss a month before cutover (sleep, folder rename) | Technical | 2 | 2 | 4 | `<owner>` | No run folder by the 2nd | Downgraded: about a month of buffer; handled by a Could story |
| R14 | Graph won't allow app-only posting to Teams | External | 3 | 2 | 6 | j.fernandez | Spike result | Monitoring |
| R15 | Memory work destabilizes the run | Technical | 2 | 3 | 6 | `<owner>` | Restart test fails; results differ before vs after | Monitoring |
| R17 | PostgreSQL 13.23 partition management or syntax differs from newer versions | Technical | 2 | 3 | 6 | `<owner>` | DDL fails on PG 13; partition closing errors | Retired 2026-11-13: target is PostgreSQL 17 (D-23). |

### Mitigation cards (score ≥ 12), accepted 2026-10-29: we'll try not to need them
**R16 — Overwhelm in S3–S4 | 16**
- **Owner:** `<owner>` (Claude nudges)
- **Trigger:** no milestone progress for 2 weeks, or a soft date slides twice.
- **Mitigation:**
  - **soft, flexible deadlines** per milestone, set at sprint start, which can slide by agreement;
  - a **weekly check-in nudge** ("what's the next small step?") and a nudge 3 days before each soft date;
  - milestones kept small (M1–M5 are already go/no-go steps); time-box exploration.
- **Contingency:** split the stalled milestone into smaller steps; pull e.cervera in for the mechanical parts; temporarily drop S4 scope to the first dashboard page.
- **Residual:** 3×3 = 9.

**R1 — Key-person dependency | 16**
- **Owner:** `<owner>`
- **Trigger:** S1 not done by the Jan 1 run, or `<owner>` out more than a week.
- **Mitigation:** e.cervera shadows one S1 story end to end; the runbook and per-stage documentation are written early; stories stay small enough to hand over.
- **Contingency:** pause S3–S6 and keep only the S1 Must items moving.
- **Residual:** 3×3 = 9.

**R2 — Exposed credentials | 10** (reduced from 15, 2026-11-11)
- **Owner:** `<owner>`
- **Trigger:** unknown LawRuler or Graph activity; rotation not done by mid-November.
- **Mitigation:** the rotation story goes first; count secret matches in the historic logs to confirm the inventory; credentials that other flows depend on get recurring reminders until rotated.
- **2026-11-11 (D-20):** Graph secret rotated; LawRuler OAuth client secret not rotatable (global/static on LawRuler's side), accepted. LawRuler API username/password pending: `<owner>` discusses with team, reports at end of S2.
- **2026-11-19 (D-26):** the same LawRuler, O365 and dncscrub credentials were typed into a new ITServer `.env` (chmod 600, owner `it.admin`) for the Postgres setup dry-run. A second host now holds them, so any later rotation must update both the Mac and the ITServer. Risk stays open until the LawRuler user is reviewed in the last sprint (D-29, 2026-11-20).
- **Contingency:** revoke access immediately and review provider audit logs.
- **Residual:** 2×5 = 10, while some credentials are still pending because other flows depend on them.

**R3 — LawRuler flakiness | 12**
- **Owner:** `<owner>`
- **Trigger:** unresolved items rise; new error types.
- **Mitigation:** shared retry policy, failed-item events, patch pass (S2), unresolved items in alerts.
- **2026-11-11:** score kept at 12 until the S2 patch pass and unresolved-items alerts ship.
- **Contingency:** manual `once --patch`.
- **Residual:** 4×2 = 8.

**R5 — "Removed by the 7th" | 12**
- **Owner:** `<owner>` + LawRuler account manager (Lindsey Humphreys, identified 2026-11-05)
- **Trigger:** request not acknowledged by the 5th.
- **Mitigation:** identify the account manager before the first automated request; the watchdog checks the request was sent; `<owner>` forwards it until then.
- **Contingency:** escalate through Matt or Brad; soften the wording to "Removal requested" until delivery is reliable.
- **Residual:** 2×3 = 6.

## 4. Monitoring schedule
| Frequency | Activity | Owner | Output |
|---|---|---|---|
| Monthly, on the 5th after the watchdog | Quick register review: did any trigger fire? | `<owner>` | Status column updated |
| Weekly during S3–S4 | Check-in nudge (R16) | Claude → `<owner>` | Next small step |
| Each sprint start (S2–S6) | Re-run Phase 4: MoSCoW + pre-mortem | `<owner>` | This file updated |
| Quarterly (with the memory review) | Full reassessment | `<owner>` + e.cervera | Revised register |
| On trigger | Activate contingency; note it in the run report | Risk owner | Incident note |
