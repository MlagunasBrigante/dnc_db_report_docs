---
type: epic
project: dnc-cron-scrub
sprint: S1
status: to-do
gaps: [GAP-01, GAP-02, GAP-03, GAP-04, GAP-05, GAP-21]
---
# Epic: Reliable alerts

## Description
Make warning, error and success alerts always arrive without exceeding Microsoft Graph size limits. Today alert emails routinely fail (400/504) because they attach logs or databases exceeding the 3 MB inline limit, so critical failures go unreported. Operators need trustworthy alerts that arrive every time, through redundant channels.

## Gaps closed
- **GAP-01** — Warning and error emails attach the full log → fail (400/504)
- **GAP-02** — Report zip includes the 164 MB database → 47 MB attachment
- **GAP-03** — No size guard or upload-session logic → unverified delivery
- **GAP-04** — Self-healed retries count as warnings → noise
- **GAP-05** — Alert-delivery failures are silent → single channel
- **GAP-21** — No runbook for failed or degraded runs (Jira DNCUP-4)

## Breakdown

### INVEST check

| Criterion | Rating | Notes |
|---|---|---|
| Independent | ✅ | No technical dependency on other S1 epics; Schema, memory and secrets can all run in parallel |
| Negotiable | ✅ | Teams spike (9) separated out; channel order is open |
| Valuable | ✅ | Operator can trust that alerts reach them |
| Estimable | ✅ | MS Graph limits verified with microsoft-learn MCP; log size, zip contents, email failures are all visible in the 10-28 run report |
| Testable | ✅ | Inject a failure and confirm both channels alert; confirm success email arrives in mailbox with right attachment and size |

### Pattern applied
**Pattern 2: Operations** (send success, send warning, send error, send Teams, handle alert-delivery failures)

Earlier patterns rejected:
- Pattern 1 (Workflow Steps): the whole epic is one stage (Stage 6); all work is within the alerting step.

### Cynefin calibration
**Low uncertainty** — Microsoft Graph attachment limits are documented; O365 library and connector exist; logs and zip contents are known from the run report.

### Split evaluation
Each story delivers observable value to the IT & Data lead: emails without oversized attachments arrive, Teams channel exists, any alert-delivery failure raises a visible alarm, runbook exists for who acts on what. Sizes are reasonably equal (1-4 AC each); no pure infrastructure stories.

> [!check] Before closing this epic, verify Michelle's actions were actually done
> - [ ] D-17 (2026-11-10): S2 started with this epic still open; finish its remaining stories in December.
> - [ ] Create the dedicated Teams channel and share its team and channel IDs (needed before this story starts). — [[Story - Teams alert channel]]
> - [ ] If the spike needs admin consent for a Graph permission, request it from j.fernandez (M365 admin). — [[Story - Spike Teams posting through Graph]]
> - [ ] Brief e.cervera that they're the backup, will receive error alerts, and should act when you're out. — [[Story - Runbook for failed or degraded runs]]
> - [ ] Identify our LawRuler account manager (name, address, domain). — [[Story - Soft-deletion request to the LawRuler team]]
> - [ ] Until then, forward each monthly request yourself. — [[Story - Soft-deletion request to the LawRuler team]]
> - [ ] Ask j.fernandez to confirm the tenant allows external sending from `it.admin@` to `jordanbelfort.com`. — [[Story - Monthly report email as a summary table]]
> - [ ] Run the container recreate command on the Mac and set the wake/no-sleep schedule. — [[Story - Mac run-night safeguards until cutover]]

## Child user stories
- [[Story - Mac run-night safeguards until cutover]] (Could; added 2026-10-29)
- [[Story - Warning and error emails always arrive]]
- [[Story - Monthly report email as a summary table]]
- [[Story - Soft-deletion request to the LawRuler team]]
- [[Story - Spike Teams posting through Graph]]
- [[Story - Teams alert channel]]
- [[Story - Alert delivery failure raises an alert]]
- [[Story - Runbook for failed or degraded runs]]

## Dependencies / sequencing
The Teams spike must complete before the Teams alert channel story. All other stories are independent. The runbook story can start any time.
