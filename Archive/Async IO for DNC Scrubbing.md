# Staging
## Data Extraction & Preprocessing
- CustomReportExtraction: Save (idempotent) csv dataset to ODS DB under table "raw_data"
	- Leave the '#TODO: Extract with API call' as is
- CustomReportCleaning: Save (idempotent) resulting dataset to STAGING DB under table "cleaned_phone_numbers"
	- Execute '#TODO: Output the Bad Numbers'; add a column is_phone_valid with boolean values (True if valid, False otherwise)

## Structure
- Data Extraction (from CRM)
- Data Preprocessing (well-handled by Pandas)
- 3rd Party API massive call (Async IO insertion needed)

![[ASYNC Scrub API Call]]