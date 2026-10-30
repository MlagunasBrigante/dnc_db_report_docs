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
# Story: Stage 2 mean memory reduced

## User Story
**As** the IT & Data lead, **I want** Stage 2 (DNC scrub), the stage with the second-highest mean memory, to use less memory **using the design lessons from Stage 4**, **so that** the job's overall mean drops further without touching its reliability.

## Acceptance Criteria
- [ ] **Given** [[Story - Stage 4 stops reloading the full lead history]] is done and stable for at least one monthly run, **when** this story starts, **then** its lessons are written down and a short investigation names what Stage 2 holds in memory and why. On 10-28, Stage 2's mean was 744 MiB, and the Stage 1 data was likely still referenced.
- [ ] **Given** the investigation, **when** the change ships, **then** Stage 2's mean memory is lower than the baseline in the memory log, with a target agreed with Michelle after the investigation, using the same formula as Stage 4.
- [ ] **Given** the change, **when** a real run completes, **then** scrub results and batch checkpointing are unchanged: same results on the same input; kill and restart mid-Stage 2 resumes without re-scrubbing completed batches.
- [ ] **Resilience and stability win over the memory target**, as in Stage 4.

## Notes
- **Priority 2** (Michelle, 2026-10-29): attended only after Stage 4 is stabilized and its key design lessons are learned.
- Focus is the **mean**; peaks only if manual runs show them to be non-negligible.
- Replaces the scaffolded "Stage 1 streams pages instead of holding them" story. Stage 1 has the lowest mean (409 MiB) and isn't prioritized; it's revisited only if the quarterly memory review shows growth.

## Dependencies
- [[Story - Stage 4 stops reloading the full lead history]] (done, stable, lessons recorded)
- [[Story - Memory log with quarterly review]]
