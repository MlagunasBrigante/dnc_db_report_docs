---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Cutover to the ITServer]]"
sprint: S2
moscow: must
s2_order: 12
status: done
gaps: []
jira: ""
size: S
---
# Story: Define cutover acceptance criteria
> [!todo] `<owner>` — action needed
> - [x] Decide the cutover acceptance criteria. ✅ 2026-11-20 (D-31, D-32)

## User Story
**As** `<owner>` (pipeline owner), **I want** to agree on the acceptance criteria for cutover before moving the job to the ITServer, **so that** the cutover is deliberate and reversible if the criteria aren't met.

## Acceptance Criteria
- [x] **Given** the S2 scope defers cutover acceptance criteria to this story, **when** `<owner>` and the team meet to define the criteria, **then** they document at least 3 concrete criteria (e.g. "parallel run comparison", "alert verification", "first scheduled run succeeds") in this story's Notes section or a linked doc (verified by the doc existing and having ≥3 testable criteria)
- [x] **Given** the criteria are defined, **when** they are reviewed, **then** each criterion has a clear pass/fail test (e.g. "row counts match within 0.1%" or "email alert received within 5 minutes") — no subjective criteria (verified by reading the criteria and confirming each is testable)
- [x] **Given** the criteria include a parallel run (Mac + ITServer both process the same month), **when** that criterion is written, **then** it specifies what must match: row counts per table, report checksums, alert content, run duration within X% (verified by the criterion's text)
- [x] **Given** the criteria include alert verification, **when** that criterion is written, **then** it specifies which alerts (email, Teams) and what content (subject, unresolved item count) must be verified (verified by the criterion's text)
- [x] **Given** the criteria are finalized, **when** Story - Cut over and complete a scheduled run on the ITServer starts, **then** the developer knows exactly what to test before declaring cutover complete (verified by the criteria being referenced in that story's acceptance criteria)

## Notes
**Suggested criteria (to be confirmed with `<owner>`):**
1. **Parallel run comparison:** Run the pipeline on both the Mac (SQLite) and ITServer (Postgres) for the same input month; row counts per table match within 0.1%, report CSVs are identical (ignoring timestamps).
2. **Alert verification:** Success email + Teams alert are received for a test run on the ITServer, with correct unresolved-item counts (inject 2 failed items, verify the alert lists them).
3. **First scheduled run:** The systemd timer on the ITServer triggers a run on the 1st, it completes successfully, and the alert is sent — all without manual intervention.
4. **Rollback plan:** Document how to roll back to the Mac (stop the ITServer timers, restart the Mac container, verify data/ is still intact).
5. **Data migration:** All SQLite runs (container-era + compatible legacy) are backfilled into Postgres and verified (checksums, row counts).

**Decisions 2026-11-20 (`<owner>`, D-31):**
1. **Parallel run: yes, once, as soon as possible.** Both machines run the same month and the real results are compared. It is the **only** parallel run; once the results are satisfactory, the Mac container is shut down. Scope still open (see Open questions).
   - Constraint (`<owner>`): Stage 4 writes `Summary: Scrubbed Date: <report_date>` to every tagged lead in LawRuler (`Dev/run.py` `_make_request_with_retry`, payload field `Summary`), and Stage 1 drops every lead whose Scrubbed Date is within the last 90 days (`Dev/run.py` `DataIngestionPipeline._select_leads`, log line "removed recently scrubbed"). So the two runs only see the same leads if **both Stage 1 downloads finish before either Stage 4 writes**; a run started after the other's Stage 4 would skip almost every lead, and runs that don't happen together leave mismatched Scrubbed Date entries.
2. **Alert verification: kept.** ITServer dry-run with 2 injected failures (`DRYRUN_INJECT_FAILURES`); the `[DRY-RUN]` email and Teams message arrive within N minutes and list 2 unresolved items. N to be set when the test is written.
3. **First scheduled run: kept and widened** to the first full cycle on the ITServer: run on the 1st, patch pass on the 3rd, watchdog heartbeat on the 5th, all without manual steps.
4. **Rollback plan: kept,** required before cutover: steps written down (stop ITServer timers, start the Mac container, `once` if a month was missed); the Mac container stays stopped, not removed, for one full cycle.
5. **Backfill: option A.** Container-era runs loaded and verified (row counts, checksums) before cutover; compatible legacy runs may follow after cutover.

**Decision 2026-11-20 (`<owner>`, D-32), parallel-run scope:** run as soon as the blocking items close, before 2026-12-01; both hosts write to LawRuler (same tags and Scrubbed Date on the same leads, harmless, and the fingerprint of the test); business recipients get two copies (`<owner>` informs the team); the ITServer run is real and writes the production `dnc` schema; the ITServer takes over the monthly job right after (option (i)). Executed in [[Story - Parallel run and early host move to the ITServer]]. Criteria 3–5 apply to phase 2 ([[Story - Cut over and complete a scheduled run on the ITServer]]); criterion 4 (rollback) applies to both phases.

**Closed 2026-11-20 (`<owner>`):** criteria in D-31/D-32. Details carried to where they're executed: the parallel-run comparison checks are an open question in [[Story - Parallel run and early host move to the ITServer]]; the alert-test time limit N is set when criterion 2 is tested in [[Story - Cut over and complete a scheduled run on the ITServer]].

**Decision authority:** `<owner>` has final say; if she defers further, the minimal criteria are #3 (first scheduled run) + #4 (rollback plan).

## Open questions
- [x] Cutover acceptance criteria (deferred by `<owner>`; this story answers it). ✅ 2026-11-20: D-31 + D-32.
- [x] ✅ 2026-11-20 (D-32): before 2026-12-01 once blockers close; both write LawRuler (identical Scrubbed Date = fingerprint); business gets two copies (`<owner>` tells the team); ITServer writes the production `dnc` schema; the ITServer takes over right after → [[Story - Parallel run and early host move to the ITServer]]. **Scope of the one parallel run (D-31):** what the ITServer instance runs (same SQLite pipeline as the Mac now, or the Postgres pipeline at the end of S2), whether it writes to LawRuler and emails business recipients, which Postgres schema it writes, when it runs, and whether the ITServer takes over the monthly job right after it.

## Dependencies
- None (this is a planning/decision story)
