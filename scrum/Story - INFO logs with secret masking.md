---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Lean and secret-free logs]]"
sprint: S1
moscow: must
s1_order: 2
status: merged — pending Mac verification (after the dry-run fix)
gaps: [GAP-06, GAP-07]
jira: ""
size: M
---
# Story: INFO logs with secret masking

## User Story
**As** the compliance/legal owner, **I want** run logs to default to INFO level with library loggers kept off request bodies, and a redaction filter to mask secrets in any level, **so that** logs are lean (MB not GB per run) and never expose credentials.

## Acceptance Criteria
- [ ] **Given** the logging config, **when** a run starts, **then** the file log level is INFO by default (not DEBUG), and library loggers (O365, requests_oauthlib, urllib3) are set to WARNING or higher to suppress request bodies.
- [ ] **Given** a DEBUG toggle environment variable (e.g. `DNC_LOG_LEVEL=DEBUG`), **when** it is set, **then** the file log level becomes DEBUG for troubleshooting (but still redacts secrets).
- [ ] **Given** any log record is written, **when** it contains a known secret pattern (e.g. `client_secret=`, `Bearer `, `password=`, `Cookie: `), **then** the redaction filter masks it (e.g. `client_secret=***REDACTED***`) before writing to the file.
- [ ] **Given** one real run with INFO logging, **when** I check the `run.log` size, **then** it is in the MB range (target: < 10 MB for a full run), not GB.
- [ ] Verified by grepping the new `run.log` for known secret patterns and confirming none appear.

## Notes
- **Code references** (verified against `Dev/run.py`):
  - Root logger level: `run.py:113-114` — currently `logging.DEBUG`
  - File handler level: `run.py:119-121` — currently `logging.DEBUG`
  - Library loggers: no explicit config (inherit root DEBUG) — O365, requests_oauthlib, urllib3 all log full request bodies at DEBUG
- **Root cause from run report 2026-10-28 (GAP-06)**: DEBUG logging for every library writes full HTTP request bodies (including base64 attachments) and OAuth exchanges (including `client_secret` and Bearer tokens) to `run.log`, inflating it ~160× (473 MB, of which only 2.9 MB is pipeline output).
- **Secret exposure (GAP-07)**: because of DEBUG logging, historic `run.log` files in `data/*/logs/` contain the LawRuler OAuth `client_secret` and Bearer tokens. Any past warning email that succeeded also attached a log with secrets. This story stops new leaks; [[Story - Scrub secrets from historic run logs]] remediates the old ones.
- **Redaction filter**: use a logging.Filter on the file handler; match regex patterns for known secret keys and mask the values. Patterns to cover: `client_secret=`, `Bearer `, `password=`, `Cookie: ASP.NET_SessionId=`, any base64 blob > 1 KB (likely an attachment).
- **Library loggers**: set `logging.getLogger('O365').setLevel(logging.WARNING)`, same for `requests_oauthlib` and `urllib3`, so they stop logging full request bodies.
- **Scope**: this story changes the logging config and adds redaction. It does NOT change the `WarningCollector` behavior (that's in [[Story - Warning and error emails always arrive]]).

## Dependencies
None — can start immediately.
