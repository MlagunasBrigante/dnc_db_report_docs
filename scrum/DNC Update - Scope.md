---
type: scope
project: dnc-cron-scrub
jira: DNCUP
owner: <owner>
status: static — source of truth (2026-10-29)
created: 2026-10-29
tags: [project/dnc-cron-scrub, scope, scrum]
---

# DNC Update — Scope

> **How to review:** annotate directly in this file (e.g. `> [!question]`, `> [!warning]`, or inline `**<owner>:** …`). The next pass reads your annotations only and fills those gaps — it does not restart the interview.

## Point
Make the monthly DNC scrub **reliable, observable and auditable** on the AlmaLinux ITServer with PostgreSQL. Then make its data ready for a simple statistical model, expose the metrics in a custom dashboard, and finally archive old months.

## Subject
The DNC Scrub Pipeline (`Dev/run.py`). Each month it:
1. ingests the full lead history from LawRuler,
2. scrubs phones against dncscrub.com,
3. interprets the result codes,
4. tags leads in LawRuler,
5. builds an Excel/CSV report, and
6. alerts by email.

It runs today as the `dnc-scrub` Docker container on `<owner-mac>` and stores each run in SQLite under `data/{date}/`.

## Development assumption (applies to every sprint)
- **All code is developed as if it already runs on the AlmaLinux ITServer:**
  - Linux paths
  - Docker (always the default runtime)
  - systemd for scheduling
  - the server's existing local PostgreSQL
- The job ran on the Mac until the early host move (2026-11-23, D-32); since then it runs on the ITServer in the image's own scheduler, and phase 2 of the cutover switches it to systemd and the Postgres pipeline. **The physical cutover is not rushed;** quality and tests take priority over speed.
- PostgreSQL is already installed on the ITServer. `<owner>` creates the database objects, with guidance; this project only defines the tables.

## Infrastructure
| Machine | OS | Role |
|---|---|---|
| ITServer | AlmaLinux | Runs the process **and** PostgreSQL (same machine). Runtime: Docker. Production host since 2026-11-23. |
| `autobot` | Ubuntu | **No role** in the process. Possible host for the dashboard (late stage, undecided). |
| `<owner-mac>` | macOS | Production host until 2026-11-23; now rollback only (`dnc-scrub` stopped, not removed). |

Because the process and the database are on the same machine, **n8n is not part of the data path.** n8n only becomes relevant if the pure-n8n rewrite decision (D-04) is taken.

## In scope (by sprint)

### S1 — Gap-closing (code targets the ITServer, still runs on the Mac)
- **Email alerting:** stop large inline attachments (zip only the report, not `scrub_ods.db`); add a size guard or Graph upload session; error email gets the same fix; warning email lists only *final* failures.
- **Report email = summary table only** (no attachments; business recipients incl. one external), plus an **internal-only soft-deletion request** to the LawRuler team (DNC leads without opt-in + litigators), sent when either count is above 0. The report states "Will be removed by the 7th of <month>" (run on the 1st → patch pass on the 3rd → watchdog on the 5th → removal by the 7th).
- **Teams alert channel** as a second channel. It reuses the existing MS Graph connector: credentials are already in `Dev/.env`, and the Graph call code exists in the email feature (`_get_o365_account`, `run.py:1120`). A failed alert delivery is itself a red alert.
- **Logging:** INFO by default, DEBUG only when turned on; mask secrets; stop library DEBUG from logging request bodies.
- **Secrets:** remove the hardcoded LawRuler session cookie; rotate exposed credentials; clean secrets out of historic logs and legacy files.
- **One retry policy for every API call (GAP-26):** a shared helper with settings per API; when retries run out it emits a structured failed-item event (stage, item key, error, attempts) that is logged and alerted. S2 persists these events.
- **Silent data loss closed (GAP-09, GAP-10):** a Stage-1 page error must not truncate the download silently, and a failed Stage-2 batch must not be skipped silently. Both are fixed and alerted in S1. The general failed-items store and scheduled patch pass for every stage still come in S2.
- **Correctness:**
  - full config validation at start
  - remove the dead "unmatched" warning
  - fix the misleading report labels
- **Observability:** record per-stage duration and memory inside the job.
- **Docs:**
  - **DNCUP-4 → runbook** for failed or degraded runs, covering who acts and through which channel.
  - **DNCUP-5 → per-stage inputs/outputs document**, which also feeds the S2 schema design.
- **Housekeeping:**
  - re-scope the DNCUP Jira tickets
  - finish the project restructure, including triaging `Scrub_API/`
  - update `README.md` and `Dev/README.md` for the `Container/` → `Dev/` and `DNC-Update` → `dnc-cron-scrub` renames
  - **credential rotation + retire the `DNC_LOCALSERVER` GitHub repo (GAP-24)**
- **Memory optimization (closes S1; GAP-18):** priority 1 is Stage 4 (initial target −20%, ≈0.06 GB per hour per thousand leads); priority 2 is Stage 2, only after Stage 4 is stable; focus on means, not peaks, unless runs show non-negligible peaks; plus a per-run memory log reviewed quarterly. Resilience always wins over memory. Baseline: about 1.1 GB held for about 1.5 h during Stage 4 (2026-10-28). **Starts only after the essential gaps above are closed and the code is stable.** The ITServer has 16 GB of RAM, so this is about efficiency, not about whether the job fits.

### S2 — Technical migration (developed for the ITServer; ends with the physical cutover)
- **PostgreSQL schema:**
  - one table per stage, **partitioned by month**
  - a `pipeline_run` table (run_id, trigger, start/end, code version, outcome, raw-snapshot SHA-256)
  - generated surrogate keys plus `run_id`; `(Lead ID, Phone)` becomes a per-run uniqueness constraint, not the primary key
  - foreign keys tracing each row back through the stages: tag → scrub result → cleaned phone → raw row
- **Full monthly raw snapshot** kept for legal traceability. **Closed months are made read-only.**
- **Failure model:** each stage records the items it couldn't process (pages, batches, leads) in a failed-items store. A **patch pass** retries them within the same run. Whatever is still failing is reported as unresolved. **A run is never marked failed just because some items failed.**
- **Scheduling (Docker + systemd):**
  - unit files and an install script are versioned in `Dev/deploy/`, so the setup is reproducible and nothing is configured by hand on the machine
  - monthly run on the **1st**, scheduled **patch pass on the 3rd**, **watchdog on the 5th**
  - `Persistent=true` on the run timer, so a run missed while the server was down fires on boot
  - the watchdog alerts if there's no successful run that month, unresolved items remain, or a run is still going
- **Backfill**, with checksums and tests, of:
  - the 4 container-era SQLite runs (2026-08-12, 09-01, 10-01, 10-28)
  - the **legacy run history in `data/legacy/`**: 16 dated runs 2025-12-09 → 2026-07-01, plus `1st_run`–`4th_run` (May–Aug 2025) and `bug/`, about 3.9 GB in total. These are from the notebook era, so their schemas may differ; each run is checked against the stage-table contract before loading, and runs that don't fit are reported, not forced in. This history is S3's main input.
- **Physical cutover** to the ITServer. Acceptance criteria are deferred (see Open items).

### S3 — Statistics: data readiness and modeling (`<owner>` leads; light engineering)
Focus: make the data accessible and fit for the statistical model `<owner>` will develop.
- Audit the folder structure, the pipelines and the data lineage.
- Ingest the notes `<owner>` feeds the project, to align the preprocessing.
- Milestones (each is a go/no-go checkpoint):
  - **M1** Data and pipeline audit complete
  - **M2** Notes ingested and preprocessing aligned
  - **M3** *Is the data clean enough?* (quality criteria to be defined)
  - **M4** *Has the model been tested with sample data?*
  - **M5** Metrics validated and ready to show in the dashboard
- Starting metric list (from legacy notes):
  - opt-in vs not
  - age buckets (+6 months / +90 days / −90 days)
  - DNC → Litigator → not a case
  - Blocked
  - invalid numbers
  - new leads in the last 30 days
  - litigators to soft-delete
  - tag completion

### S4 — Dashboard
A custom app (Streamlit, Dash or Bokeh) showing the S3 metrics from PostgreSQL. Starts only once the S2 migration is **done and stable**.

### S5 — Archiving old months
Move closed monthly partitions and old logs to an external disk and/or the cloud. Starts after the dashboard launches.

### S6 — Automated soft-deletion (optional)
After S5, and only if D-07 = automate: the job performs the soft-deletes through the LawRuler API instead of the email request. It starts with a spike to confirm API support.

## Out of scope
- Per-lead tag tracking and reconciliation as a standing activity. The focus is logs and alerts that make failures impossible to miss.
- Changes to upstream lead sources or LawRuler configuration.
- `dnctag_local_py`, the deprecated local version of this project. Archived to `Dev/Archive/dnctag_local_py/` (gitignored). Its exposed secrets are handled by GAP-24.
- Hosting anything on `autobot` before S4.

> IMPORTANT
> We should, at the end of S!, focus on optimizing memory handling. As per the last manual run's report, there's a (1h, 1GB) issue that may compromise the server's memory. This should only be done AFTER the essential gaps are closed and the code is stable.
>
> ✅ *Applied 2026-10-29: added as the closing item of S1; GAP-18 moved to S1.*

## Data sources
| Source | Used by | Notes |
|---|---|---|
| LawRuler Custom Report API (LayoutId 139, since 2022-01-01) | Stage 1 | ~422k rows and growing; flaky (disconnects, SSL/DNS errors, timeouts) |
| dncscrub.com scrub API | Stage 2 | 1,000-phone batches; API docs PDFs in `Docs/Reference/dncscrub-api/` |
| LawRuler lead-update API | Stage 4 | One sequential call per lead (~0.68 s/lead) |
| MS Graph (O365) | Alerts | `Mail.Send` today; Teams to be added through the same connector |
| Existing SQLite runs (`data/*/`) | S2 backfill | 4 container-era runs |
| Legacy run history (`data/legacy/`) | S2 backfill → S3 | 2025-05 → 2026-07; includes `1st_run/results/` (merged from `Scrub_API/results/`) |
| `<owner>`'s notes | S3 | Fed during the sprint |

## Definition of Done
- **S1:** Every warning or error path delivers an alert (email + Teams). No alert depends on a large attachment. No secrets in logs or code, and exposed secrets rotated. INFO-level log in MB, not GB. Runbook and per-stage I/O doc written. Jira DNCUP re-scoped. Verified by at least one `once --dry-run` and one real run.
- **S2:**
  - schema created on the ITServer
  - all 4 SQLite runs backfilled with matching counts and checksums
  - failed-items store, patch pass and watchdog tested, including deliberately injected failures
  - systemd and Docker setup reproducible from the repo
  - cutover done and one scheduled run completed on the ITServer
- **S3:** M1–M5 milestones passed.
- **S4:** Dashboard serves the validated metrics from PostgreSQL.
- **S5:** At least one closed month archived and restored in a test.

## Constraints
- Ask before acting (vault `CLAUDE.md` §8.1). Scope first, then build.
- Azure/M365 work (Graph email, Teams) must be checked against the `microsoft-learn` MCP (vault `CLAUDE.md` §13).
- Any push to GitHub must authenticate as `<docs-repo-owner>` (vault `CLAUDE.md` §15).
- Secrets live only in `Dev/.env`; never passed with `docker run -e`.
- `data/` stays in place while `dnc-scrub` is deployed (bind mount).
- LawRuler is flaky, so retry logic and the failure model must assume it.

## Assumptions
- The ITServer has enough memory. Measured: ~1.1 GiB working set, 2.77 GiB peak driven by the email stage. ≥2 GiB is enough; validate after S1.
  - Note: The ITServer has 16GB of RAM, but we'd still like to decrease the mean RAM usage (if possible)
- The existing PostgreSQL on the ITServer can host a dedicated database or schema for DNC.
- A run that takes longer than 4 days (the longest so far was 3.6 days) is abnormal and the watchdog should alert on it.

## Gap register (single ID scheme)
Sources: health audit 2026-10-28, `Docs/Run Reports/DNC Run Report 2026-10-28.md`, legacy-notes review.

| ID | Gap | Code / evidence | Sprint |
|---|---|---|---|
| GAP-01 | Warning and error emails attach the full log, so they fail (400/504 in Aug, Oct and 10-28) | `run.py:1259-1260`, `1307-1308` | S1 |
| GAP-02 | Report zip includes `scrub_ods.db`: a 47 MB zip for 1.6 MB of report | `run.py:1163-1169` | S1 |
| GAP-03 | Attachments sent inline in one request, no size guard; success-email delivery unverified | `run.py:1208-1209` | S1 |
| GAP-04 | Retries that eventually succeeded, and each individual failed attempt, count as warnings (noise) | `run.py:77-101`, `1420-1421` | S1 |
| GAP-05 | Alert-delivery failures are silent; email is the only channel | `run.py:1423-1446` | S1 |
| GAP-06 | DEBUG logging for every library, so logs reach GB scale | `run.py:113-125` | S1 |
| GAP-07 | Secrets exposed: OAuth secret and tokens in logs, hardcoded cookie, token/PEMs in legacy files | `run.py:683`; `data/*/logs`; `Scrub_API/` | S1 |
| GAP-08 | Failed tagging requests recorded as `DONE`, never retried | `run.py:819-838` | S2. Partial fix 2026-11-10 (code review, `s1/fix/code-review-findings`): failed leads are no longer marked `DONE`, so a same-period re-run retries them; the patch pass stays S2 |
| GAP-09 | A Stage-1 page error silently truncates the download | `run.py:306-308` | S1 |
| GAP-10 | A failed Stage-2 batch is skipped; the "next run retries" comment is false | `run.py:506-508` | S1 |
| GAP-11 | Resume only works on the same day (`report_date` = today) | `main()` | S2 |
| GAP-12 | Scheduler keeps the next run only in memory, so a restart can silently skip a month | `scheduler.py:56-66` | S2 |
| GAP-13 | Mac sleep pauses runs (19 h on 10-01, 56 min late start) | Health audit | S2 (cutover) |
| GAP-14 | "Unmatched records" warning can never fire (inner join) | `run.py:738-745` | S1 |
| GAP-15 | Config check misses `LAWRULER_URL` and `LAW_RULER_ACCOUNT_KEY` | `run.py:1363-1364` | S1 |
| GAP-16 | Misleading report labels ("Total Leads Scrubbed" = all `raw_data`) | `generate_report` | S1 |
| GAP-17 | No per-stage duration or memory instrumentation | `main()` | S1 |
| GAP-18 | Full lead history held twice in memory (Stage 1 + Stage 4 re-read) | `run.py:663-718`, `1377-1383` | S1 (closing item, after the code is stable) |
| GAP-19 | Full-history re-ingest every run, with a 2 s/page throttle | `run.py:267`, `321` | Decision D-05 |
| GAP-20 | Stage 4 sequential per-lead updates dominate run time | `run.py:958-987` | Decision D-06 |
| GAP-21 | No runbook for failed or degraded runs (DNCUP-4) | — | S1 |
| GAP-22 | No per-stage inputs/outputs document (DNCUP-5) | — | S1 |
| GAP-23 | Stale README and Jira (DNCUP-2/3/6 superseded); docs still say `Container/` | — | S1 |
| GAP-24 | Secrets committed to `github.com/<docs-repo-owner>/DNC_LOCALSERVER` (`.env`, token in `notebooks/manual_lead_creation.ipynb`); inline passwords in `Dev/Archive/notebooks/*.ipynb` | Scrub_API inspection 2026-10-29 | S1: **credential rotation + delete the deprecated GitHub repo after a local `git bundle` backup** (extends GAP-07; GitHub action as `<docs-repo-owner>` per vault `CLAUDE.md` §15). Graph secret rotated 2026-11-11; LR API username/password pending (D-20) |
| GAP-26 | Three separately coded retry loops with different rules (page: timeouts only; batch and lead: 429/5xx at 1/2/4 s); the LawRuler token call has none | `run.py:282-304`, `513-535`, `866-909`, `146` | S1: shared retry policy emitting failed-item events (S2 persists them) |
| GAP-27 | `--dry-run` still calls the paid dncscrub API for every phone | `run.py:1015`, `1510` | S1 (dry-run story) |
| GAP-28 | Dry-runs write into the real `data/<date>/` folder, including Stage 2 checkpoints (synthetic results could be reused by a same-day real run), and email real recipients | `run.py:1418-1464`, `489-506`, `1265-1400` | S1 (dry-run story) |
| GAP-29 | When every Stage 2 batch fails or is deferred, Stage 3 crashes with "No scrub results found"; the failed-items summary is skipped and the error email doesn't list the failed batches | `run.py:682`, `1750-1760`, `1805` | S1 ([[Story - Failed scrub batches are retried and reported]]) |
| GAP-31 | Literal secret masking looks up UPPERCASE config keys (`LAW_RULER_ACCOUNT_KEY`, `O365_CLIENT_SECRET`, `LOGIN_ID`) that the config dict stores in lowercase, so those three values are not literal-masked (pattern masking still applies) | `dnc/log_setup.py:50-52` | Found by the 2026-11-10 code review; fixed on `s1/fix/code-review-findings` |
| GAP-32 | Config validation reports the wrong variable names (`AUTH_URL` for `auth_url`; the password and O365 secret mapped to the literal `***`) and never checks `law_ruler_client_id`/`law_ruler_client_secret`, so a missing client secret fails late at the token call | `dnc/config_check.py:9-21`, `38-47` | Found by the 2026-11-10 code review; fixed on `s1/fix/code-review-findings` |
| GAP-33 | The error email reads the whole log into memory (`readlines()`) for its 50-line preview, including earlier same-day runs | `run.py:1728` | Found by the 2026-11-10 code review; fixed on `s1/fix/code-review-findings` |
| GAP-30 | Removal deadline is always the 7th of the run's month, so an ad-hoc run after the 7th (e.g. 2026-11-10) labels the report and the soft-deletion request with a past date | `dnc/report_summary.py:61-62`, `dnc/soft_delete_request.py:91-113` | Decided 2026-11-10 (`<owner>`): run date + 6 days (1st → 7th unchanged). Fixed on `s1/fix/removal-deadline` (`b9607c6`); tracked in [[Story - Monthly report email as a summary table]] |
| GAP-25 | About 4.1 GB of lead PII sits in OneDrive (`data/legacy/`) with no retention policy | Scrub_API inspection | S2 (moves to the ITServer at cutover) / S5 (retention) |
| GAP-34 | Two separate checksum code paths for the raw snapshot. (a) **Order dependency:** the writer hashes rows in arrival order, while `verify_raw_snapshot.py` sorts before hashing (`canonical_checksum`). They agree only because Stage 1 reads SQLite with `ORDER BY "Lead ID", "Phone"` (TEXT, BINARY collation = Python string order); `write_raw_snapshot` neither states nor checks this, so a future caller passing unsorted rows would store a checksum that later fails verification. (b) **Verify tool memory:** because `canonical_checksum` needs the full list to sort, the verify tool `fetchall()`s a whole run and copies every row into a dict (2026-11-10: 426,075 rows × 15 columns, ≈243 MB of JSON, estimated 1–2 GB peak); it cannot stream. One fix closes both: a single streaming checksum function used by the writer and the verify tool, with the verify tool reading through a server-side cursor sorted by `lead_id, phone`. (c) **Identifier hardening (same rewrite):** the schema name is put into SQL with f-strings (`pg_store.py`, `verify_raw_snapshot.py`); safe today because only `dnc`/`dnc_test` pass validation, but the rewrite should compose it with `psycopg.sql.Identifier` so safety doesn't depend on validation alone (independent review, 2026-11-13) | `Dev/dnc/pg_store.py` (`write_raw_snapshot` COPY loop vs `canonical_checksum`), `Dev/tools/verify_raw_snapshot.py:84-108`, Stage 1 `ORDER BY` in `write_raw_snapshot_from_sqlite` (branch `s2/feat/raw-snapshot-postgres`) | Decided 2026-11-17 (`<owner>`): option A — one shared streaming checksum function used by the writer and the verify tool; it enforces strictly ascending (lead_id, phone) order and fails loudly on out-of-order or duplicate keys; the verify tool reads the run through a server-side cursor ordered by lead_id, phone (columns COLLATE "C", so database order = Python order), flat memory; schema names composed with psycopg.sql.Identifier (D-25). D-21(b) unchanged. Rejected: an order-independent combined hash (redefines D-21(b), weaker against deliberate tampering for a legal record). In progress on branch s2/fix/gap-34-streaming-checksum. S2, [[Story - Raw snapshot and run record land in Postgres]] |
| GAP-35 | Git history of both repos still carries the owner's name, emails and Mac hostnames (commit identities, pre-masking blobs, deleted notebooks); the working tree was masked 2026-11-19 | `git log --all --format='%an <%ae>'`, `git log --all -S` | Deferred: [[Plan - Git history identifier rewrite]] |
| GAP-36 | Masking and the neutral commit identity are not enforced mechanically: two `Docs/` commits on 2026-11-20 used a personal identity right after the masking pass | `git log --format='%an <%ae>'` (`922b745`, `5a7a868`) | Open: enforce from now on (e.g. a pre-commit hook checking identity + masking grep), mechanism to be decided (D-30) #open-question |
| GAP-37 | `.env.example` sets `OUTPUT_DIR=/output` and `LOG_DIR=/logs`; real runs honour them (dry-runs don't), so a container `.env` copied from the example writes the report and `run.log` inside the container, lost at exit. Hit on the ITServer parallel run 2026-11-21 | `Dev/.env.example` lines 57-59, `dnc/run_paths.py` | Mitigated 2026-11-23 (lines removed from the ITServer `.env`); fix `.env.example` + config validation in [[Story - Monthly run from versioned systemd units]] |
| GAP-38 | Config validation doesn't check email address format; one recipient without `@` made the success/report email fail (Graph 400) on 2026-11-21 | `dnc/email_helpers.py`, config check | Mitigated 2026-11-23 (both `.env` fixed); add to [[Story - Config validated before any API call]] |
| GAP-39 | A monthly run can last more than 2 days when a quarterly full-scrub batch comes due (Stage 4 ~54 h for 362k leads on 2026-11-21, ~0.5 s per LawRuler write) | memory log, run 2026-11-21 | Open: [[Story - Scheduled patch pass on the 3rd]] and [[Story - Watchdog on the 5th]] must not assume the run ended #open-question |

## Decision items (not sprint work until decided)
| ID | Decision | Notes |
|---|---|---|
| D-01 | Known litigators: soft-delete or hard-delete? | Open since 2025-05 |
| D-02 | Handling of non-US numbers | Never built |
| D-03 | Reassigned-number check (6-month refresh)? | Never built |
| D-04 | Rewrite the process in pure n8n? | Decided 2026-11-10 (`<owner>`): no. S2 keeps Python + Docker and adds PostgreSQL; the tested `dnc/` package carries over. (Measured ~1.1 GiB working set and ~360k per-lead calls on a full pass argued against n8n.) |
| D-05 | Keep the 2 s/page LawRuler throttle? | The full snapshot is legally required, so only the throttle is in question |
| D-06 | Parallelize Stage 4 tagging? | Decided 2026-11-09: no, Stage 4 stays sequential (stability first; LawRuler's flakiness and rate limits) |
| D-07 | Automate the monthly soft-deletion through the LawRuler API (S6), or keep the email request to the LawRuler team? | Decided 2026-10-29: email request until at least after S5 |
| D-08 | Rotate the exposed credentials now? | Decided 2026-11-02: defer. Many other flows share them. Later options: rotate and update every flow, or create a dedicated account for this pipeline. Revisit when the *No silent data loss* epic closes. 2026-11-09: carried to S2; S1 closes without it, with a recurring reminder. Narrowed 2026-11-11 by D-20 |
| D-09 | What does a run do when every Stage 2 batch fails? | Decided 2026-11-04: finish as degraded. Skip Stages 3–4, report zero counts, warning email lists failed batches, exit 0 (GAP-29) |
| D-10 | Does the job email the soft-deletion request to LawRuler directly? | Decided 2026-11-05: no. The job sends an internal `[For review]` email (`<owner>`, CC e.cervera) with both lists. `<owner>` reviews and forwards it to the LawRuler team manually. The allowlist stays internal-only |
| D-11 | Re-verify the config-error alert path on the Mac? | Decided 2026-11-06: no; accepted on unit tests (`dnc/config_failure_alerts.py`). Both alert fixes (status line on all-channels-failed, Teams card on config failure) merged |
| D-12 | Automatic sleep prevention on the Mac until the ITServer cutover? | Decided 2026-11-06: no automatic changes beyond a daily wake at 19:45 UTC (`pmset repeat`). `<owner>` keeps the Mac awake by hand on run night |
| D-13 | Jira criteria in the README/references story? | Decided 2026-11-06: dropped (Jira `DNCUP` is deprecated); empty `Projects/DNC-Update/` deleted 2026-11-09 |
| D-14 | Where does the restored `Scrub_API/` folder go? | Decided 2026-11-06: `data/legacy/Scrub_API-2026-11-05/` after the historic-log inventory; moved 2026-11-09. Its old `.env` is covered by the rotation story (D-08) |
| D-15 | Who approves deleting historic log originals? | Decided 2026-11-09: `<owner>` approves the scrub report; the tool deletes only listed originals whose copy re-passes the checks. Done 2026-11-09: 19 deleted, 0 skipped |
| D-17 | Start S2 while S1 stories are still open? | Decided 2026-11-10 (`<owner>`): yes. S2 starts now; the remaining S1 stories (report email awaiting a business-recipient run, Mac run-night safeguards, Stage 2 mean memory) are finished in December. S1 epics stay open until then |
| D-18 | Dedicated `dnc` database or tables in the existing ITServer DB? | Decided 2026-11-10 (`<owner>`): dedicated `dnc` database (no S3/S4 joins with the existing DB needed). `<owner>` creates the database and the `dnc_*` roles from the DDL guide. Server amended by D-23 (2026-11-13). See [[Decision - Dedicated PostgreSQL database]] |
| D-16 | Stage 4 kill/restart test without a real run? | Decided 2026-11-10 (`<owner>`): accepted on the unit test (`test_checkpoint_resume_unchanged`) because the resume code (`update_status`, `_is_lead_done`) was not changed; a dry-run can't resume and a live test would write LawRuler tags |
| D-19 | Does the Mac job write to PostgreSQL before cutover? Which PostgreSQL version is the target? | Decided 2026-11-11 (`<owner>`): no. The Mac never writes to Postgres. Postgres writing is turned on during the migration and tested on the ITServer against a test table in the dedicated `dnc` database. Target: PostgreSQL **13.23** on the ITServer; DDL and driver must work on 13. Target amended by D-23 (2026-11-13): PostgreSQL 17 in a container. |
| D-20 | Credential rotation scope: which credentials from GAP-24 are rotated, accepted as-is, or deferred? | Decided 2026-11-11 (`<owner>`): Graph/O365 app client secret rotated 2026-11-11. LawRuler OAuth client secret is global/static on LawRuler's side, not rotatable, accepted. LawRuler session cookie optional, not pursued. `dnctag_local_py` API token and Postgres password: nothing to rotate (DB/service don't exist). LawRuler account key and dncscrub login ID not pursued. **LawRuler API username & password pending:** `<owner>` discusses with the team and reports at the end of S2 |
| D-21 | Raw snapshot design: row format, checksum, test target, run identity? | Decided 2026-11-11 (`<owner>`): (a) `raw_snapshot` stores key columns `lead_id`, `phone` + the full original LawRuler row as JSONB (survives LawRuler adding/renaming columns; later stories extract typed columns). (b) SHA-256 checksum = canonical serialization of the stored rows (after the (Lead ID, Phone) de-dup Stage 1 already applies), sorted by (lead_id, phone); recomputable from the DB by `Dev/tools/verify_raw_snapshot.py`. Reason: Stage 1 downloads ~1,000-row CSV pages, so there is no single 'fetched CSV' to hash, and a raw-bytes hash can't be recomputed from stored rows. (c) Test target = schema `dnc_test` in the `dnc` database with identical DDL, selected by `POSTGRES_SCHEMA`; dry-runs are always forced into `dnc_test`. (d) Every execution is its own run (new run_id, full snapshot); a resume of the same run folder reuses the stored run_id; if that run already has a stored snapshot (checksum recorded in pipeline_run), the raw write is skipped; otherwise the whole snapshot is written in one transaction (a failure rolls it back, so a partial snapshot can't exist). (d) amended 2026-11-13 (`<owner>`): skip-if-stored replaces ON CONFLICT DO NOTHING. |
| D-22 | Can a raw snapshot be deleted or edited through `pipeline_run` deletes or the archive role? | Decided 2026-11-12 (`<owner>`, branch review finding #5): no. FK `fk_raw_snapshot_run` without `ON DELETE CASCADE` (NO ACTION: deleting a run that still has snapshot rows fails); `dnc_archive` gets SELECT only on `pipeline_run` and `raw_snapshot` until S5 / [[Story - Closed months become read-only]] designs detach/export (likely a SECURITY DEFINER function owned by `dnc_owner`). `dnc_writer` grants unchanged. Applies to `Dev/schema/001_raw_snapshot.sql` on `s2/feat/raw-snapshot-postgres` |
| D-23 | Which PostgreSQL server hosts the dnc database? | Decided 2026-11-13 (`<owner>`): PostgreSQL **17** (minor version pinned in the image tag) in its own Docker container on the ITServer, data persisted on a host volume; the existing ITServer PostgreSQL 13 server is not used or touched. Reason: PostgreSQL 13 reached end of life on 2025-11-13 (13.23 is its final release); 17 is supported until 2029-11-08 (https://www.postgresql.org/support/versioning/). Amends D-19 (target was 13.23) and D-18 (the dnc database lives on its own server instead of the existing one; still a dedicated database with the dnc_* roles). Upgrade reminder: plan the next major upgrade before 2029-11-08 |
| D-24 | raw_snapshot primary key: composite or surrogate? | Decided 2026-11-13 (`<owner>`): composite primary key `(snapshot_month, run_id, lead_id, phone)`, no surrogate id. Reasons: one index on the largest table instead of two; the key is the legal fact itself (this phone for this lead in this run); on a partitioned table every unique key must include `snapshot_month`, so even a surrogate id would need `(snapshot_month, id)` to be referenced; downstream tables are small (2026-10-28 run: 421,776 raw rows, ~15.7k past filters), so 4-column references are cheap. |
| D-25 | How should the raw-snapshot checksum become one streaming function (GAP-34)? | Decided 2026-11-17 (`<owner>`): option A — one shared streaming checksum function used by the writer and the verify tool; it enforces strictly ascending (lead_id, phone) order and fails loudly on out-of-order or duplicate keys; the verify tool reads the run through a server-side cursor ordered by lead_id, phone (columns COLLATE "C", so database order = Python order), flat memory; schema names composed with psycopg.sql.Identifier. D-21(b) unchanged. Rejected: an order-independent combined hash (redefines D-21(b), weaker against deliberate tampering for a legal record). In progress on branch s2/fix/gap-34-streaming-checksum. |
| D-26 | Which LawRuler (and other service) credentials does the ITServer `.env` use for the Postgres setup dry-run? | Decided 2026-11-19 (`<owner>`): option A — the same accounts as the Mac job for the time being, typed by hand into a new `chmod 600` ITServer `.env` (never a copy of the Mac file, never committed). Rejected: B, wait for the team's LawRuler-user decision (would block Steps 10–12). D-20 stays open: the LawRuler API username/password decision is still due at the end of S2, and R2 stays open; the same credentials now live on a second host. |
| D-27 | How does the ITServer get and update the code? | Decided 2026-11-19 (`<owner>`): host git (2.52.0, installed with sudo) replaces the throwaway `alpine/git` container; the clone uses a per-clone `core.sshCommand` pointing at the key in `~/.ssh-dnc` (`<owner>`'s personal SSH key with passphrase until 2026-12-03, then a read-only deploy key in the same files). The server only pulls, never pushes. GitHub `main` is PR-only, so code lands by PR merge commit and the server fast-forwards. The container method stays in the guide as a fallback. |
| D-28 | Should the backups and restore drill come before the backfills? | Decided 2026-11-20 (`<owner>`): yes. [[Story - Backups and restore drill for the dnc database]] moves from S2 order 12 to 8 (parallel with the patch pass on the 3rd and the watchdog); the container-era backfill → 9, legacy profiling → 10, legacy backfill → 11, cutover criteria → 12. Reason: R7, no history is loaded into the legal record before a tested backup exists. The backup target decision now gates the backfills. |
| D-29 | LawRuler API username & password (D-20 pending item): rotate every flow or create a dedicated account? | Decided 2026-11-20 (`<owner>`): keep the current user for the time being; reviewed in the last sprint (S6), reminder in [[Epic - Automated soft-deletion]]. Amends D-20 and D-26; R2 stays open until then. |
| D-30 | Masking scope after the 2026-11-19 masking pass | Decided 2026-11-20 (`<owner>`): (1) history stays as is, including the two `Docs/` commits of 2026-11-20 (`922b745`, `5a7a868`, already pushed) made with a personal identity; they fall under the deferred [[Plan - Git history identifier rewrite]] (GAP-35). (2) ITServer identifiers (the `it.admin` account, `/home/it.admin/...` paths, the server hostname) are official paths of the server documentation and are not masked. (3) Masking is enforced from now on; the mechanism is GAP-36. Colleagues' names and the shared mailbox address: not decided. #open-question |
| D-31 | Cutover acceptance criteria | Decided 2026-11-20 (`<owner>`): (1) one parallel run of Mac and ITServer on real data, as soon as possible, the only one; the Mac container is shut down once its results are satisfactory. Its scope is open (#open-question in [[Story - Define cutover acceptance criteria]]). Constraint: Stage 4's `Scrubbed Date` annotation and Stage 1's 90-day filter mean both downloads must finish before either run writes to LawRuler. (2) alert verification kept; (3) first scheduled run kept, widened to the first full cycle (1st, 3rd, 5th); (4) rollback plan kept; (5) option A: container-era backfill before cutover, legacy after. |
| D-32 | Scope of the one parallel run (D-31) | Decided 2026-11-20 (`<owner>`): as soon as the blocking items close, before 2026-12-01; both hosts write to LawRuler (identical tags and Scrubbed Date, the test's fingerprint); business recipients get two copies (`<owner>` informs the team); the ITServer run is real and writes the production `dnc` schema; the ITServer then takes over the monthly job with today's pipeline (host move), and the systemd + Postgres switch stays at the end of S2. New story [[Story - Parallel run and early host move to the ITServer]], S2 order 0. Amends D-19 (no real ITServer runs before cutover) and splits the cutover epic in two phases. |
| D-33 | Parallel-run blockers (order 0) | Decided 2026-11-20 (`<owner>`): run today if nothing blocks it (report_date 2026-11-21 UTC); both images from `main` `b71cf98`; ITServer data path stays `/home/it.admin/dnc/data` → `/data`; interim off-machine copy = run folder into the Mac's OneDrive `data/` + `pg_dump` of `dnc` until the backups story ships; comparison checklist as proposed in [[Story - Parallel run and early host move to the ITServer]]. |
| D-34 | No interim off-machine copy of ITServer runs (amends D-33) | Decided 2026-11-23 (`<owner>`): no data moves from the ITServer to the Mac; run folders and `dnc` persist on the ITServer, which has no OneDrive by design. R6 stays open until [[Story - Backups and restore drill for the dnc database]] (order 8). |

## Open items (to resolve before or during backlog scaffolding)
1. **Cutover acceptance criteria.** Deferred by `<owner>`; handled during S2. 2026-11-20: decided (D-31, D-32).
2. ~~**`Scrub_API/` triage**~~ — ✅ resolved 2026-10-29: PDFs → `Docs/Reference/dncscrub-api/`; Azure-era files → `Docs/Archive/azure-function-era/`; data → `data/legacy/`; `dnctag_local_py` archived to `Dev/Archive/`; duplicates and deprecated code deleted; `Scrub_API/` removed.
> It's the local version (deprecated). It should be archived 👆🏽
3. **Teams through MS Graph.** The connector (credentials in `Dev/.env`) and the Graph calling code (email feature) already exist. To verify with `microsoft-learn` during the S1 story: which Graph permission lets the app post to a Teams channel or chat, since app-only posting has restrictions.
4. ~~**D-04 (n8n)**, discussed after the 2026-10-28 run data.~~ Decided 2026-11-10: no (D-04).
5. ~~**Leftovers in `Scrub_API/`**~~ — ✅ resolved 2026-10-29 (see item 2).
6. **Soft-deletion request recipient:** our LawRuler account manager (identity unknown). Until named, the request goes to `<owner>` to forward. Once known, add the LawRuler vendor domain to the request email's allowlist.
7. **PostgreSQL placement:** dedicated `dnc` database vs the existing DB. See [[Decision - Dedicated PostgreSQL database]].
8. **Raw-snapshot checksum: one streaming function for writer and verify tool (GAP-34).** Decided 2026-11-17 (`<owner>`): option A — one shared streaming checksum function enforcing strictly ascending (lead_id, phone) order, failing loudly on out-of-order or duplicate keys; the verify tool reads through a server-side cursor ordered by lead_id, phone, flat memory; schema names composed with psycopg.sql.Identifier (D-25). In progress on branch s2/fix/gap-34-streaming-checksum.
