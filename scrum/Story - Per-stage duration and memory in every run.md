---
type: story
project: dnc-cron-scrub
epic: "[[Epic - No silent data loss]]"
sprint: S1
moscow: should
status: in-review
gaps: [GAP-17]
jira: ""
size: M
---
# Story: Per-stage duration and memory in every run

## User Story
**As** the IT & Data lead, **I want** per-stage start/end/duration and peak memory logged and included in the Teams run summary (the ops channel), **so that** I can see resource usage trends and the memory-optimization epic has baseline data.

## Acceptance Criteria
- [x] **Given** the pipeline runs, **when** each stage starts and ends, **then** the start time, end time, duration, and peak memory are logged at INFO level (e.g. "Stage 1 Ingest: 20:46:48 → 21:15:56, duration 29m 08s, peak memory 788 MiB").
- [x] **Given** the run completes successfully, **when** the Teams run-summary message is sent, **then** it includes a summary table of per-stage durations and peak memory (e.g. "Stage 1: 29m, 788 MiB; Stage 2: 2m, 907 MiB; …").
- [x] **Given** the memory instrumentation, **when** peak memory is measured, **then** it uses cgroup `memory.peak` (Linux; most accurate) or `resource.getrusage` (fallback; macOS/Linux) or `psutil.Process.memory_info().rss` (cross-platform fallback).
- [ ] **Given** one real run completes, **when** I review the log and the Teams summary, **then** the per-stage metrics are present and match expected values (cross-check with the 2026-10-28 run report for sanity).

## Notes
- **2026-11-10: AC1–AC3 ticked on the Mac dry-runs of 2026-11-06 and 2026-11-10** (INFO stage lines and Teams per-stage table present; the card lists Stages 1–5 because it is sent during Stage 6, whose own row is in the log and the memory log). AC3 deviation, accepted: per-stage peak comes from sampling cgroup v2 `memory.current` (1 s) because `memory.peak` is a container-lifetime high-water mark and can't be split per stage; the fallback is `getrusage ru_maxrss`; `memory.peak` is still read for the run-wide peak; no psutil dependency. AC4 waits for the Dec 1 real run.
- **Mac dry-run 2026-11-06 19:58 UTC** (merged `efe2647`): source cgroup v2; Stage 1 28m 23s / 821 MiB, Stage 2 0s / 710, Stage 3 0s / 709, Stage 4 1s / 1269, Stage 5 0s / 1085, Stage 6 2s / 1073; run-wide peak 1297 MiB, exit 0. The 1 s sampler missed ~28 MiB of the run-wide peak (acceptable). Soft-deletion email (between Stages 5 and 6) is not timed. Teams success card shows the per-stage list (Michelle, 2026-11-06); container recreated from `efe2647`. Dec 1 real-run comparison with the 2026-10-28 baseline still open.
- **Gap reference**: GAP-17 — "No per-stage duration or memory instrumentation" at `main()` (`run.py:1318-1445`)
- **Root cause from run report 2026-10-28**: the per-stage timeline and memory table in the run report was built from external `docker stats` samples and manual log inspection because the pipeline itself does not instrument stages. This story adds that instrumentation into the pipeline.
- **Memory measurement** (from 2026-10-28 run report):
  - **Preferred (Linux)**: cgroup `memory.peak` — read `/sys/fs/cgroup/memory.peak` or `/sys/fs/cgroup/memory/memory.max_usage_in_bytes` (depending on cgroup v1 vs v2). This was used in the 10-28 run report and is the most accurate.
  - **Fallback (POSIX)**: `resource.getrusage(resource.RUSAGE_SELF).ru_maxrss` — returns bytes (Linux) or KB (macOS). Less accurate (does not include child processes) but portable.
  - **Fallback (cross-platform)**: `psutil.Process(os.getpid()).memory_info().rss` — requires psutil package; cross-platform but only current RSS, not peak.
- **Implementation approach**:
  - Wrap each stage in a decorator or context manager that records start time, end time, and peak memory at stage end.
  - Store metrics in a list or dict in `main()`.
  - Log metrics at INFO after each stage.
  - Include the metrics in the Teams run-summary message, formatted as a table or bullets. **Not** in the business report email ([[Story - Monthly report email as a summary table]]), which goes to management and an external recipient. Aligned 2026-10-29.
- **Scope**: this story instruments Stages 1-5 (Ingest, Scrub, Interpret, Update, Report). Stage 6 (Email) metrics are less useful (the email-sending itself is fast; memory spikes are from attachment encoding, which [[Story - Monthly report email as a summary table]] fixes). Still log Stage 6 for completeness.
- **Feeds the memory epic** (GAP-18): the memory-optimization stories ([[Story - Stage 4 stops reloading the full lead history]], [[Story - Stage 2 mean memory reduced]]) depend on this instrumentation to measure before/after.
- **Implementation (2026-11-06)**: Branch `s1/feat/per-stage-metrics`, commit a3ca7c5. Per-stage peak memory is measured by **sampling current memory usage** (not cgroup memory.peak, which is monotonic across the whole run). A lightweight daemon thread samples cgroup v2 `memory.current` (or v1 / VmRSS / ru_maxrss fallback) every `STAGE_MEMORY_SAMPLE_SECONDS` (default 1.0 s) and takes the max. This gives per-stage peaks instead of a run-wide high-water mark. The run-wide peak from `memory.peak` is logged separately at the end. Measurement errors never break a run (log DEBUG, continue with 'n/a').

## Dependencies
None — can start immediately. Should complete before the memory-optimization epic starts.
