---
type: story
project: dnc-cron-scrub
epic: "[[Epic - No silent data loss]]"
sprint: S1
moscow: must
s1_order: 5
status: to-do
gaps: [GAP-15]
jira: ""
size: S
---
# Story: Config validated before any API call

## User Story
**As** the IT & Data lead, **I want** the pipeline to validate all critical config keys at startup and fail fast with a clear alert, **so that** a missing key never causes a cryptic failure halfway through a run.

## Acceptance Criteria
- [ ] **Given** the config dict is built from `.env` (`run.py:1342-1361`), **when** validation runs, **then** it checks for `LAWRULER_URL` and `LAW_RULER_ACCOUNT_KEY` (currently missing from the check at `run.py:1363-1364`) in addition to the existing keys.
- [ ] **Given** O365/Teams settings are configured (O365_CLIENT_ID, O365_CLIENT_SECRET, O365_TENANT_ID), **when** validation runs, **then** it confirms they are set if email or Teams alerting is enabled.
- [ ] **Given** any critical config key is missing, **when** the run starts, **then** it logs CRITICAL "Missing required env vars: [list]", sends an error email (if O365 keys are present), and exits with code 1 before making any API call.
- [ ] Verified by removing one key from `.env` (e.g. `LAWRULER_URL`) and confirming the run fails immediately with a clear message.

## Notes
- **Code reference** (verified against `Dev/run.py`):
  - Config dict: `run.py:1342-1361` — built from `os.getenv()` calls
  - Current validation: `run.py:1363-1368` — checks `auth_url`, `law_ruler_username`, `law_ruler_password`, `scrub_url`, `login_id`, `custom_report_url`, but **not** `LAWRULER_URL` or `LAW_RULER_ACCOUNT_KEY`
- **Gap reference**: GAP-15 — "Config check misses `LAWRULER_URL` and `LAW_RULER_ACCOUNT_KEY`" at `run.py:1363-1364`
- **Root cause from scope**: the config validation was added for Stage 1-2 keys but never updated for Stage 4 (LawRuler tagging) keys, so a missing `LAWRULER_URL` only fails when Stage 4 tries to post, not at startup.
- **Scope**: add `lawruler_url` and `law_ruler_account_key` to the `missing` list check. Also validate O365 keys if email/Teams is configured (check the `email_sender` or a new `ENABLE_EMAIL` flag).
- **Fail fast**: the check happens in `main()` before any stage runs, so the run exits before downloading any data or calling any API.
- **Error email**: if O365 keys are present, send an error email stating the config issue before exiting. If O365 keys are missing too, just log and exit.

## Dependencies
None — can start immediately.
