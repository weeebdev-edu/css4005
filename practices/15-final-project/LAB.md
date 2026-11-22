# Lab Work 15 — Final Cloud Infrastructure Project: Review, Recover, Rotate

This laboratory work is assigned only for attendance verification.

In the practice you wrote your own design down and built its first
pieces. In this lab you are the engineer who **checks** a design and
**operates** it: a classmate's architecture that looks fine until you
hold it against the variant, a disaster-recovery game day where the
database is gone and the clock is running, a password that has lived in
Git for two years, and a database whose roles and password policy exist
only on paper. Each task is an incident or a review. You find the
problem, fix it, and prove it with a measurement — exactly what your
final project report and defense will ask of you.

You can choose 1 or 2 variant:

- **AWS Implementation** — AWS Academy Learner Lab: EC2 with
  `LabInstanceProfile`, S3, Secrets Manager (fallback: SSM Parameter
  Store), the AWS CLI.
- **Local Alternative (no AWS)** — Docker Compose on your laptop, with
  [**moto**](https://github.com/getmoto/moto) as a local AWS API (S3,
  Secrets Manager) used through the real AWS CLI image.

Both variants use the **same** incidents and the same questions. Only
the tools differ.

```
   Task 1  review     a design on paper  ──►  what breaks RPO / RTO / budget / security?
   Task 2  recover    pg_dump ──► S3 ──► (disaster) ──► new server ──► restore   ⏱ RTO, RPO
   Task 3  rotate     password in Git ──► secret manager ──► rotation with 0 failed requests
   Task 4  harden     MySQL roles admin / app_rw / app_ro + validate_password = your variant
   Task 5  defend     your own project: variant evidence, drills plan, teardown audit
```

> **Why moto and not LocalStack?** Since 23 March 2026 the
> `localstack/localstack` image needs an account and a
> `LOCALSTACK_AUTH_TOKEN` even for the free tier. moto is open source
> (Apache 2.0) and needs nothing. It is a **mock**: it stores what you
> send but does not enforce everything AWS does (lifecycle rules never
> delete anything, IAM is not checked). Where that matters, the task
> says so.

> [!WARNING]
> **What was run.** The **Local** starter files of Tasks 2, 3 and 4 were
> run end to end by the author (outputs below are from that run). The
> **AWS Implementation** steps were **[not run]** (no Learner Lab
> session): they follow the AWS documentation. Tasks 1 and 5 are
> analysis tasks. If something behaves differently, that is a finding:
> write it down with the evidence.

> **AWS Academy Learner Lab limits**
>
> - You **can't create IAM users or roles**: instances use
>   **`LabInstanceProfile`** (`LabRole`). Region **`us-east-1`**.
> - Session credentials **expire with the session (about 4 hours)**;
>   instances are **stopped** when it ends. Copy fresh credentials and
>   continue.
> - **AWS Budgets** and Cost Explorer are not available: use the lab's
>   spend counter where a task needs the bill.
> - If a call fails with `AccessDenied`, the lab doesn't allow it: write
>   the action and the error in your report and do that part on the
>   Local variant.
> - Instances and secrets cost credit. When you are done: **Cleanup**,
>   then **End Lab**.

---

## Starter (both variants)

Use **your** variant from the Canvas comment wherever a task says
`<prefix>`, `rpo`, `rto`, `backup_retention_days` or the password
policy. Where you see `ci-example-0042`, replace it with your
`resource_prefix`.

Create a repository `lab15-ops` with one folder per task (`task1/` …
`task5/`). The Local starter files below go into those folders. Add a
`.gitignore` with `.env`, `*.tfstate*` and `*.sql.gz` before the first
commit. Task 3 creates a **throwaway** Git repository with a password
in it on purpose: keep it in `task3/` with its own `.git`, and delete
that `.git` when you finish the task.

Local Alternative needs Docker with Compose v2 and `git`. The images
used: `postgres:17.12`, `mysql:8.4.13`, `motoserver/moto:5.2.5`,
`amazon/aws-cli:2.42.1`, `python:3.12-slim`,
`zricethezav/gitleaks:v8.30.1` (~2 GB in total: pull them before
class).

Keep the game-day habit from week 14: before every drill write what you
**expect** (seconds, rows lost, failed requests); after it, what you
**measured**. The difference goes into the report.

---

## Task 1 — The Design Review

Your classmate Daniyar defends tomorrow. He asks you to review his
design. His variant (from his Canvas comment) and the summary of his
`docs/` folder:

```
 variant: resource_prefix = ci-daniyar-1188     vpc_cidr = 10.20.0.0/16
          traffic_pattern = scheduled-peaks     rpo = 15 min        rto = 30 min
          monthly_budget_usd = 15               db_engine = mysql   clickhouse_required = yes
          cluster = k3s-fixed-3-nodes           hpa_cpu_target = 70
          password_min_length = 16              password_require_symbol = yes
          alert_cpu_pct = 60                    alert_5xx_per_min = 5
          backup_retention_days = 7
```

```
                          Internet
                             │ 80/443
                    ALB (us-east-1a, us-east-1b)
                             │
        ┌────────────────────┴──── VPC 10.20.0.0/16 ─────────────────┐
        │ app 10.20.11.0/24 (us-east-1a)                              │
        │   k3s node 1, node 2, node 3   (t3.medium, all in 1a)       │
        │ db  10.20.21.0/24 (us-east-1a)                              │
        │   MySQL primary ──binlog──► MySQL replica (also 1a)         │
        └─────────────────────────────────────────────────────────────┘
   NAT gateway in 1a + NAT gateway in 1b       S3: ci-daniyar-1188-backups
```

```text
docs/adr/0001-nat.md       "Two NAT gateways, one per zone, for high availability."
docs/adr/0002-cluster.md   "Three k3s nodes. If more load: the HPA adds pods (max 12)."
infra/terraform.tfvars     ... db_app_rw_password = "Daniyar-2026!" ...
infra/modules/database     SG db: ingress 3306 from 10.20.0.0/16
scripts/backup.sh          0 2 * * *  mysqldump --all-databases | aws s3 cp - s3://.../$(date +%F).sql
infra/modules/storage      lifecycle: expire backups/ after 30 days
infra/modules/monitoring   cpu alarm threshold = 60 ; HPA target = 70 ; 5xx alarm = 5 per 5 minutes
docs/runbooks/restore.md   "Restore should take about 20 minutes." (no drill recorded)
k8s/deployment.yaml        replicas: 2 ; no resources: section ; livenessProbe /readyz
```

Requirements

- Find **at least 10** problems. For each one write a row: what is
  wrong → which variant field or README rule it breaks (§ number) →
  the consequence (data lost, minutes down, dollars, points lost: see
  `RUBRIC.md` caps and penalties) → the fix.
- Answer with numbers: if the DB disk dies at 01:55, how much data is
  lost? If both DB VMs' zone fails, can he meet `rto = 30 min`, and
  what would he do first?
- Estimate his design for 730 h in the **AWS Pricing Calculator**
  (`t3.medium` ×3, 2 DB VMs of your choice, ALB, 2 NAT, disks) and
  compare it with `monthly_budget_usd = 15`. What is his hourly burn
  rate, and how many hours of building does the budget buy?
- Rewrite **one** of his ADRs (your choice) properly: context →
  decision → alternatives → consequences, using his variant.
- Find what is **missing** from the design, not only what is wrong
  (hint: one variant field has no component at all).

Useful commands:

```bash
# AWS / both: subnet math without a calculator
python3 -c 'import ipaddress as i; print([str(s) for s in i.ip_network("10.20.0.0/16").subnets(new_prefix=24)][:3])'
# Pricing: https://calculator.aws/ (no account needed) — save the share link
# Rules to check against: final-project/README.md §4, §5.1–5.13 and RUBRIC.md §4
```

Explain: Why is "the HPA adds pods" not a scaling plan on
`k3s-fixed-3-nodes`, and what is? Why does his CPU alarm fire before
his HPA reacts, and what should he write about it? Which **one**
problem would you tell him to fix tonight if he can fix only one — and
why that one?

---

## Task 2 — The DR Game Day: Restore from S3

03:10. The database server of the library app is gone: the disk is
corrupt, the replica was rebuilt last week and is not in sync. The
on-call engineer (you) has the backups in S3 and a stopwatch. Your
variant's `rto` is the deadline, `rpo` is how much data you're allowed
to lose.

```
   writer (1 loan/s) ──► db  ──pg_dump | gzip──►  s3://<prefix>-backups/backups/<time>.sql.gz
                          │                                   │
                          ✕  disaster: server + disk deleted  │
                                                              ▼
                        db-new (empty) ◄────── download, gunzip, psql ─────
                          │
                          ▼
              row count, max(created_at), the app reconnects   ⏱ RTO = now − disaster
                                                               RPO = disaster − last backup
```

Local Alternative — the starter (folder `task2/`).

`compose.yaml`:

```yaml
# Lab 15, Task 2: a database, an "S3" and nothing else.
# moto plays AWS (S3, Secrets Manager) on your laptop: no account, no bill.
name: lab15-dr

services:
  s3:
    image: motoserver/moto:5.2.5
    ports: ["127.0.0.1:5050:5000"]

  db:
    image: postgres:17.12
    environment:
      POSTGRES_DB: campus
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: lab15-local-only # Task 3 is where this changes
    volumes:
      - ./seed.sql:/docker-entrypoint-initdb.d/seed.sql:ro
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d campus"]
      interval: 2s
      retries: 30

  # The replacement server for the game day: same engine, EMPTY (no seed)
  db-new:
    image: postgres:17.12
    profiles: ["restore"]
    environment:
      POSTGRES_DB: campus
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: lab15-local-only
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d campus"]
      interval: 2s
      retries: 30

  # The AWS CLI, pointed at moto. Use: docker compose run --rm aws s3 ls
  aws:
    image: amazon/aws-cli:2.42.1
    profiles: ["tools"]
    environment:
      AWS_ENDPOINT_URL: http://s3:5000
      AWS_ACCESS_KEY_ID: test
      AWS_SECRET_ACCESS_KEY: test
      AWS_DEFAULT_REGION: us-east-1

volumes:
  pgdata:
```

`seed.sql`:

```sql
-- 50,000 library loans: enough data that a restore takes measurable time
CREATE TABLE loans (
  id         bigserial PRIMARY KEY,
  book       text        NOT NULL,
  student    text        NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO loans (book, student, created_at)
SELECT 'book-' || (g % 997), 'student-' || (g % 311), now() - (g || ' seconds')::interval
FROM generate_series(1, 50000) AS g;
```

`writer.sh` — the users:

```bash
#!/usr/bin/env bash
# "Users": one new loan per second. Leave it running in its own terminal.
while true; do
  docker compose exec -T db psql -q -U admin -d campus \
    -c "INSERT INTO loans (book, student) VALUES ('book-live', 'student-live')" 2>/dev/null \
    && printf '.' || printf 'X'
  sleep 1
done
```

`backup.sh` — one logical backup straight into the bucket (nothing is
written to the laptop's disk):

```bash
#!/usr/bin/env bash
# One logical backup: pg_dump -> gzip -> s3://$BUCKET/backups/<UTC time>.sql.gz
set -euo pipefail
BUCKET=${BUCKET:?export BUCKET=<resource_prefix>-backups}
KEY=backups/$(date -u +%Y-%m-%dT%H-%M-%SZ).sql.gz
docker compose exec -T db pg_dump -U admin -d campus --no-owner \
  | gzip \
  | docker compose run --rm -T aws s3 cp - "s3://$BUCKET/$KEY"
echo "backup: s3://$BUCKET/$KEY"
```

`lifecycle.json` — **your** `backup_retention_days` (14 in the example):

```json
{
  "Rules": [{
    "ID": "keep-backups-14-days",
    "Status": "Enabled",
    "Filter": { "Prefix": "backups/" },
    "Expiration": { "Days": 14 },
    "NoncurrentVersionExpiration": { "NoncurrentDays": 1 }
  }]
}
```

Start it and prepare the bucket:

```bash
chmod +x writer.sh backup.sh
docker compose up -d
export BUCKET=ci-example-0042-backups                     # <prefix>-backups
aws() { docker compose run --rm -T aws "$@"; }            # the AWS CLI against moto
aws s3api create-bucket --bucket "$BUCKET"
aws s3api put-bucket-versioning --bucket "$BUCKET" --versioning-configuration Status=Enabled
docker compose run --rm -T -v "$PWD/lifecycle.json:/lc.json:ro" aws \
  s3api put-bucket-lifecycle-configuration --bucket "$BUCKET" --lifecycle-configuration file:///lc.json
./writer.sh            # in a second terminal: dots = loans written
./backup.sh            # backup: s3://ci-example-0042-backups/backups/2026-11-22T19-15-36Z.sql.gz
```

AWS Implementation **[not run]**: two `t3.micro` AL2023 instances with
`LabInstanceProfile` (`db` and, later, `db-new`) in the default VPC, a
real bucket `<prefix>-backups` with versioning, Block Public Access and
the same `lifecycle.json`. On each instance (through **SSM Session
Manager**, no SSH):

```bash
sudo dnf install -y postgresql17-server postgresql17
sudo postgresql-setup --initdb && sudo systemctl enable --now postgresql
sudo -u postgres createdb campus && sudo -u postgres psql -d campus -f seed.sql      # db only
# backup.sh on the instance: the instance role writes to S3, no keys
sudo -u postgres pg_dump -d campus --no-owner | gzip | aws s3 cp - "s3://$BUCKET/backups/$(date -u +%FT%H-%M-%SZ).sql.gz"
```

Requirements

- Take **at least two** backups, a few minutes apart, while the writer
  runs. Show the bucket listing and the lifecycle configuration.
- Write your expected RTO and RPO **before** the disaster.
- The disaster: note the time and the last row (`SELECT count(*),
  max(created_at) FROM loans`), then delete the server **and** its
  disk — Local: `docker compose rm -sf db && docker volume rm
  lab15-dr_pgdata`; AWS: terminate the `db` instance (its root volume
  is deleted with it).
- Restore into the **new, empty** server from the **newest** backup in
  S3, using only what is in S3 — Local: `docker compose --profile
  restore up -d db-new`; AWS: launch `db-new`, install PostgreSQL.
- Measure **RTO** (disaster → data verified) and **RPO** (disaster time
  − `max(created_at)` of the restored data). Fill in the table:

  | | Expected | Measured | Variant | Met? |
  |---|---|---|---|---|
  | RTO | | | `rto` | |
  | RPO (data lost) | | | `rpo` | |
  | Rows lost | | | — | |

- Prove the data is right, not only "there": row count, one known
  record, `max(created_at)`.
- Change `backup.sh` + a schedule (cron line or a `while … sleep`
  loop) so that your **variant's `rpo`** would be met, and say what
  that costs in S3 (objects × size × `backup_retention_days`).

Useful commands:

```bash
# newest backup
LATEST=$(aws s3 ls "s3://$BUCKET/backups/" | sort | tail -1 | awk '{print $4}')
# restore: S3 -> gunzip -> psql on the new server (Local)
aws s3 cp "s3://$BUCKET/backups/$LATEST" - | gunzip \
  | docker compose exec -T db-new psql -q -U admin -d campus -v ON_ERROR_STOP=1
docker compose exec -T db-new psql -U admin -d campus -tAc "SELECT count(*), max(created_at) FROM loans"
# the stopwatch
T0=$(date +%s)   # at the disaster ...   echo "RTO: $(( $(date +%s) - T0 )) s"
aws s3api get-bucket-lifecycle-configuration --bucket "$BUCKET"
aws s3api list-object-versions --bucket "$BUCKET" --prefix backups/ --query 'Versions[].[Key,IsLatest]'
```

What the author's Local run showed (50,049 rows at the disaster, the
newest backup 16 s older):

```text
50035|2026-11-22 19:16:12.207345+00
RTO: 5 s
.................................................XXXX
```

— 14 loans lost, and the writer's `X`s are the users' failed requests.
Five seconds is a laptop with 50,000 rows; a real restore also needs a
server (Terraform: minutes), a bigger dump, and someone to wake up.

Explain: Why is the measured local RTO **not** your project's RTO —
which steps are missing (detection, provisioning, repointing the app)?
Your RPO came from the backup interval, not from the restore: which
mechanism would get it down to seconds (lecture, Block 4), and why does
that mechanism **not** protect you from `DELETE FROM loans`? moto
accepted the lifecycle rule — did it delete anything after 14 days?
What does that tell you about testing against a mock?

---

## Task 3 — The Password That Lived in Git

A security scan of an old service finds `app.env`, committed two years
ago, with the database password in it. The password still works. Your
job: move the password into the **secret manager**, make the app read
it from there, and **rotate** it — with no failed requests.

```
   before:   app.env (in Git) ──► client ──► PostgreSQL role app_rw
   after:    Secrets Manager <prefix>/db/app_rw ──► client (cached, refreshed) ──► app_rw
   rotate:   new value ──► put-secret-value (AWSCURRENT) ──► ALTER ROLE ──► client re-reads
```

Local Alternative — the starter (folder `task3/`, its **own** Git
repository).

`app.env` — the leak:

```bash
# "Temporary" config from 2024. It has been in git ever since.
DB_PASSWORD=Kq7#vR2m!Lx9pT4wZ3
```

`01-app-role.sh` — creates the app's role when the database is first
started:

```bash
#!/bin/sh
# Runs once, when the database is created: the app's role with the legacy password
psql -v ON_ERROR_STOP=1 -U admin -d campus -c "CREATE ROLE app_rw LOGIN PASSWORD '$DB_PASSWORD'"
```

`client.py` — a stand-in for the app: it connects twice a second and
prints `OK` or `FAIL`:

```python
"""A stand-in for the app: connects to the database twice a second, prints OK or FAIL."""

import os
import time

import psycopg

SECRET_ID = os.environ.get("SECRET_ID")  # set = read the password from Secrets Manager
CACHE_SECONDS = int(os.environ.get("CACHE_SECONDS", "30"))
REFRESH_ON_AUTH_ERROR = os.environ.get("REFRESH_ON_AUTH_ERROR") == "1"

if SECRET_ID:
    import boto3

    sm = boto3.client("secretsmanager")
cached, cached_at = None, 0.0


def password(force=False):
    global cached, cached_at
    if not SECRET_ID:
        return os.environ["DB_PASSWORD"]  # the legacy way: from app.env
    if force or cached is None or time.time() - cached_at > CACHE_SECONDS:
        cached = sm.get_secret_value(SecretId=SECRET_ID)["SecretString"]
        cached_at = time.time()
    return cached


def query(pw):
    with psycopg.connect(host="db", dbname="campus", user="app_rw", password=pw, connect_timeout=2) as conn:
        conn.execute("SELECT 1")


ok = fail = 0
print(f"password from: {'Secrets Manager ' + SECRET_ID if SECRET_ID else 'app.env'}", flush=True)
while True:
    try:
        try:
            query(password())
        except psycopg.OperationalError as e:
            if not (REFRESH_ON_AUTH_ERROR and "password authentication failed" in str(e)):
                raise
            query(password(force=True))  # rotated? fetch the secret again and retry once
        ok += 1
        print(f"{time.strftime('%T')} OK    ok={ok} fail={fail}", flush=True)
    except Exception as e:
        fail += 1
        print(f"{time.strftime('%T')} FAIL  ok={ok} fail={fail}  {str(e).split('FATAL:')[-1].strip()[:70]}", flush=True)
    time.sleep(0.5)
```

`compose.yaml`:

```yaml
# Lab 15, Task 3: a leaked password, a secret manager, a rotation.
name: lab15-secret

services:
  aws-local: # moto: Secrets Manager (and S3) on your laptop
    image: motoserver/moto:5.2.5
    ports: ["127.0.0.1:5051:5000"]

  db:
    image: postgres:17.12
    env_file: app.env # DB_PASSWORD for the init script (the legacy way)
    environment:
      POSTGRES_DB: campus
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: lab15-admin-local-only
    volumes:
      - ./01-app-role.sh:/docker-entrypoint-initdb.d/01-app-role.sh:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d campus"]
      interval: 2s
      retries: 30

  client:
    image: python:3.12-slim
    depends_on:
      db:
        condition: service_healthy
    env_file: app.env
    environment:
      AWS_ENDPOINT_URL: http://aws-local:5000
      AWS_ACCESS_KEY_ID: test
      AWS_SECRET_ACCESS_KEY: test
      AWS_DEFAULT_REGION: us-east-1
      SECRET_ID: ${SECRET_ID:-}
      CACHE_SECONDS: ${CACHE_SECONDS:-30}
      REFRESH_ON_AUTH_ERROR: ${REFRESH_ON_AUTH_ERROR:-0}
    volumes:
      - ./client.py:/client.py:ro
    command: >
      sh -c "pip install -q --root-user-action=ignore boto3==1.38.0 'psycopg[binary]==3.2.9'
      && exec python -u /client.py"

  aws:
    image: amazon/aws-cli:2.42.1
    profiles: ["tools"]
    environment:
      AWS_ENDPOINT_URL: http://aws-local:5000
      AWS_ACCESS_KEY_ID: test
      AWS_SECRET_ACCESS_KEY: test
      AWS_DEFAULT_REGION: us-east-1
```

Recreate the history the way it happened, then start it:

```bash
git init -b main && git add -A && git commit -m "legacy app"
docker compose up -d
docker compose logs -f client          # OK lines; Ctrl-C to stop watching
```

AWS Implementation **[not run]**: the same `client.py` on a `t3.micro`
with `LabInstanceProfile` (`dnf install -y python3-pip`, `pip install
boto3 psycopg[binary]`), no `AWS_ENDPOINT_URL` and no keys — boto3 uses
the instance role. PostgreSQL from Task 2's instance (or a second
small instance) with the `app_rw` role. The secret in **Secrets
Manager**; if the lab refuses it, **SSM Parameter Store
`SecureString`** with `put-parameter --overwrite` as the "rotation" (and
change `password()` to `ssm.get_parameter(..., WithDecryption=True)`).

Requirements

- **Find it with a tool, not your eyes.** Run gitleaks over the history.
  With the default rules it reports **no leaks** for `app.env` — show
  that output. Then add a rule to a `.gitleaks.toml` (`[extend]
  useDefault = true` + one `[[rules]]` with a `regex` for
  `PASSWORD=...` and `secretGroup`) so the scan **fails** on
  `app.env`. What else does your rule find, and is it a real finding?
- **Migrate:** generate a new password **with the secret manager**
  (`get-random-password`, at least your `password_min_length`, a symbol
  if `password_require_symbol = yes`, no `'`), store it as
  `<prefix>/db/app_rw`, set it on the role, restart the client with
  `SECRET_ID` set. The value must never be printed or written to a
  file. The old password is **leaked**: show that it no longer works.
- **Rotate, the naive way:** with `REFRESH_ON_AUTH_ERROR=0`
  (default), rotate once (new value → `put-secret-value` → `ALTER
  ROLE`). Count the `FAIL` lines and explain the number with
  `CACHE_SECONDS`.
- **Rotate, the right way:** restart with `REFRESH_ON_AUTH_ERROR=1`
  and rotate again. **0** `FAIL` lines is the target. Show
  `list-secret-version-ids` (`AWSCURRENT` / `AWSPREVIOUS`).
- **Clean up the leak:** remove `app.env` from the app (the client no
  longer needs `DB_PASSWORD`), and explain why rewriting the Git
  history is optional once the password is rotated — but rotation is
  not.
- Write the rotation as a **runbook** (`task3/ROTATION.md`): the order
  of the steps, what happens if the script dies between `put-secret-
  value` and `ALTER ROLE`, and how you'd undo it (`AWSPREVIOUS`).

Useful commands:

```bash
aws() { docker compose run --rm -T aws "$@" 2>/dev/null; }     # Local: against moto
docker run --rm -v "$PWD:/repo" zricethezav/gitleaks:v8.30.1 git /repo --no-banner --redact -v
docker run --rm -v "$PWD:/repo" zricethezav/gitleaks:v8.30.1 git /repo --no-banner --redact -v \
  --config /repo/.gitleaks.toml

NEW=$(aws secretsmanager get-random-password --password-length 20 --exclude-characters "'" \
  --require-each-included-type --query RandomPassword --output text)
aws secretsmanager create-secret --name ci-example-0042/db/app_rw --secret-string "$NEW" --query Name
aws secretsmanager put-secret-value --secret-id ci-example-0042/db/app_rw --secret-string "$NEW"
docker compose exec -T db psql -q -U admin -d campus -c "ALTER ROLE app_rw PASSWORD '$NEW'"; unset NEW

SECRET_ID=ci-example-0042/db/app_rw docker compose up -d client                     # migrate
SECRET_ID=ci-example-0042/db/app_rw REFRESH_ON_AUTH_ERROR=1 docker compose up -d --force-recreate client
docker compose logs --no-log-prefix client | grep -c FAIL
aws secretsmanager list-secret-version-ids --secret-id ci-example-0042/db/app_rw \
  --query 'Versions[].[VersionId,VersionStages[0]]' --output text
```

What the author's Local run showed: the naive rotation gave **56**
`FAIL` lines (`password authentication failed for user "app_rw"`) until
the 30 s cache expired; the rotation with a refresh on authentication
errors gave **0**.

Explain: Why did `ALTER ROLE` break the running client, even though the
secret manager already had the new value? Name two other ways to rotate
without downtime (hint: AWS's *alternating users* strategy; and what
PostgreSQL can't do that some databases can — two valid passwords at
once). In the AWS variant, who is allowed to read the secret, and what
would the least-privilege policy JSON look like instead of `LabRole`?
Why does `get-random-password` beat `openssl rand` typed in your
terminal?

---

## Task 4 — The Database Nobody Hardened

The final project's database must have three roles with **denied**
actions and a password policy **enforced by the server** with your
variant's numbers. A teammate says "it's in the README". The server
says otherwise. This task uses **MySQL**; if your variant is
PostgreSQL, do it anyway — then write in the report what the
PostgreSQL equivalent is and why it is weaker (`passwordcheck` /
`credcheck` only see plain-text passwords).

```
           admin   ─── owns schema campus (DDL, GRANT) — never used by the app
           app_rw  ─── SELECT, INSERT, UPDATE, DELETE on campus.*      ✗ DROP, ✗ hr.*
           app_ro  ─── SELECT on campus.*                               ✗ DELETE
   validate_password:  length = password_min_length, ≥ 1 symbol (if required),
                       ≥ 1 digit, ≥ 1 upper + lower
```

Local Alternative — the starter (folder `task4/`).

`compose.yaml`:

```yaml
# Lab 15, Task 4: one MySQL server to harden.
name: lab15-roles

services:
  db:
    image: mysql:8.4.13
    environment:
      MYSQL_ROOT_PASSWORD: lab15-root-local-only
    volumes:
      - ./schema.sql:/docker-entrypoint-initdb.d/schema.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD --silent"]
      interval: 3s
      retries: 40
```

`schema.sql`:

```sql
CREATE DATABASE campus;
CREATE TABLE campus.events (
  id    BIGINT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(100) NOT NULL
);
INSERT INTO campus.events (title) VALUES ('Club fair'), ('Hackathon');
CREATE DATABASE hr; -- another team's data: none of the app roles may see it
CREATE TABLE hr.salaries (name VARCHAR(50), amount INT);
```

```bash
docker compose up -d
docker compose ps                       # (healthy) after ~30 s
```

AWS Implementation **[not run]**: a `t3.micro` AL2023 instance with
`LabInstanceProfile`, `dnf install -y mysql8.4-server` (or the MySQL
community repository), the same `schema.sql`, in a **private** subnet
or with a security group that allows 3306 only from the client
instance's SG. The role passwords come from Secrets Manager / SSM (Task
3), never from the SQL file.

Requirements

- Write `policy.sql` (run as root): install the
  `component_validate_password` component and `SET PERSIST` the
  variables so the policy is **your variant**. Show `SHOW VARIABLES
  LIKE 'validate_password.%'`, then restart the container and show the
  values survived.
- Show the policy **rejecting** a weak password and accepting a strong
  one, three ways: `CREATE USER` with a weak password (`ERROR 1819`),
  `SELECT VALIDATE_PASSWORD_STRENGTH(...)` for both, and an app user
  trying `ALTER USER CURRENT_USER() IDENTIFIED BY '<weak>'`.
- Write `roles.sql`: `admin`, `app_rw`, `app_ro` with the grants in the
  diagram, passwords that pass your policy (local only — say where they
  would come from in the cloud).
- Show **one denied action per limited role**, plus one allowed action
  for each: `app_ro` can't `DELETE`, `app_rw` can't `DROP TABLE` and
  can't read `hr.salaries`, `app_rw` **can** `INSERT`.
- Make the same policy testable in CI: a short script (Python or bash)
  that takes a password and exits 1 unless it meets your variant's
  rules — the same rules as the template app's
  `POST /api/admin/check-password`. Run it on one good and one bad
  password.

Useful commands:

```bash
root() { docker compose exec -T db sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -t' 2>/dev/null; }
root < policy.sql
root < roles.sql
echo "SELECT VALIDATE_PASSWORD_STRENGTH('Events2026'), VALIDATE_PASSWORD_STRENGTH('Ev3nts-2026-Narxoz');" | root
echo "SELECT user, host FROM mysql.user; SHOW GRANTS FOR 'app_rw'@'%';" | root
docker compose exec -T db mysql -uapp_ro -p'<app_ro password>' campus -e "DELETE FROM events"
docker compose exec -T db mysql -uapp_rw -p'<app_rw password>' -e "SELECT * FROM hr.salaries"
docker compose restart db
```

What the author's Local run showed with `length = 14` and one symbol,
digit and mixed case required:

```text
ERROR 1819 (HY000) at line 1: Your password does not satisfy the current policy requirements
25	100
ERROR 1142 (42000) at line 1: DELETE command denied to user 'app_ro'@'localhost' for table 'events'
ERROR 1142 (42000) at line 1: DROP command denied to user 'app_rw'@'localhost' for table 'events'
ERROR 1142 (42000) at line 1: SELECT command denied to user 'app_rw'@'localhost' for table 'salaries'
```

(`25` and `100` are the strength scores of the weak and the strong
password.)

Explain: Why must the app connect as `app_rw` and never as `admin`
or `root` — what could an attacker with the app's password do in each
case? The template app creates its tables at startup: which grant does
that force on `app_rw`, and how would migrations run by `admin` remove
the need for it? Why is a policy in three places (DB, app checker, CI
test) better than one, and what goes wrong if the three use different
numbers?

---

## Task 5 — Final Challenge: Defend Your Own Design

Now do to **your** final project what you did to Daniyar's in Task 1 —
before the examiner does.

```
   your variant ──► docs/variant.md ──► evidence for every row?
        │
        ├──► rpo / rto  ──► which drill proves each one, and when will you run it?
        ├──► monthly_budget_usd ──► burn rate × planned hours ≤ budget?
        └──► teardown ──► what survives terraform destroy?
```

Requirements

- **Self-review:** apply your Task 1 checklist to your own
  `docs/variant.md`, diagram and ADRs (from the practice). Every row
  that has no evidence yet gets a planned evidence item (command,
  screenshot, file) and a milestone (M1–M7).
- **Drill plan:** for your `rpo` and `rto`, the drills you'll run in the
  project (restore test, DB failover, zone drill) with the expected
  numbers, using what you measured in Tasks 2–3 as the starting point.
- **Teardown audit.** A classmate ran `terraform destroy` and says
  "everything is gone". The account still shows this. For each line:
  why it survived destroy, what it costs per month (us-east-1 list
  prices), and the command that removes it. Then write
  `teardown-check.sh` that lists everything tagged with a given
  `Project` and the untagged usual suspects.

```text
EBS snapshots       14 × 20 GB, created by DLM, tag Project=ci-daniyar-1188
EBS volume          20 GB gp3, state "available" (was a k8s PersistentVolume)
Elastic IP          1, not associated
S3 bucket           ci-daniyar-1188-backups, versioning on, 9 GB of noncurrent versions
ECR repository      ci-daniyar-1188-app, 61 images, 11 GB
Secrets Manager     6 secrets, "scheduled for deletion in 30 days"
CloudWatch Logs     /k3s/ci-daniyar-1188, 4 GB, retention "Never expire"
Load balancer       k8s-default-campus-3fa1…, created by the AWS Load Balancer Controller
```

- **The 7-minute rehearsal:** write the outline of your presentation
  (what's on each minute, which drill you'll show) and the list of
  files you'd be least comfortable explaining line by line. Explain
  one of them in writing, line by line.

Useful commands:

```bash
# AWS
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values="$PREFIX" \
  --query 'ResourceTagMappingList[].ResourceARN' --output text
aws ec2 describe-snapshots --owner-ids self --query 'Snapshots[].[SnapshotId,VolumeSize,StartTime]'
aws ec2 describe-volumes --filters Name=status,Values=available --query 'Volumes[].[VolumeId,Size]'
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]'
aws elbv2 describe-load-balancers --query 'LoadBalancers[].[LoadBalancerName,CreatedTime]'
aws secretsmanager list-secrets --include-planned-deletion --query 'SecretList[].[Name,DeletedDate]'
aws logs describe-log-groups --query 'logGroups[].[logGroupName,storedBytes,retentionInDays]'
# Local: the same commands work against moto with AWS_ENDPOINT_URL set (Task 2 compose)
```

Explain: Which line of the teardown audit costs the most per month, and
which one would you not notice for the longest? Why did the load
balancer survive `terraform destroy`? Which part of your own design is
the weakest, and what is your honest answer if the examiner asks about
it?

---

## Cleanup

Local Alternative:

```bash
cd task2 && docker compose --profile restore --profile tools down -v && cd ..
cd task3 && docker compose --profile tools down -v && rm -rf .git && cd ..   # the leaked history goes too
cd task4 && docker compose down -v && cd ..
docker ps -a --filter name=lab15           # nothing left
```

AWS Implementation:

```bash
aws ec2 terminate-instances --instance-ids <db-new and the others>
aws s3 rm "s3://$BUCKET" --recursive
aws s3api delete-objects --bucket "$BUCKET" --delete "$(aws s3api list-object-versions \
  --bucket "$BUCKET" --query '{Objects: [Versions, DeleteMarkers][].{Key: Key, VersionId: VersionId}}' --output json)"
aws s3api delete-bucket --bucket "$BUCKET"
aws secretsmanager delete-secret --secret-id "$PREFIX/db/app_rw" --force-delete-without-recovery
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values="$PREFIX" \
  --query 'ResourceTagMappingList[].ResourceARN'
```

Then **End Lab**.

---

## Final Report (you can do it in free form)

Submit the GitHub repository:

```
GitHub Repository  lab15-ops
│
├── task1/REVIEW.md             (the problem table, cost estimate + link, the rewritten ADR)
├── task2/                      compose.yaml, backup.sh (your schedule), lifecycle.json, DRILL.md
├── task3/                      compose.yaml, client.py, .gitleaks.toml, ROTATION.md   (no .git, no app.env)
├── task4/                      policy.sql, roles.sql, check_password test
├── task5/                      SELF_REVIEW.md, teardown-check.sh
└── REPORT.md
```

Just an example of how you can show your report:

- Which variant you did (AWS or Local), and anything the Learner Lab
  didn't allow (action + error)
- Task 1: the problem table with variant field / README § / consequence
  / fix, and the cost numbers
- Task 2: the expected vs measured RTO / RPO table, the restore
  commands, the data check, and your new backup schedule vs your `rpo`
- Task 3: the gitleaks output before and after your rule, the `FAIL`
  counts of both rotations, the secret's version list
- Task 4: the policy variables, the rejected passwords, one denied
  action per role, the CI password test
- Task 5: your self-review, the drill plan and the teardown audit table
- The answers to the **Explain** questions
- Problems you ran into and how you solved them
- **Never** a real password, key or token in the repository — the
  Task 3 password must be rotated and its history deleted (say how you
  checked)
