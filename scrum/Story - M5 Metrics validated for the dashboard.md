---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Data readiness for modeling]]"
sprint: S3
moscow: tbd
status: to-do
milestone: M5
jira: ""
size: M
---
# Story: M5 Metrics validated for the dashboard

## User Story
**As** the S4 dashboard team and management, **I want** the metrics that will appear on the dashboard validated for correctness and consistency, **so that** the dashboard shows trustworthy data and we're not debugging metrics after the dashboard launches.

## Acceptance Criteria
- [ ] **Given** the starting metric list (opt-in vs not; age buckets +6mo/+90d/−90d; DNC→Litigator→not-a-case; Blocked; invalid numbers; new leads in last 30 days; litigators to soft-delete; tag completion), **when** each metric is computed from the PostgreSQL stage tables, **then** a SQL query or Python script produces the metric value for at least one sample month (e.g., October 2026).
- [ ] **Given** a metric is computed, **when** Michelle manually verifies it (spot-check against raw data, or compare to a legacy Excel report if one exists), **then** the metric matches within an acceptable tolerance (e.g., counts match exactly; percentages within 0.5%).
- [ ] **Given** any metric fails validation (wrong value, missing data, ambiguous definition), **when** the issue is debugged, **then** it is either fixed (correct the query/script) or documented as a known limitation (e.g., "legacy runs before 2025-06 lack opt-in status; those months show NULL").
- [ ] **Given** all metrics pass validation, **when** the S4 team receives the queries/scripts and sample output, **then** they can reproduce the metrics without Michelle's intervention (the logic is documented, not tribal knowledge).
- [ ] **M5 go/no-go criterion:** All metrics on the starting list are validated, documented, and reproducible. The S4 dashboard team has the SQL/Python logic and sample output to start building. Any deferred or out-of-scope metrics (e.g., Decision D-01's soft-delete list) are explicitly noted.

## Notes
- This story delivers **validated metric definitions and working queries/scripts** to the S4 team — not a dashboard, just the data layer.
- Metrics may reference **Decision D-01** (soft vs hard delete of known litigators) — if that decision is still open, the "litigators to soft-delete" metric is either stubbed out or computed with a placeholder assumption (e.g., "all litigators flagged as of today"), and the caveat is documented.
- The "new leads in last 30 days" metric requires a date filter — clarify with Michelle whether "last 30 days" means "as of the report date" (changes every time the dashboard refreshes) or "as of the end of the partition month" (fixed per partition).
- Age buckets (+6mo/+90d/−90d) are relative to *what date*? Clarify the baseline (e.g., "days since lead created" vs "days since last contact") and document it.
- Output: a metrics catalog (suggested: `Docs/S3-M5-Metrics-Catalog.md`) with each metric's definition, SQL/Python logic, sample output, and validation notes. Version the queries/scripts in `Dev/queries/` or `Dev/metrics/`.

## Dependencies
- **Depends on:** [[Story - M4 Model tested with sample data]] (metrics may be derived from or informed by the model)
- **Feeds into:** S4 Epic (dashboard consumes these validated metrics)
- **Blocks:** S4 cannot start building the dashboard until M5 passes
