---
type: reference
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
status: in-review
created: 2026-11-05
tags: [project/dnc-cron-scrub, reference, runbook, operations]
---
# DNC Pipeline Runbook — Response Guide for Alerts

> **Purpose:** Defines who acts on each alert type, through which channel, and the steps to take. Written for the IT on-call (owner m.lagunas, backup e.cervera).

---

## Alert Routing and Response Times

| Alert Type | Owner | Backup | Response Time | Channels |
|------------|-------|--------|---------------|----------|
| **Error** (run failed) | m.lagunas | e.cervera | **Next business day** | Email (TO: `it.admin@`, owner; CC: `j.fernandez@`, `e.cervera@`), Teams (dedicated channel) |
| **Warning** (run succeeded with warnings) | m.lagunas | e.cervera | Urgent warnings: **next business day**; informational: **within the week** | Email (TO: `it.admin@`, owner; CC: `j.fernandez@`, `e.cervera@`), Teams (dedicated channel) |
| **Alert-delivery failure** (one or both channels failed) | m.lagunas | e.cervera | **Next business day** | Whichever channel worked, or logs only if both failed |
| **Success** (run completed) | — | — | No action required; archive the report | Email (TO: business + team), Teams (dedicated channel) |

**Backup acts when m.lagunas is out** — m.lagunas informs them directly.

---

## 1. Success Alert ✅

### What it means
The pipeline completed successfully. All stages (ingest → scrub → tag → report → email) ran without errors or warnings.

### Who acts
**No action required.** Archive the report.

### What to check
- **Email:** `DNC Scrub Report — <date> ✅` arrives with summary table (aggregate counts: total leads, leads scrubbed, DNC without opt-in, litigators to soft-delete).
- **Teams card:** Summary table with run duration and zero failed items.

### Reports location
- **Host:** `Projects/dnc-cron-scrub/data/<YYYY-MM-DD>/processed/`
  - `leads_with_tags_<date>.xlsx` (Excel report, 3 sheets: Summary, Litigators, Raw Data)
  - `leads_with_tags_<date>.csv` (CSV export of tagged leads)
  - `dnc_without_optin_<date>.csv`, `litigators_to_soft_delete_<date>.csv` (soft-deletion lists)

---

## 2. Warning Alert ⚠️

### What it means
The run completed but emitted WARNING, ERROR, or CRITICAL log messages. Stages 1–5 finished, but something needs review.

### Who acts
**Owner:** m.lagunas | **Backup:** e.cervera (when m.lagunas is out)

### Response time
- **Urgent warnings** (see table below): **next business day**
- **Informational warnings**: **within the week**

### What to check

#### 2.1 Classify the warning

**Urgent** — act next business day:

| Warning Message | Meaning | Action |
|----------------|---------|--------|
| `Missing required env vars: …` | Config incomplete; run may have crashed or skipped stages | Check `.env` for missing credentials; re-run if needed |
| `Stage 1: raw snapshot INCOMPLETE — N page(s) failed/deferred` | Lead download truncated; lead count may be understated | Check INCOMPLETE marker file in output folder; review failed pages; manual re-run if critical |
| `Raw snapshot complete: NO — see INCOMPLETE marker` | Report row flags incomplete data | Same as above |
| `Stage <N>: M item(s) failed/deferred` | Pages, batches, or leads failed after retries | Review log excerpt attachment; check if items are critical; manual re-run if needed |
| `Stage 1: Pagination stopped at page N — breaker open` | Outage breaker tripped; remaining pages deferred | LawRuler API likely down; wait and re-run when stable |
| `Stage 2: no scrub results — all N batch(es) failed/deferred; skipping Stages 3–4` (CRITICAL) | Scrub service unavailable; no leads were scrubbed | dncscrub.com likely down; wait and re-run when stable |
| `Alert delivery failed on all channels` | Neither email nor Teams worked; alerts sent to logs only | Restore failed channel (see §4 below); check logs for the original alert content |

**Informational** — review within the week:

| Warning Message | Meaning | Action |
|----------------|---------|--------|
| `No new leads needed scrubbing this run` | Zero phones after 90-day filter | Normal if recent scrub; no action unless unexpected |
| *(none)* | Transient failure that succeeded on retry | Logged at DEBUG only, so it never reaches an alert. No action |
| `LAWRULER_SESSION_COOKIE not configured` | Optional env var missing | Confirm next real run works without it (D-08); add if needed |
| `Teams not configured; skipped` | `TEAMS_WEBHOOK_URL` unset (S1 delivery in progress) | Normal until Teams channel is set up; no action |
| `DRY-RUN: Teams skipped (DRYRUN_TEAMS_WEBHOOK_URL not set)` | Test channel unset in dry-run | Normal; no action |

#### 2.2 Check the log

**Email attachment:** `run_log_excerpt.txt` (WARNING/ERROR/CRITICAL lines only, capped at 1 MB)

**Full log location:** `data/<YYYY-MM-DD>/logs/run.log` (host path) or `docker exec dnc-scrub cat /data/<YYYY-MM-DD>/logs/run.log`

#### 2.3 Failed items summary

Warning email lists failed/deferred items by stage (e.g., "Stage 1: 2 item(s) failed/deferred: page 5, page 7"). Check the log excerpt for details.

### Manual re-run (if needed)

**Safe throwaway container** (never touches the scheduled `dnc-scrub` container):

```bash
cd /path/to/dnc-cron-scrub
docker run --rm --env-file Dev/.env \
  -v "$PWD/data:/data" \
  dnc-scrub:latest once --dry-run  # Test first
docker run --rm --env-file Dev/.env \
  -v "$PWD/data:/data" \
  dnc-scrub:latest once            # Real run
```

**Notes:**
- Dry-run first to test without paid API calls or LawRuler writes.
- Same-date reruns append to the same `data/<date>/` folder; logs grow, DBs merge via `INSERT OR IGNORE`.
- INCOMPLETE marker persists across same-date reruns unless manually deleted.

---

## 3. Error Alert ❌

### What it means
The pipeline failed with an unhandled exception. The run did not complete.

### Who acts
**Owner:** m.lagunas | **Backup:** e.cervera (when m.lagunas is out)

### Response time
**Next business day**

### What to check

#### 3.1 Email and Teams alert
- **Email:** `DNC Scrub Pipeline FAILED — <date> ❌` with traceback + last 50 log lines
- **Teams card:** Exception type + message (first 500 chars), log file path

#### 3.2 Check the log
**Full log:** `data/<YYYY-MM-DD>/logs/run.log` (host path) or `docker logs dnc-scrub` (scheduler logs)

**Attachment:** `run_log_excerpt.txt` (WARNING/ERROR/CRITICAL lines, capped 1 MB)

#### 3.3 Common error causes

| Error Type | Likely Cause | Action |
|------------|--------------|--------|
| `Missing required env vars: …` | `.env` incomplete or missing | Check `Dev/.env` for missing credentials; compare with `Dev/.env.example` |
| `LawRuler auth failed after N attempts` | OAuth credentials invalid or expired | Verify `auth_url`, `law_ruler_client_id`, `law_ruler_client_secret`, `law_ruler_username`, `law_ruler_password` in `.env` |
| `requests.exceptions.ConnectionError` | Network failure or API down | Check network; confirm API endpoints are reachable; wait and re-run |
| `sqlite3.OperationalError` | Database corruption or disk full | Check disk space on host; confirm `data/` is writable; manual cleanup if needed |
| `O365 auth failed` | Microsoft 365 app credentials invalid | Verify `O365_CLIENT_ID`, `O365_CLIENT_SECRET`, `O365_TENANT_ID` in `.env` |

#### 3.4 Check the last successful run

Last successful run folder: `data/<YYYY-MM-DD>/` (most recent with exit code 0)

**Scheduler logs:**
```bash
docker logs dnc-scrub | grep "Pipeline exited"
```

Look for `Pipeline exited with code 0 (success)` to find the last successful date.

### Manual re-run (after fix)

Same as §2.3 above — test with `--dry-run` first, then real run.

### Escalation path

If the error is unclear or requires LawRuler/Azure admin access:
- **LawRuler credentials:** m.lagunas (owner)
- **Azure app registration (O365/Teams):** j.fernandez (M365 admin)
- **Soft-deletion request approval:** Matt / Brad (escalation path, risk R5)

### S1 limitation: patch pass not yet available

**Note (S1):** The manual patch pass (re-run of failed items only) is an S2 feature. In S1, a manual re-run repeats the whole pipeline. Same-date reruns resume from checkpoints (completed batches/leads are skipped), but failed items are not automatically retried — you must trigger a full re-run.

**Coming in S2:** Patch pass on the 3rd (scheduled), systemd watchdog, and automatic failure capture for patch targeting.

---

## 4. Alert-Delivery Failure 🔕

### What it means
One or both alert channels (email, Teams) failed to deliver an alert (success, warning, or error).

### Who acts
**Owner:** m.lagunas | **Backup:** e.cervera (when m.lagunas is out)

### Response time
**Next business day**

### What to check

#### 4.1 How you'll know

**One channel failed:**
- Email failed, Teams worked → Teams card includes "Success/Warning/Error email failed to send"
- Teams failed, email worked → Follow-up email sent: `[DNC scrub] Teams alert failed to post`

**Both channels failed:**
- Exit code 3 logged in `docker logs dnc-scrub` or `data/<date>/logs/run.log`
- Scheduler logs: `Run finished but alert delivery failed on all channels (exit 3)`
- **No alerts delivered** — must check logs manually

#### 4.2 Check logs

**Scheduler logs:**
```bash
docker logs dnc-scrub | tail -50
```

**Run log:**
```bash
cat data/<YYYY-MM-DD>/logs/run.log | grep -E "CRITICAL|ERROR|WARNING" | tail -50
```

Look for:
- `Alert delivery failed on all channels` (CRITICAL)
- `Cannot send <success|warning|error|follow-up> email — auth failed: …` (ERROR)
- `Teams alert failed to post after retries` (ERROR)

#### 4.3 Check the output directory

Even if email failed, the report was written to disk:
```
data/<YYYY-MM-DD>/processed/
├── leads_with_tags_<date>.xlsx
├── leads_with_tags_<date>.csv
├── dnc_without_optin_<date>.csv
└── litigators_to_soft_delete_<date>.csv
```

**Action if email failed:** Forward the report manually to business recipients (see `.env` for `REPORT_RECIPIENTS_TO` and `REPORT_RECIPIENTS_CC` addresses).

### Restore a failed channel

#### 4.4.1 Email channel (Microsoft 365 via Graph)

**Cause:** O365 app credentials invalid, expired, or missing.

**Check:**
1. Verify `.env` has all three: `O365_CLIENT_ID`, `O365_CLIENT_SECRET`, `O365_TENANT_ID`
2. Test auth manually:
   ```python
   from O365 import Account
   account = Account((client_id, client_secret), tenant_id=tenant_id)
   if account.authenticate():
       print("Auth OK")
   else:
       print("Auth failed")
   ```
3. If expired: regenerate the client secret in Azure Portal → App registrations → `<app name>` → Certificates & secrets → New client secret
4. Update `O365_CLIENT_SECRET` in `Dev/.env`
5. Recreate the container (see below)

**Email sender address:** `EMAIL_SENDER` in `.env` (default: `it.admin@briganteimmigration.com`). Confirm this mailbox has `Mail.Send` permission in the app registration.

#### 4.4.2 Teams channel (Workflows webhook)

**Cause:** Webhook URL invalid, expired, or the workflow creator's account is disabled.

**Check:**
1. Verify `.env` has `TEAMS_WEBHOOK_URL` (production) or `DRYRUN_TEAMS_WEBHOOK_URL` (test channel)
2. Test the webhook manually:
   ```bash
   curl -X POST -H "Content-Type: application/json" \
     -d '{"text": "Test alert"}' \
     "<TEAMS_WEBHOOK_URL>"
   ```
   Expect HTTP 200 or 202.
3. If failed (403/404): the webhook may be deleted or the creator's account disabled.

**Restore:**
1. m.lagunas: create a new workflow in the dedicated channel:
   - Teams → Channel → `…` → **Workflows** → **Send webhook alerts to a channel**
   - Copy the webhook URL
2. Update `TEAMS_WEBHOOK_URL` in `Dev/.env`
3. Recreate the container (see below)

**Important:** The webhook URL is a secret — anyone with it can post to the channel. Never commit it to Git or share it in unencrypted channels.

#### 4.4.3 Recreate the container (after .env change)

**Confirm with m.lagunas before running this command** — it stops the live scheduler:

```bash
cd /path/to/dnc-cron-scrub
docker rm -f dnc-scrub
docker run -d --name dnc-scrub --restart unless-stopped \
  --env-file Dev/.env -e TZ=UTC \
  -v "$PWD/data:/data" \
  dnc-scrub:latest
```

Verify the scheduler restarted:
```bash
docker ps | grep dnc-scrub         # Should show "Up"
docker logs dnc-scrub | tail -10   # Should show "Scheduler started" + "Next run: …"
```

---

## 5. Soft-Deletion Review Email 📋

### What it means
Monthly internal review email listing DNC leads without opt-in and litigators to soft-delete. Subject: `[For review] DNC soft-deletion request — <date>`.

### Who acts
**Owner:** m.lagunas

### Response time
**Before the 7th of the run's month** (run on 1st, patch on 3rd, watchdog on 5th, removal by 7th)

### What to do
1. **Review the counts** in the email body (DNC without opt-in, litigators to soft-delete).
2. **Check the CSVs** (attached if total size < 2.5 MB; otherwise stored in `data/<date>/processed/`):
   - `dnc_without_optin_<date>.csv`
   - `litigators_to_soft_delete_<date>.csv`
3. **Forward the email to the LawRuler account manager** Lindsey Humphreys, `lindsey.humphreys@lawruler.com` (decision D-10; her address stays out of the job config) for manual removal.

**Important:**
- Recipients are internal only (`SOFT_DELETE_RECIPIENTS` TO, `SOFT_DELETE_CC` CC) and must be on the allowlist (`SOFT_DELETE_ALLOWED_DOMAINS`, default `briganteimmigration.com`).
- The job never emails LawRuler directly — m.lagunas forwards after review.
- **Coming in S6 (optional):** Automated soft-deletion (only if D-07 = automate).

---

## 6. How to Check the Last Successful Run

**Scheduler logs:**
```bash
docker logs dnc-scrub | grep "Pipeline exited with code 0"
```

**Output folders:**
```bash
ls -lt data/ | head -10  # Most recent run folders
```

Look for `data/<YYYY-MM-DD>/processed/leads_with_tags_<date>.xlsx` with today or the 1st of the month.

---

## 7. Safe Manual Re-Run

**Never stop, restart, or recreate the `dnc-scrub` container for a manual re-run** — it runs the scheduler. Always use a throwaway container:

```bash
cd /path/to/dnc-cron-scrub

# Test first (no paid API calls, isolated artifacts)
docker run --rm --env-file Dev/.env \
  -v "$PWD/data:/data" \
  dnc-scrub:latest once --dry-run

# Real run (writes tags to LawRuler, sends report email)
docker run --rm --env-file Dev/.env \
  -v "$PWD/data:/data" \
  dnc-scrub:latest once
```

**Notes:**
- `--rm` removes the container after exit (never conflicts with `dnc-scrub`)
- `--dry-run` uses synthetic scrub responses, isolated paths (`data/_dryrun/<timestamp>/`), test emails only
- Same-date real runs append to the same `data/<date>/` folder; logs grow, DBs merge via `INSERT OR IGNORE`
- Confirm the scheduler is still running afterward: `docker ps | grep dnc-scrub`

---

## 8. Where Outputs and Logs Live

| Path (host) | Purpose |
|-------------|---------|
| `data/<YYYY-MM-DD>/processed/` | Reports (xlsx, csv), soft-deletion CSVs, INCOMPLETE marker (if any) |
| `data/<YYYY-MM-DD>/interim/` | SQLite staging DB (scrub results, checkpoints) |
| `data/<YYYY-MM-DD>/logs/run.log` | Full run log |
| `data/_dryrun/<timestamp>Z/` | Dry-run artifacts (isolated, auto-purged after 7 days) |
| `docker logs dnc-scrub` | Scheduler logs (next run time, exit codes) |

**Container paths:**
- Host `data/` is bind-mounted to `/data` inside the container.
- Same paths inside: `/data/<date>/processed/`, `/data/<date>/logs/run.log`

---

## 9. S2 Features (Coming Soon, Not Yet Available)

The following are **not available in S1** but will be added in S2:

| Feature | What It Does | When |
|---------|--------------|------|
| **Patch pass** | Scheduled re-run of failed items only on the 3rd of each month | S2 |
| **Watchdog heartbeat** | Systemd timer posts a "DNC watchdog OK" card to Teams by the 5th; alerts if no heartbeat | S2 |
| **Scheduler recovery** | If watchdog fails, check `systemctl status dnc-scrub-watchdog.timer` and `journalctl` on ITServer | S2 |
| **ITServer cutover** | PostgreSQL system of record, systemd scheduling, AlmaLinux host | S2 |
| **Historical backfill** | Notebook-era run history (2025-05 → 2026-07) imported into PostgreSQL | S2 |

**For now:** manual re-runs replace the patch pass; check `docker logs dnc-scrub` instead of systemd; all runs are on m.lagunas's Mac (Docker Desktop), not ITServer.

---

## 10. Never Do This Without m.lagunas

- **Delete or recreate the `dnc-scrub` container** — it runs the scheduler; stopping it stops the monthly cadence.
- **Move or remove `data/`** — it holds the run history and is bind-mounted into the container; moving it breaks the mount.
- **Edit or delete `Dev/.env`** — it holds credentials; always back it up first.
- **Commit or push secrets** — `Dev/.env`, webhook URLs, and lead data (PII) must never enter Git.

---

## 11. Quick Reference Card

| Alert | Who | When | What |
|-------|-----|------|------|
| ✅ **Success** | — | — | No action; archive |
| ⚠️ **Warning** | m.lagunas (e.cervera backup) | Urgent: next business day; informational: within the week | Check log excerpt; classify warning; re-run if critical |
| ❌ **Error** | m.lagunas (e.cervera backup) | Next business day | Check log + traceback; fix cause; re-run |
| 🔕 **Alert-delivery failure** | m.lagunas (e.cervera backup) | Next business day | Check logs; restore failed channel; recreate container if needed |
| 📋 **Soft-deletion review** | m.lagunas | Before the 7th | Review CSVs; forward to LawRuler team manually |

**Logs:** `data/<date>/logs/run.log` or `docker logs dnc-scrub`  
**Reports:** `data/<date>/processed/*.xlsx`  
**Manual re-run:** `docker run --rm --env-file Dev/.env -v "$PWD/data:/data" dnc-scrub:latest once [--dry-run]`  
**Scheduler health:** `docker ps && docker logs dnc-scrub`

---

*Written 2026-11-05 for S1. Behaviors grounded in code (`Dev/run.py`, `Dev/dnc/*.py`). Updated as features land.*
