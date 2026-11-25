---
type: story
project: dnc-cron-scrub
epic: "[[Epic - PostgreSQL system of record]]"
sprint: S2
moscow: must
s2_order: 3
status: to-do
gaps: []
jira: ""
size: S
---
# Story: Closed months become read-only
## User Story
**As** the compliance owner, **I want** finished monthly run partitions to be read-only (no UPDATE or DELETE except by the archive role), **so that** the legal record cannot be accidentally or maliciously altered.

## Acceptance Criteria
- [ ] **Given** a run completes successfully (pipeline_run.status = 'completed'), **when** an admin attempts `UPDATE raw_snapshot_202610 SET phone = '0000000000' WHERE lead_id = 123`, **then** the query is rejected with a permission error (verified by attempting the UPDATE as the `dnc_writer` role)
- [ ] **Given** a closed partition, **when** an admin attempts `DELETE FROM scrub_results_202610 WHERE ...`, **then** the query is rejected (verified by attempting the DELETE as `dnc_writer`)
- [ ] ~~moved~~ → **moved to [[Story - Scheduled archiving of closed months]] (D-38 (6), 2026-11-25):** `dnc_archive` stays SELECT-only until S5. Original: **Given** a closed partition, **when** the `dnc_archive` role attempts the same UPDATE/DELETE, **then** it succeeds (verified by granting the permission to `dnc_archive` and running the same queries)
- [ ] **Given** a partition is closed, **when** a SELECT query runs, **then** reads are still allowed for all roles (verified by querying as `dnc_reader`)
- [ ] **Given** the pipeline writes to a new month, **when** that run is in progress (pipeline_run.status = 'running'), **then** writes to that month's partitions are still allowed (verified by inserting a test row mid-run)
- [ ] **Given** the read-only enforcement is implemented, **when** the approach is reviewed, **then** it uses either Postgres RLS policies, triggers that reject writes to closed partitions, or GRANTs/REVOKEs run by a post-run hook (verified by reading the DDL and/or run.py code)

## Notes
- **Trigger:** When `pipeline_run.status` transitions to 'completed', all partitions for that run's month become read-only
- **Implementation options:**
  1. Row-level security (RLS) policies: `CREATE POLICY no_updates ON raw_snapshot_202610 FOR UPDATE USING (FALSE)` for non-archive roles
  2. Triggers on each table that check `pipeline_run.status` before allowing UPDATEs/DELETEs
  3. `REVOKE UPDATE, DELETE ON raw_snapshot_202610 FROM dnc_writer` run by a finalize step
- **Testing:** Inject a failed UPDATE/DELETE and verify the error message; also verify the archive role can still operate (needed for S5 retention)

## Decisions (2026-11-23, D-36)
- **Mechanism = permissions only (option 3 of the Notes).** No stage updates or deletes rows mid-run, so `dnc_writer` never gets UPDATE/DELETE on stage tables: every month is read-only from the moment it is written, and no finalize step or status check is needed. AC5 (writes allowed while running) holds through INSERT. Exceptions: `pipeline_run` keeps UPDATE for `dnc_writer`; `dnc_archive` gets UPDATE/DELETE (S5 retention). Not blocked: `dnc_owner` and the superuser (accepted). Trigger-based hardening deferred.
- Role names aligned to `Dev/schema/000_roles_and_database.sql` (formerly written as app/readonly roles).
- **2026-11-25 (D-38 (6), amends D-36):** `dnc_archive` gets UPDATE/DELETE in S5, not now. This story is verification only on `dnc_test` (writer denied UPDATE/DELETE, reads allowed for every role, INSERT allowed during a run, DDL review); no code.

## Dependencies
- [[Story - All stages write to Postgres with lineage]] (partitions must exist before they can be locked)
