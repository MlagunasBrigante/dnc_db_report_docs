---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Archiving closed months]]"
sprint: S5
moscow: tbd
status: to-do
jira: ""
size: M
---
# Story: Archive and restore one closed month

## User Story
**As** IT and compliance, **I want** one closed month's data archived to external disk and/or cloud with checksums, and then restored in a test environment, **so that** we prove the archiving mechanism works and archived data is actually recoverable before automating it.

## Acceptance Criteria
- [ ] **Given** the retention policy (Story 1) defines the archive target (external disk, cloud, or both), **when** one closed month (e.g., August 2026 partition) is selected, **then** its PostgreSQL partition is dumped (`pg_dump` with `--schema` and `--table` filters for that month's partitions across all stage tables, plus the relevant `pipeline_run` rows), and the dump file is checksummed (SHA-256) and recorded (checksum stored in a manifest file or database table).
- [ ] **Given** the dump is checksummed, **when** copied to the archive target (external disk via `rsync` or `rclone`, or cloud via `restic`, `rclone`, or S3 CLI), **then** the copy is verified (checksum recalculated on the destination matches the source checksum), and the archive location is recorded (e.g., `s3://bucket/dnc-archive/2026-08.sql.gz` or `/mnt/external/dnc-archive/2026-08.sql.gz`).
- [ ] **Given** the month is archived, **when** a restore is performed **in a test environment** (scratch PostgreSQL database, not production), **then** the dump is retrieved from the archive, its checksum is verified before restoration, the dump is loaded into the test database (`pg_restore` or `psql`), and the row counts per stage table match the pre-archive counts (compare against the `pipeline_run` metadata or a pre-archive manifest).
- [ ] **Given** the restore succeeds, **when** spot-checked (query a few lead IDs, verify phone numbers and tags match pre-archive data if a sample was saved), **then** the data is confirmed intact and the restore is documented as successful.
- [ ] **Given** the archive and restore are proven, **when** the original partition is **not yet dropped** from production (this story is a drill, not a real archival), **then** the test database is cleaned up and the process is documented (suggested: `Docs/S5-Archive-Restore-Procedure.md` or a `Dev/archive/README.md`) with step-by-step commands, so Story 3 can automate it.

## Notes
- This story is a **drill** — it proves the mechanism works without committing to a real archival (the production partition stays live). Story 3 automates the real archival.
- If the retention policy (Story 1) specifies multiple tiers (e.g., "disk for 2 years, then cloud"), this story tests the **first tier only** (disk). Testing the second tier (cloud migration) can be deferred to a future story if needed.
- Legal traceability requirements: the checksum is **mandatory** (it proves the archive wasn't corrupted), and the restore drill is **mandatory** (an untested backup is a rumor). Both are acceptance criteria, not nice-to-haves.
- If the archive target is cloud, choose a provider and configure access during this story — do not defer it to Story 3. Options: S3, Azure Blob, Backblaze B2, or a `restic` repository. If the choice stalls, escalate to Michelle.
- The manifest (list of archived months, checksums, and locations) should be machine-readable (JSON, CSV, or a database table) so Story 3's automation can query it.
- Output: one month archived and restored successfully, with checksums verified at every step, and the procedure documented in the repo.

## Dependencies
- **Depends on:** [[Story - Retention policy for lead data]] (can't archive without knowing the target and retention tiers)
- **Blocks:** [[Story - Scheduled archiving of closed months]] (automation requires a proven procedure)
