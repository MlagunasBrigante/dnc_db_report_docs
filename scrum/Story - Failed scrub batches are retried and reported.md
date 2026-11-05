---
type: story
project: dnc-cron-scrub
epic: "[[Epic - No silent data loss]]"
sprint: S1
moscow: must
s1_order: 4
status: done
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
- **Mac dry-run 2026-11-04** (`data/_dryrun/2026-11-04T222929Z/`, `main` @ `ea3d03c`, `DRYRUN_INJECT_FAILURES=batch:0`): batch 0 failed after 3 attempts. The CRITICAL "no scrub results: all 1 batch(es) failed/deferred" line was logged, Stages 3–4 were skipped, and the report was written with zero scrubbed. "Failed items: 1" and the Stage 2 block listing batch 0 came before the warning email (4 warnings collected). Exit 0, SQLite DBs cleaned up. Follow-ups: a degraded run still prints "✅ Pipeline completed successfully", and the duplicate ERROR "[Batch 0] Injected failure processed" is noise.
- **In review 2026-11-04** (`a2ca346` + review fixes `ea3d03c`). The review found three more crash paths, now fixed:
  - A run with **zero phones to scrub** (every lead recently scrubbed) also crashed in Stage 3. It now skips Stages 3–4 with a WARNING.
  - The degraded report re-read an empty CSV, which would have raised `EmptyDataError`.
  - The batch count in the CRITICAL line included Stage 1 page failures.

  The report code (pandas) can't run in the sandbox, so the Mac `batch:0` dry-run is the real check.
- **Decided 2026-11-04 (Michelle, option A, GAP-29):** when every batch fails or is deferred, the run **finishes as degraded**. It logs CRITICAL "no scrub results: all N batch(es) failed/deferred" and skips Stages 3–4. The report still runs with zero scrubbed counts, and the warning email lists the failed batches. Exit 0, no error email. The S2 patch pass retries the batches. Branch: `s1/fix/all-batches-failed-degraded`, stacked on the email branches. Verify with `DRYRUN_INJECT_FAILURES=batch:0` (a dry-run with < 1,000 phones has only batch 0).
- **Mac dry-run 2026-11-04** (`data/_dryrun/2026-11-04T201311Z/`, `DRYRUN_INJECT_FAILURES=batch:0`): **bug found (GAP-29).** When *every* batch fails (here the only batch: 863 phones means one batch), Stage 3 raises `ValueError: No scrub results found. Run ScrubbingPipeline first.` The run then crashes with a misleading traceback, the failed-items summary is never logged, and the error email doesn't list the failed batches. In a real run, a dncscrub outage that defers every batch would end the same way. Partial failure (some batches OK) is still unverified on the Mac: dry-runs currently have fewer than 1,000 phones, so there's only one batch. The fix needs Michelle's decision (all-failed behaviour) before the story can close.
- **Mac dry-run 2026-11-04** (`data/_dryrun/2026-11-04T190951Z/`, `DRYRUN_INJECT_FAILURES=page:5,batch:1`; all five S1 branches merged and pushed, `main` @ `33558a0`): "Failed items: 1" and the per-stage WARNING block (page 5) reached the warning email. **The batch path was not exercised:** this dry-run had only 739 phones, so there was a single batch, and batch IDs start at **0**, so `batch:1` never existed. Re-check with `DRYRUN_INJECT_FAILURES=batch:0`. Unit tests already cover the batch path.
- **Code reference** (verified against `Dev/run.py`):
  - Batch scrub: `run.py:506-508` — exception caught, logged as ERROR, but batch is not marked done and not retried within the run
  - Comment: `run.py:509` — "Do NOT mark as done — next run will retry this batch" is false; the next run re-fetches Stage 1 data and re-determines batches, so a failed batch is only retried if the same phones appear again
- **Root cause from run report 2026-10-28 (GAP-10)**: "A failed Stage-2 batch is skipped; the 'next run retries' comment is false" — a batch exception is caught and logged, but the batch is not marked done and is not retried, so the phones in that batch are never scrubbed.
- **Scope**: this story adds within-run retry for failed batches and lists unresolved failures in alerts. The general failed-items store and scheduled patch pass for every stage are S2 features (out of scope for S1), but this story adds a minimal failed-items store (e.g. a table in the staging DB) to track which batches failed.
- **Project rule**: "LawRuler is flaky → never 'fail the run if N items fail'". This story respects that — a failed batch is retried and reported, but the run does not fail just because one batch failed. The alert makes the failure visible.
- **Within-run retry**: after the exception at `run.py:507`, attempt the batch one more time before moving to the next batch. If still failing, log CRITICAL, record in failed-items store, and continue.

## Dependencies
- [[Story - One retry policy for every API call]]
