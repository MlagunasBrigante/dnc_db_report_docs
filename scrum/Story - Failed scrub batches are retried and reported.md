---
type: story
project: dnc-cron-scrub
epic: "[[Epic - No silent data loss]]"
sprint: S1
moscow: must
s1_order: 4
status: to-do
gaps: [GAP-10]
jira: ""
size: M
---
# Story: Failed scrub batches are retried and reported

## User Story
**As** the IT & Data lead, **I want** a failed Stage 2 batch to be retried within the same run and, if still failing, listed in the alerts, **so that** batch failures are never silently skipped with a false promise that "next run will retry".

## Acceptance Criteria
- [ ] **Given** a scrub batch fails (exception in `_make_request_with_retry`, `run.py:506-508`), **when** the exception is caught, **then** it is retried by the shared retry policy ([[Story - One retry policy for every API call]]) within the same run, not deferred to the next run.
- [ ] **Given** a batch still fails after the within-run retry, **when** the run completes, **then** the alert email and Teams message list the failed batch ID and the error, and the batch appears as a failed-item event in the run log and summary. Persistent storage comes in S2 ([[Story - Failed items captured per stage]]).
- [ ] **Given** a batch fails transiently but succeeds on retry, **when** the run completes, **then** no warning is emitted (transient errors that self-heal do not count as warnings).
- [ ] **Given** the comment "Do NOT mark as done — next run will retry this batch" at `run.py:509`, **when** the fix is applied, **then** that comment is removed (it is false — the next run does NOT automatically retry a failed batch today; that's an S2 feature).
- [ ] Verified by injecting a batch failure (e.g. mocking the scrub API to fail for batch 3) and confirming the retry happens and the alert lists unresolved failures.

## Notes
- **Code reference** (verified against `Dev/run.py`):
  - Batch scrub: `run.py:506-508` — exception caught, logged as ERROR, but batch is not marked done and not retried within the run
  - Comment: `run.py:509` — "Do NOT mark as done — next run will retry this batch" is false; the next run re-fetches Stage 1 data and re-determines batches, so a failed batch is only retried if the same phones appear again
- **Root cause from run report 2026-10-28 (GAP-10)**: "A failed Stage-2 batch is skipped; the 'next run retries' comment is false" — a batch exception is caught and logged, but the batch is not marked done and is not retried, so the phones in that batch are never scrubbed.
- **Scope**: this story adds within-run retry for failed batches and lists unresolved failures in alerts. The general failed-items store and scheduled patch pass for every stage are S2 features (out of scope for S1), but this story adds a minimal failed-items store (e.g. a table in the staging DB) to track which batches failed.
- **Project rule**: "LawRuler is flaky → never 'fail the run if N items fail'". This story respects that — a failed batch is retried and reported, but the run does not fail just because one batch failed. The alert makes the failure visible.
- **Within-run retry**: after the exception at `run.py:507`, attempt the batch one more time before moving to the next batch. If still failing, log CRITICAL, record in failed-items store, and continue.

## Dependencies
- [[Story - One retry policy for every API call]]
