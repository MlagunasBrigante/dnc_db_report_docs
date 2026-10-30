---
type: epic
project: dnc-cron-scrub
sprint: S2
status: to-do
gaps: [GAP-12]
---
# Epic: Reproducible scheduling on the ITServer
## Description
Replace the in-container scheduler (whose next-run state lives only in memory) with systemd timers versioned in Dev/deploy/. Three units: monthly run on the 1st, patch pass on the 3rd, watchdog on the 5th. Persistent=true ensures a missed run fires on boot. An idempotent install script makes the setup reproducible — clone the repo, run install.sh, and the job is scheduled.

## Gaps closed
- **GAP-12**: Scheduler keeps the next run only in memory, so a restart can silently skip a month — systemd timers with Persistent=true survive reboots and container rebuilds

## Breakdown

### INVEST check
> [!question]+
> Why is the `Rating` column missing?
>
> ✅ *Applied 2026-10-29: table now uses the same `Criterion | Rating | Notes` format as the S1 epics; "Small" dropped because the method skips it at epic level.*

| Criterion | Rating | Notes |
|---|---|---|
| Independent | ⚠️ | Depends on Epic - Failures captured and patched (patch pass must exist) but otherwise standalone |
| Negotiable | ✅ | Could deploy without the watchdog initially; the 1st + 3rd timers are the core |
| Valuable | ✅ | Michelle knows the job will run even if the server reboots, and the watchdog catches stuck/missing runs |
| Estimable | ✅ | 3 stories, each is one .service + .timer pair + install script update |
| Testable | ✅ | Verify by checking timer list, simulating a reboot (Persistent triggers), deliberately missing a run (watchdog fires) |

### Splitting pattern applied
**Pattern 2: Operations** — three independent scheduled operations (main run, patch, watchdog), each with its own timer and verification path.

Earlier patterns rejected:
- Pattern 1 (Workflow Steps) — these aren't workflow steps of one process; they're separate scheduled jobs
- Pattern 6 (Major Effort) — the effort is small and uniform; no need for incremental delivery

### Cynefin calibration
**Complicated** — systemd timer syntax and semantics are well-documented; the challenge is ensuring the Docker container start, the volume mounts, and the install script's idempotency all work together. One unknown: how Persistent=true behaves if the timer's calendar doesn't align (e.g. server down on the 1st, boots on the 10th).
> [!note]+
> I think the server already has a re-run-all-containers policy in case it reboots. In case the server reboots, we could either setup a soft alert message that tell us the dnc container is up again, or a global runner that **ensures** all containers do get running, in particular the DNC container. If the original proposal than what I just said, we'll go with it.
>
> ✅ *Decision 2026-10-29: keep the original proposal (systemd timer + `Persistent=true` + watchdog).* Implications compared:
> - **The server's restart-all-containers policy** keeps a *long-lived* container alive, but that is exactly today's design. A restart re-anchors the in-container schedule, so a month can still be skipped silently (GAP-12). It answers "is the container up?", not "did this month's run happen?".
> - **A "DNC container is up again" soft alert** reports restarts but can't tell you a run was missed. You'd still need the watchdog.
> - **A global runner that ensures all containers are running** is a new server-wide component to build and maintain, and it has the same blind spot.
> - **Original:** nothing long-lived. On the 1st, systemd starts a one-shot `docker run --rm`. If the server was down then, `Persistent=true` starts the missed run right after boot. The day-5 watchdog independently checks that the run happened. It doesn't depend on the server's container policy, and it stays Docker. Cost: unit files + install script in `Dev/deploy/`, plus root once to install.

### Split evaluation
- **Does it reveal low-value work?** The watchdog (Story 3) is lower priority than the main run + patch, but it closes the observability gap ("is the job even running?"), so it's justified.
- **Are the sizes roughly equal?** All three are Small; the main run story is slightly larger (install script scaffolding) but still 1–2 days.

## Child user stories
- [[Story - Monthly run from versioned systemd units]]
- [[Story - Scheduled patch pass on the 3rd]]
- [[Story - Watchdog on the 5th]]

## Dependencies / sequencing
- **Upstream:** Epic - Failures captured and patched must deliver Story - Patch pass before Story 2 (scheduled patch pass) can be built.
- **Internal:** Story 1 (main run + install script) comes first; 2 and 3 are independent after that.
- **Downstream:** Epic - Cutover to the ITServer needs this epic done, so the ITServer scheduling is already proven.
