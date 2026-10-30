---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Metrics dashboard]]"
sprint: S4
moscow: tbd
status: to-do
jira: ""
size: L
---
# Story: Remaining metrics on the dashboard

## User Story
**As** management, compliance, and Michelle, **I want** all validated metrics from S3 M5 displayed on the dashboard, **so that** we have a complete view of DNC scrub performance and lead data quality without writing SQL queries by hand.

## Acceptance Criteria
- [ ] **Given** the first metric (Story 2) is live, **when** the remaining metrics from S3 M5 are added (opt-in vs not; age buckets +6mo/+90d/−90d; DNC→Litigator→not-a-case; Blocked; invalid numbers; new leads in last 30 days; litigators to soft-delete; tag completion), **then** each metric displays as a chart, table, or KPI card (format chosen per metric — e.g., age buckets as a bar chart, tag completion as a percentage gauge).
- [ ] **Given** a metric has a time dimension (e.g., "new leads in last 30 days"), **when** displayed, **then** it is clear what time period the metric covers (e.g., "30 days as of [date]" or "October 2026 partition").
- [ ] **Given** a metric references an unresolved decision (e.g., "litigators to soft-delete" depends on Decision D-01), **when** that metric is displayed, **then** it includes a caveat or footnote (e.g., "assumes soft-delete; policy pending") so the viewer knows the data may change.
- [ ] **Given** all metrics are displayed, **when** the dashboard is reviewed with Michelle and management, **then** they confirm the metrics match the M5 validation output and the layout is usable (no critical metric is buried or unreadable).
- [ ] **Given** the dashboard is complete, **when** a new month's data is backfilled into PostgreSQL, **then** the dashboard reflects the new month's metrics without code changes (the queries are parameterized or date-aware).

## Notes
- This is **Pattern 6: Major Effort** (scale out after proving the approach) — Story 2 de-risked the stack, this story is "more of the same" for 7–9 additional metrics.
- If this story feels too large during planning or execution, split by **metric category** or **persona**: e.g., "compliance metrics" (DNC→Litigator→not-a-case, Blocked) vs "operational metrics" (invalid numbers, tag completion, new leads) vs "statistical metrics" (opt-in, age buckets). Each split delivers a coherent view to one stakeholder group.
- The dashboard does not need to be polished (no auth, no fancy filters, no export-to-PDF) — that's future work. This story delivers **all metrics visible and correct**.
- If a metric's definition is ambiguous when building the UI (e.g., "age buckets — relative to lead creation date or scrub date?"), refer back to the S3 M5 documentation or ask Michelle; do not guess.
- Output: all S3 M5 metrics displayed on the dashboard, with the updated dashboard code versioned in the repo.

## Dependencies
- **Depends on:** [[Story - First dashboard page from validated metrics]] (can't scale out until the first slice works)
- **Feeds into:** S5 Epic (archiving old months presumes the dashboard is live and serving current data)
