---
type: story
project: dnc-cron-scrub
epic: "[[Epic - No silent data loss]]"
sprint: S1
moscow: must
s1_order: 0
status: in-review
gaps: [GAP-27, GAP-28]
jira: ""
size: M
---
# Story: Dry-run never calls paid APIs or touches real data

## User Story
**As** the pipeline owner, **I want** `--dry-run` to cost nothing and leave no trace in real run data, **so that** I can test the pipeline as often as needed without paying for dncscrub scrubs, poisoning a real run's stored results, or emailing business recipients.

## Acceptance Criteria
- [ ] **Given** `once --dry-run`, **when** Stage 2 runs, **then** no request is ever built to the dncscrub URL. A fake scrubber returns responses in dncscrub's real 17-field format, with a deterministic, realistic mix of result codes, so Stages 3–6 run end to end. A unit test checks that `requests.post` is never called with the scrub URL in dry-run mode.
- [ ] **Given** a dry-run, **when** it writes artifacts, **then** everything goes to `data/_dryrun/<date>T<time>Z/`, never to `data/<date>/`. Two dry-runs never share a folder, and `OUTPUT_DIR`/`LOG_DIR` overrides are ignored in dry-run.
- [ ] **Given** a dry-run, **when** it logs and reports, **then** it is flagged: an INFO banner "DRY-RUN: scrub responses are synthetic; no paid APIs called", and `trigger=dry-run` in the run metadata. Real runs carry `scheduled` or `manual`.
- [ ] **Given** a dry-run finishes (success or failure), **when** cleanup runs, **then** its SQLite databases are deleted and only the logs and report files are kept. At the start of each dry-run, `data/_dryrun/` folders older than `DRYRUN_RETENTION_DAYS` (default 7) are purged. Nothing outside `data/_dryrun/` is ever deleted.
- [ ] **Given** a dry-run, **when** emails would be sent, **then** they go only to `DRYRUN_EMAIL_RECIPIENTS` with a `[DRY-RUN]` subject prefix. If that's unset, they're skipped and logged. Never to business recipients, the external recipient or the LawRuler account manager.
- [ ] Verified: unit tests pass, plus one dry-run on Michelle's Mac. **The dncscrub account shows no new usage**, and `data/<today>/` is untouched.

## Notes
- **Decided 2026-11-02 (Michelle):** option B (dummy responses), and test runs must not poison stored data. The fix combines isolating and flagging test runs with cleaning up after them.
- **Found 2026-11-02:** today's `--dry-run` only skips the LawRuler *updates* (`run.py:1015`, `:1510`). It still pays for every dncscrub scrub (GAP-27). It also writes into the same `data/<date>/` folder as a real run, including the Stage 2 `batch_status` checkpoints (`run.py:489-506`). A dry-run followed by a real run on the same day would skip those batches and keep the synthetic results (GAP-28). It also emails the real recipients.
- **General rule (applies to every later story):** in dry-run, paid APIs are never called. The free calls (LawRuler token, Custom Report download) keep running.
- **Until this ships, don't run any `--dry-run`.**
- Branch: `s1/fix/dry-run-no-paid-calls`.

> [!todo] Michelle — action needed
> - [ ] Choose a test recipient for `DRYRUN_EMAIL_RECIPIENTS` (e.g. yourself), or leave it unset to skip emails in dry-runs.
> - [ ] After the first fixed dry-run, check the dncscrub account shows no new usage.

## Dependencies
- Comes before every other dry-run-based verification in S1 (`s1_order: 0`).
