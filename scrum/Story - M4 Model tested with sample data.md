---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Data readiness for modeling]]"
sprint: S3
moscow: tbd
status: to-do
milestone: M4
jira: ""
size: M
---
# Story: M4 Model tested with sample data

## User Story
**As** Michelle and management, **I want** the statistical model tested with a sample of the backfilled data, **so that** we can see whether the model works before committing to full-scale metrics production.

## Acceptance Criteria
- [ ] **Given** M3 passed (data is clean enough), **when** Michelle defines the model approach (e.g., logistic regression predicting "lead becomes a case," time-series forecasting of monthly opt-in rates, clustering leads by DNC-risk profile), **then** the approach is documented (suggested: `Docs/S3-M4-Model-Design.md`) with input features, expected outputs, and success criteria (e.g., "AUC ≥ 0.70" or "forecast error ≤10%").
- [ ] **Given** the model design is documented, **when** Michelle runs it on a sample dataset (e.g., one month of leads, or a stratified sample across all backfilled months), **then** the model produces output (predictions, clusters, or forecast) and its performance is measured against the success criteria.
- [ ] **Given** the model runs, **when** the results are reviewed, **then** a decision is recorded: (a) the model works, proceed to M5 (metrics validation), (b) the model needs refinement (iterate within S3), or (c) the model approach is wrong, revisit the design.
- [ ] **M4 go/no-go criterion:** Michelle and management agree the model has been tested with sample data and the results are promising enough to proceed. The test does not have to show *perfect* performance, but it must show the approach is viable. If the model fails badly, S3 may loop back to M2 (preprocessing) or M3 (data quality) before retrying M4.

## Notes
- This story is **Michelle's work** — engineering provides data access (SQL queries, Python scripts to extract samples) and reviews the test setup, but Michelle owns the modeling.
- The "sample data" size is negotiable: one month may be enough if the data is consistent; a stratified sample (e.g., 10% of each month) may be better if monthly patterns vary.
- **Simple statistical tools** (per the scope doc) — this is not a production ML pipeline. Michelle may use Python (pandas, scikit-learn, statsmodels), R, Excel, or even SQL aggregates, depending on the model. Engineering does not impose tooling.
- If the model reveals a data issue (e.g., "the DNC result codes are inconsistent across legacy vs container-era runs"), flag it and decide whether to loop back to M3 or proceed with a caveat.
- Output: model design doc, sample test results (notebook, script output, or spreadsheet), and a decision record stating the go/no-go outcome.

## Dependencies
- **Depends on:** [[Story - M3 Data quality gate]] (no modeling until data is confirmed clean enough)
- **Blocks:** [[Story - M5 Metrics validated for the dashboard]] (validated metrics presume the model works)
