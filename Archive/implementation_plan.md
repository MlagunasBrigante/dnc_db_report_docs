# Containerize DNC Scrub Pipeline as Monthly Automated Job

## Background

The repository contains a Jupyter notebook-based pipeline (`AsyncIO.ipynb`) that:
1. **Extracts** lead data from the LawRuler Custom Report API
2. **Scrubs** phone numbers against the DNC database (dncscrub.com API)
3. **Tags** leads with DNC/Litigator/Blocked status and updates LawRuler via API
4. **Reports** — generates a summary Excel report with a Summary sheet, Litigators sheet, and Raw Data sheet

There is also a second notebook (`global_report.ipynb`) that reads the same databases to update tags in `raw_data` and compute snapshot stats. Its report section is incomplete (the last cell errors out with a `TypeError`). This notebook's useful logic overlaps with `AsyncIO.ipynb` and will be consolidated.

---

## Proposed Changes

### Phase 1 — Port Notebook to `run.py`

#### Code to PORT (from `AsyncIO.ipynb`)

| Cell(s) | Class / Logic | Purpose |
|---------|--------------|---------|
| 0–2 | Imports, `load_dotenv`, env vars, DNC_CODES, path setup | Configuration |
| 6 | `token_generation()` | Auth for LawRuler API |
| 7 | `DataIngestionPipeline` | Extract custom report from API + clean phone numbers |
| 13 | `ScrubbingPipeline` | Batch-scrub phones against DNC API with checkpoint/retry |
| 15 | `ScrubbedDataProcessor` | Parse scrub JSON results, interpret codes, save to DB |
| 23 | `UpdateLawRulerPipeline` | Tag translation + LawRuler API updates with buffered writes |
| 28 | Export `leads_with_tags` to CSV | Intermediate output |
| 33–35 | Report data prep: `read_tags()`, masks, `active_process_mask` | Report generation |
| 37–38 | Summary counts + `summary_df` creation | Report generation |
| 41 | `litigators_to_delete` filter | Report generation |
| 43 | Excel export with 3 sheets | Final output |

#### Code to EXCLUDE

| Cell(s) | What | Why |
|---------|------|-----|
| Cell 5 (AsyncIO) — hardcoded `ODS_DB_PATH`, `STAGING_DB_PATH` override | `ODS_DB_PATH = "../data/2026-02-11/..."` | Debugging artifact — overrides the dynamic path set in Cell 2. Will use the env-variable-driven path. |
| Cell 10 | `invalid_exploration` analysis | Exploratory / pre-report diagnostics — prints counts but doesn't affect pipeline output |
| Cell 16 | `scrub_results['Phone'].duplicated().sum()` | Exploratory diagnostic, just prints |
| Cell 19 | Deprecated `ScrubbingReporting` (entire cell is commented out with `#!`) | Explicitly marked deprecated in favor of `UpdateLawRulerPipeline` |
| Cell 20 | Standalone `delete_table()` utility | Not called by any pipeline; utility for manual ad-hoc cleanup |
| Cell 22 | Deprecated `ScrubbingUpdatePipeline` (marked `#! Deprecated`) | Explicitly superseded by `UpdateLawRulerPipeline` |
| Cell 26 | `litigators` inspection | Interactive exploration — `litigators` variable displayed inline |
| Cell 30 | Empty TODO placeholder | Contains only a comment and dead string |
| Cell 39 | `summary_df` display | Interactive display only |
| `global_report.ipynb` (entire) | Tag-updating + snapshot stats | Its tag-update logic duplicates Cell 23, and its snapshot section (Cell 14 — last cell) errors with a `TypeError`. Report generation logic is already consolidated in `AsyncIO.ipynb` Cells 33–43. |

#### Structure of `run.py`

```
run.py
├── Constants/Config (DNC_CODES, ACTIVE_PROCESS_STATUSES)
├── setup_logging() → returns logger + log_file_path
├── token_generation()
├── class DataIngestionPipeline (from Cell 7)
├── class ScrubbingPipeline (from Cell 13)
├── class ScrubbedDataProcessor (from Cell 15)
├── class UpdateLawRulerPipeline (from Cell 23)
├── generate_report() — Cells 28, 33–38, 41, 43
├── main()
│   ├── 1. DataIngestionPipeline.run(pull_data=True)
│   ├── 2. ScrubbingPipeline.run()
│   ├── 3. ScrubbedDataProcessor.run() + save_idempotent()
│   ├── 4. UpdateLawRulerPipeline.run(update_lawruler=True)
│   └── 5. generate_report()
└── if __name__ == "__main__": main()
```

**Refactoring for automation:**
- Replace all `print()` with `logging` at appropriate levels (INFO/DEBUG)
- Remove interactive `pull_data=False` path override — production always pulls fresh data
- Fix the `report_date` to use `pd.Timestamp.today()` (no manual `-1 day` offset)
- All paths are relative to a configurable `DATA_DIR` env var (default: `./data`)
- The `main()` function catches top-level exceptions and logs them at CRITICAL level
- Exit code 0 on success, 1 on failure

---

### Phase 2 — Containerize

#### [NEW] `Dockerfile`
- Base image: `python:3.13-slim`
- Copy only `run.py` and `requirements.txt` into the image
- `WORKDIR /app`
- `CMD ["python", "run.py"]`
- Data, output, and logs are NOT in the image — they're bind-mounted

#### [NEW] `.dockerignore`
- Ignore: `data/`, `notebooks/`, `.git/`, `.env`, `*.db`, `*.csv`, `*.xlsx`, `__pycache__/`, `.DS_Store`

#### [NEW] `requirements.txt`
Pinned dependencies (from notebook imports):
```
pandas==2.2.3
numpy==2.2.6
requests==2.32.3
python-dotenv==1.1.0
openpyxl==3.1.5
SQLAlchemy==2.0.36
```

#### [NEW] `.env.example`
Template with all required env vars, no real secrets:
```
auth_url=https://auth.lawruler.com/brigantelaw/identity/connect/token
law_ruler_client_id=
law_ruler_client_secret=
law_ruler_username=
law_ruler_password=
SCRUB_URL=https://www.dncscrub.com/app/main/rpc/scrub?version=5&output=json
LOGIN_ID=
LAWRULER_URL=https://brigantelaw.lawruler.com/api-legalcrmapp.aspx
LAW_RULER_ACCOUNT_KEY=
CUSTOM_REPORT_URL=https://apim.lawruler.com/brigantelaw/ApiReport/GetCustomReport
DATA_DIR=/data
OUTPUT_DIR=/output
LOG_DIR=/logs
```

#### [MODIFY] `.gitignore`
Add: `*.db`, `*.xlsx`, `output/`, `logs/`, `.env`, `__pycache__/`

#### Volume Mounts & Crontab

```bash
# Build
docker build -t dnc-scrub .

# Run (one-shot)
docker run --rm \
  --env-file /path/to/.env \
  -v /host/data:/data \
  -v /host/output:/output \
  -v /host/logs:/logs \
  dnc-scrub

# Host crontab (1st of each month at 2:00 AM)
0 2 1 * * docker run --rm --env-file /path/to/.env -v /host/data:/data -v /host/output:/output -v /host/logs:/logs dnc-scrub >> /host/logs/cron.log 2>&1
```

#### Secrets Approach

**`.env` file loaded via `--env-file` at `docker run` time.**

Rationale:
- This is a **single-host, cron-scheduled** deployment — no Kubernetes, no cloud orchestrator
- `.env` file is already gitignored and the team is familiar with it (it's used today)
- `--env-file` keeps secrets out of the image, out of git, and out of shell history
- Simple to rotate: edit the file and re-run
- `.env.example` (committed) documents all required variables

---

### Phase 3 — Email via Microsoft Graph

> [!NOTE] Shipped implementation differs from this plan
> Phase 3 shipped using the **`O365` library** (`O365==2.0.38`) with the app-only
> client-credentials flow, **not** raw `msal` + Microsoft Graph. The env vars are
> `O365_CLIENT_ID` / `O365_CLIENT_SECRET` / `O365_TENANT_ID` / `EMAIL_SENDER` /
> `EMAIL_RECIPIENTS_{SUCCESS,WARNING,ERROR}` — **not** the `GRAPH_*` names below.
> There is no `msal` dependency. The underlying Graph app permission is still
> `Mail.Send` (application, admin-consented). See `DNC_db_report/README.md` §"Stage 6".
>
> Scheduling also changed: instead of a **host crontab** (see Phase 2), the schedule
> is now **baked into the image** via `scheduler.py` + `croniter` (`SCHEDULE_CRON`,
> default `0 2 1 * *`, UTC). The container self-schedules — no per-host cron setup.

#### [MODIFY] `run.py`
Add email module using `msal` + `requests` to Microsoft Graph API:
- `get_graph_token()` — client-credentials OAuth2 flow via MSAL
- `send_email_with_attachment(subject, body, recipients, attachment_path)` — POST to `/v1.0/users/{sender}/sendMail`
- `send_success_email()` — zips the output folder, emails to `it.admin@briganteimmigration.com`
- `send_warning_email(warnings)` — summarizes WARNING-level log messages
- `send_error_email(error_details)` — sends error/exception details + log excerpt

#### [MODIFY] `.env.example`
Add Graph API credentials:
```
GRAPH_TENANT_ID=
GRAPH_CLIENT_ID=
GRAPH_CLIENT_SECRET=
GRAPH_SENDER_EMAIL=it.admin@briganteimmigration.com
EMAIL_RECIPIENTS_SUCCESS=it.admin@briganteimmigration.com
EMAIL_RECIPIENTS_ERROR=it.admin@briganteimmigration.com
```

#### [MODIFY] `requirements.txt`
Add: `msal==1.31.1`

---

## Verification Plan

### Phase 1
- Run `python run.py` locally and confirm it produces the same output structure as the notebook: `data/{date}/processed/`, `data/{date}/interim/`, `data/{date}/logs/`, plus the final `.xlsx` report

### Phase 2
- `docker build -t dnc-scrub .`
- `docker run` with bind-mounted `data/`, `output/`, `logs/` volumes
- Confirm the output folder contains the `.xlsx` report and the logs are persisted on the host

### Phase 3
- Test email delivery by calling `send_success_email()` with a small test attachment
- Verify warning and error email paths by injecting test log messages

## Open Questions

> [!IMPORTANT]
> **`pull_data` flag**: The notebook has `pull_data=False` in the current usage cell (Cell 7), meaning it reads from the existing ODS database instead of calling the LawRuler API. For a fully automated monthly run, should the script **always** pull fresh data from the API (`pull_data=True`)? I'll default to `True` for production automation.

> [!IMPORTANT]  
> **`update_lawruler` flag**: The notebook's Cell 23 currently runs with `update_lawruler=True`. Should the automated monthly job always update LawRuler, or should this be configurable via env var? I'll default to `True` and make it configurable via `UPDATE_LAWRULER=true/false` in `.env`.

> [!NOTE]
> **Docker host**: Will the container run on a Mac (the current dev machine) or a Linux server? This affects the crontab instructions. I'll document both.
