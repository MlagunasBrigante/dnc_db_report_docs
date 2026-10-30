---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Memory optimization]]"
sprint: S1
moscow: could
status: to-do
gaps: [GAP-17, GAP-18]
jira: ""
size: S
---
# Story: Memory log with quarterly review

## User Story
**As** the IT & Data lead, **I want** every run to append its per-stage memory figures to a small secondary log that I review every quarter, **so that** memory regressions or growth (e.g. as the lead history grows) are caught early without anyone running a manual watcher again.

## Acceptance Criteria
- [ ] **Given** any run (scheduled, manual or patch pass), **when** it finishes (success or failure), **then** one row per stage is appended to `data/metrics/memory_log.csv`. Columns: `run_id/date, trigger, stage, start, end, duration_s, mean_mib, peak_mib, items (leads/batches/pages), leads_in_snapshot, code_version`. The mean comes from in-process sampling (e.g. every 30 s, like the 2026-10-28 manual watcher); the peak from cgroup `memory.peak` / `resource`.
- [ ] **Given** the log, **when** it's opened, **then** it's small (KB per run) and has no secrets or lead-level data.
- [ ] **Given** a quarter has ended, **when** the review is due, **then** a reminder exists (runbook entry plus a calendar or Jira recurring task), and the review compares means per stage, normalized per thousand leads, against the previous quarter and the targets. Findings are written to `Docs/Run Reports/Memory Review <YYYY>-Q<n>.md`.
- [ ] **Given** S2 (Postgres), **when** the migration lands, **then** the same rows are written to a `run_stage_metrics` table and the CSV can be retired.
- [ ] The 2026-10-28 run is back-filled as the first baseline row set, from `data/mem_2026-10-28.csv` and the run report.

## Notes
- **Decided 2026-10-29 (Michelle, Q10, note 2):** "bake a memory watching process: a simple secondary log that we analyze every quarter".
- Builds on [[Story - Per-stage duration and memory in every run]] (which emits the per-stage numbers to the log and Teams). This story makes them **persistent and reviewed**.
- The quarterly review is also where the memory metrics themselves get refined. The initial per-hour-per-thousand-leads figure is provisional (Michelle, 2026-10-29).
- Peaks are recorded but only acted on if a run or series shows non-negligible peaks (epic policy).

## Dependencies
- [[Story - Per-stage duration and memory in every run]]
