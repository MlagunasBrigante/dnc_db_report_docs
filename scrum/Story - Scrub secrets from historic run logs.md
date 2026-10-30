---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Lean and secret-free logs]]"
sprint: S1
moscow: should
status: to-do
gaps: [GAP-07]
jira: ""
size: S
---
# Story: Scrub secrets from historic run logs

## User Story
**As** the compliance/legal owner, **I want** every historic run log replaced by a small, secret-free INFO-level copy, **so that** no OAuth client secret, bearer token or password remains in files stored in OneDrive, while we still keep a record of what each run did.

## Acceptance Criteria
- [ ] **Given** each historic run log (container era: `data/2026-08-12`, `09-01`, `10-01`, `10-28` under `logs/run.log`; legacy: every log under `data/legacy/**`), **when** the clean-up runs, **then** a `run.info.log` is written next to it containing only INFO and higher lines, with secrets masked by the same masking filter as [[Story - INFO logs with secret masking]].
- [ ] **Given** the `run.info.log` has been written and checked, **when** the original is deleted, **then** only the trimmed copy remains. The original is deleted only after the check passes (non-empty copy, stage markers present, zero secret-pattern matches).
- [ ] **Given** the clean-up is complete, **when** all of `data/` is grepped for the secret patterns, **then** there are zero matches. The report records counts only and never prints the values.
- [ ] **Given** the clean-up is complete, **when** the summary report is written, **then** it lists each run: original size, trimmed size, lines kept, and the result of the secret check. The total is expected to free about 2.5 GB.
- [ ] Michelle confirms the summary report before any original is deleted (deletion is irreversible).

## Notes
- **Decided 2026-10-29 (stories interview Q4): option A.** Keep an INFO-level, secrets-masked `run.info.log` per run and delete the full DEBUG original. Applies to every run, legacy included. Credentials still have to be rotated regardless ([[Story - Rotate exposed credentials and retire the old repo]]), because the originals already synced to OneDrive and some were emailed.
- **Gap reference**: GAP-07 — "Secrets exposed: OAuth secret and tokens in logs, hardcoded cookie, token/PEMs in legacy files; `data/*/logs`"
- **Historic log locations** (from run report and scope):
  - Container-era: `data/2026-08-12/logs/`, `data/2026-09-01/logs/`, `data/2026-10-01/logs/`, `data/2026-10-28/logs/`
  - Legacy era: `data/legacy/*/` (16 dated runs, plus `1st_run`–`4th_run`, plus `bug/`) — about 3.9 GB total
- **Secret patterns to scrub**: OAuth `client_secret=`, Bearer tokens (`Bearer ` or `"access_token":"`), passwords (`password=`), session cookies (`ASP.NET_SessionId=`), any base64 blob > 1 KB (likely logged request bodies with secrets).
- **Large files:** stream the logs line by line (plain `LC_ALL=C grep`-style filtering). Never load a 1.2 GB log into memory.
- **Scope**: this story handles only logs. Other legacy files (notebooks with passwords, PEMs) are out of scope here — they are in [[Story - Rotate exposed credentials and retire the old repo]].
- **Report format**: `Docs/Reference/historic-log-scrubbing-report.md` stating counts, method, and verification grep results.

## Dependencies
- [[Story - INFO logs with secret masking]] should complete first, so new logs stop leaking secrets before old logs are scrubbed. Can start in parallel if the inventory is already known.
