---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Memory optimization]]"
sprint: S1
moscow: could
status: in-review
gaps: [GAP-17, GAP-18]
jira: ""
size: S
---
# Story: Memory log with quarterly review

## User Story
**As** the IT & Data lead, **I want** every run to append its per-stage memory figures to a small secondary log that I review every quarter, **so that** memory regressions or growth (e.g. as the lead history grows) are caught early without anyone running a manual watcher again.

## Acceptance Criteria
- [x] **Given** any run (scheduled, manual or patch pass), **when** it finishes (success or failure), **then** one row per stage is appended to `data/metrics/memory_log.csv`. Columns: `run_id/date, trigger, stage, start, end, duration_s, mean_mib, peak_mib, items (leads/batches/pages), leads_in_snapshot, code_version`. The mean comes from in-process sampling (e.g. every 30 s, like the 2026-10-28 manual watcher); the peak from cgroup `memory.peak` / `resource`.
- [x] **Given** the log, **when** it's opened, **then** it's small (KB per run) and has no secrets or lead-level data.
- [x] **Given** a quarter has ended, **when** the review is due, **then** a reminder exists (runbook entry plus a calendar or Jira recurring task), and the review compares means per stage, normalized per thousand leads, against the previous quarter and the targets. Findings are written to `Docs/Run Reports/Memory Review <YYYY>-Q<n>.md`.
- [ ] **Given** S2 (Postgres), **when** the migration lands, **then** the same rows are written to a `run_stage_metrics` table and the CSV can be retired.
- [x] The 2026-10-28 run is back-filled as the first baseline row set, from `data/mem_2026-10-28.csv` and the run report.

## Notes
- **Reviewed 2026-11-09 (branch `s1/feat/memory-log`, tip `c9dad97`, 284 tests).** Review found and fixed: the subagent had not run the tests (5 failed); baseline CSV had mixed line endings, notebook-style stage names and the filtered lead count (now live names `Stage 1 Ingest`…`Stage 6 Alerts`, `leads_in_snapshot` 421,776, `\n` only); the snapshot-lead count opened the ODS DB read-write and could create an empty DB on early failures (now read-only); README build command now runs from `Dev/`. The runbook's quarterly section quoted a made-up target; it now cites the epic's indicative Stage 4 target. Item counts per stage are left blank for now.
- **Decided 2026-10-29 (Michelle, Q10, note 2):** "bake a memory watching process: a simple secondary log that we analyze every quarter".
- Builds on [[Story - Per-stage duration and memory in every run]] (which emits the per-stage numbers to the log and Teams). This story makes them **persistent and reviewed**.
- The quarterly review is also where the memory metrics themselves get refined. The initial per-hour-per-thousand-leads figure is provisional (Michelle, 2026-10-29).
- Peaks are recorded but only acted on if a run or series shows non-negligible peaks (epic policy).
- **Implementation branch:** `s1/feat/memory-log` (commit 2c77133), backfill complete, baseline rows appended to `/workspace/vault/Projects/dnc-cron-scrub/data/metrics/memory_log.csv`. 8 new tests added to `tests/test_memory_log.py`. Tests fixed for sampler API change; lineterminator enforced to `\n`; stage names and leads_in_snapshot corrected in backfill.
- > [!todo]
  > **Michelle:** Create recurring Outlook event 'DNC memory review' for first week of Jan/Apr/Jul/Oct (quarterly).

## Dependencies
- [[Story - Per-stage duration and memory in every run]]
