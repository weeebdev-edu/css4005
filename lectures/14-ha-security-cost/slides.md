---
theme: seriph
title: "CSS4005 — Lecture 14: High Availability, Security and Cost Optimization"
info: |
  CSS4005 — Cloud Infrastructure Construction · Narxoz University
  Lecture 14 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# CSS4005 — Cloud Infrastructure Construction

## Lecture 14: High Availability, Security and Cost Optimization

<div class="pt-8 opacity-70">
Nauruzbayeva Farikha · Lesson 14
</div>

---
layout: default
---

# Recap — Lesson 13 (CI/CD)

<v-clicks>

- Why tag images with the commit SHA, not `:latest`? <span v-click class="opacity-60">(you always know which commit runs, and you can roll back to it)</span>
- Rolling vs blue-green: which needs twice the fleet? <span v-click class="opacity-60">(blue-green, during the switch; rolling needs one extra instance at most)</span>
- What starts the automatic rollback in Track A? <span v-click class="opacity-60">(the week 12 5xx alarm going to ALARM after the deploy)</span>
- Why is OIDC better than access keys in GitHub Secrets? <span v-click class="opacity-60">(short-lived credentials, nothing stored that can leak)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] Availability: nines, failure domains, RTO and RPO
- [ ] HA on AWS: the web tier across availability zones
- [ ] HA for data: RDS Multi-AZ, replicas, backups, disaster recovery
- [ ] Security: defence in depth, secrets, encryption, the security review
- [ ] Cost: how AWS bills, hidden costs, pricing models, budgets
- [ ] The trade-offs between the three → today's practice

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Registration week. At 10:05 one availability zone has a power problem.
The Narxoz Shop runs on one instance and one database — both in that
zone. The site is down for two hours.

The post-mortem finds more: the DB password is in the user data, the
database accepts connections from `0.0.0.0/0`, and last month's bill was
**$212** for a site that should cost about $50.

</div>

<div v-click class="mt-8 text-xl font-bold">
Today: survive a zone failure, close the holes — and know what it costs.
</div>

---
layout: section
transition: slide-left
---

# Block 1
## What "highly available" means

---
---

# Everything fails, all the time

<v-clicks>

- **A process** crashes (a bug, out of memory) — every day somewhere.
- **An instance** dies: the host's hardware fails, or AWS retires it.
- **A disk** is lost — EBS volumes fail rarely, but they do.
- **An availability zone** loses power, cooling or network.
- **A region** has a service outage (rare, but it has happened).
- **People**: a bad deploy, a wrong `terraform destroy`, a deleted table.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
"Everything fails, all the time" (Werner Vogels, Amazon CTO). High
availability doesn't mean nothing fails. It means <b>users don't notice</b>
when one part fails.
</div>

---
---

# Availability in "nines"

<div class="text-sm">

| Availability | Downtime per year | Per month | Typical for |
|---|---|---|---|
| 99% ("two nines") | 3.65 days | 7.3 hours | One instance, no redundancy |
| 99.9% | 8.8 hours | 44 minutes | A small web app in 2 AZs |
| 99.95% | 4.4 hours | 22 minutes | RDS Multi-AZ SLA |
| 99.99% | 53 minutes | 4.4 minutes | ALB (SLA), banking front-ends |
| 99.999% | 5 minutes | 26 seconds | Telecom, multi-region active-active |

</div>

<v-clicks>

- Each extra nine costs **much more** than the one before: more copies,
  more automation, more testing.
- An **SLA** (service level agreement) is a promise with a refund. Your
  **SLO** (objective) is the target you set for your own service.

</v-clicks>

---
---

# Availability math: in a row vs side by side

```text
 in a row (all must work):        ALB ─▶ web ─▶ DB
                                  99.99% × 99% × 99%      =  98.0 %

 side by side (one is enough):    web-a ┐
                                  web-b ┘  1 − (1 − 0.99)² =  99.99 %
```

<v-clicks>

- Every part **in a row** lowers availability: the chain is weaker than its
  weakest link.
- Two copies **side by side** multiply the *failure* chances: 1% × 1% =
  0.01%.
- That only works if the copies fail **independently**. Two instances on
  the same host, in the same AZ, or with the same bad deploy fail together.

</v-clicks>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
A <b>single point of failure</b> (SPOF) is any part with no copy. Find
them first: one instance, one DB, one NAT gateway, one person who knows
the password.
</div>

---
---

# Failure domains on AWS

```text
 Region us-east-1
 ┌───────────────────────────────────────────────────────────────┐
 │  AZ us-east-1a              AZ us-east-1b           AZ ...    │
 │  ┌────────────────────┐     ┌────────────────────┐            │
 │  │ data centre(s)     │     │ data centre(s)     │  separate  │
 │  │ own power, network │ ◀─▶ │ own power, network │  power and │
 │  │ instances, EBS     │ <2ms│ instances, EBS     │  flood risk│
 │  └────────────────────┘     └────────────────────┘            │
 └───────────────────────────────────────────────────────────────┘
```

<v-clicks>

- **Instance / host** → solved by more instances (an ASG).
- **Availability zone** → solved by spreading over **2–3 AZs** in one
  region. Fast links, so a synchronous DB copy is possible.
- **Region** → solved by a second region: much harder and costlier
  (data copies over long distances, DNS failover).
- Subnets live in **one** AZ: "multi-AZ" means subnets in several AZs.

</v-clicks>

---
---

# RTO and RPO

```text
              last good backup            failure                 service back
  ─────────────────●────────────────────────✖───────────────────────●──────▶ time
                   │◀────────── RPO ────────▶│◀──────── RTO ────────▶│
                     data written here is LOST   users see errors here
```

<v-clicks>

- **RPO** (recovery point objective): how much **data** you can lose,
  measured in time. "RPO = 5 min" → at most the last 5 minutes.
- **RTO** (recovery time objective): how long the service may be **down**.
- They come from the business, not from engineers: "registration may be
  down 15 minutes, grades may lose nothing".
- RPO decides **how** you copy data (nightly dump, PITR, sync replica);
  RTO decides how much is **already running** when disaster hits.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your web tier has 2 instances in 2 AZs (99% each), but one RDS
instance (99.5%) in one AZ. Roughly what availability does the user
get, and what should you fix first?
</div>

<div v-click class="mt-8 text-lg opacity-70">
About 99.99% × 99.5% ≈ 99.5% — the database is the weakest link in the
row. Adding a third web server changes nothing; a Multi-AZ database does.
</div>

---
layout: section
transition: slide-left
---

# Block 2
## The web tier across availability zones

---
---

# The multi-AZ web tier

```text
                       users
                         │
                ALB (nodes in BOTH AZs)        health check: GET /healthz
                 ┌───────┴────────┐
         AZ a    ▼                ▼    AZ b
      ┌────────────────┐  ┌────────────────┐
      │ public 10.14.0 │  │ public 10.14.1 │  ASG: min 1, desired 2, max 3
      │   web-1        │  │   web-2        │  subnets in both AZs
      ├────────────────┤  ├────────────────┤
      │ db 10.14.10    │  │ db 10.14.11    │  private: no internet route
      │ RDS primary    │─▶│ RDS standby    │  synchronous copy (Multi-AZ)
      └────────────────┘  └────────────────┘
```

<v-clicks>

- The **ALB** lives in every subnet you give it — it survives an AZ by
  itself (99.99% SLA).
- The **ASG** spreads instances evenly over its subnets and replaces
  unhealthy ones — in whichever AZ is still working.

</v-clicks>

---
---

# Health checks: what "healthy" should mean

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Liveness** — "is the process alive?"

- `/healthz` returns `ok` without touching the DB
- Used by the **ALB target group** and the ASG (`health_check_type = "ELB"`)
- A failed check = the instance is **replaced**

</div>
<div>

**Readiness / deep check** — "can it do real work?"

- Checks the DB, the cache, other services
- Good for **alarms and dashboards**
- Dangerous as the ALB check: when the DB is down, **every** instance
  fails it

</div>
</div>

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
If the health check needs the database, a 2-minute DB failover makes the
ASG terminate the whole fleet — and the new instances also fail the
check. A small DB problem becomes a full outage.
</div>

---
---

# When an AZ fails: the timeline

```text
 t=0      AZ a loses power. web-1 stops answering.
 t≈20s    ALB: 2 failed checks (interval 10 s) → web-1 "unhealthy" → no traffic
          users: only requests already sent to web-1 fail
 t≈30s    ASG sees the unhealthy target → terminates web-1
 t≈1min   ASG launches a replacement — in AZ b (AZ a has no capacity)
 t≈3-4min new instance passes the health check → back to 2 instances
```

<v-clicks>

- Between t=0 and the new instance, **web-2 carries all the traffic**.
- Can it? With 2 AZs × 1 instance, losing one AZ = losing **50%** of your
  capacity.
- During a real AZ outage, **everyone** launches in the other AZs at once:
  capacity can run short, and launches can be slow.

</v-clicks>

---
---

# Static stability: have the capacity *before* the failure

<div class="text-sm">

| You need | 2 AZs | 3 AZs |
|---|---|---|
| 2 instances to serve peak traffic | 2 + 2 = **4** (100% spare) | 1 + 1 + 1 = 3 (50% spare) |
| 6 instances | 6 + 6 = **12** | 3 + 3 + 3 = **9** |
| Left after losing one AZ | 50% | 67% |

</div>

<v-clicks>

- **Statically stable**: after losing an AZ, the rest still carries peak
  traffic **without launching anything** — nothing to wait for.
- Three AZs are cheaper than two for the same safety margin.
- The cheaper option: run N+1 and accept a few slow minutes while the
  ASG catches up. Decide **on purpose**, and write it down.

</v-clicks>

---
---

# The load balancer's part

<v-clicks>

- **Cross-zone load balancing** (always on for ALB): each ALB node sends
  traffic to targets in **all** AZs, so uneven instance counts are fine.
- **Deregistration delay** (`deregistration_delay = 30`): a target being
  removed finishes its open requests first — no cut-off responses during
  scale-in or a deploy.
- **Health check settings** = detection time: `interval × unhealthy_threshold`
  (10 s × 2 = 20 s). Faster checks find failures sooner, but can flap.
- The ALB doesn't retry a failed request on another target. **Clients**
  (browsers, your services) should retry idempotent requests with backoff.

</v-clicks>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
In today's Track B, HAProxy plays the ALB — and it <i>can</i> retry on
another server. You will measure the difference: errors with retries off
vs on.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your ASG has <code>desired_capacity = 2</code>, but both instances are in
<code>us-east-1a</code>. How can that happen, and why is it dangerous?
</div>

<div v-click class="mt-8 text-lg opacity-70">
<code>vpc_zone_identifier</code> lists only one subnet (or one AZ had no
capacity). One AZ failure takes both instances: the "two copies" fail
together. Always give the ASG subnets in at least two AZs — and check
where the instances really are.
</div>

---
layout: section
transition: slide-left
---

# Block 3
## High availability for data

---
---

# RDS Multi-AZ

<div class="text-sm">

| | Single-AZ | Multi-AZ **instance** | Multi-AZ **DB cluster** |
|---|---|---|---|
| Copies | 1 | Primary + 1 standby | Writer + 2 readable standbys |
| Replication | — | **Synchronous** (no data loss) | Semi-synchronous |
| Standby readable? | — | No | Yes |
| Failover | Restore / reboot: long | Typically **1–2 minutes** | Typically **under 35 s** |
| Cost | 1× | **2×** | ~3× |

</div>

<v-clicks>

- The **endpoint** (`lab14-db.xxxx.rds.amazonaws.com`) stays the same:
  on failover AWS points its DNS name at the standby.
- Multi-AZ also helps maintenance: patching happens on the standby first.
- Terraform: `multi_az = true`. Test it:
  `aws rds reboot-db-instance --force-failover`.

</v-clicks>

---
---

# Read replicas are not high availability

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Read replica**

- **Asynchronous** copy: seconds (or more) behind
- Readable: send reports and read-only traffic there
- Can be in **another region** (disaster recovery)
- Failover is **manual**: promote it, then repoint the app
- Data written since the last replicated change is **lost**

</div>
<div>

**Multi-AZ standby**

- **Synchronous**: a commit is on both before "OK"
- Not readable (instance deployment)
- Same region, another AZ
- Failover is **automatic**, same endpoint
- No committed data lost

</div>
</div>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
Replicas scale <b>reads</b>; Multi-AZ protects <b>availability</b>. A
self-hosted primary + replica (your final project, Track B today) behaves
like the left column: you promote and repoint.
</div>

---
---

# Failover from the application's side

<v-clicks>

- After a failover, the app's **open connections are broken**. It must
  reconnect — a connection pool that never retries keeps failing.
- The DNS name moves to the new primary: don't cache the IP, keep the DNS
  TTL short (RDS uses 5 s).
- **Repointing** a self-managed replica = changing one value the app reads:
  an SSM parameter `/lab14/db/host`, read on every check.
- Writes during the failover fail: retry them, or show "try again" —
  never pretend they succeeded.

</v-clicks>

```bash
# The fleet follows the parameter — no redeploy, no SSH
aws ssm put-parameter --name /lab14/db/host --overwrite \
  --type String --value lab14-db-replica.xxxx.us-east-1.rds.amazonaws.com
```

---
---

# Backups and point-in-time restore

<v-clicks>

- **Automated backups**: a daily snapshot + the transaction log every ~5
  minutes, kept `backup_retention_period` days (1–35). Gives
  **point-in-time restore** (PITR) to any second in that window.
- **Manual snapshots** stay until you delete them; copy them to another
  region or account for disaster recovery.
- A restore always creates a **new** instance with a new endpoint: repoint
  the app to it.
- **AWS Backup**: one backup plan for RDS, EBS, S3, DynamoDB…
- **Logical** backups (`pg_dump`) work everywhere and move between engines
  and versions; they're slower to restore.

</v-clicks>

<div v-click class="mt-4 p-4 rounded bg-red-500/10 text-sm">
A backup you have never restored is a hope, not a backup. Time the
restore: that number is your real RTO.
</div>

---
---

# Disaster recovery strategies

<div class="text-sm">

| Strategy | What runs in the second region | RPO | RTO | Cost |
|---|---|---|---|---|
| **Backup & restore** | Nothing — only backups copied there | Hours | Hours–a day | $ |
| **Pilot light** | The DB replica; servers ready to launch | Minutes | Tens of minutes | $$ |
| **Warm standby** | A small full copy, scaled up on failover | Seconds–minutes | Minutes | $$$ |
| **Multi-site active/active** | Full copies serving users in both | ~0 | ~0 | $$$$ |

</div>

<v-clicks>

- Choose by the business RPO/RTO — and by the budget.
- Most student projects (and many real companies) need **backup &
  restore** done well: automated, encrypted, copied away, **tested**.
- Infrastructure as Code is a DR tool: `terraform apply` in another region
  rebuilds everything except the data.

</v-clicks>

---
---

# Test it: game days

<v-clicks>

- A **game day** is a planned failure in a controlled moment: stop an
  instance, take away an AZ, fail over the DB, restore a backup.
- Before: write what you **expect** (errors, seconds, data lost). During:
  measure. After: compare and fix the gaps.
- **AWS Fault Injection Service (FIS)** runs such experiments for you:
  stop instances, add network latency, fail over RDS.
- Start small, in a test environment, with a "stop" button — and tell
  people first.

</v-clicks>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
Today's practice is a game day: you write down what you expect, cause
the failure, and measure what users actually saw.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
The business says: "Grades: RPO 0, RTO 5 minutes." Is a nightly
<code>pg_dump</code> to S3 enough? Is RDS Multi-AZ enough?
</div>

<div v-click class="mt-8 text-lg opacity-70">
The nightly dump: no — up to 24 hours of grades lost. Multi-AZ covers an
instance or AZ failure (synchronous, 1–2 min). But a wrong
<code>DELETE</code> is copied to the standby instantly: you still need
PITR to undo human mistakes.
</div>

---
layout: section
transition: slide-left
---

# Block 4
## Security

---
---

# Shared responsibility, once more

<div class="text-sm">

| | EC2 | RDS | S3 |
|---|---|---|---|
| Hardware, data centres, the hypervisor | AWS | AWS | AWS |
| OS patches | **You** | AWS | AWS |
| Database engine patches | **You** (self-hosted) | AWS (in your maintenance window) | — |
| Network access (security groups) | **You** | **You** | Bucket policy: **you** |
| Encryption switched on | **You** | **You** (at creation!) | On by default (SSE-S3) |
| Who can log in, IAM, passwords | **You** | **You** | **You** |

</div>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
Managed services move work to AWS, but <b>configuration is always
yours</b>. Most cloud breaches are a configuration mistake: a public
bucket, an open port, a leaked key.
</div>

---
---

# Defence in depth

```text
  internet ─▶ [ edge ]      WAF rules, Shield (DDoS), only 80/443 open
           ─▶ [ network ]   public subnet: ALB only · private: DB · SG references
           ─▶ [ identity ]  IAM roles, least privilege, no long-lived keys
           ─▶ [ compute ]   IMDSv2, patched AMIs, no SSH (Session Manager)
           ─▶ [ data ]      encrypted at rest (KMS) and in transit (TLS)
              [ detect ]    CloudTrail, GuardDuty, Config, alarms   ← across all layers
```

<v-clicks>

- Every layer assumes the one before it **can fail**. A leaked password
  is useless if the DB only accepts connections from the web servers.
- Today's fleet gets one change on every layer: private DB subnets, an SG
  reference, the password in SSM, IMDSv2, encrypted disks, TLS to the DB.

</v-clicks>

---
---

# Network: who can reach whom

```hcl
# The DB accepts PostgreSQL only from instances that carry the web SG
resource "aws_vpc_security_group_ingress_rule" "db_from_web" {
  security_group_id            = aws_security_group.db.id
  referenced_security_group_id = aws_security_group.web.id   # not a CIDR
  ip_protocol                  = "tcp"
  from_port                    = 5432
  to_port                      = 5432
}
```

<v-clicks>

- **SG references** follow instances: the ASG launches a new IP — the rule
  still matches. A CIDR would let anything in that range in.
- DB subnets with a route table that has **only the local route**, and
  `publicly_accessible = false`: no way in from the internet, no way out.
- Never `0.0.0.0/0` on 22, 3389, 5432, 3306. Admin access: **SSM Session
  Manager** (no port, logged in CloudTrail).

</v-clicks>

---
---

# Identity: least privilege

<v-clicks>

- **Roles, not keys**: instances get credentials from an instance profile,
  CI from OIDC. Keys that never exist can't leak.
- **Least privilege**: `ssm:GetParameter` on `/lab14/*` — not `ssm:*` on
  `*`. Start small, add what's missing (CloudTrail shows the denied calls).
- Root account: MFA, no access keys, used only for billing emergencies.
- **IAM Access Analyzer** finds resources shared outside your account and
  can generate a policy from what a role really used.
- Learner Lab: you can't create roles — `LabRole` is broad. In your own
  account, write the narrow policy.

</v-clicks>

```json
{ "Effect": "Allow",
  "Action": ["ssm:GetParameter"],
  "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/lab14/db/*" }
```

---
---

# Secrets: where the password lives

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Never here**

- Git (even "temporary", even in a private repo)
- User data — anyone with `ec2:DescribeInstanceAttribute` reads it
- Environment variables in task definitions, `docker inspect`, logs
- AMIs and container images

</div>
<div>

**Here**

- **SSM Parameter Store** `SecureString`: encrypted with KMS, free
  (standard), read by the role at runtime
- **Secrets Manager**: + automatic **rotation**, cross-account sharing;
  $0.40 per secret per month
- The app gets the **name** of the secret, never the value

</div>
</div>

```bash
# On the instance, with the instance role — no key, no password in the script
aws ssm get-parameter --name /lab14/db/password --with-decryption \
  --query Parameter.Value --output text
```

<div v-click class="mt-2 text-sm opacity-70">
A Terraform-generated password is still in the <b>state file</b>: keep
state in an encrypted, private bucket (week 13).
</div>

---
---

# Encryption: at rest and in transit

<v-clicks>

- **At rest — KMS**: EBS (`encrypted = true` in the launch template), RDS
  (`storage_encrypted = true`), S3 (SSE-S3 by default since 2023).
- RDS encryption can be set **only when the instance is created**. Later:
  snapshot → copy the snapshot encrypted → restore a new instance.
- AWS managed keys (`aws/rds`, `aws/ebs`) are free; a **customer managed
  key** ($1/month) lets you control who may use it and rotate it.
- **In transit — TLS**: HTTPS on the ALB with a certificate from **ACM**
  (free for public certificates); `sslmode=require` to the database
  (RDS PostgreSQL 15+ refuses unencrypted connections by default).

</v-clicks>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
Encryption at rest protects against a lost disk or a copied snapshot. It
does <b>not</b> protect against someone who can log in to the database —
that's what IAM, SGs and passwords are for.
</div>

---
---

# Instance hardening: IMDSv2

```text
 IMDSv1:   attacker ─▶ your app (SSRF bug: "fetch this URL for me")
                       ─▶ GET http://169.254.169.254/.../security-credentials/LabRole
                       ◀─ the role's temporary keys
 IMDSv2:   needs a PUT for a session token first — an SSRF GET can't do it
```

<v-clicks>

- 2019: an SSRF bug + IMDSv1 → stolen role credentials → data of
  **100 million** bank customers copied from S3.
- Terraform: `metadata_options { http_tokens = "required" }` — on by
  default for new AL2023 AMIs, but say it explicitly.
- Also: patched AMIs (a new launch template version = a patch), no SSH
  keys, the app runs as a non-root user, **SSM Patch Manager** for
  long-lived servers.

</v-clicks>

---
---

# Detect: knowing what happened

<div class="text-sm">

| Service | Answers | Cost |
|---|---|---|
| **CloudTrail** | Who called which API, when, from where | 90 days of events free |
| **AWS Config** | What did this resource look like last week? Is it compliant? | Per recorded change |
| **GuardDuty** | Is something behaving like an attacker? (odd API calls, crypto-mining) | 30-day free trial |
| **Security Hub** | One list of findings, checked against best-practice standards | Per check |
| **Inspector** | Known vulnerabilities (CVEs) in instances, images, Lambda | Per resource |
| **Trusted Advisor** | Open ports, no MFA on root, idle resources | Basic checks free |

</div>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
Learner Lab blocks several of these. In your own account, turn on
CloudTrail and GuardDuty on day one — they cost little and you can't
investigate the past without them.
</div>

---
---

# The security review checklist

<div class="text-sm">

| Layer | Check | How to prove it |
|---|---|---|
| Network | DB not public, SG allows 5432 only from web SG | `describe-db-instances` → `PubliclyAccessible: false`; SG rules |
| Network | Nothing open to `0.0.0.0/0` except 80/443 on the ALB | `describe-security-groups` |
| Identity | No access keys in code, CI uses roles/OIDC | `gitleaks`, repository secrets |
| Secrets | Password in SSM/Secrets Manager, not in user data | Decode the user data: no password |
| Compute | IMDSv2 required, image scanned | `MetadataOptions.HttpTokens: required`; Trivy |
| Data | Encrypted EBS and RDS, TLS to the DB | `Encrypted: true`, `StorageEncrypted: true` |
| Detect | CloudTrail on, an alarm on errors | Console / week 12 alarms |

</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
A developer says: "The DB port is open to <code>0.0.0.0/0</code>, but the
password is 24 random characters, so it's fine." What's wrong with this
argument?
</div>

<div v-click class="mt-8 text-lg opacity-70">
It depends on one layer only. Passwords leak (git, logs, a laptop), the
engine may have a bug, and anyone can try forever. With an SG reference,
a leaked password is useless from outside the VPC.
</div>

---
layout: section
transition: slide-left
---

# Block 5
## Cost optimization

---
---

# How AWS bills — and the costs nobody sees

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**The obvious**

- Instances: per second, while **running**
- RDS: per hour, **×2** for Multi-AZ, + storage per GB-month
- ALB: per hour **+ LCUs** (traffic), even with no users
- EBS: per GB-month, attached **or not**

</div>
<div>

**The hidden (us-east-1)**

- **Public IPv4**: $0.005/hour each — instances, ALB, NAT
- **NAT gateway**: $0.045/hour + $0.045 per GB through it
- **Cross-AZ traffic**: $0.01/GB in each direction
- Old snapshots, unattached volumes, idle Elastic IPs
- CloudWatch Logs kept forever

</div>
</div>

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
The $212 bill from the scenario: a forgotten NAT gateway (~$33/month
before traffic), a <code>db.m5.large</code> left from a load test
(~$130), and the real site — about $49.
</div>

---
---

# What today's architecture costs (estimate)

<div class="text-sm">

| Resource (us-east-1, on-demand, 730 h) | Single-AZ DB | Multi-AZ DB |
|---|---|---|
| 2 × EC2 `t3.micro` ($0.0104/h) | $15.18 | $15.18 |
| 2 × 8 GB gp3 root disk ($0.08/GB-month) | $1.28 | $1.28 |
| ALB ($0.0225/h, + ~1 LCU) | $16.43 + ~$5.84 | $16.43 + ~$5.84 |
| 4 public IPv4 (2 instances + 2 ALB nodes) | $14.60 | $14.60 |
| RDS `db.t3.micro` PostgreSQL ($0.018/h, ×2 Multi-AZ) | $13.14 | $26.28 |
| RDS 20 GB gp3 ($0.115/GB-month, ×2 Multi-AZ) | $2.30 | $4.60 |
| **Total per month** | **≈ $69** | **≈ $84** |

</div>

<div v-click class="mt-4 text-sm opacity-70">
List prices at the time of writing — check them in the <b>AWS Pricing
Calculator</b> before you trust them. Backups up to the DB size are
free; SSM standard parameters are free.
</div>

---
---

# Pricing models

<div class="text-sm">

| Model | What you promise | Discount vs on-demand | Good for |
|---|---|---|---|
| **On-demand** | Nothing | — | Spiky, short, unknown workloads |
| **Savings Plans** | $X/hour of compute for 1 or 3 years | up to ~72% | A steady base load |
| **Reserved Instances** | One instance type (RDS too) for 1 or 3 years | up to ~72% | Databases that run 24/7 |
| **Spot** | Nothing — AWS can take it back with 2 min notice | up to ~90% | Batch jobs, CI runners, stateless web extras |

</div>

<v-clicks>

- Typical mix: Savings Plan for the minimum fleet, on-demand for the
  peaks, Spot for what can be interrupted (`mixed_instances_policy`).
- Commit only after **right-sizing** — a discount on a too-big instance
  is still waste.

</v-clicks>

---
---

# Right-size, and use the newer option

<v-clicks>

- **Right-sizing**: look at CPU and memory for two weeks (week 12's
  metrics). 10% average CPU on an `m5.large`? Try `t3.medium`.
  **Compute Optimizer** does this analysis for you.
- **Graviton** (ARM: `t4g`, `m7g`): `t4g.micro` is ~20% cheaper than
  `t3.micro` for the same size, and often faster. Needs an `arm64` image
  (week 13: multi-arch builds).
- **gp3 instead of gp2**: ~20% cheaper per GB, and IOPS don't depend on
  the size.
- **Burstable** `t` instances earn CPU credits when idle. Constant high
  CPU → credits run out (or `unlimited` mode bills extra).
- Managed vs self-hosted: RDS costs more per hour than a DB on EC2 — but
  backups, patching and failover are not free when **you** do them.

</v-clicks>

---
---

# Turn it off when nobody uses it

```hcl
# Scale in at night, out in the morning (Almaty time) — today's stretch
resource "aws_autoscaling_schedule" "night" {
  scheduled_action_name  = "${var.prefix}-night"
  autoscaling_group_name = aws_autoscaling_group.web.name
  recurrence             = "0 20 * * *"
  time_zone              = "Asia/Almaty"
  min_size = 1
  max_size = 1
  desired_capacity = 1
}
```

<v-clicks>

- A dev environment used 08:00–20:00 on weekdays runs 60 of 168 hours a
  week: switching it off the rest saves **~64%**.
- RDS can be **stopped** for up to 7 days (then AWS starts it again).
- S3 lifecycle rules: old objects to cheaper classes, then expire them.
- Log retention: `retention_in_days = 14`, not "never expire".
- **Delete** what you don't need: `terraform destroy` is the best saving.

</v-clicks>

---
---

# See the bill: tags, Cost Explorer, Budgets

<v-clicks>

- **Tags** on everything: `Project`, `Owner`, `Environment`. Activate them
  as **cost allocation tags**, then Cost Explorer groups the bill by them.
- `default_tags` in the provider tags every resource Terraform creates —
  but **not** the instances an ASG launches: tag them in the launch
  template (`tag_specifications`).
- **AWS Budgets**: a monthly limit with alerts — on the **forecast**
  (early warning) and on the **actual** spend.
- **Cost Anomaly Detection**: "EC2 spend is 4× the usual today".
- Learner Lab: no Budgets — watch the lab's spend counter instead.

</v-clicks>

```bash
# Everything tagged Project=lab14 — is anything left after destroy?
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values=lab14 \
  --query "ResourceTagMappingList[].ResourceARN"
```

---
---

# The price of availability

<div class="text-sm">

| Change | Protects against | Extra cost / month (today's numbers) |
|---|---|---|
| 2nd web instance in another AZ | Instance or AZ failure | +$7.60 (+ $3.65 IPv4) |
| RDS Multi-AZ | DB instance or AZ failure, patching downtime | +$15.44 (×2 DB) |
| 3 AZs instead of 2 | Lower spare capacity for static stability | +1 instance |
| Second region (warm standby) | A region outage | Roughly +50–100% |
| Longer backup retention (7 → 30 days) | Mistakes found late | Snapshot storage above DB size |

</div>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
There is no "most available" answer — only the right one for the
business. Write the trade-off as an <b>ADR</b>: what you chose, what you
gave up, what it costs.
</div>

---
---

# Well-Architected: three pillars, one design

<div class="grid grid-cols-3 gap-4 text-sm">
<div>

**Reliability**

- Multiple AZs, no SPOF
- Automatic recovery (ASG, Multi-AZ)
- Backups, tested restores
- Game days

</div>
<div>

**Security**

- Least privilege, roles
- Defence in depth
- Encryption everywhere
- Traceability (CloudTrail)

</div>
<div>

**Cost optimization**

- Pay for what you use
- Right-size, then commit
- Tags and budgets
- Delete what's idle

</div>
</div>

<v-clicks>

- They pull against each other: Multi-AZ doubles the DB bill; a NAT
  gateway improves security but costs $33/month; turning dev off at
  night saves money but means a slower morning.
- The other pillars: operational excellence, performance efficiency,
  sustainability. The **Well-Architected Tool** asks the questions for you.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your instances in public subnets need to call the internet. A colleague
moves them to private subnets with a NAT gateway in each AZ "for
security". What does that add to the bill, and what do you get?
</div>

<div v-click class="mt-8 text-lg opacity-70">
2 × ($0.045 × 730) ≈ $66/month + $0.045/GB, minus the instances' IPv4
cost. You get instances with no public address. Alternatives: one NAT
(cheaper, but an AZ SPOF), or VPC endpoints for S3/SSM traffic.
</div>

---
layout: section
transition: slide-left
---

# Block 6
## Today's practice

---
---

# Today's practice — the game day

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Track A — Learner Lab**

1. A two-AZ fleet: ALB + ASG + RDS in private DB subnets
2. You write: the DB security group rule, and the "AZ down" subnet filter
3. Password in SSM `SecureString`, read by `LabRole`; IMDSv2; encrypted disks
4. Game day 1: take an AZ away — measure the errors
5. Game day 2: RDS Multi-AZ forced failover — measure the downtime
6. Security review + tag-based cost check, then **destroy**

</div>
<div>

**Track B — no AWS**

The same ideas in Docker Compose: HAProxy as the ALB, two app "zones",
a PostgreSQL primary + streaming replica, secrets as files.

Kill a zone (retries off vs on), freeze a zone, crash a process, fail
the database over, take a backup.

<div v-click class="mt-4 opacity-70">
Checked orally: your expectations vs measurements, the failover, the
security checks and the cost table.
</div>

</div>
</div>

---
---

# Your final project

<v-clicks>

- **Zone drill** (R12): stop a node or a zone, show pods/instances moving
  and what users saw.
- **Database** (R7): primary + replica in 2 zones, backups that meet your
  `rpo`, a **timed** restore test and failover drill vs your `rto`.
- **Security**: 3 DB roles with denied actions, secrets out of Git, the
  checklist from today with evidence.
- **Cost** (R12): a Pricing Calculator estimate, the actual spend, a budget
  alert at your `monthly_budget_usd`, ≥ 3 optimizations, teardown proof.
- Write the trade-offs as **ADRs**: Multi-AZ or not, NAT or not, managed
  or self-hosted DB.

</v-clicks>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Explain availability in nines, SPOFs, failure domains, RTO and RPO
- [ ] Design a web tier that survives losing an AZ, and say how much spare capacity it needs
- [ ] Choose between Multi-AZ, read replicas, backups and DR strategies for an RPO/RTO
- [ ] Apply defence in depth: SG references, private subnets, least privilege, IMDSv2
- [ ] Keep secrets out of code and user data; encrypt at rest and in transit
- [ ] Estimate a monthly bill, find the hidden costs, and pick a pricing model
- [ ] Use tags, schedules and budgets to keep the bill under control

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Finish today's practice: **`terraform destroy`**, check that nothing
      tagged `Project=lab14` is left, then **End Lab** (Track A) — or the
      Track B cleanup
- [ ] In your final project: write the RPO/RTO from your variant into the
      report, and plan the zone drill and the restore test
- [ ] Make the cost estimate for your project in the Pricing Calculator
      and set the budget alert **before** you build

<div class="mt-8 text-sm opacity-60">
Next week: the final project — everything from weeks 1–14 in one platform.
</div>

---
layout: end
---

# Next lecture

Final Cloud Infrastructure Project — build, defend and tear down your
platform.
