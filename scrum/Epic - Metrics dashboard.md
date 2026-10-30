---
type: epic
project: dnc-cron-scrub
sprint: S4
status: to-do
gaps: []
---
# Epic: Metrics dashboard

> [!important] Sprint-start ritual
> At the **start of this sprint**, re-run **Prioritize & De-risk** (scrum-scoping Phase 4): MoSCoW the stories (their `moscow` property is `tbd` until then), re-run the pre-mortem and update [[Prioritization and Risk Register]].

> [!tip] Soft, flexible deadlines (risk R16)
> Michelle's biggest risk is getting stuck or overwhelmed in S3–S4. Each milestone/story gets a **soft target date** set at sprint start; deadlines can slide by agreement. **Nudges:** a weekly check-in nudge ("what's the next small step?") and a nudge 3 days before each soft date. If a milestone stalls for 2 weeks, split it into smaller steps rather than push harder.

## Description
A custom web application displaying the S3-validated metrics from PostgreSQL. Built with Streamlit, Dash or Bokeh (tech choice deferred to a spike story). May be hosted on the ITServer or on `autobot` (Ubuntu, Docker + NPM reverse proxy + n8n) — undecided. **Docker is always the runtime.**

Starts only after the S2 migration is **done and stable** (not just complete — at least one scheduled run has passed and the system is operating reliably).

The dashboard reads directly from PostgreSQL (same machine as the process if hosted on ITServer; remote connection if hosted on `autobot`). Metrics include opt-in status, age buckets, DNC→Litigator→not-a-case flows, blocked/invalid numbers, new leads in 30 days, litigators flagged for soft-delete, and tag completion.

## Breakdown

### INVEST check
| Criterion | Rating | Notes |
|---|---|---|
| **Independent** | ✅ | Depends on S2 done+stable and S3 M5 (validated metrics); then self-contained |
| **Negotiable** | ✅ | Stack choice (Streamlit/Dash/Bokeh) and hosting (ITServer/autobot) decided in spike |
| **Valuable** | ✅ | Management and compliance see live metrics without SQL queries |
| **Estimable** | ⚠️ | Spike bounds risk; hosting decision affects complexity |
| **Testable** | ✅ | Dashboard serves data from PostgreSQL, displayed correctly |
| (Small skipped) | | Epic-level check |

### Splitting pattern applied
**Pattern 9: Spike** (for the first story), then **Pattern 6: Major Effort** (for the remaining stories).

- **Story 1 (spike):** time-boxed exploration of Streamlit vs Dash vs Bokoh and ITServer vs `autobot` hosting. Delivers a **recommendation** with trade-offs, not working code. Bounds risk before committing to build.
- **Story 2:** one **complete vertical slice** — single metric, PostgreSQL → backend → frontend, deployed and visible in a browser. Proves the stack works end-to-end.
- **Story 3:** remaining metrics. This is a Major Effort split (one slice done, now scale out), acceptable here because Story 2 de-risks the approach and Story 3 is "more of the same."

**Why earlier patterns didn't fit:**
- Pattern 1 (Workflow Steps): no sequential workflow; all metrics are peers
- Pattern 2 (Operations): no CRUD operations; this is read-only display
- Pattern 3 (Business Rule Variations): metrics differ in *content* (opt-in vs age bucket), not in *logic*
- Pattern 4 (Data Variations): we're not splitting by data source; PostgreSQL is the single source

**Alternative split (if Story 3 feels too large):** split by **persona** or **metric category** (e.g., "compliance metrics" vs "operational metrics" vs "lead-quality metrics"), so each story delivers a coherent view to one stakeholder group.

### Cynefin calibration
**Complicated** (known unknowns, sense-analyze-respond). The stack choice and hosting decision are unknowns, but solvable with a spike. Once decided, building a read-only dashboard from PostgreSQL is a solved problem. Keep the spike time-boxed (3–5 days); if it runs long, escalate the decision to Michelle rather than extending the exploration.

### Split evaluation
Three stories is light for an epic. If the spike reveals significant hosting complexity (e.g., `autobot` networking, SSL, auth), split Story 2 into "dashboard running locally in Docker" + "dashboard deployed and accessible." If Story 3 grows (8+ metrics), split by metric category or add a fourth story for polish (error states, loading indicators, responsive layout).

## Child user stories
- [[Story - Spike dashboard stack and hosting]]
- [[Story - First dashboard page from validated metrics]]
- [[Story - Remaining metrics on the dashboard]]

## Dependencies / sequencing
- **Starts after:** S2 migration done+stable **and** S3 M5 passed (metrics validated)
- **Feeds into:** S5 (archiving old months presumes the dashboard is live and serving current data)
- **Infrastructure decision:** hosting on ITServer vs `autobot` resolved in Story 1
