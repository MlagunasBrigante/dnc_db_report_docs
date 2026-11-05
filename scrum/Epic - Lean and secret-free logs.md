---
type: epic
project: dnc-cron-scrub
sprint: S1
status: to-do
gaps: [GAP-06, GAP-07, GAP-24]
---
# Epic: Lean and secret-free logs

## Description
Stop DEBUG-level logging from writing secrets and full HTTP request bodies to run logs, remove hardcoded credentials from the code, scrub secrets from historic logs, and rotate every exposed credential. Today logs reach GB scale and contain OAuth client_secret and Bearer tokens, and legacy files hold plaintext passwords — all stored in OneDrive.

## Gaps closed
- **GAP-06** — DEBUG logging for every library → logs reach GB scale
- **GAP-07** — Secrets exposed: OAuth secret and tokens in logs, hardcoded cookie, token/PEMs in legacy files
- **GAP-24** — Secrets committed to GitHub, LawRuler client_secret, passwords in archived notebooks

## Breakdown

### INVEST check

| Criterion | Rating | Notes |
|---|---|---|
| Independent | ✅ | No technical dependency on other S1 epics |
| Negotiable | ✅ | Historic log scrubbing (delete vs redact) is open for Michelle's call |
| Valuable | ✅ | Security & compliance owner can trust that secrets are not leaking and have been rotated |
| Estimable | ✅ | Log size, secret locations, and redaction patterns are known from the run report and legacy file inspection |
| Testable | ✅ | Grep historic and new logs for known secret patterns (client_secret, Bearer, passwords); verify rotated credentials work |

### Pattern applied
**Pattern 4: Data Variations** (different secret types, different remediation: logs, code, git history)

Earlier patterns rejected:
- Pattern 1 (Workflow Steps): spans multiple stages (logging config, code change, git history, credential rotation) but the stages are not a single workflow.
- Pattern 2 (Operations): the operations are on different artifacts (code, logs, git repo, Azure) and can be independent stories.

### Cynefin calibration
**Low uncertainty** — Log redaction patterns are known; retiring the old repo is a `git bundle` backup plus a GitHub delete; credential rotation is Azure/LawRuler API key generation.

> [!note]+
> COSMOS DB is deprecated for the foreseeable future. We must delete any reference to it.
>
> ✅ *Applied 2026-10-29: Cosmos removed from this epic, its rotation story and the scope's GAP-24.*

### Split evaluation
Each story delivers value to the compliance/legal owner: INFO logs are lean and safe, hardcoded secrets are gone, historic exposure is remediated, rotated credentials work. Sizes vary (S to L) because git-history purge and rotation are inherently larger, but all are vertical (each closes a security exposure end-to-end).

> [!check] Before closing this epic, verify Michelle's actions were actually done
> - [x] Confirm who can rotate each credential (LawRuler admin, dncscrub, Entra app). Michelle can (2026-11-02); rotation deferred, D-08. — [[Story - Rotate exposed credentials and retire the old repo]]
> - [ ] Delete the `DNC_LOCALSERVER` repo on GitHub as `MlagunasBrigante`, after the `git bundle` backup is verified. — [[Story - Rotate exposed credentials and retire the old repo]]
> - [ ] Set the recurring reminders for any credential that can't be rotated yet. — [[Story - Rotate exposed credentials and retire the old repo]]
> - [ ] Approve the summary report before any original log is deleted. — [[Story - Scrub secrets from historic run logs]]

## Child user stories
- [[Story - INFO logs with secret masking]]
- [[Story - Remove the hardcoded LawRuler session cookie]]
- [[Story - Scrub secrets from historic run logs]]
- [[Story - Rotate exposed credentials and retire the old repo]]

## Dependencies / sequencing
The hardcoded-cookie story can start immediately. The INFO-logs story should complete before the historic-log scrubbing (so new logs stop leaking before old logs are cleaned). The rotation story depends on knowing which secrets are exposed (so it can start after the other three are scoped, or in parallel if the secret inventory is already known from GAP-07/GAP-24).
