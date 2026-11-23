# Lab Work 14 — High Availability, Security and Cost Optimization

**Submission: a report with screenshots is required.** Upload one PDF to
Canvas with your repository link and, for **every** task you did,
screenshots of the commands, their output and the result, each with a
one-line caption. A lab without a screenshot report is not accepted.

In the practice you broke a zone and a database on purpose and measured
what users saw. In this lab you are the engineer who is called **after**
the review: an audit that found open doors, a zone outage during
registration week, a failover that only one person can do, a backup that
nobody has ever restored, and a bill nobody can explain. Each task is an
incident. You find the cause, change the system so it can't happen
again, and prove it with a measurement.

You can choose 1 or 2 variant:

- **AWS Implementation** — AWS Academy Learner Lab + Terraform: the
  practice's two-AZ fleet (ALB, ASG, RDS PostgreSQL, SSM Parameter
  Store).
- **Local Alternative (no AWS)** — the practice's Docker Compose
  "region": HAProxy, two app zones, a PostgreSQL primary and a streaming
  replica, secrets as files.

Both variants use the **same** incidents and the same questions. Only
the tools differ.

```
            users ──► load balancer ──► web (zone a)   web (zone b)   [web (zone c): Task 2]
                                              │               │
                                              └──► DB primary ──► DB copy in the other zone
   AWS:     ALB + ASG + RDS Multi-AZ + SSM            Local:  HAProxy + Compose + replica + files
```

> [!WARNING]
> **Not run.** This lab was written from the practice 14 code. The Local
> practice was run end to end by the author; the AWS practice code was
> never applied to an account, and **none of the tasks below were run**
> on either variant. Expected results come from the practice run (Local)
> or from the AWS documentation (AWS). If something behaves differently,
> that is a finding: write it down with the evidence.

> **AWS Academy Learner Lab limits**
>
> - You **can't create IAM users or roles**: instances use
>   **`LabInstanceProfile`** (`LabRole`). Region **`us-east-1`**.
> - Session credentials **expire with the session (about 4 hours)**.
>   Terraform state is local: after `ExpiredToken`, copy new credentials
>   and run the same command again.
> - **AWS Budgets**, Cost Explorer and some security services
>   (GuardDuty, Security Hub) are not available. Where a task needs them,
>   write what you would do and why.
> - If a call fails with `AccessDenied`, the lab doesn't allow it: write
>   the action and the error in your report and do that part on the
>   Local variant.
> - An RDS instance and an ALB bill every hour. When you are done:
>   **Cleanup**, then **End Lab**.

---

## Starter (both variants)

Start from your **practice 14** project:

- AWS Implementation: the `lab14/` Terraform project after **A2** (both
  TODOs done), applied, with `terraform.tfvars` holding your `prefix`
  and `owner`. Single-AZ database to begin with.
- Local Alternative: the `lab14ha/` Compose project after **B4** (the
  retry lines in `haproxy.cfg`), started with a fresh database:

  ```bash
  docker compose down -v && echo db-primary > config/db_host && docker compose up -d --build
  ```

If you didn't finish the practice, the practice handout has every file.

Put the project in a Git repository `lab14-review` **after Task 1**
(Local: Task 1 makes a throwaway repository with the secrets in it on
purpose). Add `secrets/`, `backups/` and `.env` to `.gitignore` before
the first commit, and never push a real password.

Keep the game-day habit from the practice: before every drill, write
what you **expect** (failed requests, seconds, rows lost); after it,
what you **measured**. Every task below ends with such a row.

Useful in every task:

```bash
./watch.sh "$ALB/"                       # AWS: ALB=$(terraform output -raw alb_dns)
./watch.sh http://localhost:8080/        # Local
```

---

## Task 1 — The Audit Nobody Passed

Monday, 09:00. An external audit of the Narxoz Shop arrives: "**FAILED** —
database reachable from untrusted networks, credentials stored in plain
text, administrative access open to the internet, instance metadata
unprotected." The previous engineer left a note: *"Temporary debug
access, I'll remove it on Friday."* That was a year ago.

```
   audit finding                         where it hides
   ─────────────────────────────────     ───────────────────────────
   DB reachable from anywhere       ──►  a forgotten SG rule / a published port
   credentials in plain text        ──►  a String parameter / an env var / Git
   admin access open                ──►  port 22 from 0.0.0.0/0 / a root container
   metadata unprotected             ──►  IMDSv1 allowed
```

Recreate the "previous engineer's" changes (apply them exactly like
this).

AWS Implementation — add `legacy.tf`:

```hcl
# "Temporary" debug access. Added 2025-11, remove on Friday.
resource "aws_vpc_security_group_ingress_rule" "db_debug" {
  security_group_id = aws_security_group.db.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "tcp"
  from_port         = 5432
  to_port           = 5432
}

resource "aws_vpc_security_group_ingress_rule" "web_ssh" {
  security_group_id = aws_security_group.web.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "tcp"
  from_port         = 22
  to_port           = 22
}

# For the reporting script, which "can't decrypt"
resource "aws_ssm_parameter" "db_password_plain" {
  name  = "/${var.prefix}/db/password-plain"
  type  = "String"
  value = random_password.db.result
}

output "db_password" {
  value = nonsensitive(random_password.db.result) # "so I can copy it"
}
```

and in `compute.tf`, in `metadata_options`, change `"required"` to
`"optional"`. Then `terraform apply`.

Local Alternative — create `compose.override.yaml` (Compose merges it
into `compose.yaml` automatically: that's why nobody noticed it):

```yaml
# Debug access, added 2025-11 "for one day"
services:
  db-primary:
    ports:
      - "5432:5432"              # so I can connect from DBeaver
  web-b:
    user: root                   # "pip install in the container, quick fix"
    environment:
      DB_PASSWORD: ${APP_DB_PASSWORD}   # the old reporting cron reads it
```

and commit the secrets on purpose, the way it happened:

```bash
echo "APP_DB_PASSWORD=$(cat secrets/app_db_password.txt)" > .env
git init -b main && git add -A && git add -f secrets/    # -f: past the .gitignore, "just this once"
git commit -m "working config"
docker compose up -d
```

Requirements

- Find **every** problem with commands, not by reading the files: for
  each audit finding, the command that proves it and its output. (There
  are at least five in each variant. AWS: one of them is in
  `terraform output`, one is in the state, one is visible with
  `curl` from an instance. Local: one of them is in `git log`.)
- Fix them all, and say for each one whether removing it is enough or
  the password must be **rotated** too. A password that was in Git, in a
  `String` parameter or in an output **has leaked**: rotate it.
- Rotate the database password without downtime, and measure the
  downtime anyway:
  - AWS: `terraform apply -replace=random_password.db` (the RDS master
    password and the SSM parameter change in one apply). Watch the pages:
    for how long does any instance show `db=DOWN`, and why (think about
    the order of the two changes, and the 15 s check)?
  - Local: `ALTER ROLE app PASSWORD '<new>'` on the primary, then write
    the new value into `secrets/app_db_password.txt` (`openssl rand -hex
    16 > ...` rewrites the same file: the containers see it at once —
    an editor that replaces the file would break the bind mount). The
    app reads the file on every request: does it need a restart? Look
    at the writer too: when does it read the password?
- Write `audit.sh`: it checks every finding and **exits 1** if any check
  fails. Run it before the fix (FAIL lines, exit 1) and after (all PASS,
  exit 0).
- Local: when you're done, `rm -rf .git` — that history holds the old
  passwords and must never be pushed (once it is pushed, rewriting it
  with `git filter-repo` is possible, but the password still counts as
  leaked). Then start `lab14-review` with the `.gitignore`.

A start for `audit.sh` (AWS):

```bash
#!/usr/bin/env bash
# Security audit of the lab14 fleet. Exit 1 = at least one check failed.
# Usage: P=lab14-aidana ./audit.sh
set -uo pipefail
P=${P:?export P=<your prefix>}
fail=0
check() { # name, actual, expected
  if [ "$2" = "$3" ]; then echo "PASS  $1"; else echo "FAIL  $1 (got: $2)"; fail=1; fi
}

DB_SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values="$P-db-sg" \
  --query 'SecurityGroups[0].GroupId' --output text)
check "db-sg has no CIDR ingress rules" "$(aws ec2 describe-security-group-rules \
  --filters Name=group-id,Values="$DB_SG" \
  --query 'length(SecurityGroupRules[?!IsEgress && CidrIpv4!=`null`])' --output text)" 0

check "instances require IMDSv2" "$(aws ec2 describe-instances \
  --filters Name=tag:Project,Values="$P" Name=instance-state-name,Values=running \
  --query 'length(Reservations[].Instances[?MetadataOptions.HttpTokens!=`required`][])' --output text)" 0

# TODO: no port 22 from 0.0.0.0/0; only String parameter under /$P/ is db/host;
#       RDS not publicly accessible and encrypted; no "password" in terraform output
exit $fail
```

A start for `audit.sh` (Local):

```bash
#!/usr/bin/env bash
# Security audit of the lab14ha stack. Exit 1 = at least one check failed.
set -uo pipefail
fail=0
check() { if [ "$2" = "$3" ]; then echo "PASS  $1"; else echo "FAIL  $1 (got: $2)"; fail=1; fi; }

check "db-primary publishes no port" "$(docker inspect $(docker compose ps -q db-primary) \
  --format '{{json .HostConfig.PortBindings}}')" "{}"
check "no PASSWORD= in any container's environment" "$(docker inspect $(docker compose ps -q) \
  --format '{{json .Config.Env}}' | grep -c 'PASSWORD=')" 0

# TODO: web-a and web-b don't run as root (id -u); nothing under secrets/ or .env
#       is tracked by git (git ls-files); .env and secrets/ never appear in git log --all
exit $fail
```

Useful commands:

```bash
# AWS
terraform output
grep -m1 -o '"result": "[^"]*"' terraform.tfstate        # what is in the state?
aws ssm describe-parameters --parameter-filters Key=Name,Option=BeginsWith,Values="/$P/" \
  --query 'Parameters[].[Name,Type]' --output table
aws ssm send-command --instance-ids <id> --document-name AWS-RunShellScript \
  --parameters 'commands=["curl -s -m 2 http://169.254.169.254/latest/meta-data/iam/info"]'
# Local
docker compose config | less            # the MERGED configuration: the override is in it
docker inspect $(docker compose ps -q) --format '{{.Name}} {{json .Config.Env}}'
git log --all --oneline -- .env secrets/
```

Explain: The database is in a subnet with no internet route and
`publicly_accessible = false`. Was the `0.0.0.0/0` rule on 5432
exploitable from the internet? Why do you remove it anyway? The app logs
in to RDS as the **master user** `dbadmin` (AWS) — what would a least-
privilege setup look like, and what could an attacker do with the
master password that they couldn't do with an app role? Why is deleting
`.env` from the latest commit not enough?

---

## Task 2 — The Zone Outage at Registration

Registration week, 10:05. One availability zone loses power. The Narxoz
Shop doesn't go down — but it gets so slow that students give up. The
metrics show why: before the outage two servers were at 60% CPU each;
after it, one server got all the traffic.

```
   before:   zone a [60%]   zone b [60%]              = fine
   10:05:    zone a  ✖      zone b [120%] → timeouts   = "up", but useless
   rule:     after losing ANY one zone, the rest must carry the peak
```

The business says: **at peak, the shop needs 2 web servers**. Your job
is a web tier that is **statically stable**: after losing any one zone,
2 servers are still running **without launching anything**.

Requirements

- A SPOF table for the whole system: every component, its copies, what
  happens when it fails, and whether you fix it in this lab or accept it
  (with a reason). At least: the load balancer, the web servers, the
  database, the DB address (SSM parameter / `config/db_host`), the
  Terraform state (AWS) or the laptop (Local), the region.
- AWS Implementation: change the fleet to **3 AZs with 1 instance each**
  (`slice(..., 0, 3)` in `network.tf` gives a third public and DB subnet;
  `desired_capacity = 3`). Why is this cheaper than 2 AZs × 2 for the
  same rule? Prove it: `-var az_down=<az>` → 2 instances stay
  `healthy`, and `describe-scaling-activities` shows what the ASG did.
  Why might it **still** launch an instance, and is that a problem?
- Local Alternative: add a third zone `web-c` (`ZONE: c`) to
  `compose.yaml` and to the HAProxy backend. Prove it: kill zone a, then
  zone b — the shop still answers from zone c. What is the one component
  in your Compose "region" that has no copy at all, and what would AWS
  use in its place?
- Run the zone drill once more with **watch.sh** on and write the row:
  expected vs measured failed requests and seconds.

Useful commands:

```bash
# AWS
terraform output subnet_ids
TG=$(aws elbv2 describe-target-groups --names "$P-tg" --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 describe-target-health --target-group-arn "$TG" \
  --query 'TargetHealthDescriptions[].[Target.Id,TargetHealth.State]' --output text
aws autoscaling describe-scaling-activities --auto-scaling-group-name "$P-asg" --max-items 5 \
  --query 'Activities[].[StartTime,Description]' --output text
# Local
docker compose up -d                     # after editing compose.yaml
docker compose restart lb                # after editing haproxy.cfg
docker compose kill web-a && docker compose kill web-b
```

Explain: With 2 AZs × 1 instance, how much capacity is left after losing
a zone, and with 3 AZs × 1? What does "statically stable" buy you during
a **real** AZ outage, when every other customer is launching instances in
the remaining zones at the same time? What extra cost per month does
your change add (use the practice's price table)?

---

## Task 3 — The Failover Only Aidar Knows

Friday, 18:40. The database primary dies. The runbook says: *"Call
Aidar."* Aidar is on a plane. The shop shows `db=DOWN` for 47 minutes,
until someone finds the two commands in Aidar's shell history.

```
   18:40  primary ✖ ──► pages: db=DOWN ──► writes fail
   18:41  ...nobody knows the commands...
   19:27  promote + repoint ──► db=ok          RTO = 47 min, RPO = ?
```

Requirements

- AWS Implementation: make the database **Multi-AZ**
  (`db_multi_az = true` in `terraform.tfvars`). The pages check the
  database only every 15 s, so they can't measure a failover precisely:
  run a 1-second probe **on one instance** through SSM Run Command (no
  SSH), force a failover, and read the probe's log. Do it **twice**.
- Local Alternative: write `failover.sh`: it checks the primary every
  second, and after **3 failed checks in a row** promotes the replica,
  repoints `config/db_host`, and prints the time of each step. Kill the
  primary with `failover.sh` running. Do it **twice** (reset with
  `docker compose down -v`, `echo db-primary > config/db_host`, `up -d`).
- For every run: the RTO (from the failure to the first `db=ok`), the
  RPO (rows the writer reported as written that are not in the new
  primary), and what users saw (watch.sh). A table with both runs.
- `RUNBOOK.md`: what happens automatically, what a human must still do
  (rebuild the old primary as a replica / nothing on Multi-AZ), and how
  to check that it worked.

A start for `failover.sh` (Local):

```bash
#!/usr/bin/env bash
# Automatic failover: 3 failed checks of the primary in a row = promote the replica.
fails=0
echo "$(date +%T.%N | cut -c1-12) watching db-primary"
while true; do
  if docker compose exec -T db-replica pg_isready -h db-primary -t 1 -q; then
    fails=0
  else
    fails=$((fails + 1))
    echo "$(date +%T.%N | cut -c1-12) primary check failed ($fails/3)"
  fi
  if [ "$fails" -ge 3 ]; then
    # TODO: promote db-replica (pg_promote), repoint config/db_host,
    #       print a timestamp after each step
    break
  fi
  sleep 1
done
```

The 1-second probe (AWS) — `probe.json`:

```json
{
  "commands": [
    "for i in $(seq 1 120); do /usr/local/bin/dbcheck.sh; cat /usr/share/nginx/html/index.html >> /var/tmp/failover.log; sleep 1; done"
  ],
  "executionTimeout": ["900"]
}
```

```bash
IID=$(aws ec2 describe-instances --filters Name=tag:Project,Values="$P" Name=instance-state-name,Values=running \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)
aws ssm send-command --instance-ids "$IID" --document-name AWS-RunShellScript --parameters file://probe.json
aws rds reboot-db-instance --db-instance-identifier "$P-db" --force-failover
# ~3 minutes later: read the log
CMD=$(aws ssm send-command --instance-ids "$IID" --document-name AWS-RunShellScript \
  --parameters 'commands=["grep -o \"db=[A-Za-z]* .*checked [0-9:]*\" /var/tmp/failover.log | uniq -c -w 7"]' \
  --query Command.CommandId --output text)
aws ssm get-command-invocation --command-id "$CMD" --instance-id "$IID" --query StandardOutputContent --output text
aws rds describe-events --source-type db-instance --source-identifier "$P-db" --duration 30 \
  --query 'Events[].[Date,Message]' --output text
```

(Each `dbcheck.sh` run calls SSM twice, so one loop takes 2–3 s, not 1:
that's your measurement resolution. Say so in the report.)

Explain: Multi-AZ (AWS) is synchronous, your replica (Local) is
asynchronous: which one can lose committed data, and how much? What
happens if `failover.sh` promotes the replica because of a **network
blip**, and the old primary is still alive (split brain)? What would you
add to prevent it ("fencing")? Why does the AWS endpoint name stay the
same, and what must the application still do after a failover?

---

## Task 4 — The Backup Nobody Has Restored

Wednesday, 14:12. A support script meant for one test order runs
`DELETE FROM notes;` without a `WHERE`. Multi-AZ (Task 3) copies the
delete to the standby in milliseconds. The team says: "We have backups."
Nobody has ever restored one, and nobody knows how long it takes.

```
   14:00 ──────── 14:12 DELETE ✖ ──────────────────── restored ──►
         backups?      │◀─ RPO: rows written since the restore point are lost
                       │◀──────────────── RTO: how long to get them back ───────▶│
```

Requirements

- Make the data real first: let the system write rows for at least 10
  minutes (Local: the writer does it; AWS: create the table with SSM Run
  Command on an instance and insert one row every 2 s for 10 minutes —
  `psql "host=$HOST dbname=app user=dbadmin sslmode=require"` with the
  password from SSM, like `dbcheck.sh` does).
- Run the `DELETE`, and **write down the exact time** (UTC).
- Restore the data and measure RTO and RPO:
  - AWS: **point-in-time restore** to one minute before the `DELETE`
    into a **new** instance `<prefix>-db-restored` (same subnet group and
    security group, not public, single-AZ), then **repoint the fleet**
    through the SSM parameter `/<prefix>/db/host`. Count the rows.
  - Local: add a `backup` service that runs `pg_dump -Fc` every **2
    minutes** into `backups/` and keeps only the **last 5** dumps. After
    the `DELETE`, restore the newest dump into a new database, repoint,
    count the rows.
- A table: restore point, rows before the `DELETE`, rows after the
  restore, rows lost, minutes from `DELETE` to "users see the data again".
- Clean up the restore (AWS: delete `<prefix>-db-restored`, put the
  parameter back with `terraform apply`. Why does Terraform show a
  change?).

Useful commands:

```bash
# AWS
aws rds describe-db-instances --db-instance-identifier "$P-db" \
  --query 'DBInstances[0].[LatestRestorableTime,BackupRetentionPeriod]'
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier "$P-db" --target-db-instance-identifier "$P-db-restored" \
  --restore-time 2026-11-25T09:11:00Z \
  --db-subnet-group-name "$P-db" --vpc-security-group-ids "$(terraform output -raw db_sg_id)" \
  --no-publicly-accessible --no-multi-az
aws rds wait db-instance-available --db-instance-identifier "$P-db-restored"     # ~10-20 min
aws ssm put-parameter --name "/$P/db/host" --type String --overwrite --value <restored endpoint>
aws rds delete-db-instance --db-instance-identifier "$P-db-restored" --skip-final-snapshot
# Local
docker compose exec -T db-primary pg_dump -U postgres -Fc app > backups/manual.dump
docker compose exec -T db-primary createdb -U postgres app_restored
docker compose exec -T db-primary pg_restore -U postgres -d app_restored --no-owner < backups/<file>.dump
```

(Local: the app and the writer connect to the database named `app`. To
"repoint" to `app_restored`, either rename the databases, or restore
into `app` after `DROP DATABASE`/`CREATE DATABASE`. Choose one and
explain why.)

Explain: Why didn't Multi-AZ (Task 3) protect you here? What decides
your RPO with PITR (AWS) and with dumps every 2 minutes (Local)? Your
measured RTO: which step took the longest, and how would you make it
shorter? What does `backup_retention_days` (7, 14 or 30 in your final
project variant) change, and what does it cost?

---

## Task 5 — Final Challenge: The Bill and the Review

The finance office sends last month's bill for the shop's AWS account —
**$212.97** — and asks two questions: "Why?" and "What will it cost
next month?" The head of IT asks for one document that shows the shop
is now available, secure and affordable.

The bill (an exercise, us-east-1 list prices):

| # | Line | Usage | Cost |
|---|---|---|---|
| 1 | EC2 `t3.micro` "shop-web" | 730 h | $7.59 |
| 2 | EBS gp2 8 GB, attached to shop-web | 8 GB-month | $0.80 |
| 3 | EBS gp2 100 GB "old-db-disk", **unattached** | 100 GB-month | $10.00 |
| 4 | Public IPv4: shop-web + one **unattached** Elastic IP | 1,460 h | $7.30 |
| 5 | NAT gateway "nat-test" (nothing in its private subnet) | 730 h + 20 GB | $33.75 |
| 6 | RDS `db.m5.large` PostgreSQL "loadtest-db" | 730 h | $129.94 |
| 7 | RDS `db.t3.micro` PostgreSQL "shop-db" + 20 GB gp2 | 730 h | $15.44 |
| 8 | RDS backup storage beyond the free amount | 30 GB-month | $2.85 |
| 9 | CloudWatch Logs: 10 GB ingested, 10 GB stored | | $5.30 |
| | **Total** | | **$212.97** |

Requirements

- For each line: keep, resize, delete or replace — with the new monthly
  cost and the reason. The total for next month **with today's
  architecture** (2 AZs or 3 from Task 2, Multi-AZ database from Task
  3), from your own estimate. Which lines were "hidden costs" from the
  lecture?
- AWS Implementation: prove that everything you run is tagged
  (`resourcegroupstaggingapi`), add the night/weekend schedule from the
  practice's Stretch (`schedule.tf`), and calculate the instance-hours it
  saves. Make the estimate in the **AWS Pricing Calculator** and put the
  share link in the report.
- Local Alternative: the same estimate on paper for "this Compose
  region, built on AWS" (which AWS service replaces each container?).
- `REVIEW.md` — one document for the head of IT:
  - the architecture diagram as it is **now**;
  - the game-day table: every drill from Tasks 2–4, expected vs measured;
  - the `audit.sh` output before and after (Task 1);
  - the cost: last month, next month, and the three biggest savings;
  - at least **three ADRs** (decision, alternatives, trade-off, cost):
    e.g. 3 AZs vs 2, Multi-AZ vs replica, PITR vs dumps, NAT vs public
    subnets, scheduling dev at night;
  - the risks you **accept** (from the SPOF table) and why.

Useful commands:

```bash
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values="$P" \
  --query 'ResourceTagMappingList[].ResourceARN' --output text | tr '\t' '\n'
aws ec2 describe-volumes --filters Name=status,Values=available --query 'Volumes[].[VolumeId,Size]'
aws ec2 describe-addresses --query 'Addresses[?AssociationId==`null`].PublicIp'
aws autoscaling describe-scheduled-actions --auto-scaling-group-name "$P-asg"
```

Explain: Which of your changes made the system **more** available and
**more** expensive at the same time? Which made it cheaper **and** safer?
If the business cut the budget to $40/month, what would you give up
first, and what would you never give up?

---

## Cleanup

AWS Implementation:

```bash
aws rds delete-db-instance --db-instance-identifier "$P-db-restored" --skip-final-snapshot 2>/dev/null
terraform destroy
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values="$P" \
  --query 'ResourceTagMappingList[].ResourceARN'
aws rds describe-db-instances --query 'DBInstances[].DBInstanceIdentifier'
```

Then **End Lab**.

Local Alternative:

```bash
docker compose down -v
docker image rm lab14ha-web
rm -f compose.override.yaml .env
docker ps -a --filter name=lab14ha      # nothing left
```

---

## Final Report (you can do it in free form)

Submit the GitHub repository:

```
GitHub Repository  lab14-review
│
├── (AWS)   *.tf, terraform.tfvars        (legacy.tf removed, 3 AZs, Multi-AZ, schedule.tf)
├── (Local) compose.yaml, haproxy.cfg, app/, db/   (web-c, the backup service)
├── audit.sh
├── failover.sh               (Local) / probe.json (AWS)
├── watch.sh
├── RUNBOOK.md
├── REVIEW.md
└── REPORT.md
```

Just an example of how you can show your report:

- Which variant you did (AWS or Local), and anything the Learner Lab
  didn't allow (action + error)
- For every task: the incident, the evidence (command output,
  screenshots), what you changed and why
- The audit findings with their proof, and the rotation measurement
  from Task 1
- The SPOF table from Task 2
- The failover and restore tables from Tasks 3 and 4 (RTO, RPO, what
  users saw)
- The bill analysis and the cost estimate from Task 5
- The answers to the **Explain** questions
- Problems you ran into and how you solved them
- **Never** a real password, key or token in the repository — the Task 1
  secrets must be rotated before you push (and say how you checked)
