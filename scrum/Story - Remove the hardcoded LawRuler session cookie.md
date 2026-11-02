---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Lean and secret-free logs]]"
sprint: S1
moscow: must
s1_order: 2
status: merged — awaiting a real (non-dry-run) run
gaps: [GAP-07]
jira: ""
size: S
---
# Story: Remove the hardcoded LawRuler session cookie

## User Story
**As** the compliance/legal owner, **I want** the hardcoded LawRuler session cookie removed from the code, **so that** credentials live only in `Dev/.env` and not in source control.

## Acceptance Criteria
- [ ] **Given** the UpdateLawRulerPipeline class, **when** it initializes, **then** the `lawruler_headers` dict does not contain a hardcoded `Cookie` value (`run.py:681` currently has `'Cookie': 'ASP.NET_SessionId=vsmlthcod0f4nkjea22vtpa3'`).
- [ ] **Given** the LawRuler API requires a session cookie, **when** the cookie is needed, **then** it is read from `Dev/.env` (e.g. `LAWRULER_SESSION_COOKIE`) or verified to be unnecessary (consult LawRuler API docs or test without it).
- [ ] **Given** the hardcoded cookie is removed, **when** one real run completes, **then** Stage 4 successfully tags leads in LawRuler (confirming the cookie was not required or is now read from .env).
- [ ] Verified by grepping the codebase for `ASP.NET_SessionId` and confirming it appears only in comments or docs, not in live code.

## Notes
- **Verified 2026-11-02** with the dry-run `data/_dryrun/2026-11-02T194943Z/` (image built from `main` @ `b3b5a85`): the dry-run skips LawRuler updates, so the cookie removal is still unverified. It gets verified on the next real run.
- **Code reference** (verified against `Dev/run.py`):
  - Hardcoded cookie: `run.py:681` — `'Cookie': 'ASP.NET_SessionId=vsmlthcod0f4nkjea22vtpa3'` in `lawruler_headers`
- **Gap reference**: GAP-07 — "Secrets exposed: OAuth secret and tokens in logs, hardcoded cookie, token/PEMs in legacy files"
- **Uncertainty**: it is not clear whether the LawRuler lead-update API requires a session cookie, or whether the Bearer token (from OAuth) is sufficient. The code includes both. Test by removing the cookie and running `--dry-run` against LawRuler, or consult LawRuler API documentation.
- **Scope**: if the cookie is needed, move it to `Dev/.env` and read it from there. If it is not needed, delete the `Cookie` header entirely. Do NOT commit a new hardcoded value.
- This is a quick fix (S-sized) to remove one source of credential exposure.

## Dependencies
None — can start immediately.
