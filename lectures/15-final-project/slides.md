---
theme: seriph
title: "CSS4005 — Lecture 15: Final Cloud Infrastructure Project"
info: |
  CSS4005 — Cloud Infrastructure Construction · Narxoz University
  Lecture 15 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# CSS4005 — Cloud Infrastructure Construction

## Lecture 15: Designing and Defending a Production Cloud Architecture

<div class="pt-8 opacity-70">
Nauruzbayeva Farikha · Lesson 15
</div>

---
layout: default
---

# Recap — Lesson 14 (HA, security, cost)

<v-clicks>

- Two instances in one AZ, or one in each of two AZs? <span v-click class="opacity-60">(two AZs: one failure domain can't take both)</span>
- RPO vs RTO? <span v-click class="opacity-60">(RPO: how much data you may lose; RTO: how long you may be down)</span>
- A read replica or Multi-AZ for availability? <span v-click class="opacity-60">(Multi-AZ fails over by itself; a replica must be promoted)</span>
- What bills when nobody uses the site? <span v-click class="opacity-60">(load balancer, NAT gateway, public IPv4, disks: by the hour or GB-month)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] The reference architecture of the final project, end to end
- [ ] Your variant as a requirements document: every field → a decision
- [ ] Network, cluster, database, backups, secrets: the design choices
- [ ] Writing it down: diagram, ADRs, cost estimate, teardown
- [ ] How it is graded, and what the defense looks like
- [ ] Common failures → today's practice → course wrap-up

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Defense day. Aidana's platform works: the site answers through the
load balancer, the pods spread over two zones. The examiner opens her
variant: <code>rpo = 15 min</code>. Then her backup script: one
<code>pg_dump</code>, every night at 03:00.

"The primary's disk dies at 02:50. How much data do you lose?"

</div>

<div v-click class="mt-8 text-xl font-bold">
Today: a design is a list of decisions, each one tied to a requirement —
and you must be able to defend every one.
</div>

---
layout: section
transition: slide-left
---

# Block 1
## The reference architecture

---
---

# What "production-grade" means in this project

<div class="text-sm">

| Area | Must have | Week |
|---|---|---|
| Network | Your CIDR, 3 tiers × 2 zones, SG chaining, a second layer | 4 |
| Compute + cluster | Templates + startup scripts, k8s over 2 zones, HPA | 2–3, 9–10 |
| Database | **Self-hosted** primary + replica, 3 roles, password policy | 7 |
| Storage + backups | Bucket with versioning, lifecycle, snapshots, logical dumps | 5 |
| Identity + secrets | Instance role, secret manager, nothing secret in Git | 6 |
| Delivery | Terraform modules, CI: lint → test → build → scan → deploy | 11, 13 |
| Operations | Dashboards, ≥ 4 alerts, logs by `request_id`, drills, cost | 12, 14 |

</div>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
The app is given. What is graded is the <b>platform</b> around it — and
your evidence that it works.
</div>

---
---

# The reference architecture

```text
                  Internet ── 80/443 only
                      │
   ┌──────────────────▼─────────────── VPC 10.X.0.0/16 ──────────────────────┐
   │   ALB  (public 10.X.1.0/24 + 10.X.2.0/24)          NAT ─► internet      │
   │      │ NodePort / Traefik, health check /readyz                         │
   │   ┌──▼──────── zone a ──────┐        ┌──────── zone b ──────────┐       │
   │   │ app 10.X.11.0/24        │        │ app 10.X.12.0/24         │       │
   │   │ k3s server + agent      │        │ k3s agent    (ASG)       │       │
   │   │ pods: campus-app ×N     │        │ pods: campus-app ×N      │       │
   │   ├─────────────────────────┤        ├──────────────────────────┤       │
   │   │ db 10.X.21.0/24         │ stream │ db 10.X.22.0/24          │       │
   │   │ DB primary (+ClickHouse)├───────►│ DB replica               │       │
   │   └─────────────────────────┘        └──────────────────────────┘       │
   └─────────────────────────────────────────────────────────────────────────┘
   S3: attachments + backups   ECR: images   Secrets Manager / SSM   CloudWatch
   GitHub Actions ─► build, scan, push ─► kubectl rollout    Terraform ─► all of it
```

<div class="text-sm opacity-70 mt-2">
GCP track: VPC network, MIG, HTTP(S) LB, Cloud NAT, GCS, Artifact Registry,
Secret Manager, Cloud Monitoring — the same boxes, other names.
</div>

---
---

# One request, end to end

```text
browser ─▶ ALB :443 ─▶ target group (nodes, port 30080) ─▶ Traefik ingress
        ─▶ Service campus-app ─▶ a Ready pod (zone a or b)
        ─▶ DB primary :5432  (SG: only from the app SG)
        ─▶ S3 attachment (instance role, presigned URL back to the browser)
```

<v-clicks>

- Every arrow is a **rule** you wrote: listener, target group, NodePort,
  Ingress, Service selector, SG reference, bucket policy.
- Only **Ready** pods are endpoints: readiness on `/readyz` keeps a pod
  without a database out of rotation; liveness on `/healthz` restarts a
  stuck one.
- At the defense you must be able to walk this path — and say what
  breaks at each arrow.

</v-clicks>

---
---

# Learner Lab or GCP: what changes

<div class="text-sm">

| | AWS Academy Learner Lab | Own AWS / GCP account |
|---|---|---|
| Roles | Only `LabRole` / `LabInstanceProfile`; write the policy JSON you **would** attach | Your own least-privilege role / service account |
| CI credentials | Session keys in GitHub Secrets (~4 h) or a self-hosted runner | OIDC / Workload Identity Federation |
| Secrets | Secrets Manager, or SSM `SecureString` if blocked | Secrets Manager / Secret Manager |
| Budget alert | Not available: show the lab's spend counter | AWS Budgets / GCP budget at your cap |
| Sessions | Instances **stop** after the session: design for restart | Running until you destroy |
| Region | `us-east-1` (or `us-west-2`) | One region of your choice |

</div>

<div v-click class="mt-4 text-sm opacity-70">
A Learner Lab limitation that is <b>documented</b> (what, why, real-account
design) costs no points.
</div>

---
layout: section
transition: slide-left
---

# Block 2
## The variant is your requirements document

---
---

# Variant → decisions (1): network and compute

<div class="text-sm">

| Field | Example | Decision it forces |
|---|---|---|
| `resource_prefix` | `ci-example-0042` | Every name + `Project` tag; one `var.resource_prefix` in Terraform |
| `vpc_cidr` | `10.42.0.0/16` | `cidrsubnet()` → six /24s; no overlap with a classmate (VPC peering later) |
| `cluster` | `k3s-on-autoscaled-vms` | k3s server + agent **ASG** (min 2, 2 AZs), join token from the secret store |
| `traffic_pattern` | `spiky` | Target tracking on the ASG; HPA reacts first; a load test with bursts |
| `hpa_cpu_target` | `60` | HPA `averageUtilization: 60` — needs CPU **requests** on the pods |

</div>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
<code>scheduled-peaks</code> → scheduled actions (scale out before 09:00 Almaty time)
+ target tracking. <code>steady</code> → a modest target, fewer moving parts.
<code>k3s-fixed-3-nodes</code> → no node scaling: a written capacity plan.
</div>

---
---

# Variant → decisions (2): data

<div class="text-sm">

| Field | Example | Decision it forces |
|---|---|---|
| `db_engine` | `postgresql` | Streaming replication; `pg_dump`; port 5432 in the DB SG |
| `clickhouse_required` | `yes` | A ClickHouse VM in the db tier, `CLICKHOUSE_URL` from the secret store |
| `rpo` | `15 min` | Nightly dumps are **not** enough: replica + WAL/binlog shipped to S3 ≤ 15 min |
| `rto` | `1 h` | A restore you have **timed**: VM from Terraform + restore + repoint < 1 h |
| `backup_retention_days` | `14` | S3 lifecycle `Expiration: 14`; snapshot policy keeps 14 days |
| `password_min_length` / `_require_symbol` | `14` / `yes` | DB policy + app checker + a CI test, same numbers in all three |

</div>

<div v-click class="mt-4 p-4 rounded bg-red-500/10 text-sm">
The variant is applied only if you can <b>show</b> it: a screenshot, an
output or a file line for every row — the report's first table.
</div>

---
---

# Variant → decisions (3): operations and money

<div class="text-sm">

| Field | Example | Decision it forces |
|---|---|---|
| `alert_cpu_pct` | `80` | CloudWatch alarm on node CPU > 80 % — **above** the scaling target |
| `alert_5xx_per_min` | `10` | ALB `HTTPCode_Target_5XX_Count` Sum / 60 s > 10, `notBreaching` |
| `monthly_budget_usd` | `25` | Budget alert at $25; instance sizes, one NAT or none, destroy between sessions |

</div>

<v-clicks>

- Scaling target vs alarm: the HPA keeps CPU near **60 %**. An alarm at
  80 % means "scaling is not keeping up" — a human should look.
- An alarm **below** the target (e.g. 50 % with a target of 70 %) fires
  every busy afternoon: people learn to ignore it.
- Same thresholds in Terraform variables: `var.alert_cpu_pct`, never a
  hard-coded `80` the examiner has to hunt for.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Variant: <code>hpa_cpu_target = 70</code>, <code>alert_cpu_pct = 60</code>.
You set both exactly as given. What will you see during the load test,
and what do you write in the report?
</div>

<div v-click class="mt-8 text-lg opacity-70">
The CPU alarm fires before the HPA even starts to scale. Keep the variant
values (they're required) and explain in an ADR: the alarm warns of
<i>sustained</i> load (e.g. 3 × 5 min), or watch a second signal — node CPU
vs pod CPU.
</div>

---
layout: section
transition: slide-left
---

# Block 3
## Network, cluster, scaling

---
---

# The subnet plan from your CIDR

```hcl
locals {
  tier_base = { public = 0, app = 10, db = 20 }
  subnets = merge([for tier, base in local.tier_base : {
    for i, az in var.azs : "${tier}-${az}" => {
      tier = tier, az = az, cidr = cidrsubnet(var.vpc_cidr, 8, base + i + 1)
    }
  }]...)
}
```

<div class="text-sm">

| Tier | us-east-1a | us-east-1b | Route |
|---|---|---|---|
| public | 10.42.1.0/24 | 10.42.2.0/24 | `0.0.0.0/0` → IGW |
| app | 10.42.11.0/24 | 10.42.12.0/24 | `0.0.0.0/0` → NAT |
| db | 10.42.21.0/24 | 10.42.22.0/24 | `0.0.0.0/0` → NAT (updates only) |

</div>

---
---

# Security chaining and the second layer

<div class="text-sm">

| SG | Inbound | From |
|---|---|---|
| `<prefix>-lb` | 80, 443 | `0.0.0.0/0` — the **only** such rule |
| `<prefix>-app` | 30080 (ingress NodePort) | SG `<prefix>-lb` |
| `<prefix>-app` | 6443, 8472/udp, 10250 (k3s) | SG `<prefix>-app` (nodes talk to nodes) |
| `<prefix>-db` | 5432 / 3306 | SG `<prefix>-app` |
| `<prefix>-db` | 5432 / 3306 (replication) | SG `<prefix>-db` |

</div>

<v-clicks>

- By **SG reference**, not by `10.42.0.0/16`: a new node is trusted because
  of its SG, and the public tier is not.
- Second layer: a **NACL** on the db subnets (allow the DB port from the
  app CIDRs, ephemeral return ports, deny the rest). Stateless — forget
  the return ports and replication breaks.
- Evidence: `nc -vz <db-ip> 5432` from a public-tier host **times out**.

</v-clicks>

---
---

# NAT: two, one or none?

<div class="text-sm">

| Option | Cost (us-east-1, 730 h) | Zone failure | Fits a $15–40 budget? |
|---|---|---|---|
| 2 NAT gateways (one per AZ) | 2 × $32.85 + 2 IPv4 $7.30 + $0.045/GB | Each zone keeps its egress | Not 24/7 |
| **1 NAT gateway** | $32.85 + IPv4 $3.65 + $0.045/GB | Zone a down → zone b loses egress (not ingress!) | Only if destroyed between sessions |
| No NAT, VPC endpoints for S3/ECR | S3 gateway endpoint free; interface endpoints ~$7.30 each/AZ | No egress dependency | Yes, but no `dnf update` |
| Create NAT only while building | Hours × $0.045 | — | Yes |

</div>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
This is a classic ADR: the README allows one NAT <b>if you explain the
trade-off</b>. "Users still reach the site, but nodes in zone b can't pull
new images" is the sentence the examiner wants to hear.
</div>

---
---

# Your `cluster` variant

<div class="text-sm">

| | `k3s-on-autoscaled-vms` | `k3s-fixed-3-nodes` | `managed-k8s-if-credits` |
|---|---|---|---|
| Control plane | 1 k3s server VM | 1 server + 2 agents, or 3 servers | EKS / GKE ($0.10/h on EKS) |
| Nodes | Agent **ASG / MIG**, min 2, 2 AZs | 3 VMs from one template, 2 AZs | Node group / node pool, 2 AZs |
| Node scaling | ASG target tracking | None: capacity plan in an ADR | Cluster autoscaler |
| Join | Token read **at boot** from the secret store | Same | Managed |
| Learner Lab | Works | Works | Fall back to the first, say so in an ADR |

</div>

<v-clicks>

- Single k3s server = a single point of failure for the **API**, not for
  traffic: running pods keep serving. Say that in an ADR — or run 3 servers.
- The database never goes into the cluster: VMs in the db subnets.

</v-clicks>

---
---

# The ALB in front of k3s

```text
ALB :443 (ACM / self-signed cert) ─▶ target group  port 30080, health check /readyz
                                      │  targets: every node (the agent ASG registers itself)
                                      ▼
              Traefik (k3s ingress) NodePort 30080 ─▶ Ingress ─▶ Service ─▶ Ready pods
```

<v-clicks>

- `target_group_arns` on the agent ASG: new nodes join the target group
  by themselves; terminated nodes drain.
- The health check goes **through** the ingress to the app's `/readyz`:
  a node is healthy only if it can route to a Ready pod.
- `externalTrafficPolicy: Cluster` (default): any node can forward to a
  pod on another node — a node with no local pod still passes.
- No domain? A self-signed certificate imported into ACM; document the
  browser warning and why a real domain fixes it.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your one NAT gateway is in us-east-1a, and that zone goes down. The
ALB, the nodes in 1b and the DB replica are fine. What do users see?
What stops working?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Users see nothing: traffic comes in through the ALB, not the NAT.
What stops: outbound traffic from zone b — image pulls, updates, and
calls to AWS APIs without a VPC endpoint (backups to S3!).
</div>

---
layout: section
transition: slide-left
---

# Block 4
## Data: database, backups, RPO and RTO

---
---

# RPO and RTO on one timeline

```text
 last good copy            failure            detected        service back
       │◄─────── RPO ───────►│◄──────────────── RTO ─────────────►│
 ──────●─────────────────────✕──────────────────●──────────────────●──────▶ time
       backup / WAL segment      data written here     decide, promote or
       / replica position        is LOST               restore, repoint app
```

<v-clicks>

- **RPO** is decided by *how often* data leaves the primary: replication
  (seconds), WAL / binlog shipping (minutes), dumps (hours).
- **RTO** is everything after the failure: noticing (alarms!), deciding,
  the technical steps, repointing the app, checking it works.
- Week 14 showed the drills. In the project you **measure** both and put
  the numbers next to your variant.

</v-clicks>

---
---

# Which mechanism meets which RPO?

<div class="text-sm">

| Mechanism | Typical data loss | Protects against | Meets `rpo` |
|---|---|---|---|
| Async replica in the other AZ | Seconds (replication lag) | Primary VM / AZ loss — **not** `DROP TABLE` | 15 min, 1 h, 4 h |
| WAL archive / binlogs to S3 every ≤ 15 min (`archive_timeout = 900`) | ≤ 15 min | Everything, incl. human error (point in time) | 15 min, 1 h, 4 h |
| EBS snapshots (DLM: at most every 1 h) | ≤ interval | Disk loss; crash-consistent | 1 h, 4 h |
| `pg_dump` / `mysqldump` to S3 by cron | ≤ interval | Human error; portable | 4 h if every ≤ 4 h |

</div>

<div v-click class="mt-4 p-4 rounded bg-red-500/10 text-sm">
A replica copies mistakes in seconds: <code>DELETE FROM items</code> is gone on
both. Replication is for <b>availability</b>; backups are for <b>recovery</b>.
You need both.
</div>

---
---

# The self-hosted database tier

<v-clicks>

- **Placement:** primary in db-a, replica in db-b; data on a **separate
  EBS volume** (encrypted), mounted by the startup script.
- **Replication:** PostgreSQL streaming (`pg_basebackup -R`) or MySQL
  GTID (`SOURCE_AUTO_POSITION = 1`); replication user's password from
  the secret store.
- **Roles:** `admin` owns the schema (migrations, never the app),
  `app_rw` = DML on the app schema, `app_ro` = `SELECT` (reports, the
  replica). Show one **denied** action per limited role.
- **Failover:** promote (`pg_promote()` / `STOP REPLICA; RESET REPLICA
  ALL; read_only = OFF`), repoint the app (DB host in SSM or a ConfigMap
  → rollout), rebuild the old primary as the new replica.
- **Report:** compare with RDS / Cloud SQL — backups, patching, failover,
  cost, effort.

</v-clicks>

---
---

# Password policy: three places, one set of numbers

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

MySQL (built in):

```sql
INSTALL COMPONENT 'file://component_validate_password';
SET PERSIST validate_password.length = 14;
SET PERSIST validate_password.special_char_count = 1;
SET PERSIST validate_password.mixed_case_count = 1;
SET PERSIST validate_password.number_count = 1;
-- ERROR 1819: Your password does not satisfy ...
```

PostgreSQL: `passwordcheck` or the `credcheck` extension
(`shared_preload_libraries`). Both can only check a password sent in
**plain text** — a client that sends a SCRAM hash skips them.

</div>
<div>

<v-clicks>

- **DB:** the server rejects a weak password (show it).
- **App:** `POST /api/admin/check-password` with
  `PASSWORD_MIN_LENGTH` from your variant.
- **CI:** a test that every generated password (`random_password`,
  `get-random-password`) passes the same rules.
- Same `password_min_length` and symbol rule in all three — the
  examiner compares them.

</v-clicks>

</div>
</div>

---
---

# Backups to S3 with your retention

```bash
# /etc/cron.d/campus-backup on the primary (user postgres, role from the instance profile)
*/30 * * * * postgres pg_dump -Fc campus | aws s3 cp - "s3://ci-example-0042-backups/backups/$(date -u +\%FT\%H-\%MZ).dump"
```

```json
{ "Rules": [{ "ID": "backups", "Status": "Enabled", "Filter": { "Prefix": "backups/" },
  "Expiration": { "Days": 14 }, "NoncurrentVersionExpiration": { "NoncurrentDays": 1 } }] }
```

<v-clicks>

- `Expiration` = `backup_retention_days`; snapshots: DLM retain count =
  days × snapshots per day.
- Versioning + Block Public Access + encryption on the bucket; the
  instance role may `PutObject` to `backups/` — **not** `DeleteObject`
  (ransomware can't wipe them). Object Lock is the stricter version.
- Same region: no transfer cost. Another region: protection from a
  region outage, at a price — an ADR.

</v-clicks>

---
---

# A backup you haven't restored is not a backup

<div class="text-sm">

| Restore step (`rto = 1 h`) | Example time |
|---|---|
| Alarm fires → you start | 3 min |
| `terraform apply -target=module.database` (new VM + empty disk) | 6 min |
| Download the latest dump from S3 | 1 min |
| `pg_restore` (50 k rows) | 2 min |
| Check: row count, a known record, `max(created_at)` | 1 min |
| Repoint the app (secret / ConfigMap) + `kubectl rollout status` | 2 min |
| **RTO** | **15 min** ✓ — data loss = time since the last dump |

</div>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
Write the table <b>before</b> the drill (your guess), then fill in the real
numbers. The difference is the most honest paragraph of the report.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Back to Aidana: <code>rpo = 15 min</code>, a nightly <code>pg_dump</code> at 03:00
and a streaming replica. The primary's disk dies at 02:50. And if,
instead, someone runs <code>DROP TABLE items</code> at 02:50?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Disk: promote the replica — seconds of data lost, RPO met. DROP TABLE:
the replica drops it too; the last copy is 23 h 50 min old — RPO missed
by far. She needs WAL archiving to S3 (≤ 15 min) or a far more frequent
dump.
</div>

---
layout: section
transition: slide-left
---

# Block 5
## Secrets, IaC, CI/CD, monitoring

---
---

# Secrets: from the store to a file, never to Git

```text
Secrets Manager / SSM SecureString   /<prefix>/db/app_rw, /<prefix>/app/secret_key, ...
        │  instance role (LabInstanceProfile) or the CI job's credentials
        ▼
startup script / deploy step ──► kubectl create secret generic campus-app --from-file=...
        ▼
pod: volume /run/secrets (0400, uid 10001) ──► DB_PASSWORD_FILE, SECRET_KEY_FILE
```

<v-clicks>

- The app reads `*_FILE`: the value never sits in an env var that
  `kubectl describe` or `docker inspect` would show.
- Kubernetes Secrets are only **base64** in etcd: k3s
  `--secrets-encryption`, and RBAC so only the deploy job reads them.
- Rotation plan: new value in the store → `ALTER ROLE` → update the
  Secret → rolling restart → verify → the old value is gone.

</v-clicks>

---
---

# What Terraform knows about your secrets

```hcl
resource "random_password" "app_rw" {
  length      = var.password_min_length + 4
  special     = var.password_require_symbol
  min_special = var.password_require_symbol ? 1 : 0
  min_upper   = 1
  min_numeric = 1
}
resource "aws_ssm_parameter" "app_rw" {
  name  = "/${var.resource_prefix}/db/app_rw"
  type  = "SecureString"
  value = random_password.app_rw.result
}
```

<v-clicks>

- `random_password.result` is in the **state** in plain text: state in
  an encrypted, versioned S3 bucket with a lock, never in Git.
- Outputs with a secret: `sensitive = true` — hidden in the log, still
  in the state.
- Terraform 1.10+: **ephemeral** resources and write-only arguments keep
  a value out of the state — a good stretch, and a good ADR.

</v-clicks>

---
---

# The repository the examiner opens

```text
infra/
  modules/{network,cluster,database,storage,iam,monitoring}/
  terraform.tfvars        # your variant, no secrets (un-ignore it in .gitignore)
k8s/                      # Deployment, Service, Ingress, HPA, ConfigMap (or a Helm chart)
scripts/                  # node + DB startup scripts, backup, restore, failover
app/sql/                  # roles, password policy
.github/workflows/        # ci.yml (PR) + deploy.yml (main)
docs/
  variant.md              # field → decision → where (file / screenshot)
  architecture.md         # the diagram
  adr/0001-….md …         # ≥ 4 ADRs
  cost.md                 # estimate vs actual
README.md   REPORT.md
```

<div v-click class="mt-2 text-sm opacity-70">
You must be able to explain <b>every</b> file here. If you can't explain
it, it doesn't count — however it was written.
</div>

---
---

# The pipeline (week 13, now to Kubernetes)

```text
PR:    ruff · pytest · gitleaks (all history) · terraform fmt/validate · trivy config · plan
main:  build ─▶ trivy image (CRITICAL = stop) ─▶ push ECR :<sha> ─▶ secrets → k8s Secret
       ─▶ kubectl set image deploy/campus-app app=…:<sha> ─▶ kubectl rollout status --timeout=5m
                                                                 │ fail
                                                                 ▼
                                                      kubectl rollout undo  (+ git revert)
```

<v-clicks>

- `rollout status` waits for **readiness**: a release whose pods never
  get Ready fails the job and never takes traffic.
- The password-policy test runs here too (§5.10).
- Learner Lab: session keys in Secrets (expire ~4 h) or a self-hosted
  runner with `LabInstanceProfile` — write which one and the OIDC design.

</v-clicks>

---
---

# Monitoring: your thresholds, in code

```hcl
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${var.resource_prefix}-node-cpu-high"
  namespace           = "AWS/EC2"
  metric_name         = "CPUUtilization"
  dimensions          = { AutoScalingGroupName = module.cluster.agent_asg_name }
  statistic           = "Average"
  period              = 300
  evaluation_periods  = 2
  comparison_operator = "GreaterThanThreshold"
  threshold           = var.alert_cpu_pct
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]   # shows "resolving" too
}
```

<div v-click class="text-sm mt-2">

≥ 4 alarms: CPU > `alert_cpu_pct`, 5xx > `alert_5xx_per_min`, + two of
unhealthy targets, p95 latency, replication lag, disk. Dashboards: LB,
app RED from `/metrics`, nodes + pods + HPA, DB. Logs: one `request_id`
from the LB to the pod.

</div>

---
layout: section
transition: slide-left
---

# Block 6
## Writing it down: diagram, ADRs, cost, teardown

---
---

# The architecture diagram

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Must show**

- Both zones, and the 6 subnets **with CIDRs**
- ALB, NAT, IGW, route directions
- Cluster nodes (which group), pods
- DB primary / replica (+ ClickHouse), the replication arrow
- Bucket, registry, secret store, monitoring
- CI/CD and where its credentials come from

</div>
<div>

<v-clicks>

**Common errors**

- The diagram from a blog post: other CIDRs, RDS, three AZs
- Arrows without a direction or a port
- No zones: "HA" you can't see
- The DB drawn inside the cluster
- Pretty, but not what `terraform state list` says

</v-clicks>

</div>
</div>

<div v-click class="mt-4 text-sm opacity-70">
Any tool: diagrams.net, Mermaid in the README, even clean ASCII. It must
match what you built.
</div>

---
---

# Architecture Decision Records

```markdown
# ADR 0001 — One NAT gateway for both zones

Status: accepted · 2026-11-23
## Context
Private nodes and DB VMs need egress (image pulls, updates, S3). Budget: $25/month.
Two NAT gateways cost ~$73/month before data; one ~$36.
## Decision
One NAT gateway in us-east-1a; both private route tables use it. Created only while
I build and demo; destroyed with the rest of the stack.
## Alternatives considered
Two NATs (zonal HA, 2× cost) · no NAT + VPC endpoints (no OS updates) · public nodes (fails §5.1)
## Consequences
If us-east-1a fails, zone b keeps serving users but loses egress: no new image pulls,
no backups to S3 until the zone returns. Accepted for the budget; with 2× budget I'd use two.
```

<div v-click class="text-sm mt-2 opacity-70">
Context → decision → alternatives → consequences. An ADR with no
downside is a sign that you didn't consider one.
</div>

---
---

# The cost estimate: Pricing Calculator, 24/7

<div class="text-sm">

| Line item (us-east-1, on-demand, 730 h) | Monthly |
|---|---|
| k3s server + 2 agents, `t3.small` ($0.0208/h) | $45.55 |
| DB primary + replica, `t3.micro` ($0.0104/h) | $15.18 |
| ALB ($0.0225/h) + ~1 LCU + 2 public IPv4 | $16.43 + ~$5.84 + $7.30 |
| 1 NAT gateway + its IPv4 (+ $0.045/GB) | $32.85 + $3.65 |
| gp3: 3 × 20 GB nodes + 2 × (8 + 20) GB DB ($0.08/GB) | $9.28 |
| S3, snapshots, ECR, 6 secrets ($0.40), CloudWatch beyond free | ~$5 |
| **Total 24/7** | **≈ $141 ≈ $0.19 per hour** |

</div>

<div v-click class="mt-2 text-sm opacity-70">
List prices at the time of writing — rebuild it in calculator.aws (or the
Google Cloud calculator) and save the <b>share link</b> in <code>docs/cost.md</code>.
</div>

---
---

# Fitting it into `monthly_budget_usd`

<v-clicks>

- The 24/7 estimate is **above** every variant budget ($15 / 25 / 40) —
  that is expected. The budget limits what you **spend**, not the design.
- **Burn rate:** ≈ $0.19/h for the full stack. A $25 budget ≈ 130 hours
  of everything running; minus what bills even when "off" (disks, S3,
  secrets, an Elastic IP you forgot).
- **Plan:** "build sessions 3 h, destroy after each; full stack for the
  load test and drills; ~40 h in total ≈ $8 + $3 always-on."
- **≥ 3 optimizations** applied or proposed: one NAT or none, `t4g`
  (Graviton) nodes, scheduled scale-in at night, lifecycle to cheaper
  storage, destroy between sessions.
- **Actual spend** from Billing / Cost Explorer (Learner Lab: the spend
  counter) next to the estimate.

</v-clicks>

---
---

# Teardown: what survives `terraform destroy`

<div class="text-xs">

| Leftover | Why it stays | Check |
|---|---|---|
| EBS snapshots, DLM / manual | Not in your state, or retention | `aws ec2 describe-snapshots --owner-ids self` |
| Unattached volumes, Elastic IPs | Created by hand or by k8s | `describe-volumes --filters Name=status,Values=available` |
| S3 with versions | `force_destroy = false`, noncurrent versions | `list-object-versions` |
| ECR images, log groups | Created by CI / the agents | `ecr describe-repositories`, `logs describe-log-groups` |
| Secrets Manager | **Scheduled deletion** (7–30 days): the name stays taken, the next `apply` fails | `--force-delete-without-recovery` |
| Load balancers made by k8s | Owned by the cluster, not Terraform | `elbv2 describe-load-balancers` |

</div>

```bash
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values="$PREFIX"   # teardown proof
```

---
layout: section
transition: slide-left
---

# Block 7
## Grading and the defense

---
---

# How the project is graded

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

| Part | Points |
|---|---|
| Report (RUBRIC.md, R1–R12) | 20 |
| Presentation & demo (~7 min) | 8 |
| Oral defense (~8 min) | 12 |
| **Total = the final exam** | **40 %** |

</div>
<div>

<v-clicks>

- **Individual.** Discuss ideas; never share code, Terraform or text.
- **Deadline:** any day before the final exam, **including the exam
  day**. Book your defense when you're ready.
- **Your variant** arrives as a **private Canvas comment**. Built
  someone else's variant → **0 for the report**.
- Submit: public repo, `REPORT.pdf`, and a demo video (≤ 10 min) if the
  stack is already destroyed.

</v-clicks>

</div>
</div>

---
---

# The report: where the 20 points are

<div class="text-sm">

| | Area | Pts | | Area | Pts |
|---|---|---|---|---|---|
| R1 | Variant + diagram + ≥ 4 ADRs | 2 | R7 | **Self-hosted DB** | 3 |
| R2 | Networking | 2 | R8 | IAM & secrets | 2 |
| R3 | Compute, Linux hardening | 1 | R9 | Containers & CI/CD | 1 |
| R4 | **Kubernetes cluster** | 3 | R10 | Security checks | 1 |
| R5 | LB & auto scaling | 2 | R11 | IaC | 1 |
| R6 | Storage | 1 | R12 | Monitoring, drills, cost | 1 |

</div>

<v-clicks>

- Each area: **full** (done + evidence + variant), **partial** (≈ half),
  **none**. "It works, trust me" is none.
- Caps: DB reachable from the internet → report ≤ 10; RDS / Cloud SQL →
  R7 = 0 and ≤ 12; variant ignored → ≤ 12; a committed secret → −4.

</v-clicks>

---
---

# The presentation (~7 minutes)

```text
0:00  Who, which track, the variant in one slide (prefix, CIDR, cluster, rpo/rto, budget)
1:00  The diagram: one request from the browser to the database
2:30  Two or three decisions you'd defend (ADRs) — with the downside
3:30  Live: the site through the ALB, kubectl get pods -o wide, the HPA
5:00  One drill: node/zone failure, DB failover or rollback — with the measured time
6:15  The hardest problem, cost: estimate vs actual
7:00  Stop.
```

<v-clicks>

- Start `terraform apply` **before** your slot if you rebuild — or bring
  the demo video. A live demo that fails with no video costs the demo
  points.
- Readable: the diagram large enough to read from the back of the room.

</v-clicks>

---
---

# The oral defense: what kind of questions

<div class="text-sm">

| Type | What you do | You should be ready to |
|---|---|---|
| **Your own file** (always) | Walk through a file the examiner picks, line by line | Say what each line does, where its values come from, what happens if it fails |
| **The cluster** | Explain a mechanism in *your* cluster | Talk about probes, requests, the HPA, the request path, zone spreading, your drill |
| **The database** | Explain a mechanism in *your* DB tier | Talk about roles, the policy, replication and your RPO, your timed restore/failover |
| **A weak spot** | A question on the thinnest part of your report | Defend or honestly improve that area: network, IAM, CI/CD, monitoring, cost |
| **What-if** (strong students) | Reason about a change or an incident | Think aloud with your own numbers: growth, a lost zone, a leak, a smaller budget |

</div>

<div v-click class="mt-2 text-sm opacity-70">
4 questions × 3 points: 3 = right, in your own words, tied to your build. Can't explain
your own code → defense capped at 4 / 12.
</div>

---
---

# How to prepare for the defense

<v-clicks>

- **The random-file drill:** open a file you didn't touch for a week and
  explain it out loud. Every line you skip is a question.
- For every value, know its **source**: "60 is `hpa_cpu_target` from my
  variant, passed as `var.hpa_cpu_target` into the manifest".
- For every component, know **what happens when it fails** — and which
  drill proved it.
- Have the evidence one click away: the report PDF, the repo, the
  terminal with `kubectl` and `aws` ready.
- "I don't know, but here's how I would find out" beats a confident
  wrong answer.

</v-clicks>

---
---

# Common failures

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**In the build**

- **Variant drift:** template CIDR, a classmate's prefix, a hard-coded `80`
  where `alert_cpu_pct` should be
- **HPA with no CPU requests**; liveness and readiness both on `/healthz`
- **The DB in the cluster**, or RDS "just to save time"
- **SG rules by CIDR** (`10.42.0.0/16` to the DB), `0.0.0.0/0` to the app
- **Secrets in Git**: `.env`, state, a token in user data — "deleted in
  the next commit" is still a leak
- **Join token / DB password in the launch template** instead of read at boot

</div>
<div>

**In the evidence**

- **Drills that "would work":** never restored, never failed over, no times
- **Backups that can't meet the RPO** (one nightly dump for `rpo = 15 min`)
- **A diagram that doesn't match** `terraform state list`
- **ADRs without alternatives** or consequences
- **No cost numbers**, or a budget alert at the wrong amount
- **No teardown proof**, or a NAT billing in a forgotten region
- **Screenshots without your prefix**

</div>
</div>

---
---

# Today's practice — architecture & project kickstart

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Both tracks**

1. Your repository from the template
2. `docs/variant.md`: every field → a decision
3. A diagram, **3 ADRs**, a cost estimate within `monthly_budget_usd`
4. Terraform network skeleton with **your** CIDR and prefix:
   `fmt`, `validate`, `plan`
5. CI green: tests + Terraform + gitleaks

</div>
<div>

**Track A — Learner Lab:** a secret in SSM / Secrets Manager, read by a
VM through `LabInstanceProfile` into a `0400` file.

**Track B — no AWS:** Vault dev in Docker Compose plays the secret
manager; the template app starts with its secrets as files.

<div v-click class="mt-4 opacity-70">
Checked orally: your variant table, one ADR, the plan line, a green CI
run, and where the secret ended up.
</div>

</div>
</div>

---
---

# Course wrap-up: what you built in 15 weeks

<div class="text-sm">

| Weeks | You learned to | In the project |
|---|---|---|
| 1–3 | Use the cloud, Linux and VMs | Templates, startup scripts, hardening |
| 4–6 | Build networks, storage and identity | 3-tier VPC, buckets, roles, secrets |
| 7–8 | Run databases, design architectures | Self-hosted primary + replica, ADRs |
| 9–10 | Balance, scale, containerise | ALB, ASG, k3s, HPA |
| 11–13 | Automate everything | Terraform modules, CI/CD with rollback |
| 14 | Survive failures, secure, pay less | Drills, security review, cost |
| 15 | Design and **defend** | This project |

</div>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
The skill isn't one service. It is: requirement → decision → code →
evidence → "here's what breaks, and here's how I know".
</div>

---
---

# Where to go next

<v-clicks>

- **Portfolio:** your public repo with a good README, the diagram and a
  demo video is a stronger CV line than a certificate.
- **Certifications** that follow this course: AWS Solutions Architect –
  Associate, HashiCorp Terraform Associate, CNCF CKA / KCNA, Google
  Associate Cloud Engineer.
- **Next skills:** GitOps (Argo CD / Flux), service meshes, policy as
  code (OPA, Checkov in CI), observability with OpenTelemetry, FinOps.
- **Practice the habit:** every project gets a diagram, ADRs, a cost
  line and a `destroy`.

</v-clicks>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Draw the project's reference architecture and walk one request through it
- [ ] Turn every variant field into a concrete, testable design decision
- [ ] Choose backup and replication mechanisms that meet a given RPO and RTO
- [ ] Explain how secrets reach the app without passing through Git or the image
- [ ] Write an ADR with real alternatives and consequences
- [ ] Estimate the monthly cost, plan the spend within the budget, and prove teardown
- [ ] Say how the project is graded and prepare for each kind of defense question

</v-clicks>

---
layout: default
---

# Before the final exam

- [ ] Finish today's practice and its cleanup (Track A: terminate the
      instance, **End Lab**; Track B: `docker compose down -v`)
- [ ] Follow the milestones: network → DB → cluster → CI/CD → monitoring
      → drills → report. Save evidence (with your prefix) **as you go**
- [ ] Record the demo video while the stack runs; then `terraform destroy`
      and keep the teardown proof
- [ ] Book your defense — any day up to and including the exam day

<div class="mt-8 text-sm opacity-60">
Questions about your variant: a private Canvas comment, never in a group chat.
</div>

---
layout: end
---

# Good luck at the defense

Design it, build it, break it on purpose, explain every line — and
tear it down.
