---
type: reference
project: dnc-cron-scrub
epic: "[[Epic - PostgreSQL system of record]]"
sprint: S2
status: draft
created: 2026-11-11
updated: 2026-11-19
tags: [project/dnc-cron-scrub, reference, postgresql, itserver, setup, docker]
---
# ITServer PostgreSQL 17 Setup for DNC Pipeline (Docker)

> **Purpose:** Step-by-step guide for `<owner>` to set up PostgreSQL 17 in its own Docker container on the ITServer, create the dedicated `dnc` database, roles, schemas, and tables. Run this once during the S2 migration, before the first pipeline execution with `POSTGRES_DSN` set.

---

## Prerequisites

**ITServer facts (2026-11-13, updated 2026-11-19):**
- RAM: 12 GiB total (~10 GiB available, mostly page cache), 5.8 GiB swap (1.0 GiB used)
- Storage: `/` 70 GB (29 GB free, holds Docker images), `/home` 224 GB (221 GB free)
- Docker Engine 29.3.0, Docker Compose v5.1.0, daemon reachable as `it.admin`
- Existing PostgreSQL 13.23 server (socket at `/var/run/postgresql`, port 5432) — **not used by the DNC pipeline**
- The `schema/` DDL files from the code repo clone on `main` (after the PostgreSQL 17 composite-key merge and the GAP-34 streaming-checksum merge)
- Git 2.52.0 installed on the host (verified 2026-11-19); `it.admin` has sudo

**What you'll set up:**
- PostgreSQL **17.12** (pinned minor version; current as of 2026-11-17 per https://www.postgresql.org/support/versioning/) in its own Docker container
- Data directory on `/home` (221 GB free, not on `/` with only 29 GB free)
- Server socket shared through Docker named volume `dnc-pgsocket` (not the host's `/var/run/postgresql`, which belongs to the existing PG13 server)
- TCP disabled (`listen_addresses=''`), no port published → no collision with the existing PG13 server on port 5432
- Superuser password from file (never `-e POSTGRES_PASSWORD`)
- Local (socket) authentication via scram-sha-256 (not trust)
- Memory tuning: shared_buffers=512MB, maintenance_work_mem=256MB, work_mem=16MB, max_connections=20
- Estimated storage per stored run: ~0.3–0.5 GB; ~4–6 GB/year (not measured, estimate)

**Checklist**:
- [x] Step 1: Prepare host directories
- [x] Step 2: Create the named volume for the socket
- [x] Step 3: Run the PostgreSQL 17.12 server container
- [x] Step 4: Get the code onto the ITServer
- [x] Step 5: Copy the DDL into the container
- [x] Step 6: Create the database and roles
- [ ] Step 7: Set passwords for the roles
- [ ] Step 8: Create the schema (dnc and dnctest)
- [ ] Step 9: Verify the schema structure
- [ ] Step 10: Configure the pipeline's .env on the ITServer
- [ ] Step 11: Run one dry-run of the pipeline
- [ ] Step 12: Verify the raw snapshot checksum
---

## Step 1: Prepare host directories

Create the data directory on `/home` (221 GB free) and a secrets directory:

```bash
# Data on /home (plenty of space)
mkdir -p <pg-data-dir>

# Secrets directory
mkdir -p <pg-secrets-dir>
chmod 700 <pg-secrets-dir>

# Create the password file without the password touching shell history
install -m 600 /dev/null <pg-secrets-dir>/postgres_superuser_password

# Edit it with an editor (nano, vi, etc.) and save a strong password
nano <pg-secrets-dir>/postgres_superuser_password
```

**What `install -m 600 /dev/null` does:** Copies the always-empty `/dev/null`, creating an empty file already restricted to owner read/write (600) in one step — no window with default permissions (unlike create-then-chmod), and the password never touches shell history because it is typed in the editor afterwards (unlike `echo pw > file`). The container mounts it read-only and `POSTGRES_PASSWORD_FILE` passes only the path, never the secret. The image reads it only on first initialisation. If first start fails reading `/run/secrets/pg_superuser`, check SELinux (see SELinux note below) and then file ownership.

Replace `<pg-data-dir>` and `<pg-secrets-dir>` with your chosen paths on `/home` (e.g., `/home/it.admin/dnc-postgres-data` and `/home/it.admin/dnc-postgres-secrets`). These are placeholders; exact paths are your decision.

---

## SELinux note

Run `getenforce`. If `Enforcing`, check whether Docker applies SELinux labels with:
```bash
docker info --format '{{.SecurityOptions}}'
```

If the output includes `name=selinux`, a container may be denied reading/writing bind-mounted host folders even with correct permissions. Add `:z` to bind mounts of dedicated folders (e.g., `-v <pg-secrets-dir>/postgres_superuser_password:/run/secrets/pg_superuser:ro,z`, `-v <pg-data-dir>:/var/lib/postgresql/data:z`, `-v <clone>:...:z`). Never use `:z`/`:Z` on all of `/home/it.admin` (relabels everything in it).

**Verified 2026-11-19:** `getenforce` = Enforcing, but `docker info` output `[name=seccomp,profile=builtin name=cgroupns]` (no selinux listed), so Docker on this host does not apply SELinux labels; `:z` is not needed here. The check above is kept for other hosts.

---

## Step 2: Create the named volume for the socket

```bash
docker volume create dnc-pgsocket
```

This volume will hold the PostgreSQL Unix socket (`/var/run/postgresql` inside the server container). The pipeline container will mount this same volume to connect without TCP.

---

## Step 3: Run the PostgreSQL 17.12 server container

**Image defaults verified on 2026-11-17** from the [official image docs](https://github.com/docker-library/docs/blob/master/postgres/README.md) and [Docker Hub](https://hub.docker.com/_/postgres):

- **Data path:** `/var/lib/postgresql/data` (PG 17 and below; PG 18+ changed to `/var/lib/postgresql/18/docker`)
- **Default local authentication:** `trust` (any container mounting the socket can connect without a password), so we override with `--auth-local=scram-sha-256` via `POSTGRES_INITDB_ARGS`
- **POSTGRES_PASSWORD_FILE:** Supported
- **`-c` configuration flags:** Passed after the image name, applied to the postgres server

**Quick check** (optional): After `docker pull postgres:17.12`, verify the version with:
```bash
docker run --rm postgres:17.12 postgres --version
# Expected: postgres (PostgreSQL) 17.12
```

```bash
docker run -d --name dnc-postgres --restart unless-stopped \
  --shm-size=256m \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/pg_superuser \
  -e POSTGRES_INITDB_ARGS="--auth-local=scram-sha-256" \
  -v <pg-secrets-dir>/postgres_superuser_password:/run/secrets/pg_superuser:ro \
  -v <pg-data-dir>:/var/lib/postgresql/data \
  -v dnc-pgsocket:/var/run/postgresql \
  postgres:17.12 \
  -c listen_addresses='' \
  -c shared_buffers=512MB \
  -c maintenance_work_mem=256MB \
  -c work_mem=16MB \
  -c max_connections=20
```

**What this does:**
- `--shm-size=256m`: Shared memory for Postgres (default Docker `/dev/shm` is 64 MB)
- `-e POSTGRES_PASSWORD_FILE`: Reads the superuser password from the mounted secret file
- `-e POSTGRES_INITDB_ARGS="--auth-local=scram-sha-256"`: Forces local socket auth to require a password (not `trust`)
- `-v <pg-secrets-dir>/...:/run/secrets/pg_superuser:ro`: Mounts the password file read-only
- `-v <pg-data-dir>:/var/lib/postgresql/data`: Persists database data on the host
- `-v dnc-pgsocket:/var/run/postgresql`: Shares the socket with other containers
- `postgres:17.12`: Pinned to PG 17.12 (current minor as of 2026-11-17)
- `-c listen_addresses=''`: Disables TCP, only Unix socket connections allowed
- Other `-c` flags: Memory tuning (see Implementation calls, 2026-11-13)

**Check the logs:**

```bash
docker logs dnc-postgres
```

You should see:
```
PostgreSQL init process complete; ready for start up.
...
LOG:  database system is ready to accept connections
```

**Verify it's running:**

```bash
docker exec -it dnc-postgres psql -U postgres -c 'SELECT version();'
```

Enter the superuser password when prompted. Expected: `PostgreSQL 17.12 on ...`

---

## Step 4: Get the code onto the ITServer

**Verified 2026-11-19:** Git 2.52.0 is installed on the host; `it.admin` has sudo. Use host git directly (simpler, persistent config). The alpine/git container method is kept below as a fallback for servers without git.

**Folder:**

```bash
mkdir -p /home/it.admin/dnc/git_container && chmod 700 /home/it.admin/dnc/git_container
```

(On `/home`: `/` has only 29 GB free.)

### Method A: Host git (primary, verified 2026-11-19)

**Install git** (if not present):
```bash
git --version    # check first
sudo dnf install -y git    # AlmaLinux/RHEL; Ubuntu/Debian: sudo apt install -y git
```

**Authentication:** GitHub rejects account passwords for git (`remote: Invalid username or token. Password authentication is not supported for Git operations.`). Three options (two SSH, one HTTPS):

**Option 1 — read-only deploy key over SSH** (recommended long-term: one repo, read-only, not tied to a person's account, no enforced expiry; needs repo admin to add)

Generate key (no passphrase so unattended pulls work; risk bounded: read-only, one repo):
```bash
mkdir -p ~/.ssh-dnc && chmod 700 ~/.ssh-dnc
ssh-keygen -t ed25519 -N "" -C "itserver dnc_db_report deploy" -f ~/.ssh-dnc/id_ed25519
cat ~/.ssh-dnc/id_ed25519.pub
```

GitHub: repo Settings → Deploy keys → Add deploy key, paste the .pub line, leave 'Allow write access' UNCHECKED.

Clone:
```bash
cd /home/it.admin/dnc/git_container
GIT_SSH_COMMAND="ssh -i /home/it.admin/.ssh-dnc/id_ed25519 -o IdentitiesOnly=yes -o UserKnownHostsFile=/home/it.admin/.ssh-dnc/known_hosts" \
  git clone git@github.com:BriganteLaw/dnc_db_report.git
cd dnc_db_report
git config core.sshCommand "ssh -i /home/it.admin/.ssh-dnc/id_ed25519 -o IdentitiesOnly=yes -o UserKnownHostsFile=/home/it.admin/.ssh-dnc/known_hosts"
```

The `git config core.sshCommand` persists in this clone's `.git/config` (per-clone, not global), so later `git pull` inside this folder works without `GIT_SSH_COMMAND`.

**Option 2 — personal SSH key with passphrase, temporary** (in use on the ITServer 2026-11-18–2026-12-03)

- **Broad access:** grants read+write to every repo the account can reach; passphrase is typed at each clone/pull
- **Manual expiry:** GitHub account SSH keys have no expiry date — set a calendar reminder for **2026-12-03**, then delete the key on GitHub (Settings → SSH and GPG keys → Delete) and switch the server to the deploy key (see *Switching from Option 2 to Option 1* below)
- Use this only for a short trial period (e.g. 2–3 weeks); switch to Option 1 before the systemd-units story

Generate key (NOTE: passphrase is prompted, do NOT add `-N ""`):
```bash
mkdir -p ~/.ssh-dnc && chmod 700 ~/.ssh-dnc
ssh-keygen -t ed25519 -C "itserver dnc TEMP - remove 2026-12-03" -f ~/.ssh-dnc/id_ed25519
cat ~/.ssh-dnc/id_ed25519.pub
```

GitHub: your account Settings → SSH and GPG keys → New SSH key, type 'Authentication key', title `ITServer dnc TEMP – remove 2026-12-03`. If a 'Configure SSO' button appears next to the key, authorise BriganteLaw.

Clone: **same commands as Option 1** (the same SSH flags and git config line).

**To remove the temporary key on 2026-12-03:**
1. GitHub: Settings → SSH and GPG keys → find the key → Delete
2. Server: if the deploy key is already in `~/.ssh-dnc`, nothing else to do (it overwrote the personal key). Only if you are NOT switching to a deploy key: `rm -rf ~/.ssh-dnc` (it would also delete a deploy key stored there).

**Verified 2026-11-19:** Personal SSH key (Option 2) in use; `~/.ssh-dnc/id_ed25519` with passphrase, temporary until 2026-12-03, then replaced by a read-only deploy key (Option 1) in the same folder. The existing clone works with host git as-is. Pull:
```bash
cd /home/it.admin/dnc/git_container/dnc_db_report
git pull
```
Prompts for the passphrase; output: `Already up to date.` (or the merge/fast-forward line). Remote: `origin git@github.com:BriganteLaw/dnc_db_report.git`.

**Pitfall (verified 2026-11-19):** The personal key has WRITE access, so accidental pushes/branch-deletes succeed (e.g. `git push origin --delete <branch>` worked). The deploy key (read-only) will make such mistakes fail. The server only pulls; never push/branch-delete from it.

**Switching from Option 2 to Option 1 later:** No remote change; replace the key files in `~/.ssh-dnc` with the deploy key (re-run the Option 1 keygen, overwrite `id_ed25519` and `.pub`), add the new `.pub` as a deploy key on the repo, then delete the personal key on GitHub. The `core.sshCommand` setting stays.

**Option 3 — HTTPS + fine-grained personal access token** (fast for initial setup; prompts on every pull)

- Username = GitHub username (not an email)
- Password prompt = token, **Resource owner = BriganteLaw** (not your personal account), restricted to repo `BriganteLaw/dnc_db_report` only, permission 'Contents: Read-only', short custom expiry (e.g. 15 days; GitHub allows up to a year unless the org sets a lower limit); the org may need to approve it (see Troubleshooting for the 403 error)
- Typed at the prompt only: never put it in the URL (`https://user:token@...` saves it in plain text in `.git/config`) and never set `credential.helper store` (plain text in `~/.git-credentials`)
- Nothing is saved, so each pull prompts again

Clone:
```bash
cd /home/it.admin/dnc/git_container
git clone https://github.com/BriganteLaw/dnc_db_report.git
```

Pull:
```bash
cd /home/it.admin/dnc/git_container/dnc_db_report
git pull
```

**Check the clone** (before continuing; local read only, no GitHub contact, no passphrase):
```bash
cd /home/it.admin/dnc/git_container/dnc_db_report
git log --oneline -5
```

Expected (top four commits, 2026-11-19):
```
b35a022 Fix bug: cannot add UUID + str in Postgres writer
967308d GAP-34: shared streaming checksum, streaming verify tool, sql.Identifier for schema names
04d42b0 Updated for Postgres 17
d6b7454 Fix skip-if-stored rollback and SECURITY DEFINER pg_temp
```

If b35a022 (or a later commit) is missing, push `main` from the Mac, then pull on the server.

**GitHub `main` is protected (PR-only).** Code changes land via PR merge commit; the server then `git pull`s (fast-forward). After a PR merge the server HEAD SHA is the merge commit (e.g. b35a022), not the branch commit.

**Placeholder:** `<clone>` = `/home/it.admin/dnc/git_container/dnc_db_report` (verified path; final location decided in the systemd-units story; any folder on `/home` works for setup).

---

### Method B: alpine/git container (fallback for servers without git)

**Do NOT install git inside `dnc-postgres`** (lost when that container is recreated, e.g. on minor updates).

**Pitfall:** The `-v` host side must be an absolute path. A bare name like `-v git_container:/work` makes Docker create a root-owned NAMED VOLUME instead, and git fails with `fatal: could not create work tree dir 'dnc_db_report': Permission denied`. Cleanup: `docker volume rm git_container`.

**Authentication:** GitHub rejects passwords. HTTPS tokens and SSH deploy keys work (see Method A for key generation and GitHub setup). Clone and pull commands:

**HTTPS clone:**
```bash
docker run --rm -it \
  --user "$(id -u):$(id -g)" -e HOME=/tmp \
  -v /home/it.admin/dnc/git_container:/work -w /work \
  alpine/git clone https://github.com/BriganteLaw/dnc_db_report.git
```

**SSH clone:**
```bash
docker run --rm -it \
  --user "$(id -u):$(id -g)" -e HOME=/tmp \
  -v /etc/passwd:/etc/passwd:ro \
  -v "$HOME/.ssh-dnc":/keys \
  -v /home/it.admin/dnc/git_container:/work -w /work \
  -e GIT_SSH_COMMAND="ssh -i /keys/id_ed25519 -o UserKnownHostsFile=/keys/known_hosts -o StrictHostKeyChecking=accept-new" \
  alpine/git clone git@github.com:BriganteLaw/dnc_db_report.git
```

**HTTPS pull:**
```bash
docker run --rm -it --user "$(id -u):$(id -g)" -e HOME=/tmp \
  -v /home/it.admin/dnc/git_container:/work -w /work/dnc_db_report \
  alpine/git pull
```

**SSH pull:**
```bash
docker run --rm -it \
  --user "$(id -u):$(id -g)" -e HOME=/tmp \
  -v /etc/passwd:/etc/passwd:ro \
  -v "$HOME/.ssh-dnc":/keys \
  -v /home/it.admin/dnc/git_container:/work -w /work/dnc_db_report \
  -e GIT_SSH_COMMAND="ssh -i /keys/id_ed25519 -o UserKnownHostsFile=/keys/known_hosts -o StrictHostKeyChecking=accept-new" \
  alpine/git pull
```

**Check the clone:**
```bash
docker run --rm --user "$(id -u):$(id -g)" -e HOME=/tmp \
  -v /home/it.admin/dnc/git_container:/work -w /work/dnc_db_report \
  alpine/git log --oneline -5
```

Expected: same commits as Method A (b35a022, 967308d, 04d42b0, d6b7454).

**Why these flags (SSH clones):**
- `/etc/passwd` mount avoids 'No user exists for uid 1000'
- `known_hosts` persists GitHub's host key after first use
- If port 22 times out, use URL `ssh://git@ssh.github.com:443/BriganteLaw/dnc_db_report.git`

**Comparison:**

| Option | Enforced expiry? | Access granted | Org approval? |
|--------|------------------|----------------|---------------|
| 1 (deploy key) | No (manual) | Read-only, one repo | No (repo admin adds it) |
| 2 (personal SSH key) | No (manual) | Read+write, all repos | No (SSO auth if button appears) |
| 3 (fine-grained token) | Yes (custom, e.g. 15d) | Read-only, one repo | If the org requires it (token shows "pending") |

**In use on the ITServer since 2026-11-18:** Option 2 (passphrase-protected personal SSH key, remove by 2026-12-03). Switch to Option 1 before the systemd-units story.

The server only pulls, never commits, so no identity is recorded; files on disk belong to Linux user `it.admin` regardless. Switch to Option 1 (or `it.admin`'s own credentials — that needs a GitHub account with repo access) before the systemd-units story, when the server pulls unattended; a personal token or personal SSH key stops working when it expires or the person's access is removed.

---

## Step 5: Copy the DDL into the container

```bash
docker cp <clone>/schema dnc-postgres:/tmp/schema
```

Replace `<clone>` with your repository clone path (defined in Step 4).

**Expected output (verified 2026-11-18):** `Successfully copied 25.6kB to dnc-postgres:/tmp/schema`

---

## Step 6: Create the database and roles

Run `000_roles_and_database.sql` as the `postgres` superuser:

```bash
docker exec -it dnc-postgres psql -U postgres \
  -v ON_ERROR_STOP=1 \
  -f /tmp/schema/000_roles_and_database.sql
```

**Never run with `-1` or `--single-transaction`** (CREATE DATABASE cannot run inside a transaction).

This creates:
- Four roles: `dnc_owner`, `dnc_writer`, `dnc_reader`, `dnc_archive` (all LOGIN-capable, no passwords set yet)
- The `dnc` database, owned by `dnc_owner`
- `REVOKE CONNECT ... FROM PUBLIC` so only the four DNC roles can connect
- `GRANT CONNECT` to the four roles

The script is **safe to re-run** (idempotent).

**Expected output (verified 2026-11-18):** Prompts `Password for user postgres:` (the superuser password from Step 1). Output: four `DO` (the four roles), `CREATE DATABASE`, `You are now connected to database "dnc" as user "postgres".`, `REVOKE`, `GRANT`; no `ERROR`. Local socket auth uses scram-sha-256, so the password prompt itself proves `POSTGRES_INITDB_ARGS` worked. On re-run the database is skipped without error.

---

## Step 7: Set passwords for the roles

**Do NOT set passwords in the SQL files** (they would be committed to version control). Use `\password` interactively:

```bash
docker exec -it dnc-postgres psql -U postgres -c '\password dnc_owner'
docker exec -it dnc-postgres psql -U postgres -c '\password dnc_writer'
docker exec -it dnc-postgres psql -U postgres -c '\password dnc_reader'
docker exec -it dnc-postgres psql -U postgres -c '\password dnc_archive'
```

Each command prompts three times: `Password for user postgres:` (superuser), `Enter new password for user "dnc_...":`, `Enter it again:`. Success prints nothing.

**Before running:** Create four strong passwords in your password manager (one per role). The `dnc_writer` password is needed again in Step 10.

**Optional alternative (one superuser login instead of four):**
```bash
docker exec -it dnc-postgres psql -U postgres
# Then inside psql:
\password dnc_owner
\password dnc_writer
\password dnc_reader
\password dnc_archive
\q
```

**Check passwords are set (without revealing them):**
```bash
docker exec -it dnc-postgres psql -U postgres -c \
  "SELECT rolname, rolpassword IS NOT NULL AS has_password FROM pg_authid WHERE rolname LIKE 'dnc\_%' ORDER BY 1;"
```
Expected: four rows, all with `has_password | t`.

- **`dnc_owner`**: for `<owner>` (DDL changes, schema migrations)
- **`dnc_writer`**: for the pipeline container (write runs + snapshots)
- **`dnc_reader`**: for read-only analysis, S3 statistics, S4 dashboard
- **`dnc_archive`**: for S5 archiving (detach/export closed partitions)

---

## Step 8: Create the schemas (dnc and dnc_test)

Run `001_raw_snapshot.sql` as `dnc_owner` for **both schemas**:

```bash
# Production schema (dnc)
docker exec -it dnc-postgres psql -U dnc_owner -d dnc \
  -v ON_ERROR_STOP=1 \
  -v schema=dnc \
  -f /tmp/schema/001_raw_snapshot.sql

# Test schema (dnc_test, used by dry-runs)
docker exec -it dnc-postgres psql -U dnc_owner -d dnc \
  -v ON_ERROR_STOP=1 \
  -v schema=dnc_test \
  -f /tmp/schema/001_raw_snapshot.sql
```

This creates:
- Schema `dnc` / `dnc_test`
- Table `pipeline_run` (run metadata, PK = run_id UUID)
- Table `raw_snapshot` (partitioned by `snapshot_month`, composite PK `(snapshot_month, run_id, lead_id, phone)` per D-24)
- Function `ensure_raw_snapshot_partition(month date)` (SECURITY DEFINER, allows `dnc_writer` to create partitions without CREATE privilege)
- Grants for `dnc_writer`, `dnc_reader`, `dnc_archive`
- Foreign key `raw_snapshot.run_id → pipeline_run` without cascade (D-22)

**Expected output per schema (verified 2026-11-18):** Prompts for the **dnc_owner** password (not postgres), which also proves that password. Output: `CREATE SCHEMA`, `SET`, `CREATE TABLE`, 6×`COMMENT`, `CREATE TABLE`, 6×`COMMENT`, `DO`, `CREATE FUNCTION`, `COMMENT`, `ALTER TABLE`, `ALTER TABLE`, `ALTER FUNCTION`, 8×`GRANT`; no `ERROR`. Safe to re-run.

---

## Step 9: Verify the schema structure

Run the verification script as `dnc_reader`:

```bash
docker exec -it dnc-postgres psql -U dnc_reader -d dnc \
  -P pager=off \
  -v schema=dnc \
  -f /tmp/schema/verify_raw_snapshot.sql

docker exec -it dnc-postgres psql -U dnc_reader -d dnc \
  -P pager=off \
  -v schema=dnc_test \
  -f /tmp/schema/verify_raw_snapshot.sql
```

Prompts for the dnc_reader password. **Verified structure (2026-11-18):**

- **`pipeline_run`:** 12 columns (run_id, trigger, is_dry_run, started_at, finished_at, status, code_version, report_date, snapshot_month, raw_row_count, raw_snapshot_checksum, stage1_incomplete), PK `pipeline_run_pkey PRIMARY KEY (run_id)`, referenced by FK `fk_raw_snapshot_run`
- **`raw_snapshot`:** Partitioned table, 6 columns (run_id, snapshot_month, lead_id, phone, payload, created_at), `lead_id` and `phone` show Collation `C`, PK `raw_snapshot_pkey PRIMARY KEY, btree (snapshot_month, run_id, lead_id, phone)`, Partition key `LIST (snapshot_month)`, Number of partitions: 0, FK `fk_raw_snapshot_run`
- **Function `ensure_raw_snapshot_partition`:** owner `dnc_owner`, Security `definer`, access privileges `=X/dnc_owner` (PUBLIC EXECUTE), `dnc_owner=X/dnc_owner`, `dnc_writer=X/dnc_owner`. (`=X` = PUBLIC EXECUTE is Postgres's default for functions; harmless because only the four DNC roles can connect to `dnc`.)
- **Before first dry-run:** partitions / run-count / uniqueness / sample queries all `(0 rows)`. The tables are empty so this output contains no PII and may be shared.
- **After Step 11:** the sample query shows lead data — do not paste it.

**Keep `-P pager=off`:** without it, the wide `\d+`/`\df+` sections and the run-count table print nothing (seen 2026-11-18: psql hands wide output to a pager that drops it silently inside the container).

**Note:** Once runs exist, the sample query in `verify_raw_snapshot.sql` shows PII (lead IDs, phones). From then on, do not paste its output anywhere.

---

## Step 10: Configure the pipeline's `.env` on the ITServer

The ITServer `.env` is a NEW file you create there with `chmod 600` (start from `<clone>/.env.example`), never a copy of the Mac's `.env`, never committed. **Credentials decision D-26 (2026-11-19, `<owner>`):** the ITServer `.env` uses the same LawRuler/O365/dncscrub accounts as the Mac for now, typed by hand from the password manager into a new file, never a copy of the Mac's `.env`, never committed. R2/D-20 stay open (LawRuler user decision end of S2); any later rotation must update both Mac and ITServer.

**Create the file:**

```bash
cd <clone>
install -m 600 .env.example .env
nano .env
```

**What to fill:**

| Key | Required? | Notes |
|-----|-----------|-------|
| LawRuler keys (8 total) | **Required, even in dry-run** | auth_url, law_ruler_client_id, law_ruler_client_secret, law_ruler_username, law_ruler_password, CUSTOM_REPORT_URL, LAWRULER_URL, LAW_RULER_ACCOUNT_KEY. Dry-run still pulls real leads from LawRuler (Stage 1). |
| O365_CLIENT_ID, O365_CLIENT_SECRET, O365_TENANT_ID | Optional | Set all three or none. If ANY O365_* key is set, email is enabled and all three are required. If none are set, email is off. |
| SCRUB_URL, LOGIN_ID | Optional for dry-run | Dry-run uses synthetic scrubbing (Stage 2 faked, no paid calls even if set). |
| POSTGRES_DSN | **Required for this guide** (unset = the pipeline runs but skips Postgres, logging `Postgres writing disabled (POSTGRES_DSN not set)`) | `postgresql://dnc_writer:<dnc_writer password>@/dnc?host=/var/run/postgresql` (replace `<dnc_writer password>` with the password set in Step 7) |
| POSTGRES_SCHEMA | Optional (default `dnc`) | `dnc` (dry-runs switch to `dnc_test` automatically) |
| DRYRUN_EMAIL_RECIPIENTS | Optional | In dry-run, emails go only to these addresses; if empty they're logged only. |
| DRYRUN_TEAMS_WEBHOOK_URL | Optional | In dry-run, Teams posts only to this webhook (test channel), never production. |

**Format rules:**
- `KEY=value`, no quotes (docker --env-file keeps quotes literally), no spaces around `=`, no `export`
- Key must start at column 1 (no leading `#` or space — a commented or indented key is silently ignored)
- The dnc_writer password inside the DSN breaks the URL if it contains `@ : / ? # % &` or a space: simplest is letters+digits only, 30+ chars; reset with `\password dnc_writer` (Step 7) if needed. The pipeline masks the DSN password in logs.

**Check without showing values:**

```bash
ls -l .env
awk -F= '/^[A-Za-z_][A-Za-z0-9_]*=/{print $1, ($2==""?"(empty)":"set")}' .env
```

**Expected:** `-rw-------` owner it.admin; the 8 LawRuler keys, POSTGRES_DSN, POSTGRES_SCHEMA = set; O365_* all set or all empty. **Verified 2026-11-19:** file mode -rw------- it.admin; all required keys set; O365 set; DRYRUN_EMAIL_RECIPIENTS empty (dry-run emails logged only); DRYRUN_TEAMS_WEBHOOK_URL set (dry-run posts to the test channel).

---

## Step 11: Run one dry-run of the pipeline

### 11a. Build the image on the ITServer

**Verified 2026-11-19:** Use host git to read the commit SHA (local read only, no passphrase):

```bash
cd /home/it.admin/dnc/git_container/dnc_db_report
SHA=$(git rev-parse --short HEAD)
echo "$SHA"
docker build -t dnc-scrub:latest --build-arg GIT_SHA="$SHA" .
```

**Expected:** echo = the latest main commit (b35a022 as of 2026-11-19); build ends with `naming to docker.io/library/dnc-scrub:latest`. The .env is not copied into the image (Dockerfile copies only requirements.txt, run.py, scheduler.py, entrypoint.sh, dnc/, tools/). **Rebuild after every pull.**

**Pitfall (verified 2026-11-19):** Forgetting the trailing ` .` (space-dot) gives `ERROR: docker: 'docker buildx build' requires 1 argument`. The `.` is the build context (current directory).

**Fallback for servers without host git:** Replace the SHA line with the alpine/git container read (see Method B in Step 4).

### 11b. Create the data folder

```bash
mkdir -p /home/it.admin/dnc/data && chmod 700 /home/it.admin/dnc/data
```

**Data folder:** `/home/it.admin/dnc/data` (on /home; separate from the Mac's OneDrive data/; holds lead PII). Note the systemd-units story may still move it.

### 11c. Run detached so a dropped SSH session doesn't matter

```bash
docker run -d --rm --name dnc-dryrun \
  --env-file /home/it.admin/dnc/git_container/dnc_db_report/.env \
  -v /home/it.admin/dnc/data:/data \
  -v dnc-pgsocket:/var/run/postgresql \
  dnc-scrub:latest once --dry-run

docker logs -f dnc-dryrun     # Ctrl-C only stops following; the run continues
```

**Notes:**
- **SELinux:** `:z` not needed on this host (see SELinux note above); on hosts where `docker info` shows `name=selinux`, use `-v /home/it.admin/dnc/data:/data:z`
- The image runs as root so files under data/ are root-owned (revisit in the systemd-units story)
- **Duration (verified 2026-11-19):** Stage 1 downloads ~429 pages of 1000 leads at ~3-4 s/page = ~27 minutes; the Postgres write happens right after Stage 1 (~9 s for 428,641 rows); stages 2-5 take seconds
- The run log is also saved under `/home/it.admin/dnc/data/_dryrun/<timestamp>Z/logs/run.log`; may need sudo to read (root-owned)
- With --rm the container and its `docker logs` disappear at exit, the file log stays
- Dry-runs always use a fresh timestamped folder; SQLite .db files are deleted at the end, logs/reports and processed/.run_id kept; folders purged after DRYRUN_RETENTION_DAYS (default 7) — so Step 12 should be done within that window
- Don't paste the full log (share Postgres lines and final status only); on failure share the first ERROR/CRITICAL line

**Progress check:**
```bash
docker logs --tail 1 dnc-dryrun
```

**Wait for completion:**
```bash
while docker ps -q --filter name=dnc-dryrun | grep -q . && \
      ! docker logs dnc-dryrun 2>&1 | grep -qE 'Wrote .* rows|Failed to write raw snapshot|CRITICAL'; do
  sleep 60
done
grep -E 'Postgres|run_id|Partition|Wrote .* rows|ERROR|CRITICAL' \
  "$(ls -d /home/it.admin/dnc/data/_dryrun/*/ | tail -1)logs/run.log"
```

This checks every 60 seconds; exits when the container stops OR a final Postgres line/error appears. The grep shows key lines from the log file.

**Expected log lines (verified 2026-11-19):**
- `INFO Dry-run mode: overriding POSTGRES_SCHEMA=dnc with dnc_test` → correct (dry-runs forced into `dnc_test`)
- `INFO Partition check: created: raw_snapshot_202611` (or `exists: raw_snapshot_202611` if the partition already exists)
- `INFO Created new run_id <uuid>`
- `INFO Wrote run_id to /data/_dryrun/<timestamp>Z/processed/.run_id`
- `INFO Wrote 428641 rows to dnc_test.raw_snapshot (run_id <uuid>, checksum <64-char hex prefix>)`
- `INFO Raw snapshot written to Postgres (dnc_test schema)` → success!

**Check the database:**

```bash
docker exec -it dnc-postgres psql -U dnc_reader -d dnc \
  -P pager=off \
  -v schema=dnc_test \
  -f /tmp/schema/verify_raw_snapshot.sql
```

You should see:
- One row in `dnc_test.pipeline_run` (trigger = 'dry-run', is_dry_run = true)
- ~400k+ rows in `dnc_test.raw_snapshot` (depending on snapshot size)
- One partition created (`raw_snapshot_YYYYMM` for the current month)
- Row count matches `pipeline_run.raw_row_count`
- Zero duplicates in the uniqueness check

---

## Step 12: Verify the raw snapshot checksum

Get the `run_id` from the dry-run logs (`Created new run_id <uuid>`) or from the run folder:

```bash
ls /home/it.admin/dnc/data/_dryrun/          # newest <timestamp>Z folder = this dry-run
cat /home/it.admin/dnc/data/_dryrun/<timestamp>Z/processed/.run_id    # may need sudo
```

### 12a. Count query (no PII)

**Verified 2026-11-19:** This narrow query shows only counts and checksums (safe to share; no lead data):

```bash
docker exec -it dnc-postgres psql -U dnc_reader -d dnc -P pager=off -c "
SELECT p.run_id, p.trigger, p.is_dry_run, p.code_version, p.snapshot_month,
       p.raw_row_count,
       (SELECT count(*) FROM dnc_test.raw_snapshot r WHERE r.run_id = p.run_id) AS actual_rows,
       left(p.raw_snapshot_checksum, 16) AS checksum16
FROM dnc_test.pipeline_run p;"
```

Prompts for the dnc_reader password. **Expected (verified 2026-11-19):** one row: trigger `dry-run`, is_dry_run `t`, code_version `b35a022`, snapshot_month `2026-11-01`, raw_row_count `428641` = actual_rows `428641`, checksum16 `851bcb8b32a9a221`.

**Note:** The full `verify_raw_snapshot.sql` now prints sample lead rows (PII); after a run, use the narrow query above or don't share the sample section.

### 12b. Checksum recompute

Then run the verification tool **inside the pipeline container** (psycopg is only available there):

```bash
docker run --rm --env-file /home/it.admin/dnc/git_container/dnc_db_report/.env \
  -v dnc-pgsocket:/var/run/postgresql \
  --entrypoint python \
  dnc-scrub:latest \
  tools/verify_raw_snapshot.py --schema dnc_test --run-id <run_id>
echo "exit=$?"
```

**Expected output (verified 2026-11-19):**
```
Verifying raw_snapshot for schema dnc_test
Stored checksum: <64-char hex>
Stored row count: NNNN
Actual row count: NNNN
Recomputed checksum: <64-char hex>
✓ Checksum and row count MATCH — data integrity verified
exit=0
```

Exit code 0 means success. Prints only counts and hashes (safe to share). Took ~5 s for 428,641 rows (verified 2026-11-19).

### 12c. Resume skip check (AC4)

**To be verified on the server.** Dry-runs always use a fresh folder, so the skip-if-stored path is exercised by calling the writer directly on a finished dry-run folder that still has processed/.run_id; it passes a row iterator that raises if consumed, so if the skip didn't happen, the transaction rolls back and nothing changes:

```bash
Q="SELECT started_at, raw_row_count, left(raw_snapshot_checksum,16) FROM dnc_test.pipeline_run;"
docker exec -it dnc-postgres psql -U dnc_reader -d dnc -P pager=off -c "$Q"

docker run --rm --env-file /home/it.admin/dnc/git_container/dnc_db_report/.env \
  -v /home/it.admin/dnc/data:/data -v dnc-pgsocket:/var/run/postgresql \
  --entrypoint python dnc-scrub:latest -c "
import logging, psycopg
logging.basicConfig(level=logging.INFO, format='%(levelname)s %(message)s')
from dnc.pg_store import get_postgres_settings, write_raw_snapshot
s = get_postgres_settings(is_dry_run=True)
def rows():
    raise RuntimeError('rows consumed: skip did NOT happen (rolled back)')
    yield
write_raw_snapshot(lambda: psycopg.connect(s['dsn']), s['schema'],
  {'trigger':'dry-run','is_dry_run':True,'code_version':'<sha>','report_date':'<YYYY-MM-DD of the run>',
   'stage1_incomplete':False,'started_at':None},
  rows(), '/data/_dryrun/<timestamp>Z/processed')
print('done')"

docker exec -it dnc-postgres psql -U dnc_reader -d dnc -P pager=off -c "$Q"
```

Expected: `INFO Reusing run_id <run_id> from …/.run_id`, `INFO Raw snapshot for run <run_id> already stored; skipping`, `done`; before/after query identical.

**This run (2026-11-19):** timestamp folder `2026-11-19T223911Z`, report_date `2026-11-19`, code_version `b35a022`.

---

## No real runs on the ITServer before cutover

> [!warning] Don't run `once` (without `--dry-run`) on the ITServer during S2.
> A real run spends dncscrub credits, writes tags to LawRuler and emails business recipients, in addition to the Mac's scheduled run. Writing to the production `dnc` schema starts with the cutover ([[Story - Cut over and complete a scheduled run on the ITServer]]); until then only dry-runs into `dnc_test` are used (D-19, D-21).

---

## Minor updates (changing the pinned tag)

To update to a newer PostgreSQL 17 minor version (e.g., 17.13):

1. **Check the current 17.x minor:** https://www.postgresql.org/support/versioning/
2. **Read the [PostgreSQL release notes](https://www.postgresql.org/docs/release/)** for that minor version
3. Pull the new image: `docker pull postgres:17.13`
3. Stop and remove the container (data stays in `<pg-data-dir>`):
   ```bash
   docker stop dnc-postgres
   docker rm dnc-postgres
   ```
4. Re-run the `docker run` command from **Step 3** with the new tag (e.g., `postgres:17.12`)
5. Verify with `docker exec -it dnc-postgres psql -U postgres -c 'SELECT version();'`

**Never delete `<pg-data-dir>`** during an update — it holds the legal record.

---

## Backups

Backups are covered in [[Story - Backups and restore drill for the dnc database]].

**Quick pointer:** `pg_dump` runs **inside the container**:

```bash
docker exec -it dnc-postgres pg_dump -U dnc_owner -d dnc -Fc -f /tmp/dnc_backup.dump
docker cp dnc-postgres:/tmp/dnc_backup.dump ./dnc_backup_$(date +%Y%m%d).dump
```

Encrypt and store off-machine (details in the story).

---

## Rollback

**To remove the test schema** (safe to re-run after fixing issues):

```bash
docker exec -it dnc-postgres psql -U dnc_owner -d dnc -c "DROP SCHEMA dnc_test CASCADE"
```

**To remove the production schema** (**DANGER: deletes all data**):

```bash
docker exec -it dnc-postgres psql -U dnc_owner -d dnc -c "DROP SCHEMA dnc CASCADE"
```

**To remove the entire database, container, and volume** (**DANGER: cannot be undone**):

```bash
# Stop and remove the container
docker stop dnc-postgres
docker rm dnc-postgres

# Remove the socket volume
docker volume rm dnc-pgsocket

# Remove the data directory (only after <owner> confirms — it's the legal record)
# DO NOT RUN without explicit approval:
# rm -rf <pg-data-dir>
```

**To remove the roles** (must run AFTER dropping the database):

Reconnect to the `postgres` superuser in the container (if it still exists) or on another PG server, then:

```bash
docker exec -it dnc-postgres psql -U postgres -d postgres \
  -c "REASSIGN OWNED BY dnc_owner, dnc_writer, dnc_reader, dnc_archive TO postgres; DROP OWNED BY dnc_owner, dnc_writer, dnc_reader, dnc_archive;"

docker exec -it dnc-postgres psql -U postgres \
  -c "DROP ROLE dnc_owner, dnc_writer, dnc_reader, dnc_archive"
```

---

## Troubleshooting

### "could not connect to server: No such file or directory"

- The socket volume is not mounted. Check:
  - `docker volume ls | grep dnc-pgsocket` (should exist)
  - The pipeline container mounts it: `-v dnc-pgsocket:/var/run/postgresql`
  - The server container is running: `docker ps | grep dnc-postgres`

### "Missing required env vars: [...]" at startup

- A required key is empty/commented/indented in .env (names only are logged)
- Check format: key must start at column 1, no leading `#` or space
- Run the awk check in Step 10 to see which keys are set

### "password authentication failed for user ..." (postgres or any dnc_ role)

- **Pipeline (`dnc_writer`):** the password in `POSTGRES_DSN` doesn't match what you set in Step 7. Re-run `docker exec -it dnc-postgres psql -U postgres -c '\password dnc_writer'` and update `.env`
- **Almost always a typo at a silent prompt:** Right after the same password worked a step earlier? Retry, check for stray spaces when pasting. Failed attempts don't lock the account.
- **`POSTGRES_PASSWORD_FILE` is read only on first initialisation:** editing the secrets file later changes nothing, and recreating the container doesn't either (the data directory is already initialised). To change the superuser password, run `\password postgres` in a superuser psql session, then update the secrets file to match.
- **Check logs without exposing the password:** `docker logs --tail 20 dnc-postgres` shows rejected logins without the password text.

### "permission denied for schema dnc"

- The grants weren't applied. Re-run `001_raw_snapshot.sql` as `dnc_owner` (Step 8).

### "relation dnc.raw_snapshot_202611 does not exist"

- The partition function wasn't called. The pipeline calls it automatically; if testing manually:
  ```bash
  docker exec -it dnc-postgres psql -U dnc_owner -d dnc \
    -c "SELECT dnc.ensure_raw_snapshot_partition('2026-11-01'::date);"
  ```

### "fatal: could not create work tree dir 'dnc_db_report': Permission denied"

- Bare `-v` name → Docker created a named volume instead of binding a host folder. Use absolute path: `-v /home/it.admin/dnc/git_container:/work`. Cleanup: `docker volume rm git_container`.
- If still failing with absolute path, check SELinux (see SELinux note) and add `:z` to the bind mount.

### "remote: Write access to repository not granted" / 403 error (fine-grained token)

GitHub returns this misleading message whenever a fine-grained token cannot reach the repo at all, even for a read-only clone. Most common causes (check in order):

1. **Token's Resource owner is your personal account instead of BriganteLaw** — cannot be changed after creation: create a new token with Resource owner = BriganteLaw, delete the old one
2. **Token pending org approval** — org owner approves under BriganteLaw → Settings → Personal access tokens → Pending requests
3. **Org policy blocks fine-grained tokens** — use Option A2 (classic token) or Option B/C (SSH) instead
4. **Repo not selected** — token settings → Repository access → Only select repositories → ensure `BriganteLaw/dnc_db_report` is checked

### "Password authentication is not supported for Git operations"

- GitHub rejects account passwords for git. Use a fine-grained personal access token (Option A), classic token (Option A2), or SSH keys (Options B/C). Username is the GitHub username, not an email.

### "No user exists for uid 1000"

- SSH clone needs `/etc/passwd` mounted read-only: `-v /etc/passwd:/etc/passwd:ro`.

### Permission denied on bind mounts with correct file modes

- Check SELinux: `getenforce`. If `Enforcing`, add `:z` to dedicated folder bind mounts (see SELinux note).

### Verify script prints empty sections (no table structure, no error)

- psql's pager swallowed the wide output inside the container. Re-run with `-P pager=off` (already in Steps 9 and 12).

### "Checksum MISMATCH"

- The row order or deduplication logic changed between write and verify
- Check the `payload` column: all values should be JSON objects with the same keys as the CSV
- Verify `lead_id` and `phone` match between the indexed columns and the payload

### "Failed to write raw snapshot to Postgres: ..."

The run continues (SQLite is system of record in Stage 2 and beyond); nothing is left in Postgres (the whole transaction — partition + run row — rolls back), so a re-run starts clean. Read the error, fix, `git pull`, rebuild, rerun dry-run.

**Example (verified 2026-11-19, fixed in b35a022):** `ERROR Failed to write raw snapshot to Postgres: unsupported operand type(s) for +: 'UUID' and 'str'` — a code bug where a UUID was incorrectly concatenated with a string. Ensure the image is built from b35a022 or later.

### docker build: 'docker buildx build' requires 1 argument

Forgetting the trailing ` .` (space-dot) in `docker build -t dnc-scrub:latest --build-arg GIT_SHA="$SHA" .` gives this error. The `.` is the build context (current directory).

### Wrong server reached: pipeline connects to PostgreSQL 13 instead of 17

- If a container mounts the **host's** `/var/run/postgresql:/var/run/postgresql`, it reaches the existing PG13 server.
- The pipeline must mount the **named volume** `dnc-pgsocket:/var/run/postgresql` (not the host path).
- The server container also mounts `dnc-pgsocket:/var/run/postgresql`, so both containers share the same socket directory and reach the PG17 server.

---

## What's next (S2 follow-up stories)

- **All-stages lineage:** Write Stage 2–5 intermediate results to Postgres (scrub_results, leads_with_tags, etc.)
- **Read-only months:** Closed partitions become read-only after the watchdog verifies them
- **Report from Postgres:** Monthly report draws from Postgres instead of SQLite
- **Failed-items store:** Log all FailedItems to Postgres for trend analysis and alerting

---

*Guide written 2026-11-11, rewritten for PostgreSQL 17 in Docker 2026-11-13, updated 2026-11-17 (code clone via git container, `<clone>` paths, SELinux), updated 2026-11-18 (clone options A2/C, verified clone on the ITServer, Steps 5–9 verified on the ITServer: expected outputs, pager fix); updated 2026-11-19 (Step 4 rewritten for host git as primary method, Step 11a/11c verified build and dry-run, Step 12 narrow count query and checksum verification, Step 12c resume skip check, troubleshooting entries for Postgres write failures and docker build missing dot, SELinux check updated, D-26). DDL and scripts live in `schema/` of the code repo, on `main`.*
