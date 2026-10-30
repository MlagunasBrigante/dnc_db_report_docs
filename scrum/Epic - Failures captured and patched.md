---
type: epic
project: dnc-cron-scrub
sprint: S2
status: to-do
gaps: [GAP-08, GAP-11]
---
# Epic: Failures captured and patched
## Description
Every stage records the items (pages, batches, leads) it couldn't process. A scheduled patch pass on the 3rd retries only the failed items within the same monthly run. Unresolved items after the patch pass are reported in alerts. A run is never marked failed only because some items failed — the flaky LawRuler API makes transient failures inevitable, so the pipeline must absorb them.

## Gaps closed
- **GAP-08**: Failed tagging requests recorded as DONE, never retried — S2 creates the failed-items store and patch pass
- **GAP-11**: Resume only works on the same day (report_date = today) — patch pass takes run_id, not date, so it can retry a September run in October

## Breakdown

### INVEST check
> [!question]+
> Why is the `Rating` column missing?
>
> ✅ *Applied 2026-10-29: table now uses the same `Criterion | Rating | Notes` format as the S1 epics; "Small" dropped because the method skips it at epic level.*

| Criterion | Rating | Notes |
|---|---|---|
| Independent | ⚠️ | Depends on Epic - PostgreSQL system of record (needs run_id, stages tables) but independent of scheduling/cutover |
| Negotiable | ✅ | Could start with fewer stages (e.g. Stage 4 tagging only), but all 4 data-touching stages need it |
| Valuable | ✅ | Michelle sees every failure and knows which leads weren't tagged; compliance risk eliminated |
| Estimable | ✅ | 3 stories, each testable by injecting failures |
| Testable | ✅ | Inject API errors, verify failed-items records, run patch pass, check unresolved count in alert |

### Splitting pattern applied
**Pattern 1: Workflow Steps** — capture (main run), patch (scheduled retry), alert (unresolved summary). Each is a distinct observable step in the failure lifecycle.

Earlier patterns rejected:
- Pattern 4 (Data Variations) — per-stage failures are handled uniformly (same schema, same patch logic), so splitting by stage creates duplication
- Pattern 6 (Major Effort) doesn't apply; the stories are already small

### Cynefin calibration
**Complicated** — the design (failed-items table, patch pass as a CLI arg, alert integration) is known, but coordinating it with the existing retry logic and ensuring the patch pass is idempotent (doesn't re-tag already-tagged leads) requires careful implementation.
> [!note]+
> Note that we can update the retry logic so this implementation is more streamline. That'll mean, maybe, that a gap in the retry logic exists. We should discuss this.
>
> ✅ *Decided 2026-10-29: new S1 story [[Story - One retry policy for every API call]] (GAP-26). One shared helper emits a structured failed-item event when retries run out; this epic's capture story only persists those events and the patch pass replays them.*

### Split evaluation
- **Does it reveal low-value work?** No — legal/compliance exposure from silent failures is the project's highest risk after secrets.
- **Are the sizes roughly equal?** Story 1 (capture) is Small, 2 (patch) is Medium (new CLI mode + idempotency), 3 (alert) is Small; acceptable.

## Child user stories
- [[Story - Failed items captured per stage]]
- [[Story - Patch pass reprocesses a run's failed items]]
- [[Story - Unresolved items reach the alerts]]

## Dependencies / sequencing
- **Upstream:** Epic - PostgreSQL system of record must land Story - All stages write to Postgres with lineage, so run_id and stage tables exist.
- **Internal:** Stories run in order (capture → patch → alert).
- **Downstream:** Epic - Reproducible scheduling on the ITServer needs Story - Patch pass done to schedule the 3rd-of-month timer.
