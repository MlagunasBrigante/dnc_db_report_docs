---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Cutover to the ITServer]]"
sprint: S2
status: to-do
moscow: must
s2_order: 8
gaps: [GAP-25]
jira: ""
size: M
---
# Story: Backups and restore drill for the dnc database

> [!todo] `<owner>` — action needed
> - [ ] Choose the backup target (cloud and/or external SSD).
> - [ ] Store the backup encryption key in the team password manager, with a second copy.

## User Story
**As** the compliance/legal owner, **I want** the DNC database (the legal record: raw monthly snapshots, lineage, run records) backed up off the ITServer every night, with at least one real restore performed, **so that** losing the server, its disk or the database doesn't lose the legal record.

## Acceptance Criteria
- [ ] **Given** the `dnc` database (or schema, per [[Decision - Dedicated PostgreSQL database]]), **when** the nightly backup runs, **then** an encrypted `pg_dump` is written **off the ITServer** (target decided near this story: cloud and/or external SSD), from a systemd timer and script versioned in `Dev/deploy/` and installed by `install.sh`.
- [ ] **Given** a retention policy (e.g. 30 daily and 12 monthly, aligned with [[Story - Retention policy for lead data]] once it exists), **when** backups age, **then** old ones are pruned and the most recent ones are always kept.
- [ ] **Given** a backup fails, **when** its service exits non-zero, **then** the shared `OnFailure=` notifier sends an email and a Teams alert. A missing backup is never silent.
- [ ] **Given** the latest backup, **when** the **restore drill** runs (into a scratch database), **then** row counts per table and the `pipeline_run` checksums match production, and the restore steps are written down in the runbook. **This drill happens before the physical cutover.**
- [ ] The backup encryption key or password is stored where it can be recovered by someone other than `<owner>` (e.g. the team's password manager), with a second copy.

## Open questions
- [x] ✅ 2026-11-23 (`<owner>`, D-35): **external SSD** plugged into the ITServer, mounted only during the backup (Azure Blob lean dropped). Residual risk accepted: losing the whole server (theft, fire, ransomware reaching attached drives) loses the backups too; R6 is re-scored when this story ships, not closed. Encryption key: **Vaultwarden (autobot) + a sealed offline copy**. ~~#open-question~~ Backup target: cloud, external SSD, or both?
- Stays at order 8 (D-35, path B: the systemd story keeps its dependency on order 2). Align the first AC ("off the ITServer") with the SSD decision when the story starts.

## Notes
- **Azure cold storage, facts to plan with (Microsoft Learn, [Access tiers for blob data](https://learn.microsoft.com/azure/storage/blobs/access-tiers-overview), checked 2026-11-20):** cool 30 days, cold 90 days, archive 180 days minimum retention; deleting, overwriting or re-tiering earlier is billed pro rata as early deletion. Cool and cold are online (millisecond reads, higher per-GB read cost); archive is offline and needs rehydration of up to 15 hours before a restore can even start. Nightly dumps pruned after 30 days must not sit in a tier with a longer minimum (or the prune is billed as early deletion); the restore drill must be timed against the tier's read latency. Migrating old records later: see the note in [[Epic - Archiving closed months]].
- **No interim copy (D-34, 2026-11-23, replaces D-33's):** data never leaves the ITServer, so since the host move (2026-11-23) the legal record has no off-machine copy until this story ships.
- **Moved before every backfill 2026-11-20 (`<owner>`, D-28):** s2_order 12 → 8, so the legal record has a tested backup before history is loaded into it (R7). The backup target decision now gates the backfill stories too.
- **Added 2026-10-29** from the pre-mortem (risk R6): the process and the database live on one machine, and S5 is archiving, not backup.
- Model: HAM's `Projects/HAM/Hardware Asset Management/Story - Backups and restore drill.md`. Lessons there: read `.env` instead of sourcing it (a `$` in a password breaks `set -u`), and make sure retention grouping actually prunes.

## Dependencies
- [[Story - Raw snapshot and run record land in Postgres]] (database exists)
- Before [[Story - Backfill the container-era runs]] and [[Story - Backfill compatible legacy runs]] (D-28)
- Before [[Story - Cut over and complete a scheduled run on the ITServer]]
