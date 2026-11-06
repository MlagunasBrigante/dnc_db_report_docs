---
status: static — source of truth (2026-10-29)
type: backlog
project: dnc-cron-scrub
jira: DNCUP
created: 2026-10-29
---
# Backlog — dnc-cron-scrub

14 epics, 53 stories. Priorities (MoSCoW, in each story's `moscow` property) and risks: [[Prioritization and Risk Register]]. Scope, gap register (GAP-01…29) and decisions (D-01…10) live in [[DNC Update - Scope]].

Sprints are sequential. Each one starts only when the previous one is stable. All code is developed as if it already runs on the AlmaLinux ITServer (Docker, systemd, local PostgreSQL), even though the job stays on the Mac until the S2 cutover.

> **Review gates (mandatory, Michelle):** annotate the epic files first (**Satisfactory Epics?**), then the story files (**Satisfactory Stories?**). Annotate in the files, not in chat. The next pass only touches what you annotate.

> **Open questions** live in the story or epic that needs them, tagged `#open-question`. Search that tag in Obsidian to list them. They're answered when that story starts, not before.

> **Checkboxes track completion.** A box is ticked only when that story's or epic's `status` property is `done`. After closing a story or epic, re-check every box in its epic so ticks match the `status` fields.

## S1 — Gap-closing
- [ ] [[Epic - Reliable alerts]]
    - [x] [[Story - Warning and error emails always arrive]]
    - [ ] [[Story - Monthly report email as a summary table]]
    - [x] [[Story - Soft-deletion request to the LawRuler team]]
    - [x] [[Story - Spike Teams posting through Graph]]
    - [ ] [[Story - Teams alert channel]]
    - [ ] [[Story - Alert delivery failure raises an alert]]
    - [ ] [[Story - Mac run-night safeguards until cutover]]  (Could)
    - [x] [[Story - Runbook for failed or degraded runs]]
- [ ] [[Epic - Lean and secret-free logs]]
    - [x] [[Story - INFO logs with secret masking]]
    - [x] [[Story - Remove the hardcoded LawRuler session cookie]]
    - [ ] [[Story - Scrub secrets from historic run logs]]
    - [ ] [[Story - Rotate exposed credentials and retire the old repo]]
- [ ] [[Epic - No silent data loss]]
    - [x] [[Story - Dry-run never calls paid APIs or touches real data]]  ← before any dry-run
    - [x] [[Story - One retry policy for every API call]]  ← do first
    - [x] [[Story - Lead download never truncates silently]]
    - [x] [[Story - Failed scrub batches are retried and reported]]
    - [x] [[Story - Dry-run failure injection]]  (Should; verification tool)
    - [x] [[Story - Config validated before any API call]]
    - [x] [[Story - Report labels and unmatched warning fixed]]
    - [ ] [[Story - Per-stage duration and memory in every run]]
- [ ] [[Epic - Project documentation and housekeeping]]
    - [ ] [[Story - Per-stage inputs and outputs reference]]
    - [ ] [[Story - README, Jira and vault references re-scoped]]
- [ ] [[Epic - Memory optimization]]: closes S1, **only after the other S1 epics are done and the code is stable**
    - [ ] [[Story - Stage 4 stops reloading the full lead history]]  ← priority 1
    - [ ] [[Story - Memory log with quarterly review]]
    - [ ] [[Story - Stage 2 mean memory reduced]]  ← only after Stage 4 is stable

## S2 — Technical migration
> Key decision: [[Decision - Dedicated PostgreSQL database]]
- [ ] [[Epic - PostgreSQL system of record]]
    - [ ] [[Story - Raw snapshot and run record land in Postgres]]
    - [ ] [[Story - All stages write to Postgres with lineage]]
    - [ ] [[Story - Closed months become read-only]]
    - [ ] [[Story - Report generated from Postgres]]
- [ ] [[Epic - Failures captured and patched]]
    - [ ] [[Story - Failed items captured per stage]]
    - [ ] [[Story - Patch pass reprocesses a run's failed items]]
    - [ ] [[Story - Unresolved items reach the alerts]]
- [ ] [[Epic - Reproducible scheduling on the ITServer]]
    - [ ] [[Story - Monthly run from versioned systemd units]]
    - [ ] [[Story - Scheduled patch pass on the 3rd]]
    - [ ] [[Story - Watchdog on the 5th]]
- [ ] [[Epic - Historical backfill]]
    - [ ] [[Story - Backfill the container-era runs]]
    - [ ] [[Story - Profile legacy runs against the stage contract]]
    - [ ] [[Story - Backfill compatible legacy runs]]
- [ ] [[Epic - Cutover to the ITServer]]: physical move, not rushed
    - [ ] [[Story - Define cutover acceptance criteria]]
    - [ ] [[Story - Backups and restore drill for the dnc database]]  (before cutover)
    - [ ] [[Story - Cut over and complete a scheduled run on the ITServer]]

> S3–S6: at each sprint start, re-run **Prioritize & De-risk** (MoSCoW + pre-mortem). Stories carry `moscow: tbd` until then.

## S3 — Statistics: data readiness and modeling (Michelle leads)
- [ ] [[Epic - Data readiness for modeling]]: milestones M1–M5 are go/no-go checkpoints
    - [ ] [[Story - M1 Data and pipeline audit]]
    - [ ] [[Story - M2 Notes ingested and preprocessing aligned]]
    - [ ] [[Story - M3 Data quality gate]]
    - [ ] [[Story - M4 Model tested with sample data]]
    - [ ] [[Story - M5 Metrics validated for the dashboard]]

## S4 — Dashboard (after the migration is done and stable)
- [ ] [[Epic - Metrics dashboard]]
    - [ ] [[Story - Spike dashboard stack and hosting]]
    - [ ] [[Story - First dashboard page from validated metrics]]
    - [ ] [[Story - Remaining metrics on the dashboard]]

## S5 — Archiving (after the dashboard launches)
- [ ] [[Epic - Archiving closed months]]
    - [ ] [[Story - Retention policy for lead data]]
    - [ ] [[Story - Archive and restore one closed month]]
    - [ ] [[Story - Scheduled archiving of closed months]]

## S6 — Automated soft-deletion (optional, after S5; only if D-07 = automate)
- [ ] [[Epic - Automated soft-deletion]]
    - [ ] [[Story - Spike LawRuler soft-delete via API]]

## Decisions (not sprint work until decided)
D-01 litigator soft vs hard delete · D-02 non-US numbers · D-03 reassigned-number check · D-04 pure-n8n rewrite · D-05 LawRuler page throttle · D-06 parallel Stage 4 tagging · D-07 automate the soft-deletion (S6) or keep the email request. Details are in [[DNC Update - Scope#Decision items (not sprint work until decided)]].
