---
type: reference
project: dnc-cron-scrub
epic: "[[Epic - Project documentation and housekeeping]]"
sprint: S1
status: draft
created: 2026-11-05
tags: [project/dnc-cron-scrub, reference, stages]
---
# Pipeline Stages Reference — Inputs, Outputs, Keys, Rules, Failures

> **Purpose:** Describes each stage's inputs, outputs, keys, business rules and failure modes as implemented in the current SQLite-based pipeline. Written 2026-11-05 from the S1 code (`run.py` + `dnc/*.py`).

---

## Run-level

### Run paths and triggers

**Real runs:**
- Artifacts: `data/<report_date>/` where `report_date` = run date (YYYY-MM-DD)
- Subdirs: `processed/` (ODS DB, reports), `interim/` (staging DB), `logs/run.log`
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
- `1`: unhandled exception or missing config

**Final status line** (`dnc/run_status.py`): logged at end, format varies by outcome and failed-item count.

## Stage 1: Data Ingestion

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

## Stage 2: DNC Scrubbing

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

## Stage 3: Interpret Scrub Results

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

## Stage 4: Update LawRuler Tags

### Inputs

**SQLite table: `raw_data` (ODS DB)**
- Re-read from disk (not reused from Stage 1 in-memory copy; GAP-18)
- All columns, all rows (~421k)

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

## Stage 5: Report Generation

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
- `dnc_without_optin_<report_date>.csv`: columns `leadId`, `First Name`, `Last Name`, `Phone`, `Status` (`run.py:1373-1378`, `SOFT_DELETE_CSV_COLUMNS` at `run.py:1250`)
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

**Deadline in report** (`dnc/report_summary.py:61-62`): "Will be removed by the 7th of <month>" (run on 1st → patch on 3rd → watchdog on 5th → removal by 7th).

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

> **Note:** This is a distinct email from the main report email. It is sent after Stage 5 report generation and before the Stage 6 success email. Subject: `[For review] DNC soft-deletion request — <report_date>`. Recipients: internal only (`SOFT_DELETE_RECIPIENTS` TO, `SOFT_DELETE_CC` CC), allowlisted by domain (`SOFT_DELETE_ALLOWED_DOMAINS`, default `briganteimmigration.com`). Michelle reviews and forwards to the LawRuler team manually (D-10).

### Inputs

**CSV files** (output dir):
- `dnc_without_optin_<report_date>.csv`
- `litigators_to_soft_delete_<report_date>.csv`
- Written by Stage 5, read by this email step

**Env vars:**
- `SOFT_DELETE_RECIPIENTS` (TO), `SOFT_DELETE_CC` (CC), `SOFT_DELETE_ALLOWED_DOMAINS` (comma-separated)
- Dry-run override: `DRYRUN_EMAIL_RECIPIENTS`

### Outputs

**Email via MS Graph** (`sendMail` API):
- Subject: `[For review] DNC soft-deletion request — <report_date>` (dry-run: prefixed `[DRY-RUN]`)
- Body: HTML table with counts + "Please review and remove by the 7th of <month>"
- Attachments: both CSVs if total size < 2.5 MB; otherwise note in body with file paths
- Line ref: `run.py:1427-1521`

### Keys and uniqueness

**None** (email only).

### Business rules

**Domain allowlist** (`dnc/soft_delete_request.py`):
- All recipients (TO + CC) must match `SOFT_DELETE_ALLOWED_DOMAINS`
- Rejected domain → ERROR log, send skipped (`run.py:1461-1467`)

**Attachment size guard** (`dnc/soft_delete_request.py`): 2.5 MB total limit; over limit → email sent without attachments, body states file paths (`run.py:1473-1489`).

**Send condition:** skipped if both counts = 0 (`run.py:1452-1454`), or degraded mode (`run.py:1436-1438`), or allowlist violation.

### Failure modes and handling

**Best-effort:** email send failure logged as ERROR, run continues (`run.py:1969-1970`).

**Dry-run:** routes to `DRYRUN_EMAIL_RECIPIENTS` if set; logs only if unset (`run.py:1469`).

---

## Stage 6: Email Notifications

### Inputs

**Summary table rows** (from Stage 5)
- Dict: `{'Metric': [...], 'Count': [...], 'Action': [...]}`

**Log file** (`run.log`)
- For warning/error emails: compact excerpt (WARNING/ERROR/CRITICAL lines only, capped 1 MB) attached

**Warning collector** (`WarningCollector` in-memory):
- Captures WARNING+ log records during Stages 1–5
- Snapshot taken before Stage 6 (`run.py:1988`)

**Env vars (email):**
- `O365_CLIENT_ID`, `O365_CLIENT_SECRET`, `O365_TENANT_ID` (app-only client credentials, `Mail.Send` permission)
- `EMAIL_SENDER` (default `it.admin@briganteimmigration.com`)
- `EMAIL_RECIPIENTS_SUCCESS`, `EMAIL_RECIPIENTS_WARNING`, `EMAIL_RECIPIENTS_ERROR`
- `ALERT_EMAIL_CC` (for warning/error), `REPORT_RECIPIENTS_TO` (business + team, falls back to SUCCESS), `REPORT_RECIPIENTS_CC`
- Dry-run override: `DRYRUN_EMAIL_RECIPIENTS`

### Outputs

**Three email types** (MS Graph `sendMail` API):

1. **Success / Report email** (`run.py:1523-1567`):
   - When: run completes (normal or degraded)
   - Subject: `DNC Scrub Report — <report_date> ✅` (dry-run: prefixed `[DRY-RUN]`)
   - Body: HTML summary table (same as Excel Summary sheet), report date, run duration; degraded notice if applicable
   - Attachments: **none** (XLSX/CSV written to output folder, not attached)
   - Recipients: `REPORT_RECIPIENTS_TO` (TO), `REPORT_RECIPIENTS_CC` (CC)

2. **Warning email** (`run.py:1645-1716`):
   - When: run completes AND `warning_collector.messages` non-empty
   - Subject: `DNC Scrub Report — <report_date> ⚠️ (<N> warning(s))` (dry-run: prefixed `[DRY-RUN]`)
   - Body: bulleted warning summary
   - Attachments: compact log excerpt (WARNING/ERROR/CRITICAL lines, capped 1 MB)
   - Recipients: `EMAIL_RECIPIENTS_WARNING` (TO), `ALERT_EMAIL_CC` (CC)

3. **Error email** (`run.py:1569-1643`):
   - When: unhandled exception in try block (`run.py:2041-2055`)
   - Subject: `DNC Scrub Pipeline FAILED — <report_date> ❌` (dry-run: prefixed `[DRY-RUN]`)
   - Body: traceback + last 50 log lines
   - Attachments: compact log excerpt (WARNING/ERROR/CRITICAL lines, capped 1 MB)
   - Recipients: `EMAIL_RECIPIENTS_ERROR` (TO), `ALERT_EMAIL_CC` (CC)

### Keys and uniqueness

**None** (email only).

### Business rules

**Failed-items summary** (`dnc/failed_item_formatting.py`):
- Logged before Stage 6 warning snapshot (`run.py:1973-1985`)
- Per-stage WARNING with item keys (failed vs deferred), max 50 keys shown
- Always logged even when zero (`run.py:1977`)

**Notice for degraded runs** (`run.py:1993-1998`):
- `degraded_no_results` → "Scrub service unavailable this run — no leads were scrubbed; the batches will be retried."
- `nothing_to_scrub` → "No new leads needed scrubbing this run."

**Dry-run recipient override:**
- All emails route to `DRYRUN_EMAIL_RECIPIENTS` (comma-separated) if set; skipped (logged only) if unset (`dnc/email_helpers.py`)

### Failure modes and handling

**Best-effort:** email send failures logged as WARNING, never fail the run or change exit code (`run.py:2012-2013`, `2025-2026`, `2053-2054`).

**Auth failure:** O365 auth failure logged as ERROR, email skipped (`run.py:1494-1496`, `1584-1587`, `1667-1669`).

**Log excerpt:** extracted via `extract_log_excerpt()` (`dnc/log_excerpt.py`), capped 1 MB; attached as temp file (`run.py:1624-1639`, `1696-1712`).

---

## Known Quirks

1. **Phone deduplication quirk:** `scrub_results_translated` dedupes on `Phone` (PRIMARY KEY), so "attempted 1158, total 1154" when Stage 2 batches overlap (`run.py:752-753`).

2. **Same-day real runs append:** multiple `once` calls on the same date write to the same `data/<date>/` folder; `run.log` grows, DBs merge via `INSERT OR IGNORE`. INCOMPLETE marker persists for same-day reruns (not cleared unless manually removed).

3. **Dry-run timestamp isolation:** each dry-run gets a unique UTC timestamp folder (`data/_dryrun/YYYY-MM-DDTHHMMSSZ/`), so parallel dry-runs never collide.

4. **Stage 4 re-reads `raw_data`:** the full `raw_data` table is read from disk in Stage 4 (`run.py:795-796`) even though Stage 1 already has it in memory; both coexist (~1.1 GB plateau, GAP-18). This is for stage isolation (Stage 4 can resume independently).

5. **Failed leads marked DONE:** currently, leads that fail Stage 4 API calls are still checkpointed as `status = DONE` in `update_status` (`run.py:959`), so they're never retried in the same run.

6. **Email failures silent:** send failures logged as WARNING, run continues and reports success (`run.py:2012-2013`); email delivery never verified.

7. **INCOMPLETE marker sticky:** once written, the marker file stays in the output dir for same-date reruns unless manually deleted.

---

## Code Line References

> All line numbers from `Dev/run.py` and `Dev/dnc/*.py` as of 2026-11-05 (S1 code, commit preceding this doc).

**Key files:**
- `run.py`: main pipeline (2,069 lines)
- `dnc/retry.py`: retry policies, OutageBreaker, FailedItemCollector (464 lines)
- `dnc/run_paths.py`: real vs dry-run path resolution (136 lines)
- `dnc/report_summary.py`: Summary table builder (70 lines)
- `dnc/fake_scrub.py`: dry-run synthetic responses (74 lines)
- `dnc/incomplete_snapshot.py`: INCOMPLETE marker I/O
- `dnc/run_outcome.py`: outcome decision logic
- `dnc/soft_delete_request.py`: allowlist, attachment guard, email builder
- `dnc/email_helpers.py`: recipient resolution (TO/CC, dry-run override)
- `dnc/log_excerpt.py`: compact log extractor (WARNING+ lines only)

---

*Written 2026-11-05 from the S1 code. The code is the source of truth; this doc is derived from it.*
