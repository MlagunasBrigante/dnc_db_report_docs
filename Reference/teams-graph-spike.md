---
type: reference
project: dnc-cron-scrub
story: "[[Story - Spike Teams posting through Graph]]"
date: 2026-11-05
status: in-review
---
# Spike: posting alerts to Teams

## Question
Can the pipeline post alerts to a Teams channel using the existing app registration (`O365_CLIENT_ID` / `_SECRET` / `_TENANT_ID`, client-credentials flow, already used for email)? If so, which permission and which target?

## Findings (verified with Microsoft Learn, 2026-11-05)
1. **Graph cannot post live channel or chat messages with an app-only token.** [Send chatMessage in channel](https://learn.microsoft.com/graph/api/channel-post-messages?view=graph-rest-1.0) and [Send message in a chat](https://learn.microsoft.com/graph/api/chat-post-messages?view=graph-rest-1.0) list `ChannelMessage.Send` / `ChatMessage.Send` as **delegated** only. The only application permission, `Teamwork.Migrate.All`, is "only supported for migration". The same pages say it's against the terms of use to use Teams as a log file: only send messages people will read.
   → Our client-credentials app can't post to Teams through Graph, whatever permission we grant it.
2. **Office 365 Connectors (the old "Incoming Webhook") were retired in May 2026.** Microsoft's replacement is the **Workflows app** in Teams: the *Send webhook alerts to a channel* template, or a workflow with the **"When a Teams webhook request is received"** trigger. It gives a webhook URL; an HTTP POST with an **Adaptive Card** (or MessageCard) payload posts to the chosen channel ([Create Incoming Webhooks → Workflows](https://learn.microsoft.com/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook#create-webhooks-using-workflows)).
3. Other routes: a **bot** (Azure Bot Framework, proactive messages; the bot must be added to the channel), or a **delegated** token from a service account with `ChannelMessage.Send`. Both are heavier than this pipeline needs.

## Options
| Option | What's needed | Implications |
|---|---|---|
| **A. Workflows webhook** (recommended) | Michelle creates the workflow in the dedicated channel (Channel → … → Workflows → *Send webhook alerts to a channel*) and puts the URL in `Dev/.env` as `TEAMS_WEBHOOK_URL` | No Graph permission change and no admin consent. The code is one `requests.post` with an Adaptive Card. **The URL is a secret**: anyone with it can post, so `.env` only, masked in logs. The workflow runs under the creator's Power Automate context, so if that account is disabled the webhook stops; the alert-delivery-failure story catches this. |
| B. Bot | App registration as a bot, bot added to the team | Most robust identity, but far more setup and maintenance for one alert stream |
| C. Delegated service account | A licensed user plus a stored refresh token (or username/password, which we don't want) | Token refresh, MFA and credential-rotation burden; conflicts with keeping secrets minimal |
| D. Email only | Nothing | Drops Teams; *Alert delivery failure raises an alert* would then need another second channel |

## Recommendation
**Option A.** Store the URL as `TEAMS_WEBHOOK_URL` in `Dev/.env`. Post an Adaptive Card (status, report date, counts, failed items, run folder) with the shared retry policy, timeout 30 s. **Never fail the run** if Teams fails; an email reports that Teams failed. In dry-run, post only when `DRYRUN_TEAMS_WEBHOOK_URL` is set (a separate test channel or workflow), otherwise skip, matching the email dry-run rule.

> [!todo] Michelle — action needed
> - [ ] Create the dedicated channel, then in that channel: **… → Workflows → "Send webhook alerts to a channel"** → save → copy the webhook URL into `Dev/.env` as `TEAMS_WEBHOOK_URL`. Use a **standard** channel: workflow posting to private channels has had limitations.
> - [ ] Optional: a second workflow in a test channel for dry-runs, saved as `DRYRUN_TEAMS_WEBHOOK_URL`.
> - [ ] Ask j.fernandez whether tenant policy allows the Workflows app or Power Automate for your account. *Unverified: whether this trigger needs a premium Power Automate licence in our tenant.*

## Not verified
- Licensing of the Workflows webhook trigger in this tenant (see the todo above).
- Adaptive Card size limits for our payload. Keep cards to a summary plus at most 20 failed-item keys.
