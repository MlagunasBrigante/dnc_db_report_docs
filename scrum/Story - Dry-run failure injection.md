---
type: story
project: dnc-cron-scrub
epic: "[[Epic - No silent data loss]]"
sprint: S1
moscow: should
status: done
gaps: []
jira: ""
size: S
---
# Story: Dry-run failure injection

## User Story
**As** the pipeline owner, **I want** a dry-run to simulate specific API failures from a setting in `.env`, **so that** I can watch the real failure handling (retries, failed items, outage pause, INCOMPLETE marker, report row, warning email) end to end on my Mac without editing code or waiting for LawRuler to fail.

## Acceptance Criteria
- [ ] **Given** `DRYRUN_INJECT_FAILURES` in `.env` (comma list of `page:N`, `batch:N`, `lead:N`, `outage:stage1|stage2|stage4`), **when** a `--dry-run` runs, **then** the matching calls fail on every attempt *instead of* calling the API, and all downstream handling is the real production code.
- [ ] **Given** `outage:<stage>`, **when** the breaker trips in a dry-run, **then** pauses are shortened to seconds so the test finishes quickly; real runs keep the configured pause.
- [ ] **Given** the setting is present in a **real** (non-dry-run) run, **when** the run starts, **then** it is ignored and a WARNING says so. A unit test proves a non-dry-run never injects.
- [ ] **Given** malformed tokens, **when** the run starts, **then** it logs which tokens were ignored and continues.
- [ ] Verified on Michelle's Mac with `DRYRUN_INJECT_FAILURES=page:5,batch:1`: the `INCOMPLETE` marker, "Raw snapshot complete: NO", and page 5 and batch 1 in the failed-items block of the log and warning email (or the "would send" line if `DRYRUN_EMAIL_RECIPIENTS` is unset).

## Notes
- **Mac dry-run 2026-11-04** (`data/_dryrun/2026-11-04T201311Z/`, `DRYRUN_INJECT_FAILURES=batch:0`): batch 0 failed after 3 attempts with no waits and no network call. Together with the earlier `page:5` run, both injection kinds are proven on the Mac. The injection also exposed a real bug (GAP-29, see [[Story - Failed scrub batches are retried and reported]]).
- **Mac dry-run 2026-11-04** (`data/_dryrun/2026-11-04T190951Z/`, `DRYRUN_INJECT_FAILURES=page:5,batch:1`; all five S1 branches merged and pushed, `main` @ `33558a0`): `page:5` worked end to end with no retry waits. `batch:1` did not fire because only batch 0 existed: batch IDs are 0-based (`phone index // 1000`). README and `.env.example` give `batch:1` as the example; fix that wording to note 0-based IDs and recommend `batch:0`. The story closes after the `batch:0` re-check.
- Requested by Michelle 2026-11-02. It enables Mac verification of [[Story - Lead download never truncates silently]], [[Story - Failed scrub batches are retried and reported]] and every later failure-related story (alerts, Teams, delivery-failure alert, S2 patch pass and watchdog).
- Free by construction: injected failures make no calls, and dry-runs never call paid APIs.
- Done out of `s1_order` sequence on purpose. It's a Should, but it's the tool that lets the in-review Musts be verified.
- Branch: `s1/fix/dry-run-failure-injection`, off `main` once the two in-review branches are merged (or rebased onto them).

## Dependencies
- [[Story - Dry-run never calls paid APIs or touches real data]] (done)
- Best after [[Story - Lead download never truncates silently]] and [[Story - Failed scrub batches are retried and reported]] are merged, so their behaviour can be demonstrated.
