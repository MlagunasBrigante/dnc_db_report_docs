---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Memory optimization]]"
sprint: S1
moscow: could
status: to-do
gaps: [GAP-18]
jira: ""
size: M
---
# Story: Stage 4 stops reloading the full lead history

> [!todo] Michelle — action needed
> - [x] Decide which memory figure governs the target (per hour per thousand leads, or plain MiB). → plain Stage 4 mean MiB (2026-11-09).

## User Story
**As** the IT & Data lead, **I want** Stage 4 (LawRuler tagging) to hold only the leads it will tag, not the full lead history, during its long API loop, **so that** its mean memory drops while the tagging stays exactly as reliable as today.

## Acceptance Criteria
- [ ] **Given** Stage 1 has saved the raw snapshot and the cleaned leads to the database, **when** Stage 2 starts, **then** the full-history DataFrames are released and not kept in `main()` or on the pipeline objects (`run.py:205`, `251`, `339`, `1377-1383`).
- [ ] **Given** Stage 4 starts, **when** it builds its working set, **then** it loads only the leads to tag (the join of the cleaned leads and `scrub_results_translated`) and only the columns the tagging and report need. It no longer reads all of `raw_data` (`run.py:663-718`).
- [ ] **Given** the 10-28 baseline (Stage 4: ~1 GB held for ~1.5 h for ~8k leads), **when** a comparable run completes, **then** Stage 4 meets Michelle's **initial target: a 20% reduction**, from ~0.083 to **≤ ~0.06 GB per hour per thousand leads** (see Notes for the formula and caveat). The before/after figures are recorded in the memory log.
- [ ] **Given** the change, **when** a real run completes, **then** the tagged leads, the report counts and the soft-deletion lists **match** a pre-change run on the same data (dry-run comparison), and resume/checkpoint behavior is unchanged: kill and restart mid-Stage 4, and nothing is tagged twice or skipped.
- [ ] **Given** the target is missed but stability is intact, **when** it's reviewed, **then** the change ships anyway if it's a net improvement, and the lessons are written down for [[Story - Stage 2 mean memory reduced]]. **Resilience and stability always win over the memory target.**

## Notes
- **Decided 2026-11-09 (Michelle):** (1) the governing figure is the **plain Stage 4 mean MiB** from the memory log, not GB per hour per thousand leads (not distorted by speed or volume). Baseline 2026-10-28: mean 1,120.6 MiB; dry-run 2026-11-09: mean 1,103 / peak 1,207 MiB. The 20% target becomes **mean ≤ ~900 MiB** (indicative; the AC 3 ratio is superseded). (2) D-06: **no parallel tagging**, Stage 4 stays sequential (stability first, LawRuler is flaky). (3) Build now; merge only after a Mac dry-run with matching outputs and a kill/restart test. The Dec 1 real run is the 'after', Oct 28 the 'before'.
- **Target set by Michelle (stories interview Q10, 2026-10-29):** 10-28 ≈ (1 GB / 1.5 h) / 8 kLeads ≈ 0.083 → −20% (optimistic) ≈ 0.06. It's an **initial** target only.
- **Caveat on the formula (flagged for Michelle):** dividing memory by **hours** means a *faster* Stage 4 at the same memory scores *worse* (e.g. if D-06 parallel tagging is ever adopted). Stage 4's memory is also mostly a **fixed cost** of loading the full history, not a per-lead cost. The memory log therefore also records the plain **Stage 4 mean in MiB** (10-28 baseline 1,121 MiB; −20% ≈ 900 MiB) so either view can be used. Michelle chooses which one governs.
- **The metric is indicative, not final (Michelle, 2026-10-29):** GB per hour per thousand leads reads more like a rate of change and doesn't fully capture every situation. Very large runs, e.g. a ~200k-lead pass, distort it badly. **The real measurable metrics will be adjusted along the way**, informed by the memory log. The figure exists to give an order of magnitude, not a pass/fail gate.
- **Design aspiration:** if the new design can *atomize* Stage 4, processing leads in fixed-size chunks with a bounded working set, its memory becomes **independent of the number of leads and of run time**. That's the preferred outcome and makes the per-lead/per-hour metric moot.
- **Earlier design rejected (2026-10-29):** the scaffolded version passed Stage 1's full DataFrame into Stage 4. That keeps all ~422k leads in memory through the ~1.5 h loop, the opposite of the goal.
- **Root cause (run report 2026-10-28):** Stage 4 start jumped 907 → 1,295 MiB when it re-read 421,776 `raw_data` rows; it then held ~1.1 GiB flat for 88 min.

## Open questions
- [x] #open-question Which memory figure governs the target: GB per hour per thousand leads, or plain Stage 4 mean MiB? Adjust as the memory log matures.
- [x] #open-question D-06: parallelize Stage 4 tagging? Interacts with the memory metric and LawRuler's flakiness.

## Dependencies
- [[Story - Per-stage duration and memory in every run]] (before/after measurement)
- [[Story - Memory log with quarterly review]] (where the result is recorded)
