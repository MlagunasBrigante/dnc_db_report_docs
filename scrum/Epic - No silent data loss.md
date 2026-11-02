---
type: epic
project: dnc-cron-scrub
sprint: S1
status: to-do
gaps: [GAP-26, GAP-09, GAP-10, GAP-14, GAP-15, GAP-16, GAP-17]
---
# Epic: No silent data loss

## Description
Close every gap where the pipeline silently truncates data, skips failed items, or proceeds with invalid config. Today a Stage 1 page error can truncate the download, a failed Stage 2 batch is skipped, and the config check misses critical keys — all without loud alerts. Add fail-fast config validation, per-stage instrumentation, and fix misleading report labels.

## Gaps closed
- **GAP-26** — Three separately coded retry loops with different rules; the token call has none
- **GAP-09** — A Stage 1 page error silently truncates the download
- **GAP-10** — A failed Stage 2 batch is skipped; the "next run retries" comment is false
- **GAP-14** — Dead "unmatched" warning (inner join can never fire it)
- **GAP-15** — Config check misses LAWRULER_URL and LAW_RULER_ACCOUNT_KEY
- **GAP-16** — Misleading report labels (e.g. "Total Leads Scrubbed" = all raw_data)
- **GAP-17** — No per-stage duration and memory instrumentation

## Breakdown

### INVEST check

| Criterion | Rating | Notes |
|---|---|---|
| Independent | ✅ | No technical dependency on other S1 epics |
| Negotiable | ✅ | Memory instrumentation method is open (cgroup vs psutil vs resource); report label wording is open |
| Valuable | ✅ | Operator can trust that data loss is impossible to miss and config errors fail early |
| Estimable | ✅ | Code locations are known from the run report and scope; config keys are listed; instrumentation is additive |
| Testable | ✅ | Inject a page error, batch error, missing config key, and confirm loud failures; verify per-stage metrics in log |

### Pattern applied
**Pattern 3: Business Rule Variations** (different failure modes: page error, batch error, config missing, report labels, instrumentation)

Earlier patterns rejected:
- Pattern 1 (Workflow Steps): these are not sequential stages of one workflow; they are independent failure modes across different stages.
- Pattern 4 (Data Variations): the variations are in failure *modes* (different ways data can be lost), not in data types.

### Cynefin calibration
**Low uncertainty** — All code locations are known from the run report; retry logic and config validation are standard patterns; instrumentation is proven (cgroup memory.peak was used in the 10-28 run report).

### Split evaluation
Each story delivers value to the IT & Data lead: page errors can't truncate downloads, batch errors can't be skipped, invalid config fails before any API call, report labels are honest, per-stage metrics feed the memory epic. Sizes are reasonably equal (S to M); all are vertical (each closes one silent-failure mode end-to-end).

## Child user stories
- [[Story - Dry-run never calls paid APIs or touches real data]] ← first (s1_order 0)
- [[Story - One retry policy for every API call]]
- [[Story - Lead download never truncates silently]]
- [[Story - Failed scrub batches are retried and reported]]
- [[Story - Config validated before any API call]]
- [[Story - Report labels and unmatched warning fixed]]
- [[Story - Per-stage duration and memory in every run]]

## Dependencies / sequencing
**Start with [[Story - One retry policy for every API call]]** (added 2026-10-29 at Michelle's request). The page and batch stories build on its failed-item event.
The config-validation story should complete early (fail-fast principle). The per-stage instrumentation story feeds the memory-optimization epic (GAP-18), so it should complete before memory work starts. All other stories are independent.
