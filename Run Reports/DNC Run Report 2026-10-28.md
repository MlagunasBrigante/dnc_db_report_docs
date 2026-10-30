---
type: run-report
project: DNC-Update
run_date: 2026-10-28
trigger: manual (one-shot `once` container, outside the monthly schedule)
created: 2026-10-28
tags: [project/DNC-Update, run-report, gap-closure, migration-scoping]
---

# DNC Run Report — 2026-10-28 (manual run)

> **Point:** The run **completed (exit 0) but degraded**: all 7,713 leads were tagged in LawRuler, yet the warning email failed with a Graph `504` (same failure as 2026-10-01). Both the email failure and the run's memory peak are caused by Stage 6 email handling, not by the data work itself. The pipeline's real working set plateaus at **~1.1 GiB**; the **2.77 GiB** peak is email/log overhead.
>
> Purpose: baseline for **gap-closure and migration scoping**. No code was changed. Line references are to `Container/run.py` as of 2026-10-28.

## Evidence sources

| Source | Path | Notes |
|---|---|---|
| Pipeline log | `data/2026-10-28/logs/run.log` | 473 MB total; only the first **2.9 MB** (lines 1–25,506) is pipeline output — see §2 |
| Memory samples | `data/mem_2026-10-28.csv` | `docker stats` + cgroup `memory.peak`, every ~31 s, 238 samples, no gaps (20:47:03 → 22:51:14 UTC) |
| Scheduler history | `dnc_logs.txt` (vault root) | `docker logs dnc-scrub`, 2026-08-12 → 2026-10-28 |
| Run artifacts | `data/2026-10-28/{interim,processed}/` | DBs, CSV/XLSX report, zip |
| Code | `Container/run.py`, `Container/requirements.txt` (`O365==2.0.38`, `requests==2.32.3`) | |

All times are UTC.

---

## 1. Run summary

| Item | Value |
|---|---|
| Trigger | Manual one-shot container (`dnc-scrub:latest once`) |
| Start → end | 20:46:48 → 22:51:38 (**2 h 05 m**) |
| Exit | `✅ Pipeline completed successfully` (exit 0) |
| Verdict | **Degraded**: warning email failed (504); success email accepted but oversized (unverified delivery) |
| Records fetched from LawRuler | 421,776 (423 pages × 1,000) |
| Scrubbed against DNC | 7,768 phones (8 batches) |
| Leads tagged in LawRuler | 7,713 processed, 0 skipped, 0 final errors (1 transient connect timeout on lead 475667, retried OK at 21:35:44) |
| Leads with DNC tags | 3,028 |
| Warnings collected | 1 (the transient ERROR above → triggered the warning email) |

### Per-stage timeline and volume

| Stage | Start | Duration | Volume | Throughput | Code |
|---|---|---|---|---|---|
| 1 Ingest (LawRuler Custom Report) | 20:46:48 | 29 m 08 s | 421,776 rows fetched → 15,737 not recently scrubbed → 7,969 bad numbers removed → 7,768 to scrub | ~14.5 pages/min | `run.py:262-324` |
| 2 DNC scrub | 21:15:56 | 1 m 53 s | 7,768 phones, 8 batches of 1,000 | ~69 phones/s | `run.py:423-511`, call `run.py:1386-1393` |
| 3 Interpret | 21:17:49 | <1 s | 7,768 → 7,749 unique in `scrub_results_translated` | — | `run.py:545-633` |
| 4 LawRuler tag update | 21:17:50 | **1 h 27 m 50 s** | 7,713 leads (one HTTP call each) | **~1.46 leads/s (~0.68 s/lead), sequential** | `run.py:638-997` (`_update_lawruler` `run.py:958-987`) |
| 5 Report | 22:45:40 | 15 s | 7,713 rows → CSV + XLSX | — | `run.py:1005-1113` |
| 6 Email | 22:45:55 | 5 m 43 s | success email 202; warning email **504** | — | `run.py:1117-1313`, call `run.py:1423-1446` |

**Scoping notes from the timeline (not fixes):**
- Stage 1 re-downloads the **full lead history since 2022-01-01** on every run (`run.py:267`) and sleeps 2 s per page (`run.py:321`) → ~14 of the 29 minutes are deliberate throttle. Volume grows each month (408,100 → 413,984 → 421,776 for 09-01 → 10-01 → 10-28).
- Stage 4 is the wall-clock bottleneck: strictly sequential per-lead requests. Run duration scales with *leads to tag* (10-01: 28,982 leads → ~18 h; 10-28: 7,713 → ~1.5 h).

---

## 2. Email failure (Stage 6)

### What happened

| Step | Time | What was sent | HTTP | Effect on `run.log` |
|---|---|---|---|---|
| Stages 1–5 | 20:46:48–22:45:55 | — | — | **2.9 MB** (actual pipeline log) |
| Success email | 22:45:55 → 22:48:09 | `DNC_Report_2026-10-28.zip` (47 MB) → **63 MB** base64 JSON request | **202** Accepted | +126 MB (request body logged twice, lines 25523 and 25527) |
| Warning email | 22:48:09 → 22:51:38 | `run.log` (~129 MB at that moment) → **172 MB** JSON request, single POST | **504** Gateway Timeout after 3 m 16 s | +344 MB (logged twice, lines 25548 and 25552) |
| Final | 22:51:38 | | | **473 MB** |

Microsoft Graph limits (official docs, learn.microsoft.com — Azure/M365 MCP was unavailable; verified via web search of `attachment` / `createUploadSession` pages):
- Inline attachment in a single request: **< 3 MB**
- 3–150 MB: must use an **upload session** (chunked, ≤ 4 MB per PUT)
- Exchange Online default **max message size: 35 MB** (tenant-configurable)

Both emails are sent as a single `sendMail` POST with the attachment inline. The warning email (172 MB) exceeds every limit; the success email (63 MB request / 47 MB attachment) exceeds the inline limit and the default message-size limit but was accepted (`202` = accepted for processing, **not** delivered — receipt is unverified).

### Root causes — code responsible

| # | Cause | Code | Effect |
|---|---|---|---|
| E1 | Root logger and file handler set to **DEBUG** for every library, including `O365`, `requests_oauthlib`, `urllib3` | `run.py:113-114`, `run.py:119-121` | Full HTTP request bodies (base64 attachments) and OAuth exchanges (incl. `client_secret`, Bearer tokens) are written to `run.log`. Inflates the log ~160× and stores secrets in OneDrive. |
| E2 | `_zip_output` zips the **entire** output dir, including `scrub_ods.db` (164 MB) | `run.py:1152-1171` (walk at `run.py:1163-1169`) | Report zip is 47 MB; the actual report (XLSX 0.66 MB + CSV 0.93 MB) is ~1.6 MB. Verified: zip contains `scrub_ods.db`, `.xlsx`, `.csv`. |
| E3 | Success email attaches the zip inline in one request | `run.py:1190`, `run.py:1208-1209` | 63 MB single request; over the 3 MB inline and 35 MB default limits. |
| E4 | Warning email attaches the **full `run.log`** inline | `run.py:1307-1310` | Attaches the log already inflated by E1 + E3 → 172 MB request → 504. The **error** email has the same pattern (`run.py:1259-1260`), so a failed run would likely lose its failure alert the same way. |
| E5 | Warning email fires on any WARNING-or-above record, including errors that were retried successfully | `WarningCollector` `run.py:77-101`; snapshot `run.py:1420-1421` | A single transient, self-healed timeout triggered the oversized email. |
| E6 | Email failures are caught and logged as WARNING; run still reports success | `run.py:1423-1446`; README "email failures never fail the run" | Alerting fails silently, exactly when there is something to alert about. |

**Security exposure (scoping item):** because of E1 + E4, any *successful* past warning email attached a log containing the LawRuler OAuth `client_secret` and Bearer tokens. Historic `run.log` files in `data/` also contain them.

**Recurrence:** identical 504 on 2026-10-01 (`dnc_logs.txt` line 2568ff, `run.py:1310` call site at that version). Expected to recur on every run that emits ≥1 warning.

---

## 3. Memory usage

Container limit (Docker Desktop VM): **7.75 GiB**. `entrypoint.sh` `exec`s `run.py`, so container memory ≈ pipeline process memory.

### Overall

| Metric | Value | Source |
|---|---|---|
| **Mean** | **966 MiB** (median 1,121 MiB) | `docker stats` samples |
| **Max sampled** | **1,718 MiB (1.68 GiB)** | `docker stats`, 22:48:36–22:50:43 |
| **True peak** | **2,841 MiB (2.77 GiB)**, ≈ 36 % of limit | cgroup `memory.peak`, reached between 22:48:05 and 22:48:36 |
| Time ≥ 1 GiB | ~92 min (Stage 4 + Stage 6) | 179 samples × 31 s |
| Time ≥ 1.5 GiB | ~3 min (Stage 6 only) | 6 samples |
| Time ≥ 2 GiB | < 31 s (only seen by `memory.peak`) | 0 samples |
| OOM risk | None observed this run | exit 0; peak < 40 % of limit |

> `docker stats` reports usage excluding inactive page cache; `memory.peak` is the kernel high-water mark and may include file cache. Treat 2.77 GiB as an upper bound for the process.

### Per stage

| Stage | Duration | Samples | Mean | Min → Max | `memory.peak` at stage end |
|---|---|---|---|---|---|
| 1 Ingest | 29 m 08 s | 56 | 409 MiB | 108 → 721 MiB (linear climb) | 788 MiB |
| 2 Scrub | 1 m 53 s | 3 | 744 MiB | 744 → 745 MiB | 907 MiB |
| 3 Interpret | 1 s | 0 | — | — | — |
| 4 LawRuler tags | 1 h 27 m 50 s | 168 | **1,121 MiB** | 1,104 → 1,122 MiB (flat) | 1,360 MiB |
| 5 Report | 15 s | 0 | — | — | — |
| 6 Email | 5 m 43 s | 11 | 1,511 MiB | 1,159 → **1,718 MiB** | **2,841 MiB** |

`memory.peak` step changes: 21:17:53 (907 → 1,295 MiB, Stage 4 load), 22:46:30 (1,360 → 1,865 MiB, success email), 22:48:36 (1,865 → 2,841 MiB, warning email).

### Code responsible for the numbers

| Memory behaviour | Likely cause | Code | Confidence |
|---|---|---|---|
| Stage 1 linear climb 108 → 721 MiB | All 421,776 rows accumulated as a Python `list` of `dict`s (one dict per CSV row), then converted to a DataFrame — both briefly coexist | `run.py:280` (`leads_created = []`), `run.py:309-317` (`list(csv_reader)` + `extend`), `run.py:324` (`pd.DataFrame(leads_created)`) | High (climb tracks pages fetched) |
| Stage 1 result never released | Raw DataFrame kept on the ingestion object (`self._raw_df`) and copied in `clean()`; `ingestion` and `cleaned_df` stay referenced in `main()` for the whole run | `run.py:205`, `run.py:251`, `run.py:339`; `run.py:1377-1383` | Medium (inferred from code + plateau) |
| Stage 4 jump +~390 MiB and 1.1 GiB plateau | Re-reads the **entire** `raw_data` table (421,776 rows) from SQLite into a second DataFrame, plus `scrub_results_translated`, then builds the merged tagged-leads frame; all held for the 88-minute API loop | `run.py:663-666` → `run.py:696-718` (`pd.read_sql_query`, `run.py:710`); `run.py:670-672`; `run.py:929` | High for the jump (log "Pulled 421776 rows" at 21:17:54 matches the 21:17:53 peak step); medium for plateau composition |
| Stage 4 flat during API loop | Per-lead updates are buffered and flushed every 100 → no growth while tagging | `run.py:958-987`, `_flush_buffer` `run.py:819-838` | High |
| Stage 6 peak to 2.77 GiB | Attachment read + base64-encoded + JSON-serialised in memory by O365, and the whole request body then formatted into DEBUG log strings (multiple full-size copies) | Attach: `run.py:1208`, `run.py:1307-1308`; DEBUG config: `run.py:113-121`; zip contents: `run.py:1163-1169` | High (peak steps coincide exactly with the two `sendMail` request-build timestamps) |

**Takeaways for migration sizing (inference, not measured):**
- Real data working set ≈ **1.1 GiB** for ~422k raw rows, dominated by holding the full lead history in memory twice (Stage 1 + Stage 4 re-read). It will grow with history volume (`run.py:267` start date fixed at 2022-01-01).
- Everything above ~1.2 GiB in this run is attributable to Stage 6 email/log handling (E1–E4).
- A target environment with **≥ 2 GiB** would have absorbed this run as-is; ~1.5 GiB would likely suffice once E1–E4 are addressed. Validate with a measured run before committing.

---

## 4. Artifacts produced

| File | Size |
|---|---|
| `data/2026-10-28/processed/scrub_ods.db` | 164 MB |
| `data/2026-10-28/processed/DNC_Report_2026-10-28.zip` | 47 MB (contains `scrub_ods.db` — see E2) |
| `data/2026-10-28/processed/leads_with_tags_2026-10-28.xlsx` | 0.66 MB |
| `data/2026-10-28/processed/leads_with_tags_2026-10-28.csv` | 0.93 MB |
| `data/2026-10-28/interim/scrub_staging.db` | 10.9 MB |
| `data/2026-10-28/logs/run.log` | 473 MB (2.9 MB real; contains secrets — see E1) |
| `data/mem_2026-10-28.csv` | 13.7 KB (238 samples) |

---

## 5. Gap-closure candidates (no fixes applied)

| ID | Gap | Code | Category |
|---|---|---|---|
| G1 | Library DEBUG logging writes secrets and full request bodies to `run.log` | `run.py:113-121` | Security / reliability |
| G2 | Report zip includes the 164 MB ODS database | `run.py:1163-1169` | Email reliability |
| G3 | Attachments sent inline in a single `sendMail` (no size guard / upload session) | `run.py:1208`, `run.py:1307-1308` | Email reliability |
| G4 | Warning email attaches the full log | `run.py:1307-1308` | Email reliability / security |
| G5 | Self-healed retries count as warnings and trigger the warning email | `run.py:77-101`, `run.py:1420-1421` | Alert noise |
| G6 | Email failures are non-fatal and only logged → silent alert loss | `run.py:1423-1446` | Observability |
| G7 | No memory/duration instrumentation per stage (this report needed an external sampler) | `main()` `run.py:1318-1445` | Observability |
| G8 | Full-history re-ingest every run (since 2022-01-01) with 2 s/page throttle | `run.py:267`, `run.py:321` | Performance / cost |
| G9 | Full `raw_data` re-loaded into memory in Stage 4 while Stage 1 copy is still alive | `run.py:663-666`, `run.py:1377-1383` | Memory |
| G10 | Stage 4 sequential per-lead updates (~0.68 s/lead) dominate run time | `run.py:958-987` | Performance |
| G11 | Success-email delivery unverified (47 MB > 35 MB default) | `run.py:1208-1209` | Verify in mailbox / tenant limit |
| G12 | Historic logs and possibly past warning emails contain the LawRuler `client_secret` | `data/*/logs/run.log` | Secret rotation |
