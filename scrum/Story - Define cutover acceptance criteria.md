---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Cutover to the ITServer]]"
sprint: S2
moscow: must
status: to-do
gaps: []
jira: ""
size: S
---
# Story: Define cutover acceptance criteria
> [!todo] Michelle — action needed
> - [ ] Decide the cutover acceptance criteria.

## User Story
**As** Michelle (pipeline owner), **I want** to agree on the acceptance criteria for cutover before moving the job to the ITServer, **so that** the cutover is deliberate and reversible if the criteria aren't met.

## Acceptance Criteria
- [ ] **Given** the S2 scope defers cutover acceptance criteria to this story, **when** Michelle and the team meet to define the criteria, **then** they document at least 3 concrete criteria (e.g. "parallel run comparison", "alert verification", "first scheduled run succeeds") in this story's Notes section or a linked doc (verified by the doc existing and having ≥3 testable criteria)
- [ ] **Given** the criteria are defined, **when** they are reviewed, **then** each criterion has a clear pass/fail test (e.g. "row counts match within 0.1%" or "email alert received within 5 minutes") — no subjective criteria (verified by reading the criteria and confirming each is testable)
- [ ] **Given** the criteria include a parallel run (Mac + ITServer both process the same month), **when** that criterion is written, **then** it specifies what must match: row counts per table, report checksums, alert content, run duration within X% (verified by the criterion's text)
- [ ] **Given** the criteria include alert verification, **when** that criterion is written, **then** it specifies which alerts (email, Teams) and what content (subject, unresolved item count) must be verified (verified by the criterion's text)
- [ ] **Given** the criteria are finalized, **when** Story - Cut over and complete a scheduled run on the ITServer starts, **then** the developer knows exactly what to test before declaring cutover complete (verified by the criteria being referenced in that story's acceptance criteria)

## Notes
**Suggested criteria (to be confirmed with Michelle):**
1. **Parallel run comparison:** Run the pipeline on both the Mac (SQLite) and ITServer (Postgres) for the same input month; row counts per table match within 0.1%, report CSVs are identical (ignoring timestamps).
2. **Alert verification:** Success email + Teams alert are received for a test run on the ITServer, with correct unresolved-item counts (inject 2 failed items, verify the alert lists them).
3. **First scheduled run:** The systemd timer on the ITServer triggers a run on the 1st, it completes successfully, and the alert is sent — all without manual intervention.
4. **Rollback plan:** Document how to roll back to the Mac (stop the ITServer timers, restart the Mac container, verify data/ is still intact).
5. **Data migration:** All SQLite runs (container-era + compatible legacy) are backfilled into Postgres and verified (checksums, row counts).

**Decision authority:** Michelle has final say; if she defers further, the minimal criteria are #3 (first scheduled run) + #4 (rollback plan).

## Open questions
- [ ] #open-question Cutover acceptance criteria (deferred by Michelle; this story answers it).

## Dependencies
- None (this is a planning/decision story)
