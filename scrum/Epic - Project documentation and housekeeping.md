---
type: epic
project: dnc-cron-scrub
sprint: S1
status: to-do
gaps: [GAP-22, GAP-23]
---
# Epic: Project documentation and housekeeping

## Description
Complete the project restructure and bring documentation current: write the per-stage inputs/outputs reference (feeds S2 Postgres schema), update READMEs for the Container→Dev and DNC-Update→dnc-cron-scrub renames, re-scope Jira DNCUP tickets to the new backlog, and fix the Jira↔vault sync for the renamed folder.

## Gaps closed
- **GAP-22** — No per-stage inputs/outputs document (Jira DNCUP-5)
- **GAP-23** — Stale README and Jira (DNCUP-2/3/6 superseded); docs still say `Container/`; Jira↔vault sync broken for the renamed folder

## Breakdown

### INVEST check

| Criterion | Rating | Notes |
|---|---|---|
| Independent | ✅ | No technical dependency on other S1 epics; pure documentation |
| Negotiable | ✅ | Jira ticket disposition (close vs re-scope) is open; sync-SOP update method is open |
| Valuable | ✅ | S2 schema designer has the stage contract; new contributors find current docs; Jira↔vault sync works |
| Estimable | ✅ | Scope lists exactly which docs, Jira tickets, and sync mappings to update |
| Testable | ✅ | READMEs read correctly; Jira tickets map to stories or are closed; sync SOP references the renamed path |

### Pattern applied
**Pattern 2: Operations** (write reference doc, update READMEs, re-scope Jira, fix sync)

Earlier patterns rejected:
- Pattern 1 (Workflow Steps): these are independent documentation tasks, not sequential stages of one workflow.

### Cynefin calibration
**Low uncertainty** — All documentation locations and Jira tickets are known; the per-stage I/O contract is visible in the code and run report; README updates are text edits; Jira re-scoping is a mapping exercise.

### Split evaluation
Each story delivers value to a different persona: the per-stage reference doc serves the S2 schema designer and future maintainers; the README updates serve new contributors; the Jira re-scoping serves the backlog consumer (Michelle); the sync fix serves the Jira↔vault automation. Sizes are S to M; all are vertical (each closes one documentation gap end-to-end).

> [!check] Before closing this epic, verify Michelle's actions were actually done
> - [ ] Approve the Jira DNCUP re-scope and the Jira↔vault sync mapping fix (then run `reset sync baseline`). — [[Story - README, Jira and vault references re-scoped]]

## Child user stories
- [[Story - Per-stage inputs and outputs reference]]
- [[Story - README, Jira and vault references re-scoped]]

## Dependencies / sequencing
The per-stage reference doc should complete early (it feeds S2 schema design). The README/Jira story can start any time.
