---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
moscow: should
status: to-do
gaps: []
jira: ""
size: S
---
# Story: Spike Teams posting through Graph

> [!todo] Michelle — action needed
> - [ ] If the spike needs admin consent for a Graph permission, request it from j.fernandez (M365 admin).

## User Story
**As** the IT & Data lead, **I want** a time-boxed investigation into which Microsoft Graph permission and target (channel vs chat) the existing app registration can use to post Teams messages, **so that** the Teams alert channel story knows which Graph API to call and whether any permission changes are needed.

## Acceptance Criteria
- [ ] **Given** the existing MS Graph connector in `Dev/.env` (O365_CLIENT_ID, O365_CLIENT_SECRET, O365_TENANT_ID) and the `_get_o365_account` code path (`run.py:1120`), **when** the spike completes, **then** it produces a recommendation document stating:
  - Which Graph permission (e.g. `ChannelMessage.Send`, `Chat.ReadWrite.All`, or app-only equivalent) the app registration needs
  - Which target (Teams channel, Teams chat, or both) is accessible with app-only posting restrictions
  - Whether the existing app registration already has the required permission or needs a tenant admin to grant it
- [ ] All claims about Graph permissions, app-only posting restrictions, and message-size limits are verified with the microsoft-learn MCP (vault CLAUDE.md §13).
- [ ] Time-boxed to 1 day; if Graph API access is blocked or requires a tenant-wide permission the app cannot get, the recommendation states "use email only" and the Teams alert channel story is dropped.

## Notes
- **Alert routing (decided 2026-10-29, stories interview Q1):** owner **Michelle**, backup **e.cervera**, who acts when Michelle is out (Michelle informs them). Email goes **to** `it.admin@` and the owner, **CC** `j.fernandez@` and `e.cervera@` (same domain as `it.admin@`; recipients from `Dev/.env`, e.g. `EMAIL_RECIPIENTS_*` plus a new `EMAIL_CC`). Teams goes to a **dedicated channel that Michelle sets up**, not a 1:1 chat. Response times: **errors, alert-delivery failures and watchdog alerts** are acted on the next business day; **warnings and unresolved items** within the week, before the patch pass on the 3rd or the watchdog on the 5th. The target is a **channel**, so the spike answers which Graph permission lets the app post to a channel, verified with the microsoft-learn MCP.
- **Code reference** (verified against `Dev/run.py`):
  - `_get_o365_account`: `run.py:1120` — existing Graph connector using O365 library and client credentials flow
- **Scope decision open item 3**: "The connector (credentials in `Dev/.env`) and the Graph calling code (email feature) already exist. To verify with `microsoft-learn` during the S1 story: which Graph permission lets the app post to a Teams channel or chat, since app-only posting has restrictions."
- **Pattern 9** (Spike): exploratory work with a fixed time-box and a decision output, not production code.
- Output location: `Docs/Reference/teams-graph-spike.md`
- The existing O365 library and credentials already work for sending email via Graph (`Mail.Send` delegated or app permission). The question is whether the same connector can post to Teams, and if so, to what target.

## Open questions
- [ ] #open-question Which Graph permission lets the existing app registration post to Michelle's dedicated channel? Answered by this spike; verify with microsoft-learn.

## Dependencies
None — can start immediately.
