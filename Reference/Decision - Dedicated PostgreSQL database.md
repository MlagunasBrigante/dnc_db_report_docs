---
type: decision
project: dnc-cron-scrub
sprint: S2
epic: "[[Epic - PostgreSQL system of record]]"
status: proposed — awaiting Michelle (stories interview Q8)
created: 2026-10-29
tags: [project/dnc-cron-scrub, decision, postgresql]
---
# Decision — Dedicated PostgreSQL database for DNC

> **Proposal:** create a dedicated `dnc` database on the ITServer's existing PostgreSQL server (same server, new database) instead of adding the DNC tables to an existing database.
> **Applies to:** S2 · [[Epic - PostgreSQL system of record]] · [[Story - Raw snapshot and run record land in Postgres]]

## What's the same either way
Both options run on **the same PostgreSQL server** (the same Postgres installation on the ITServer). So these are identical:
- shared CPU, RAM and disk
- one superuser
- one version to upgrade
- point-in-time recovery from the transaction log (WAL), which works for the whole server, not per database

The choice is only about **how separate the DNC data is inside that server.**

## Why a dedicated database is better here

| # | Reason | Dedicated `dnc` database | Tables in the existing database |
|---|---|---|---|
| 1 | **Who can even see the data** (lead PII, legal record) | `REVOKE CONNECT ON DATABASE dnc FROM PUBLIC`: only DNC roles can connect at all | Every role that can connect to the existing DB is one grant away from the DNC tables. You have to audit that app's roles, default privileges and `public` schema rights, and keep them audited forever. |
| 2 | **Backups and restores** | `pg_dump dnc` is exactly the legal record: complete, nothing else in it. Restoring it never touches the other app. | The DNC data lives inside another app's backup. Restoring that app (e.g. after a bad migration) also rolls back the DNC data, and the reverse. Dumping only DNC needs careful `--schema` filters. |
| 3 | **Legal traceability and retention** (closed months read-only, S5 archiving, the retention policy) | Proving "this is everything we hold, and here's how it's protected" is simple: one database, one set of roles, one size figure. | The proof has to exclude everything else in the DB, and the other app's admins become part of the audit scope. |
| 4 | **Blast radius of the other app** | Another app's migration, `DROP`, mistaken `TRUNCATE` or runaway table can't touch DNC objects. | The same DB means the same mistakes reach both, e.g. migration tools that "reset" a database. |
| 5 | **Settings and maintenance** | `ALTER DATABASE dnc SET …` (timezone UTC, statement timeouts) without affecting the other app. VACUUM and partition upkeep on about 400k rows a month are easy to monitor in isolation. | Settings are shared or must be set per role. DNC's monthly bulk loads show up in the other app's performance. |
| 6 | **Naming** | No clashes with the other app's tables, schemas or `search_path` | You need a dedicated schema anyway to avoid clashes |

## What you give up with a dedicated database (honest counterpoints)
- **No direct joins with data in the existing database.** Cross-database queries need `postgres_fdw`/`dblink`. **This is the one case where the existing DB wins:** if S3 or S4 needs to join DNC data with tables that already live there.
- **One more connection string** in `Dev/.env`, and one more database in backups and monitoring. Negligible.

## A middle option
**A dedicated `dnc` schema inside the existing database, with its own roles** covers reasons 4 and 6 and part of 1, if done carefully (revoke `CREATE`/`USAGE` on the schema from `PUBLIC`, separate roles). It does **not** cover 2 and 3: backups, restores and the legal-record scope stay entangled with the other app.

## Recommendation
**Use a dedicated `dnc` database, unless S3/S4 must join DNC data with tables in the existing database.** For DNC, the deciding factors are reasons 1–3: lead PII and a legal record are much easier to protect and prove when they're on their own. The cost is close to zero.

## Proposed layout (for [[Story - Raw snapshot and run record land in Postgres]])
| Item | Role / setting | Purpose |
|---|---|---|
| Database | `dnc` | Dedicated, `REVOKE CONNECT … FROM PUBLIC` |
| `dnc_owner` | Owns the tables, runs the DDL (Michelle) | Table changes only through reviewed scripts |
| `dnc_writer` | The job: insert into stage tables; update only open partitions and `failed_items` | Closed months stay read-only |
| `dnc_reader` | Read-only (S3 statistics, S4 dashboard) | Analysis can't change data |
| `dnc_archive` | Detaches and exports closed partitions (S5) | The only role that can touch closed months |
| Connection | Postgres Unix socket (`/var/run/postgresql`) mounted into the container; password auth, password only in `Dev/.env` | No database port exposed; requires the job and the DB on the same machine (current plan) |

## Open points before deciding
1. **What does the existing database hold, and does S3/S4 need to join with it?** If not, dedicated wins outright.
2. **Does Michelle have admin rights (`CREATEDB` / `CREATEROLE`)** on that PostgreSQL server? Needed for either option's roles.

## Decision log
- 2026-10-29 — Proposed by Claude in the stories interview (Q8). Michelle's initial preference was the existing DB; she asked for this justification note before deciding.
