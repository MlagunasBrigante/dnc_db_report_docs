---
type: story
project: dnc-cron-scrub
epic: "[[Epic - No silent data loss]]"
sprint: S1
moscow: must
s1_order: 3
status: to-do
gaps: [GAP-26, GAP-09, GAP-10, GAP-08]
jira: ""
size: M
---
# Story: One retry policy for every API call

## User Story
**As** the IT & Data lead, **I want** every external API call (LawRuler token, LawRuler Custom Report pages, dncscrub batches, LawRuler lead updates) to go through one shared retry policy that emits a structured "failed item" event when retries run out, **so that** every stage handles failures the same way, nothing is silently dropped, and the S2 failed-items store only has to persist these events.

## Acceptance Criteria
- [ ] **Given** any of the four call sites, **when** it receives a 429, 5xx, timeout or connection error, **then** it is retried by the shared helper with that API's configured policy. LawRuler gets a longer, more patient backoff than dncscrub. All values live in config or constants in one place, not in each stage.
- [ ] **Given** a non-retryable client error (4xx other than 429), **when** it occurs, **then** it is not retried and produces a failed-item event immediately, with the response body truncated and secrets masked.
- [ ] **Given** retries are exhausted, **when** the helper gives up, **then** it emits one structured event `{stage, item_key, error, attempts, first_seen, last_seen}`, which is logged once at ERROR level and counted in the run summary and alerts. The calling stage continues with the next item; it doesn't abort or break its loop.
- [ ] **Given** an item fails and then succeeds on a later attempt, **when** the run completes, **then** it is **not** counted as a warning or failure; it only shows up in a DEBUG-level attempt log.
- [ ] **Given** injected failures (mocked 500 on a Stage-1 page, timeout on a Stage-2 batch, connection error on a Stage-4 lead, 503 on the token call), **when** the tests run, **then** each produces exactly one failed-item event after the configured attempts, and recovered calls produce none. Verified by unit tests plus one `once --dry-run` with injection.

## Notes
- **Today, three separately coded loops with different rules, plus one call with none (GAP-26):**
  - Stage 1 page `run.py:282-304`: retries timeouts and connection errors only, 5/10/20 s; a non-200 response `break`s (GAP-09).
  - Stage 2 batch `run.py:513-535`: retries 429/5xx, 1/2/4 s; when retries run out the batch is skipped (GAP-10).
  - Stage 4 lead `run.py:866-909`: retries 429/5xx and connection errors, 1/2/4 s; when retries run out it logs and the lead is still marked `DONE` (GAP-08, fixed in S2).
  - LawRuler token `run.py:146`: no retry, so a failure kills the run.
- **Implementation choice is open:** a small in-house helper vs `tenacity` vs a `urllib3.Retry` adapter on a shared `requests.Session`. Pick the simplest that can emit the structured event. Add any new dependency to `Dev/requirements.txt`, pinned.
- **The flaky-API rule applies:** the helper never fails the whole run because items failed. Whatever is still failing after retries is reported, not fatal. The exception is the token call: without a token nothing can run, so token exhaustion is fatal and raises the error alert.
- **S1 storage of events:** the run log plus the run summary is enough. **S2** ([[Story - Failed items captured per stage]]) persists the same events in Postgres, and the patch pass replays them.

## Dependencies
- None. **Do this first in the epic**; [[Story - Lead download never truncates silently]] and [[Story - Failed scrub batches are retried and reported]] build on it.
- Works with [[Story - Warning and error emails always arrive]] (only final failures count) and [[Story - INFO logs with secret masking]] (attempt details at DEBUG level).
