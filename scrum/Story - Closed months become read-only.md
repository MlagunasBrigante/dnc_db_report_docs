---
type: story
project: dnc-cron-scrub
epic: "[[Epic - PostgreSQL system of record]]"
sprint: S2
moscow: must
status: to-do
gaps: []
jira: ""
size: S
---
# Story: Closed months become read-only
## User Story
**As** the compliance owner, **I want** finished monthly run partitions to be read-only (no UPDATE or DELETE except by the archive role), **so that** the legal record cannot be accidentally or maliciously altered.

## Acceptance Criteria
- [ ] **Given** a run completes successfully (pipeline_run.status = 'completed'), **when** an admin attempts `UPDATE raw_snapshot_202610 SET phone = '0000000000' WHERE lead_id = 123`, **then** the query is rejected with a permission error (verified by attempting the UPDATE as the `dnc_app` role)
- [ ] **Given** a closed partition, **when** an admin attempts `DELETE FROM scrub_results_202610 WHERE ...`, **then** the query is rejected (verified by attempting the DELETE as `dnc_app`)
- [ ] **Given** a closed partition, **when** the `dnc_archive` role attempts the same UPDATE/DELETE, **then** it succeeds (verified by granting the permission to `dnc_archive` and running the same queries)
- [ ] **Given** a partition is closed, **when** a SELECT query runs, **then** reads are still allowed for all roles (verified by querying as `dnc_readonly`)
- [ ] **Given** the pipeline writes to a new month, **when** that run is in progress (pipeline_run.status = 'running'), **then** writes to that month's partitions are still allowed (verified by inserting a test row mid-run)
- [ ] **Given** the read-only enforcement is implemented, **when** the approach is reviewed, **then** it uses either Postgres RLS policies, triggers that reject writes to closed partitions, or GRANTs/REVOKEs run by a post-run hook (verified by reading the DDL and/or run.py code)

## Notes
- **Trigger:** When `pipeline_run.status` transitions to 'completed', all partitions for that run's month become read-only
- **Implementation options:**
  1. Row-level security (RLS) policies: `CREATE POLICY no_updates ON raw_snapshot_202610 FOR UPDATE USING (FALSE)` for non-archive roles
  2. Triggers on each table that check `pipeline_run.status` before allowing UPDATEs/DELETEs
  3. `REVOKE UPDATE, DELETE ON raw_snapshot_202610 FROM dnc_app` run by a finalize step
- **Testing:** Inject a failed UPDATE/DELETE and verify the error message; also verify the archive role can still operate (needed for S5 retention)

## Dependencies
- [[Story - All stages write to Postgres with lineage]] (partitions must exist before they can be locked)
