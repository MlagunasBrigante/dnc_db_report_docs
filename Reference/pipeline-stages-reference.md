---
type: reference
project: dnc-cron-scrub
epic: "[[Epic - Project documentation and housekeeping]]"
sprint: S1
status: draft
created: 2026-11-05
updated: 2026-11-10
tags: [project/dnc-cron-scrub, reference, stages]
---
# Pipeline Stages Reference — Inputs, Outputs, Keys, Rules, Failures

> **Purpose:** Describes each stage's inputs, outputs, keys, business rules and failure modes as implemented in the current SQLite-based pipeline. Written 2026-11-05 from the S1 code (`run.py` + `dnc/*.py`), updated 2026-11-10 for Stage 4 SQL working set, Stage 6 alerts, and per-stage metrics.

---

## Run-level

### Run paths and triggers

**Real runs:**
- Artifacts: `data/<report_date>/` where `report_date` = run date (YYYY-MM-DD)
- Subdirs: `processed/` (ODS DB, reports), `interim/` (staging DB), `logs/run.log`, `metrics/memory_log.csv`
- Trigger: `RUN_TRIGGER` env var (`scheduled` / `manual` / `dry-run`); default `manual`

**Dry-runs** (`--dry-run`):
- Artifacts: `data/_dryrun/<timestamp>Z/` (ISO8601 UTC timestamp)
- Same subdirs; SQLite DBs deleted at end, logs + reports kept
- Auto-purge: folders > `DRYRUN_RETENTION_DAYS` (default 7) removed at start
- Behaviour: synthetic scrub responses (`dnc/fake_scrub.py:12`), no paid API calls (`dnc/fake_scrub.py:73` disabled URL), isolated paths (`dnc/run_paths.py:27-41`), test-only emails (`DRYRUN_EMAIL_RECIPIENTS`)

### Run status and exit codes

**Outcomes** (`dnc/run_outcome.py`):
- `normal`: scrub results exist; Stages 1–6 run
- `nothing_to_scrub`: zero phones after 90-day filter; Stages 3–4 skipped, report with zero counts
- `degraded_no_results`: all Stage 2 batches failed/deferred; Stages 3–4 skipped, warning email lists failed batches, exit 0 (`run.py:1904-1918`)

**Exit codes:**
- `0`: success or degraded (partial success)
- `1`: unhandled exception or missing config (`run.py:2154`, `run.py:2609`)
- `3`: alert delivery failed on all channels (email AND Teams both failed for at least one due alert; `run.py:2506`)

**Final status line** (`dnc/run_status.py`): logged at end, format varies by outcome and failed-item count.

### Run-level metrics output

**Memory log CSV:** `data/metrics/memory_log.csv` (real runs) or `data/_dryrun/<ts>/metrics/memory_log.csv` (dry-runs)
- One row per stage per run; appended after Stage 6 (`run.py:2436-2480`)
- Columns (`dnc/memory_log.py:17-32`):
  - `run_id`: run start timestamp (UTC ISO format)
  - `report_date`: YYYY-MM-DD report date
  - `trigger`: `scheduled` / `manual` / `dry-run`
  - `stage`: stage name (e.g. `Stage 1 Ingest`)
  - `status`: `completed` / `failed`
  - `start_utc`, `end_utc`: stage start/end times (UTC ISO format)
  - `duration_s`: stage duration in seconds (decimal)
  - `mean_mib`, `peak_mib`: mean and peak memory usage in MiB during stage (sampled every `STAGE_MEMORY_SAMPLE_SECONDS`, default 1.0 s; `dnc/stage_metrics.py:94-151`)
  - `items`: stage-specific item count (pages/batches/leads/alerts; currently blank for all stages)
  - `leads_in_snapshot`: total leads from Stage 1 snapshot (`SELECT COUNT(*) FROM raw_data` opened read-only; `run.py:2462-2466`)
  - `code_version`: Git SHA from `GIT_SHA` build arg → `CODE_VERSION` env var (`Dockerfile:6-8`, `run.py:2450`)
  - `memory_source`: memory measurement source (`cgroup v2 memory.current` / `cgroup v1 memory.usage_in_bytes` / `/proc/self/status VmRSS` / `resource.getrusage ru_maxrss`; `dnc/stage_metrics.py:46-91`)
- Per-stage metrics also logged at INFO (`run.py:2180-2345` with `stage_timer` context manager; `dnc/stage_metrics.py:206-302`)
- Per-stage metrics included in Teams success card (`dnc/stage_metrics.py:165-202`), but **not** in the monthly report email (which goes to management)

---

## Stage 1: Data Ingestion (Ingest)

### Inputs

**LawRuler Custom Report API** (`CUSTOM_REPORT_URL`):
- Purpose: extract full lead history (LayoutId 139, start date 2022-01-01 → `report_date`)
- Auth: OAuth2 password grant (`auth_url`, `law_ruler_client_id`, `law_ruler_client_secret`, `law_ruler_username`, `law_ruler_password`) via `token_generation()` (`run.py:150`)
- Paginated CSV: 1,000 rows/page, throttled 2 s between pages (`run.py:385`)

**Env vars (config):**
- `auth_url`, `law_ruler_client_id`, `law_ruler_client_secret`, `law_ruler_username`, `law_ruler_password`, `CUSTOM_REPORT_URL`

### Outputs

**SQLite table: `raw_data` (ODS DB)**
- Schema: dynamic from CSV columns; typical columns include `Lead ID`, `Phone`, `Case Description`, `Scrubbed Date`, `First Name`, `Last Name`, `Tags`, `Status`, `Lead Created Date`, `Case Created Date`, `Call Outcome`
- Types: all TEXT (`run.py:238`)
- Key: PRIMARY KEY (`Lead ID`, `Phone`) (`run.py:278`)
- Write: idempotent via `INSERT OR IGNORE` (`run.py:230-258`)
- Line ref: `run.py:273-279`
- **Memory lifecycle:** DataFrame released after persistence (`run.py:485`); Stage 4 reads via SQL join (see Stage 4 Known Quirks item, resolved 2026-11-10)

**SQLite table: `cleaned_phone_numbers` (staging DB)**
- Schema: same columns as `raw_data` + `is_phone_valid` (boolean), `batch_id` (integer)
- Key: PRIMARY KEY (`Lead ID`, `Phone`) (`run.py:452`)
- Filters: recently-scrubbed removed (< 90 days), bad numbers removed (`Call Outcome` = 'Bad Number')
- Line ref: `run.py:449-454`

**Side effect:** INCOMPLETE marker file written to output dir when pages fail/deferred (`dnc/incomplete_snapshot.py:19`); checked by report (`run.py:1338`).

### Keys and uniqueness

- **Primary key:** (`Lead ID`, `Phone`) — composite key; one lead may have multiple phones
- **Merge logic:** `INSERT OR IGNORE` ensures same (`Lead ID`, `Phone`) pair never duplicates in `raw_data` or `cleaned_phone_numbers`
- **Batch assignment:** `batch_id = (row index // 1000)` after sorting by `Lead ID` DESC (`run.py:443`)

### Business rules

**90-day recently-scrubbed filter** (`run.py:457-474`):
- Extracts `Scrubbed Date` from `Case Description` field (regex `r'Scrubbed Date: \s*(\d{4}-\d{2}-\d{2})'`)
- Keeps only leads where: `Scrubbed Date` is null OR `Scrubbed Date < (report_date - 90 days)`

**Bad-number filter** (`run.py:434-438`):
- Phone validation: length must be exactly 10 digits after cleaning (`run.py:414-425`)
- Cleaning: remove `.0` suffix, `+` prefix, leading `+1` country code (`run.py:414-420`)
- Invalid phones marked `Call Outcome = 'Bad Number'`, then filtered out before scrubbing

**2022-01-01 start date** (`run.py:287`): firm creation date, hardcoded.

### Failure modes and handling

**Retry policy:** `LAWRULER_POLICY` (`dnc/retry.py:72-79`)
- Max attempts: 5 (env `LAWRULER_RETRY_ATTEMPTS`, default 5)
- Backoff: 5 s base, 60 s cap, exponential + jitter (env `LAWRULER_RETRY_BASE_BACKOFF`, `LAWRULER_RETRY_BACKOFF_CAP`)
- Retryable: HTTP 429/500/502/503/504, `requests.RequestException` (network errors)
- Line ref: `run.py:338-345` (call site)

**Outage breaker** (`dnc/retry.py:198-290`):
- Threshold: 20 consecutive failures → pause 900 s (15 min), max 3 pauses → open (defer all)
- Env overrides: `OUTAGE_CONSECUTIVE_FAILURES`, `OUTAGE_PAUSE_SECONDS`, `OUTAGE_MAX_PAUSES`
- When open: remaining pages deferred as `FailedItem` with `failure_type=DEFERRED` (`run.py:306-319`)

**Failed pages** (GAP-09 closed):
- Logged to `FailedItemCollector` (`run.py:348`) with item key `f"page {page}"`
- Tracked in `stage1_failed_pages` list (`run.py:349`)
- INCOMPLETE marker written if any pages failed (`run.py:1863-1876`)
- Run continues (no crash); next pages attempted

**Dry-run fault injection** (`dnc/fault_injection.py`):
- `DRYRUN_INJECT_FAILURES=page:5` → page 5 fails after retries
- No actual network call; fast retry sleep (`run.py:326-336`)

**Volumes (2026-10-28 baseline):**
- 421,776 raw rows → 15,737 after 90-day filter → 7,768 after bad-number removal (~424k growing)

---

## Stage 2: DNC Scrubbing (Scrub)

### Inputs

**SQLite table: `cleaned_phone_numbers` (staging DB)**
- Read from: output of Stage 1 clean step
- Columns used: `Phone`, `batch_id`

**dncscrub.com API** (`SCRUB_URL`):
- Purpose: scrub up to 1,000 phones per batch
- Auth: `LOGIN_ID` header
- Payload: `{'phoneList': [phone1, phone2, ...]}`
- Response: JSON array of scrub result objects (one per phone)
- Line ref: `run.py:624-629`

**Env vars:**
- `SCRUB_URL`, `LOGIN_ID`
- Dry-run: `SCRUB_URL` replaced with `dryrun-disabled://paid-api-not-called` (`run.py:1834`) to prevent accidental paid calls

### Outputs

**SQLite table: `batch_status` (staging DB)**
- Schema: `batch_id INTEGER PRIMARY KEY`, `status TEXT`, `updated_at TIMESTAMP`
- Purpose: checkpoint completed batches for resume-safety
- Line ref: `run.py:511-516`

**SQLite table: `scrub_results` (staging DB)**
- Schema: `batch_id INTEGER`, `response_json TEXT`, `FOREIGN KEY(batch_id) REFERENCES batch_status(batch_id)`
- Content: raw JSON response from dncscrub.com (or synthetic in dry-run), one row per batch
- Line ref: `run.py:517-523`

**Side effect (dry-run):** synthetic responses generated via `generate_fake_scrub_batch()` (`dnc/fake_scrub.py:12-69`), deterministic mix (~60% open, ~35% DNC, ~3% litigator, ~2% blocked).

### Keys and uniqueness

- **Primary key:** `batch_id` (0-based: phone index // 1000)
- **Resume logic:** `SELECT 1 FROM batch_status WHERE batch_id = ? AND status = 'DONE'` (`run.py:526-533`); skips already-done batches
- **Idempotency:** same batch never re-scrubbed in same run unless checkpoint missing

### Business rules

**1,000-phone batches** (`run.py:553`): fixed batch size, matches dncscrub.com API limit.

**Same-day resume** (GAP-11): works only if `report_date` = today; different-date reruns need manual cleanup.

### Failure modes and handling

**Retry policy:** `DNCSCRUB_POLICY` (`dnc/retry.py:82-89`)
- Max attempts: 3 (env `DNCSCRUB_RETRY_ATTEMPTS`)
- Backoff: 2 s base, 8 s cap (env `DNCSCRUB_RETRY_BASE_BACKOFF`, `DNCSCRUB_RETRY_BACKOFF_CAP`)
- Retryable: HTTP 429/500/502/503/504, `requests.RequestException`
- Line ref: `run.py:631-637` (real), `run.py:601-614` (dry-run injected)

**Outage breaker** (same threshold as Stage 1):
- When open: remaining batches deferred as `FailedItem` with `failure_type=DEFERRED` (`run.py:577-588`)
- Logged to `FailedItemCollector` (`run.py:587`) with item key `f"batch {batch_id}"`

**Failed batches** (GAP-10 closed):
- Logged to `FailedItemCollector` (`run.py:640-642`)
- Batch skipped (not checkpointed as DONE)
- If all batches fail/deferred: outcome = `degraded_no_results`, Stages 3–4 skipped (`run.py:1904-1918`)

**Dry-run:** `DRYRUN_INJECT_FAILURES=batch:0` → batch 0 fails; `outage:stage2` → all calls fail, breaker opens.

**Volumes (2026-10-28):** 7,768 phones → 8 batches; typical runs ~1k–10k phones.

---

## Stage 3: Interpret Scrub Results (Interpret)

### Inputs

**SQLite table: `scrub_results` (staging DB)**
- All batches' JSON responses, one row per batch
- Each batch is a JSON array of phone result objects

### Outputs

**SQLite table: `scrub_results_translated` (staging DB)**
- Schema: dynamic from JSON keys; typical columns: `Phone`, `ResultCode`, `Reason`, `DoNotCallToday`, `LineType`, `IsWirelessOrVoIP`, `Country`, `RegionAbbrev`, `Locale`, `TZCode`, `UTCOffset`, `CallingWindow`, `CallingTimeRestrictions`, `CarrierInfo`, `EBRType`, `NewReassignedAreaCode`, `Reserved`
- Key: PRIMARY KEY (`Phone`) (`run.py:735`, `run.py:1935`)
- Write: idempotent via `INSERT OR IGNORE` (`run.py:741-744`)
- Line ref: `run.py:720-758`

### Keys and uniqueness

- **Primary key:** `Phone` (10-digit US phone, no country code)
- **Deduplication:** Stage 2 batches may overlap phones (rare); `scrub_results_translated` dedupes → "attempted 1158, total 1154" quirk (log `run.py:749-753`)

### Business rules

**DNC_CODES map** (`run.py:68-75`):
- Open codes → `'Open to call'`: C, E, F, G, H, O, V, W, Y
- DNC codes → `'Do not call'`: D, I, L, M, P
- Blocked → `'Blocked'`: B

**Litigator flag** (`run.py:705-709`):
- When `ResultCode = 'Do not call'` AND `Reason = 'Litigator'` → `ResultCode` replaced with `'Known Litigator'`

### Failure modes and handling

**None expected:** pure in-memory data transform (JSON → DataFrame → translate → save).

**Degraded case (GAP-29):** if `scrub_results` table is empty (all batches failed), Stage 3 skipped with INFO log (`run.py:1917`). Used to crash with "No scrub results found".

**Volumes:** 7,749 unique phones in typical run (slight dedup from 7,768 attempted).

---

## Stage 4: Update LawRuler Tags (Update)

### Inputs

**SQLite table: `raw_data` (ODS DB)**
- Accessed read-only via SQL join (ATTACHed to staging DB; `run.py:852`)
- Stage 4 builds working set via `_build_working_set_via_sql()` (`run.py:835-876`): inner join with `scrub_results_translated` on `raw.Phone = '+1' || scrub.Phone`, selecting 10 columns (leadId, Phone, First Name, Last Name, Tags, Status, Lead Created Date, Case Created Date, ResultCode, Reason)
- **Not loaded in full** (Stage 1 releases its DataFrame after persistence; `run.py:485`)

**SQLite table: `scrub_results_translated` (staging DB)**
- All scrubbed phones with interpreted result codes

**LawRuler lead-update API** (`LAWRULER_URL`):
- Purpose: tag one lead with DNC/Litigator/Blocked
- Auth: `LAW_RULER_ACCOUNT_KEY` payload param
- Optional: `LAWRULER_SESSION_COOKIE` (ASP.NET_SessionId value only) in Cookie header (`run.py:815`)
- Payload: `Key`, `LeadID`, `FirstName`, `CellPhone`, `Tags`, `Summary` (contains `Scrubbed Date: <report_date>`)
- Line ref: `run.py:1005-1020`

**Env vars:**
- `LAWRULER_URL`, `LAW_RULER_ACCOUNT_KEY`, `LAWRULER_SESSION_COOKIE` (optional)

### Outputs

**SQLite table: `leads_with_tags` (staging DB)**
- Schema: `leadId INTEGER PRIMARY KEY`, `Phone TEXT`, `names TEXT`, `First Name TEXT`, `Last Name TEXT`, `ResultCode TEXT`, `Tags TEXT`, `Status TEXT`, `Lead Created Date TEXT`, `Case Created Date TEXT`, `created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP`
- Content: merged raw + scrubbed data with assigned DNC tag
- Line ref: `run.py:924-937`, `run.py:969-991`

**SQLite table: `update_status` (staging DB)**
- Schema: `leadId INTEGER PRIMARY KEY`, `status TEXT`, `updated_at TIMESTAMP`
- Purpose: checkpoint completed lead updates for resume-safety
- Line ref: `run.py:909-915`

**SQLite table: `update_results` (staging DB)**
- Schema: `leadId INTEGER`, `response_json TEXT`, `FOREIGN KEY(leadId) REFERENCES update_status(leadId)`
- Content: API response per lead (success or error)
- Line ref: `run.py:916-922`

**Side effect (real runs):** LawRuler leads tagged via API; `Case Description` updated with `Scrubbed Date: <report_date>`.

**Side effect (dry-run):** Stage 4 skipped unless fault injection active (`run.py:1086-1090`); if injected, simulates the loop without real API calls.

### Keys and uniqueness

- **Primary key:** `leadId` (LawRuler lead ID, integer)
- **Join:** inner join on `Phone` (`raw_data.Phone = '+1' + scrub_results_translated.Phone`) (`run.py:868-873`)
- **Resume logic:** `SELECT 1 FROM update_status WHERE leadId = ? AND status = 'DONE'` (`run.py:940-946`); skips already-done leads
- **Buffered writes:** every 100 leads flushed to DB (`run.py:1147-1148`, `FLUSH_INTERVAL` at `run.py:777`)

### Business rules

**Tag translation** (`run.py:876-890`):
- `ResultCode = 'Known Litigator'` → `names = 'DNC, Litigator'`
- `ResultCode = 'Blocked'` → `names = 'DNC, Blocked'`
- `ResultCode = 'Do not call'` → `names = 'DNC'`
- `ResultCode = 'Open to call'` → `names = NULL` (no tag)

**Active process statuses list** (`run.py:77-98`):
- Used by report only (Stage 5) to exclude active cases from soft-deletion lists
- 18 statuses: `$225 Pitched - Pending`, `Case Pitched - Pending`, `2025 BF - Case Pitched Pending`, `Appointment Set`, `Case Completed`, `Case Completed - Paid In Full`, `Case Completed - Stop Work`, `Case Submitted`, `Case Submitted - Paid In Full`, `Case Submitted - Past Due`, `Case Submitted - Stop Work`, `Existing Case Client`, `New Case`, `New Case - Paid In Full`, `Open Consult`, `Sent e-Sign`, `Signed e-Sign`, `Staff`, `Submitted - T Visa Derivative`, `T Visa Derivative`

### Failure modes and handling

**Retry policy:** `LAWRULER_POLICY` (same as Stage 1; `dnc/retry.py:72-79`)
- 5 attempts, 5 s base backoff, 60 s cap
- Line ref: `run.py:1022-1028` (real), `run.py:1198-1205` (dry-run injected)

**Outage breaker:**
- When open: remaining leads deferred as `FailedItem` with `failure_type=DEFERRED` (`run.py:1119-1131`)
- Logged to `FailedItemCollector` (`run.py:1129`)

**Failed leads** (GAP-08):
- Currently: marked `status = DONE` even if API call failed (`run.py:959`, line 1142 buffer append); never retried same run
- Logged to `FailedItemCollector` (`run.py:1031-1032`)

**Sequential execution** (GAP-20): strictly one lead at a time (~0.68 s/lead, measured 2026-10-28); dominates run time for large tag counts (28,982 leads = ~18 h).

**Dry-run:** `DRYRUN_INJECT_FAILURES=lead:3,lead:10` → leads 3 and 10 fail (by 1-based index in `leads_with_tags`).

**Volumes (2026-10-28):** 7,713 leads processed, 3,028 with DNC tags.

---

## Stage 5: Report Generation (Report)

### Inputs

**SQLite table: `leads_with_tags` (staging DB)**
- All tagged leads from Stage 4
- In degraded mode: empty dataframe (Stage 4 skipped) (`run.py:1276-1278`)

**SQLite table: `raw_data` (ODS DB)**
- Used only for total lead count (`SELECT COUNT(*) FROM raw_data`) (`run.py:1334`)

**INCOMPLETE marker** (file in output dir):
- Written by Stage 1 if pages failed/deferred
- Checked by `is_snapshot_complete()` (`dnc/incomplete_snapshot.py:26`)
- Line ref: `run.py:1338`

### Outputs

**CSV file:** `leads_with_tags_<report_date>.csv` (output dir)
- All rows from `leads_with_tags` table, all columns
- Line ref: `run.py:1286`

**Excel file:** `leads_with_tags_<report_date>.xlsx` (output dir, 3 sheets)
- **Summary sheet:** 6-row table (Metric, Count, Action)
  - Total leads in LawRuler (from `raw_data` count)
  - Leads scrubbed this run (from `leads_with_tags`)
  - …of which not a case
  - DNC leads without opt-in (to be soft-deleted)
  - Litigators to soft-delete
  - Raw snapshot complete (YES / NO — see INCOMPLETE marker)
- **Litigators sheet:** litigators to soft-delete (DNC + litigator + not a case + not single opt-in + not active process status)
- **Raw Data sheet:** full `leads_with_tags` table
- Line ref: `run.py:1382-1387`

**Soft-deletion CSV files** (output dir):
- `dnc_without_optin_<report_date>.csv`: columns `leadId`, `First Name`, `Last Name`, `Phone`, `Status` (`run.py:1373-1378`, `SOFT_DELETE_CSV_COLUMNS` at `run.py:1266`)
- `litigators_to_soft_delete_<report_date>.csv`: same columns
- Written always when report data exists; email only reads them if counts > 0

### Keys and uniqueness

**None** (report stage; no primary keys). Report masks:
- `case_mask = (Case Created Date notna)`
- `dnc_mask = (DNC == True)` (tag contains "DNC")
- `optin_mask = (Single Opt-In == True)` (tag contains "Single Opt-In")
- `litigator_mask = (Litigator == True)` (tag contains "Litigator")
- `active_process_mask = (Status in ACTIVE_PROCESS_STATUSES)`
- Line ref: `run.py:1318-1322`

### Business rules

**Summary counts logic** (`dnc/report_summary.py:9-69`):
- Total leads: `SELECT COUNT(*) FROM raw_data` (ODS)
- Leads scrubbed: len(`leads_with_tags`)
- Not a case: `~case_mask`
- DNC without opt-in: `~optin_mask & dnc_mask & ~case_mask`
- Litigators to delete: `~case_mask & dnc_mask & ~optin_mask & litigator_mask & ~active_process_mask`

**Deadline in report** (`dnc/report_summary.py`, `removal_deadline()` / `deadline_label()`): "Will be removed by the <Nth> of <month>", where the deadline is **run date + 6 days** (GAP-30, 2026-11-10). The scheduled run on the 1st gives the 7th (run on 1st → patch on 3rd → watchdog on 5th → removal by 7th); ad-hoc runs never show a past date. The soft-deletion email uses the same date.

**Degraded mode:** zero counts for scrubbed/derived metrics; total leads and completeness status unchanged (`run.py:1922-1928`).

### Failure modes and handling

**None expected:** file I/O only; no network calls. Excel write failure would raise unhandled exception → error email.

**Volumes (2026-10-28):**
- 421,776 total leads
- 7,713 scrubbed
- 3,028 with DNC tags
- CSV 0.93 MB, XLSX 0.66 MB

---

## Stage 5b: Soft-deletion Request Email

> **Note:** This is a distinct email from the main report email. It is sent after Stage 5 report generation and before the Stage 6 alerts (before the warning snapshot, so an unset/blocked recipient or a send failure reaches the alert email; `run.py:2301-2313`). Subject: `[For review] DNC soft-deletion request — <report_date>`. Recipients: internal only (`SOFT_DELETE_RECIPIENTS` TO, `SOFT_DELETE_CC` CC), allowlisted by domain (`SOFT_DELETE_ALLOWED_DOMAINS`, default `briganteimmigration.com`). Michelle reviews and forwards to the LawRuler team manually (D-10).

### Inputs

**CSV files** (output dir):
- `dnc_without_optin_<report_date>.csv`
- `litigators_to_soft_delete_<report_date>.csv`
- Written by Stage 5, read by this email step (`run.py:1456-1457`)

**Env vars:**
- `SOFT_DELETE_RECIPIENTS` (TO), `SOFT_DELETE_CC` (CC), `SOFT_DELETE_ALLOWED_DOMAINS` (comma-separated; `run.py:2109-2111`)
- Dry-run override: `DRYRUN_EMAIL_RECIPIENTS`

### Outputs

**Email via MS Graph** (`sendMail` API):
- Subject: `[For review] DNC soft-deletion request — <report_date>` (dry-run: prefixed `[DRY-RUN]`)
- Body: HTML table with counts + the removal deadline (run date + 6 days, same as the report)
- Attachments: both CSVs if total size < 2.5 MB; otherwise note in body with file paths (`run.py:1490-1505`)
- Line ref: `run.py:1443-1522`

### Keys and uniqueness

**None** (email only).

### Business rules

**Domain allowlist** (`dnc/soft_delete_request.py`):
- All recipients (TO + CC) must match `SOFT_DELETE_ALLOWED_DOMAINS`
- Rejected domain → ERROR log, send skipped (`run.py:1481-1483`)

**Attachment size guard** (`dnc/soft_delete_request.py`): 2.5 MB total limit; over limit → email sent without attachments, body states file paths (`run.py:1492-1505`).

**Send condition:** skipped if both counts = 0 (`run.py:1468-1470`), or degraded mode (`run.py:1452-1454`), or allowlist violation.

### Failure modes and handling

**Best-effort:** email send failure logged as ERROR, run continues (`run.py:2312-2313`).

**Dry-run:** routes to `DRYRUN_EMAIL_RECIPIENTS` if set; logs only if unset (`run.py:1485`).

---

## Stage 6: Alerts (email + Teams)

### Inputs

**Summary table rows** (from Stage 5)
- Dict: `{'Metric': [...], 'Count': [...], 'Action': [...]}`

**Log file** (`run.log`)
- For warning/error emails: compact excerpt (WARNING/ERROR/CRITICAL lines only, capped 1 MB) attached

**Warning collector** (`WarningCollector` in-memory):
- Captures WARNING+ log records during Stages 1–5
- Snapshot taken before Stage 6 (`run.py:2331`)

**Per-stage metrics** (`StageMetrics` in-memory):
- Duration, mean/peak memory for each stage (collected via `stage_timer` context manager; `dnc/stage_metrics.py:206-302`)
- Included in Teams success card only

**Env vars (email):**
- `O365_CLIENT_ID`, `O365_CLIENT_SECRET`, `O365_TENANT_ID` (app-only client credentials, `Mail.Send` permission)
- `EMAIL_SENDER` (default `it.admin@briganteimmigration.com`)
- `EMAIL_RECIPIENTS_SUCCESS`, `EMAIL_RECIPIENTS_WARNING`, `EMAIL_RECIPIENTS_ERROR`
- `ALERT_EMAIL_CC` (for warning/error), `REPORT_RECIPIENTS_TO` (business + team, falls back to SUCCESS), `REPORT_RECIPIENTS_CC`
- Dry-run override: `DRYRUN_EMAIL_RECIPIENTS`

**Env vars (Teams):**
- `TEAMS_WEBHOOK_URL` (real runs), `DRYRUN_TEAMS_WEBHOOK_URL` (dry-runs; `run.py:2115-2116`)

### Outputs

**Three email types** (MS Graph `sendMail` API):

1. **Success / Report email** (`run.py:1539-1567`, sent at `run.py:2354-2362`):
   - When: run completes (normal or degraded)
   - Subject: `DNC Scrub Report — <report_date> ✅` (dry-run: prefixed `[DRY-RUN]`)
   - Body: HTML summary table (same as Excel Summary sheet), report date, run duration; degraded notice if applicable
   - Attachments: **none** (XLSX/CSV written to output folder, not attached)
   - Recipients: `REPORT_RECIPIENTS_TO` (TO), `REPORT_RECIPIENTS_CC` (CC)

2. **Warning email** (`run.py:1779-1850`, sent at `run.py:2397-2403`):
   - When: run completes AND `warning_collector.messages` non-empty
   - Subject: `DNC Scrub Report — <report_date> ⚠️ (<N> warning(s))` (dry-run: prefixed `[DRY-RUN]`)
   - Body: bulleted warning summary
   - Attachments: compact log excerpt (WARNING/ERROR/CRITICAL lines, capped 1 MB)
   - Recipients: `EMAIL_RECIPIENTS_WARNING` (TO), `ALERT_EMAIL_CC` (CC)

3. **Error email** (`run.py:1687-1777`, sent in exception handler):
   - When: unhandled exception in try block (`run.py:2541-2555`)
   - Subject: `DNC Scrub Pipeline FAILED — <report_date> ❌` (dry-run: prefixed `[DRY-RUN]`)
   - Body: traceback + last 50 log lines
   - Attachments: compact log excerpt (WARNING/ERROR/CRITICAL lines, capped 1 MB)
   - Recipients: `EMAIL_RECIPIENTS_ERROR` (TO), `ALERT_EMAIL_CC` (CC)

**Three Teams alert types** (Workflows webhook POST):

1. **Success card** (`run.py:1606-1685`, sent at `run.py:2368-2379`):
   - When: run completes
   - Contents: summary table rows, run duration, failed items count, final status line, per-stage metrics (duration + memory)
   - Webhook: `TEAMS_WEBHOOK_URL` (real), `DRYRUN_TEAMS_WEBHOOK_URL` (dry-run)

2. **Warning card** (`run.py:1923-1992`, sent at `run.py:2409-2416`):
   - When: run completes with WARNING-level log messages
   - Contents: warning counts by level, first 20 warning lines (truncated to 200 chars)

3. **Error card** (`run.py:1994-2088`, sent in exception handler at `run.py:2562-2569`):
   - When: unhandled exception
   - Contents: exception type + message (first 500 chars), log file path

**Cross-reporting:** when one channel fails, the working channel reports it (`dnc/alert_delivery.py`):
- Email fails, Teams works → Teams card includes "Success/Warning/Error email failed to send"
- Teams fails, email works → Follow-up email sent to alert recipients '[DNC scrub] Teams alert failed to post' (`run.py:2384-2389`, `2421-2426`)
- Both fail → exit 3 logged (`run.py:2486`)

### Keys and uniqueness

**None** (email/Teams only).

### Business rules

**Failed-items summary** (`dnc/failed_item_formatting.py`):
- Logged before Stage 6 warning snapshot (`run.py:2315-2328`)
- Per-stage WARNING with item keys (failed vs deferred), max 50 keys shown
- Always logged even when zero (`run.py:2320`)

**Notice for degraded runs** (`run.py:2336-2341`):
- `degraded_no_results` → "Scrub service unavailable this run — no leads were scrubbed; the batches will be retried."
- `nothing_to_scrub` → "No new leads needed scrubbing this run."

**Dry-run recipient override:**
- All emails route to `DRYRUN_EMAIL_RECIPIENTS` (comma-separated) if set; skipped (logged only) if unset (`dnc/email_helpers.py`)

**Teams skipped when not configured:**
- Real runs: `TEAMS_WEBHOOK_URL` unset → logged as "Teams not configured; skipped" (INFO, not a failure)
- Dry-runs: `DRYRUN_TEAMS_WEBHOOK_URL` unset → logged as "DRY-RUN: Teams skipped (DRYRUN_TEAMS_WEBHOOK_URL not set)" (INFO; `run.py:1638`, `1956`, `2023`)

### Failure modes and handling

**Alert delivery failure (exit 3):**
- If both email and Teams fail for the same alert (success/warning/error) → `any_alert_failed_all_channels = True` (`run.py:2392-2393`, `2429-2430`)
- CRITICAL logged before final status: "Alert delivery failed on all channels" (`run.py:2486`)
- Exit code 3 (`run.py:2506`)
- Config validation failure (exit 1) takes precedence over exit 3 if both occur (`run.py:2608`)

**Best-effort email:** email send failures logged as WARNING, never fail the run or change exit code (unless all channels fail for a due alert; see exit 3 above).

**Auth failure:** O365 auth failure logged as ERROR, email skipped (`run.py:1510-1512`).

**Log excerpt:** extracted via `extract_log_excerpt()` (`dnc/log_excerpt.py`), capped 1 MB; attached as temp file.

---

## Known Quirks

1. **Phone deduplication quirk:** `scrub_results_translated` dedupes on `Phone` (PRIMARY KEY), so "attempted 1158, total 1154" when Stage 2 batches overlap (`run.py:752-753`).

2. **Same-day real runs append:** multiple `once` calls on the same date write to the same `data/<date>/` folder; `run.log` grows, DBs merge via `INSERT OR IGNORE`. INCOMPLETE marker persists for same-day reruns (not cleared unless manually removed).

3. **Dry-run timestamp isolation:** each dry-run gets a unique UTC timestamp folder (`data/_dryrun/YYYY-MM-DDTHHMMSSZ/`), so parallel dry-runs never collide.

4. **Stage 4 SQL working set (resolved 2026-11-10):** Stage 4 now builds its working set via SQL join (`_build_working_set_via_sql()` at `run.py:835-876`) instead of loading all of `raw_data` into memory. Stage 1 releases its DataFrames after persistence (`run.py:485`), and main() releases `cleaned_df` after Stage 2. Measured effect on 2026-11-10 dry-run: Stage 4 mean 731.5 MiB / peak 799 MiB vs 1,103 / 1,207 on 2026-11-09 (before the fix). The SQL join ATTACHes the ODS DB read-only and selects only the 10 columns needed for tagging + report.

5. **Failed leads marked DONE:** currently, leads that fail Stage 4 API calls are still checkpointed as `status = DONE` in `update_status` (`run.py:959`), so they're never retried in the same run.

6. **Email failures silent:** send failures logged as WARNING, run continues and reports success (unless every alert channel fails for a due alert, triggering exit 3; see Stage 6 exit codes); email delivery never verified.

7. **INCOMPLETE marker sticky:** once written, the marker file stays in the output dir for same-date reruns unless manually deleted.

---

## S2 Mapping

For each stage, the SQLite tables and files it writes, with suggested Postgres table names (S2 schema design starts from this doc):

| Stage | SQLite table / file | Suggested Postgres table | Notes |
|-------|---------------------|-------------------------|-------|
| **Stage 1 Ingest** | `raw_data` (ODS) | `raw_data_snapshots` | Partitioned by `run_id`; add `run_id`, `snapshot_timestamp` |
| | `cleaned_phone_numbers` (staging) | `scrub_queue` | Ephemeral; can be a temp table or derived view |
| | INCOMPLETE marker file | `run_metadata.snapshot_complete` | Boolean column in run metadata |
| **Stage 2 Scrub** | `batch_status` (staging) | `batch_checkpoints` | Partitioned by `run_id`; resume-safety |
| | `scrub_results` (staging) | `scrub_responses_raw` | Partitioned by `run_id`; raw JSON per batch |
| **Stage 3 Interpret** | `scrub_results_translated` (staging) | `scrub_results` | Partitioned by `run_id`; interpreted scrub results |
| **Stage 4 Update** | `leads_with_tags` (staging) | `tagged_leads` | Partitioned by `run_id`; join result + tag assignments |
| | `update_status` (staging) | `update_checkpoints` | Partitioned by `run_id`; resume-safety |
| | `update_results` (staging) | `update_responses` | Partitioned by `run_id`; API responses |
| **Stage 5 Report** | `leads_with_tags_<date>.csv` | (file artifact; not persisted in Postgres) | Archive in object storage |
| | `leads_with_tags_<date>.xlsx` | (file artifact; not persisted in Postgres) | Archive in object storage |
| | `dnc_without_optin_<date>.csv` | `soft_delete_lists` | Partitioned by `run_id`, `list_type` ENUM |
| | `litigators_to_soft_delete_<date>.csv` | `soft_delete_lists` | Same table, `list_type = 'litigator'` |
| **Stage 6 Alerts** | (email/Teams; no persistence) | `alert_log` | Log alert outcomes (sent/failed/skipped) per run |
| **Run-level** | `data/metrics/memory_log.csv` | `stage_metrics` | Partitioned by `report_date`; per-stage timeseries |

All staging tables (`batch_status`, `scrub_results`, `scrub_results_translated`, `leads_with_tags`, `update_status`, `update_results`) should include `run_id` (UUID or ISO timestamp) as part of the primary key or partition key. The ODS `raw_data` table becomes `raw_data_snapshots` partitioned by `run_id` or `snapshot_date`. The S2 schema designer can map each stage to Postgres tables without reading `run.py`.

---

## Code Line References

> All line numbers from `Dev/run.py` and `Dev/dnc/*.py` as of 2026-11-10 (main @ a03e293, commit `Stage 4 review: parameterized ATTACH, drop dead _translate_tags...`).

**Key files:**
- `run.py`: main pipeline (2,613 lines)
- `dnc/retry.py`: retry policies, OutageBreaker, FailedItemCollector (464 lines)
- `dnc/run_paths.py`: real vs dry-run path resolution (136 lines)
- `dnc/report_summary.py`: Summary table builder (70 lines)
- `dnc/fake_scrub.py`: dry-run synthetic responses (74 lines)
- `dnc/incomplete_snapshot.py`: INCOMPLETE marker I/O
- `dnc/run_outcome.py`: outcome decision logic
- `dnc/soft_delete_request.py`: allowlist, attachment guard, email builder
- `dnc/email_helpers.py`: recipient resolution (TO/CC, dry-run override)
- `dnc/log_excerpt.py`: compact log extractor (WARNING+ lines only)
- `dnc/stage_metrics.py`: stage duration and memory tracking (332 lines)
- `dnc/memory_log.py`: memory log CSV persistence (172 lines)
- `dnc/teams.py`: Teams Workflows webhook posting
- `dnc/alert_delivery.py`: cross-reporting logic for failed alert channels

---

*Written 2026-11-05 from the S1 code, updated 2026-11-10 for Stage 4 SQL working set, Stage 6 alerts (email + Teams with cross-reporting and exit 3), per-stage metrics, and memory log CSV. The code is the source of truth; this doc is derived from it.*
