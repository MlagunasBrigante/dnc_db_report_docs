---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Cutover to the ITServer]]"
sprint: S2
status: to-do
moscow: must
gaps: [GAP-25]
jira: ""
size: M
---
# Story: Backups and restore drill for the dnc database

## User Story
**As** the compliance/legal owner, **I want** the DNC database (the legal record: raw monthly snapshots, lineage, run records) backed up off the ITServer every night, with at least one real restore performed, **so that** losing the server, its disk or the database doesn't lose the legal record.

## Acceptance Criteria
- [ ] **Given** the `dnc` database (or schema, per [[Decision - Dedicated PostgreSQL database]]), **when** the nightly backup runs, **then** an encrypted `pg_dump` is written **off the ITServer** (target decided near this story: cloud and/or external SSD), from a systemd timer and script versioned in `Dev/deploy/` and installed by `install.sh`.
- [ ] **Given** a retention policy (e.g. 30 daily and 12 monthly, aligned with [[Story - Retention policy for lead data]] once it exists), **when** backups age, **then** old ones are pruned and the most recent ones are always kept.
- [ ] **Given** a backup fails, **when** its service exits non-zero, **then** the shared `OnFailure=` notifier sends an email and a Teams alert. A missing backup is never silent.
- [ ] **Given** the latest backup, **when** the **restore drill** runs (into a scratch database), **then** row counts per table and the `pipeline_run` checksums match production, and the restore steps are written down in the runbook. **This drill happens before the physical cutover.**
- [ ] The backup encryption key or password is stored where it can be recovered by someone other than Michelle (e.g. the team's password manager), with a second copy.

## Open questions
- [ ] #open-question Backup target: cloud, external SSD, or both? Settle near this story (Michelle, 2026-10-29).

## Notes
- **Added 2026-10-29** from the pre-mortem (risk R6): the process and the database live on one machine, and S5 is archiving, not backup.
- Model: HAM's `Projects/HAM/Hardware Asset Management/Story - Backups and restore drill.md`. Lessons there: read `.env` instead of sourcing it (a `$` in a password breaks `set -u`), and make sure retention grouping actually prunes.

## Dependencies
- [[Story - Raw snapshot and run record land in Postgres]] (database exists)
- Before [[Story - Cut over and complete a scheduled run on the ITServer]]
