---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reproducible scheduling on the ITServer]]"
sprint: S2
moscow: must
status: to-do
gaps: []
jira: ""
size: S
---
# Story: Watchdog on the 5th
## User Story
**As** Michelle (pipeline owner), **I want** a watchdog that alerts if no successful run occurred this month, unresolved items remain, or a run is still going after 4 days, **so that** silent failures and stuck runs are caught automatically.

## Acceptance Criteria
- [ ] **Given** the repo is updated with the watchdog units, **when** `Dev/deploy/install.sh` is executed, **then** it installs `dnc-scrub-watchdog.service` and `dnc-scrub-watchdog.timer` to `/etc/systemd/system/`, and enables the timer (verified by `systemctl list-timers` showing `dnc-scrub-watchdog.timer` with next run on the 5th at 08:00 UTC)
- [ ] **Given** the watchdog runs on the 5th, **when** no completed run exists for the current month (simulated by clearing `pipeline_run` or setting all runs' status to 'failed'), **then** the watchdog sends an email + Teams alert "DNC watchdog: No successful run in [Month]" (verified by checking the alert was sent)
- [ ] **Given** a completed run exists but has unresolved failed items, **when** the watchdog runs, **then** it queries `failed_items WHERE run_id=<current_month_run> AND resolved_at IS NULL` and alerts "DNC watchdog: 3 unresolved items from 2026-10 run (lead_123, lead_456, batch_7)" (verified by injecting unresolved items and checking the alert)
- [ ] **Given** a run started on the 1st and is still in status='running' on the 5th (4+ days later), **when** the watchdog runs, **then** it alerts "DNC watchdog: Run <run_id> still running after 4 days (started 2026-10-01 02:00)" — abnormally long run detected (verified by manually setting a run's status to 'running' with an old start_time)
- [ ] **Given** the current month's run completed, has no unresolved items, and its soft-deletion request was sent (or wasn't needed because both counts were 0), **when** the watchdog runs, **then** it **always posts a heartbeat** to the Teams channel: "✅ DNC watchdog — all clear for [Month]: run completed, 0 unresolved, soft-deletion request sent". **Silence on the 5th means the watchdog didn't run**, and the runbook says what to do. Verified by checking the channel after a clean test run.
- [ ] **Given** the watchdog itself fails (e.g. Postgres unreachable, bad config), **when** its service exits non-zero, **then** systemd's `OnFailure=` starts `dnc-scrub-notify-failure@%n.service`, a small notifier in `Dev/deploy/` that reuses the Graph code. It sends an email and a Teams alert "DNC watchdog FAILED: <unit> — see journalctl". Verified by breaking the DB connection string in a test and confirming both alerts arrive. The journal entry remains as a secondary record.
- [ ] **Given** a completed run where "DNC leads without opt-in" or "Litigators to soft-delete" is above 0, **when** the watchdog runs, **then** it checks that the soft-deletion request was sent for that run ([[Story - Soft-deletion request to the LawRuler team]]); if not, it alerts "DNC watchdog: soft-deletion request not sent for [Month]; removal by the 7th at risk".

## Notes
- **Decided 2026-10-29 (stories interview Q9):** the watchdog sends a monthly heartbeat even when all is clear; its own failure is escalated through `OnFailure=` to email and Teams; it also checks that the soft-deletion request was sent. This closes the "who watches the watchdog" gap, since a silent watchdog can no longer look like a healthy month.
- The same `OnFailure=` notifier should be attached to the monthly run and patch-pass units ([[Story - Monthly run from versioned systemd units]], [[Story - Scheduled patch pass on the 3rd]]), so a crash at the systemd level is never silent either.
- **Alert routing (decided 2026-10-29, stories interview Q1):** owner **Michelle**, backup **e.cervera**, who acts when Michelle is out (Michelle informs them). Email goes **to** `it.admin@` and the owner, **CC** `j.fernandez@` and `e.cervera@` (same domain as `it.admin@`; recipients from `Dev/.env`, e.g. `EMAIL_RECIPIENTS_*` plus a new `EMAIL_CC`). Teams goes to a **dedicated channel that Michelle sets up**, not a 1:1 chat. Response times: **errors, alert-delivery failures and watchdog alerts** are acted on the next business day; **warnings and unresolved items** within the week, before the patch pass on the 3rd or the watchdog on the 5th.
- **Unit files:** `dnc-scrub-watchdog.service` and `dnc-scrub-watchdog.timer` (OnCalendar=*-*-05 08:00:00 UTC, Persistent=true)
- **Watchdog script:** `Dev/deploy/watchdog.py` or equivalent; queries Postgres and calls the existing email + Teams alert functions
- **Alert channel:** Reuses the existing S1 email + Teams infrastructure; subject line "DNC Watchdog Alert"
- **4-day threshold:** Based on the scope's assumption "A run that takes longer than 4 days is abnormal"; the longest run so far was 3.6 days (2026-10-01)
- **Current month:** `WHERE start_time >= date_trunc('month', CURRENT_DATE)` finds runs that started this calendar month

## Dependencies
- [[Story - Unresolved items reach the alerts]] (failed_items table and alert logic must exist)
- [[Story - Monthly run from versioned systemd units]] (install.sh and unit-file pattern are established)
- S1 email + Teams alerting
