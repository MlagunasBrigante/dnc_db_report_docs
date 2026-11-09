---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Lean and secret-free logs]]"
sprint: S1
moscow: should
status: in-review
gaps: [GAP-07]
jira: ""
size: S
---
# Story: Scrub secrets from historic run logs

> [!todo] Michelle — action needed
> - [ ] Approve the summary report before any original log is deleted.

## User Story
**As** the compliance/legal owner, **I want** every historic run log replaced by a small, secret-free INFO-level copy, **so that** no OAuth client secret, bearer token or password remains in files stored in OneDrive, while we still keep a record of what each run did.

## Acceptance Criteria
- [ ] **Given** each historic run log (container era: `data/2026-08-12`, `09-01`, `10-01`, `10-28` under `logs/run.log`; legacy: every log under `data/legacy/**`), **when** the clean-up runs, **then** a `run.info.log` is written next to it containing only INFO and higher lines, with secrets masked by the same masking filter as [[Story - INFO logs with secret masking]].
- [ ] **Given** the `run.info.log` has been written and checked, **when** the original is deleted, **then** only the trimmed copy remains. The original is deleted only after the check passes (non-empty copy, stage markers present, zero secret-pattern matches).
- [ ] **Given** the clean-up is complete, **when** all of `data/` is grepped for the secret patterns, **then** there are zero matches. The report records counts only and never prints the values.
- [ ] **Given** the clean-up is complete, **when** the summary report is written, **then** it lists each run: original size, trimmed size, lines kept, and the result of the secret check. The total is expected to free about 2.5 GB.
- [ ] Michelle confirms the summary report before any original is deleted (deletion is irreversible).

## Notes
- **Reviewed 2026-11-09 (branch `s1/feat/scrub-historic-logs`, tip `29dfdee`, 298 tests).** Review found and fixed: the first run re-read its own copies and truncated 9 legacy copies to 0 bytes; two originals in one folder collided on one copy name (copies are now `<stem>.info.log`); legacy `Access Token: <hex>` / `Token: <hex>` lines were not masked (patterns added to `dnc/log_masking.py`, so live logs are covered too); delete mode was not implemented (now: deletes only originals listed in the approved report whose copy re-passes every check at that moment); the secret count loaded whole files and was OOM-killed (now streamed, split copies vs originals). Report regenerated: 19 originals, 19/19 pass, 2.41 GB to free, **0 secret matches in copies**, all counted mechanically. Awaiting Michelle's approval and the Mac delete run.
- **Decided 2026-10-29 (stories interview Q4): option A.** Keep an INFO-level, secrets-masked `run.info.log` per run and delete the full DEBUG original. Applies to every run, legacy included. Credentials still have to be rotated regardless ([[Story - Rotate exposed credentials and retire the old repo]]), because the originals already synced to OneDrive and some were emailed.
- **Gap reference**: GAP-07 — "Secrets exposed: OAuth secret and tokens in logs, hardcoded cookie, token/PEMs in legacy files; `data/*/logs`"
- **Historic log locations** (from run report and scope):
  - Container-era: `data/2026-08-12/logs/`, `data/2026-09-01/logs/`, `data/2026-10-01/logs/`, `data/2026-10-28/logs/`
  - Legacy era: `data/legacy/*/` (16 dated runs, plus `1st_run`–`4th_run`, plus `bug/`) — about 3.9 GB total
- **Secret patterns to scrub**: OAuth `client_secret=`, Bearer tokens (`Bearer ` or `"access_token":"`), passwords (`password=`), session cookies (`ASP.NET_SessionId=`), any base64 blob > 1 KB (likely logged request bodies with secrets).
- **Large files:** stream the logs line by line (plain `LC_ALL=C grep`-style filtering). Never load a 1.2 GB log into memory.
- **Scope**: this story handles only logs. Other legacy files (notebooks with passwords, PEMs) are out of scope here — they are in [[Story - Rotate exposed credentials and retire the old repo]].
- **Report format**: `Docs/Reference/historic-log-scrubbing-report.md` stating counts, method, and verification grep results.
- **Implementation** (2026-11-09):
  - Initial version: Branch `s1/feat/scrub-historic-logs`, commit `b1c48d4`. Tool: `Dev/tools/scrub_historic_logs.py`. 
  - Fixes (commit `c7ea4c5`): Excluded *.info.log from inputs (tool was treating its own outputs as inputs, re-scrubbing and truncating to 0 bytes). Named copies after originals (`<stem>.info.log`, e.g., `logs.log` → `logs.info.log`) to avoid name collisions. Added masking patterns for legacy "Access Token: <hex>" format (22 occurrences found in legacy notebook logs). Updated report format: relative paths, clearer delete instructions, per-file output names. Deleted 10 stale 0-byte `run.info.log` files from buggy first run.
  - Re-ran tool 2026-11-09: 19 logs processed, all checks passed, 2.41 GB to free. Secret counts in originals: 7 client_secret, 7 Bearer, 21 access_token, 22 Access Token (legacy), 7 Token (legacy), 42 base64 blobs >1KB. All copies verified clean (0 unmasked secrets mechanically checked). Report: [[historic-log-scrubbing-report]]. Awaiting Michelle's approval for deletion.

## Dependencies
- [[Story - INFO logs with secret masking]] should complete first, so new logs stop leaking secrets before old logs are scrubbed. Can start in parallel if the inventory is already known.
