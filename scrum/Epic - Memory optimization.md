---
type: epic
project: dnc-cron-scrub
sprint: S1
status: to-do
gaps: [GAP-18]
---
# Epic: Memory optimization

## Description
Lower the job's **mean** memory use, one stage at a time, **always favoring resilience and stability over memory savings**. Priority 1 is Stage 4 (LawRuler tagging), the longest and heaviest stage. Priority 2 is Stage 2 (DNC scrub), started only after Stage 4 is stable and its design lessons are written down. A lightweight **memory log with a quarterly review** keeps watching afterwards. This is the closing item of S1: it starts only after the other S1 epics are done and the code is stable. The ITServer has 16 GB of RAM, so the goal is efficiency, not making the job fit.

## Optimization policy (Michelle, 2026-10-29)
- **Optimize means, not peaks.** On 2026-10-28, each stage's mean sat very close to its own peak, so the mean is the lever. Peaks get attention **only if** a manual run, or a series of runs, shows non-negligible peaks.
- **The Stage 6 email peak (2.77 GiB) is removed by design**, not by this epic: no report attachments and no full-log attachments ([[Story - Monthly report email as a summary table]], [[Story - Warning and error emails always arrive]]).
- **Resilience first.** A memory change that makes the run less robust (e.g. weaker checkpointing or retries) is rejected even if it saves memory.
- **Metrics are indicative and will be adjusted along the way.** The initial Stage 4 target (≈0.06 GB per hour per thousand leads) only gives an order of magnitude and breaks down at extreme volumes. Preferred design outcome: an atomized, chunked process whose memory doesn't depend on lead count or run time.
- Stage 1 (mean 409 MiB, the lowest) is **not** prioritized. It's revisited only if the quarterly review shows it growing.

## Gaps closed
- **GAP-18** — Full lead history held twice in memory (Stage 1 + Stage 4 re-read)

## Breakdown

### INVEST check
| Criterion | Rating | Notes |
|---|---|---|
| Independent | ⚠️ | Starts only after the other S1 epics are done and the code is stable. Stage 2 waits for Stage 4. |
| Negotiable | ✅ | Implementation approach per stage is open; targets are initial and can be revised |
| Valuable | ✅ | Lower mean memory on the shared ITServer, plus a standing watch on memory |
| Estimable | ✅ | Baseline in `Docs/Run Reports/DNC Run Report 2026-10-28.md`; code locations known |
| Testable | ✅ | Before/after with [[Story - Per-stage duration and memory in every run]] |

### Pattern applied
**Pattern 8: Defer Performance** plus a priority order. Correctness and stability come from the rest of S1. Memory work then goes stage by stage, Stage 4 first, so its lessons inform Stage 2. Pattern 6 (Major Effort) was considered, but the stages differ too much for "one + the rest"; Pattern 1 doesn't fit because these aren't workflow steps a user sees.

### Cynefin calibration
**Low–medium uncertainty.** The Stage 4 cause is known (full `raw_data` re-read held through the tagging loop). Stage 2's cause is less clear, so its story starts with a short investigation informed by the Stage 4 lessons.

### Split evaluation
- **Reveals low-value work?** Yes: Stage 1 streaming was dropped as low value (lowest mean).
- **Equal sizes?** Roughly: Stage 4 is M, Stage 2 is M, and the memory log is S.

> [!check] Before closing this epic, verify Michelle's actions were actually done
> - [ ] Decide which memory figure governs the target (per hour per thousand leads, or plain MiB). — [[Story - Stage 4 stops reloading the full lead history]]

## Child user stories
1. [[Story - Stage 4 stops reloading the full lead history]] — priority 1
2. [[Story - Memory log with quarterly review]] — ships with priority 1 so Stage 4's result is recorded
3. [[Story - Stage 2 mean memory reduced]] — priority 2, only after Stage 4 is stable

## Dependencies / sequencing
1. The other four S1 epics are done and the code is stable (at least one verified run with the S1 fixes).
2. [[Story - Per-stage duration and memory in every run]] is complete, so before/after is measurable.
3. Stage 4 → stabilize and record lessons → Stage 2.
