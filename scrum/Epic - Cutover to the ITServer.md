---
type: epic
project: dnc-cron-scrub
sprint: S2
status: to-do
gaps: [GAP-13, GAP-25]
---
# Epic: Cutover to the ITServer
## Description
Move the job and data from Michelle's Mac to the AlmaLinux ITServer. The old Mac container is retired, and the first scheduled run on the 1st completes successfully on the ITServer. Cutover acceptance criteria are defined collaboratively with Michelle before the move (e.g. parallel run comparison, alert verification). The physical cutover is not rushed — this epic closes S2 only after quality is proven.

## Gaps closed
- **GAP-13**: Mac sleep pauses runs (19 h on 10-01, 56 min late start on 10-28) — ITServer is a server, stays on
- **GAP-25**: About 4.1 GB of lead PII sits in OneDrive (data/legacy/) with no retention policy — moves to the ITServer at cutover; S5 handles retention

## Breakdown

### INVEST check
| Criterion | Rating | Notes |
|---|---|---|
| Independent | ⚠️ | Depends on every other S2 epic (Postgres, failures, scheduling, backfill must all be done) |
| Negotiable | ✅ | Acceptance criteria are deferred; could be as simple as "one successful run" or as thorough as "parallel Mac + ITServer runs with diff" |
| Valuable | ✅ | Michelle's Mac is freed; job reliability improves (no sleep); PII off OneDrive |
| Estimable | ⚠️ | Story 1 (define criteria) is a decision story, quick; Story 2 (cutover + first run) is concrete but depends on Story 1's criteria |
| Testable | ⚠️ | The acceptance criteria themselves define the test |

### Splitting pattern applied
**Pattern 1: Workflow Steps** — define criteria, then execute cutover. Story 1 is atypical (produces a decision, not code), but it's a necessary prerequisite to Story 2.

Earlier patterns rejected:
- Pattern 6 (Major Effort) — cutover isn't incremental; it's binary (job runs on Mac or ITServer, not both)
- Pattern 7 (Simple/Complex) — the complexity is in the preparation (other epics), not in the cutover itself

### Cynefin calibration
**Story 1 (define criteria): Complicated** — the options are known (parallel run, alert test, data diff), but Michelle's risk tolerance and the legal/compliance bar aren't yet stated.

**Story 2 (cutover): Complicated** — assuming the other S2 epics are done, cutover is: rsync data/, docker pull on the ITServer, systemctl start, watch the first run. The unknowns are environmental (will the ITServer's Postgres credentials, Docker setup, and network reach LawRuler/dncscrub/Graph all work?), not conceptual.

### Split evaluation
- **Does it reveal low-value work?** No — cutover is the point of S2.
- **Are the sizes roughly equal?** Story 1 is XS (a meeting + a doc update), Story 2 is Small-to-Medium (cutover + verification); imbalance is acceptable given Story 1's nature.

> [!check] Before closing this epic, verify Michelle's actions were actually done
> - [ ] Choose the backup target (cloud and/or external SSD). — [[Story - Backups and restore drill for the dnc database]]
> - [ ] Store the backup encryption key in the team password manager, with a second copy. — [[Story - Backups and restore drill for the dnc database]]
> - [ ] Decide the cutover acceptance criteria. — [[Story - Define cutover acceptance criteria]]

## Child user stories
- [[Story - Backups and restore drill for the dnc database]] (added 2026-10-29, pre-mortem R6; before the cutover)
- [[Story - Define cutover acceptance criteria]]
- [[Story - Cut over and complete a scheduled run on the ITServer]]

## Dependencies / sequencing
- **Upstream:** Every other S2 epic must be done — Postgres schema deployed, failures + patch working, systemd timers installed, backfill complete.
- **Internal:** Story 1 (criteria) before Story 2 (cutover).
- **Downstream:** S3 starts after cutover (ITServer is the stable production environment).
