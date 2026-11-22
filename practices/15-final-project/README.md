# Practice 15 — Architecture & Project Kickstart

**Objective:** start your final project the way a real platform starts:
with **decisions on paper** and a first piece of code that proves them.
From the template repository, write your personal variant into a design
(a variant table, a diagram, three ADRs and a cost estimate within your
`monthly_budget_usd`), write the **Terraform network** with your own
`vpc_cidr` and `resource_prefix` so it passes `fmt`, `validate` and
`plan`, make **CI green**, and read one **secret from a secret manager**
into a file the app can use. These are the skills from today's lecture.

**Timebox:** in class: ~60 min on either track. Parts 1–3 are the same
for everyone (~40 min); Part 4 is per track (~20 min). If you've been
stuck on one step for more than 10 minutes, stop and ask. Don't push
through alone.

> **How you're checked:** this practice is done in class and checked
> **orally**. There is no report. When you finish, keep your terminal,
> your repository and the Actions page open. The instructor will ask you
> to show the items in the *Definition of done* and to explain what you
> did.

Everything you make today stays in your **final project repository**:
it is milestones M1–M2 done properly, not a throwaway exercise.

## Pick your track

You don't need AWS access to pass. Pick **one** track for Part 4:

| | Track A — AWS Academy Learner Lab | Track B — no AWS |
|---|---|---|
| Needs | Learner Lab, GitHub account, `git`, `gh` (logged in), Terraform ≥ 1.10, AWS CLI v2 | GitHub account, `git`, `gh`, Docker with Compose v2, Terraform ≥ 1.10 (or the `hashicorp/terraform` image) |
| `terraform plan` against | The real AWS API with your lab session | An **offline** provider (`offline_override.tf`): no account, nothing created |
| The secret manager | SSM Parameter Store `SecureString` (or Secrets Manager) | A **Vault dev server** in Docker Compose |
| Who reads the secret | A `t3.micro` with `LabInstanceProfile`, at boot | A `fetch-secrets` container with a **limited Vault token**, before the app starts |
| Slow step | The instance boot (~3 min) | Building the app image the first time (~1 min) |

> **Why not LocalStack?** Since **23 March 2026** the
> `localstack/localstack` image requires an account and a
> `LOCALSTACK_AUTH_TOKEN` even for the free tier, and the Community
> repository is archived. Track B uses Vault (the open-source fork
> **OpenBao** works the same way) so you need no account at all. The lab
> uses **moto** as a local AWS API for the same reason.

The ideas are the same on both tracks:

| Concept | Track A (Learner Lab) | Track B (your laptop) |
|---|---|---|
| Variant → design | `docs/variant.md`, diagram, ADRs, cost | Same |
| Network as code | `infra/` + `modules/network`, `plan` against AWS | Same code, `plan` with the offline override |
| CI | GitHub Actions: tests, Terraform, gitleaks | Same |
| Secret store | SSM `SecureString` `/<prefix>/app/secret_key` | Vault KV `secret/<prefix>/app` |
| Identity of the reader | Instance role (`LabInstanceProfile`) | A Vault token with a read-one-path policy |
| Least privilege | The policy JSON you **would** attach (LabRole can read everything) | Real: the token gets `403` on another path |
| The secret on the server | `/run/campus/secret_key`, `0400`, owner `app` | `/run/secrets/secret_key`, `0400`, uid 10001 |
| Cleanup | Terminate the instance, **End Lab** | `docker compose down -v` |

## Definition of done

Be ready to **show** each of these:

- [ ] Your repository, created from the template, **public**, with
      `APP_OWNER` = your name and ID.
- [ ] `docs/variant.md`: every variant field with your value, the
      decision it forces and where it will be applied.
- [ ] `docs/architecture.md` with a diagram that shows **your** CIDRs,
      and three ADRs in `docs/adr/` — one of them on the NAT choice.
- [ ] `docs/cost.md`: a Pricing Calculator link, the 24/7 total, the
      hourly burn rate and a spending plan that stays within
      `monthly_budget_usd`.
- [ ] `infra/`: `terraform fmt -check` clean, `validate` OK, `plan`
      shows `Plan: 28 to add` with **your** prefix and **your** subnet
      CIDRs — and you can read one resource in the plan.
- [ ] The part you wrote yourself: the DB security-group rule (the TODO
      in Part 2).
- [ ] A green CI run with three jobs: `test`, `terraform`, `gitleaks`.
- [ ] The secret: where it is stored, who read it, where the file is,
      its mode and owner — and that you never printed it.
- [ ] Cleanup done.

---

## Part 0 — Your repository (both tracks)

Create your repository from the course template and clone it:

```bash
gh repo create <you>/css4005-<resource_prefix> --public \
  --template weeebdev-edu/css4005-final-project --clone
cd css4005-<resource_prefix>
```

(Or on GitHub: **weeebdev-edu/css4005-final-project → Use this template
→ Create a new repository**, public.) Public, because it is your
portfolio and because GitHub Free enforces environment reviewers only on
public repositories (week 13).

Open your **variant**: it was sent to you as a **private comment in
Canvas**. Keep it open in a second window for the whole practice. If you
don't have one, ask now: a project built on someone else's variant gets
**0 for the report**.

> **Example variant used in this handout.** All outputs below use this
> made-up row. Yours is different — replace **every** value.
>
> | Field | Example | Field | Example |
> |---|---|---|---|
> | `theme` | `library` | `cluster` | `k3s-on-autoscaled-vms` |
> | `resource_prefix` | `ci-example-0042` | `hpa_cpu_target` | `60` |
> | `vpc_cidr` | `10.42.0.0/16` | `password_min_length` | `14` |
> | `traffic_pattern` | `spiky` | `password_require_symbol` | `yes` |
> | `rpo` / `rto` | `1 h` / `1 h` | `alert_cpu_pct` | `80` |
> | `monthly_budget_usd` | `25` | `alert_5xx_per_min` | `10` |
> | `db_engine` | `postgresql` | `backup_retention_days` | `14` |
> | `clickhouse_required` | `no` | | |

Set your name in the app's example config (it is shown on `/` and
`/info`):

```bash
cp .env.example .env                  # .env is ignored by git; never commit it
# edit .env: APP_NAME=<your theme>, APP_OWNER=Your Name (student ID)
mkdir -p docs/adr infra/modules/network
```

---

## Part 1 — The design on paper (both tracks)

### 1.1 `docs/variant.md` — the requirements table

This is the first table of your report (R1). Every row: your value,
the **decision** it forces, and **where** you will show it. Start from
this; the first rows are filled for the example variant:

```markdown
# My variant

| Field | My value | Design decision | Where it is applied (evidence) |
|---|---|---|---|
| `theme` | library | Items are books; `APP_NAME=library` | `k8s/configmap.yaml`, screenshot of `/` |
| `resource_prefix` | ci-example-0042 | Every name + tag `Project`, one Terraform variable | `infra/terraform.tfvars`, `terraform state list` |
| `vpc_cidr` | 10.42.0.0/16 | 6 × /24: public .1/.2, app .11/.12, db .21/.22 | `infra/modules/network/main.tf`, plan output |
| `traffic_pattern` | spiky | HPA + ASG target tracking; a load test with bursts | `k8s/hpa.yaml`, `infra/modules/cluster/`, k6 graph |
| `rpo` | 1 h | Replica (seconds) + `pg_dump` to S3 every 30 min + hourly snapshots | `scripts/backup.sh`, cron, S3 listing |
| `rto` | 1 h | Restore and failover runbooks, both timed | `docs/runbooks/`, drill tables |
| `monthly_budget_usd` | 25 | One NAT, t3.small nodes, destroy after each session | `docs/cost.md`, budget alert screenshot |
| `db_engine` | postgresql | ... | ... |
| ... every other field ... | | | |
```

Two rules:

- A decision must be **testable**: "fast backups" is not a decision;
  "`pg_dump` every 30 min to `s3://<prefix>-backups/backups/`, kept 14
  days" is.
- Check the combinations, not only the fields: does your backup
  schedule meet the `rpo`? Is `alert_cpu_pct` above `hpa_cpu_target`
  (lecture, Block 2)? Fits the `monthly_budget_usd`?

### 1.2 `docs/architecture.md` — the diagram

Draw the target architecture **with your CIDRs**: diagrams.net, Mermaid
or clean ASCII, as long as GitHub shows it. It must have both zones,
the six subnets with CIDRs, ALB, IGW, NAT, the cluster nodes and which
group they belong to, DB primary and replica (+ ClickHouse if
`clickhouse_required = yes`), bucket, registry, secret store,
monitoring and the CI/CD path. A starting point (example CIDRs):

```text
                         Internet ── 80/443
                             │
 ┌───────────────────────────▼──────── VPC 10.42.0.0/16 (ci-example-0042) ───────┐
 │  public  10.42.1.0/24 (1a)   10.42.2.0/24 (1b)    ALB · NAT (1a only) · IGW   │
 │                │ :30080, health check /readyz                                 │
 │  app     10.42.11.0/24 (1a)  10.42.12.0/24 (1b)   k3s server (1a)             │
 │                                                   agent ASG min 2 (1a + 1b)   │
 │                │ :5432 (SG app → SG db)                                       │
 │  db      10.42.21.0/24 (1a)  10.42.22.0/24 (1b)   PostgreSQL primary (1a)     │
 │                                                   ──stream──► replica (1b)    │
 └───────────────────────────────────────────────────────────────────────────────┘
  S3 ci-example-0042-files / -backups · ECR · SSM /ci-example-0042/* · CloudWatch
  GitHub Actions ──(session keys / OIDC)──► ECR push, kubectl rollout
```

### 1.3 Three ADRs in `docs/adr/`

One file per decision, numbered. Use this template:

```markdown
# ADR 000N — <the decision in one line>

Status: accepted · <date>

## Context
What forces a decision: the requirement (variant field), the constraint (budget, Learner Lab).

## Decision
What you do, concretely enough to check in the code.

## Alternatives considered
At least two, each with why not.

## Consequences
What gets better, what gets worse, what you'll watch. An ADR with no downside is suspicious.
```

Write these three today (you need ≥ 4 for the report):

1. **`0001-nat-gateway.md`** — one NAT, two, or none (lecture, Block 3:
   prices and the zone-failure consequence). Required.
2. **`0002-cluster.md`** — how you build your `cluster` variant: number
   of servers, how agents join, how the ALB reaches the ingress.
3. **`0003-backups-vs-rpo.md`** — which mechanisms together meet your
   `rpo`, and where the backups live and for how long
   (`backup_retention_days`).

### 1.4 `docs/cost.md` — the estimate within your budget

Open the **AWS Pricing Calculator** (<https://calculator.aws/>, no
account needed) or the **Google Cloud Pricing Calculator**. Add every
item of **your** design for **730 hours** in your region. Save the
estimate and copy its **share link**. Then fill this:

```markdown
# Cost

Pricing Calculator estimate: <share link> (prices of <date>)

| Item | Size / count | Monthly, 24/7 |
|---|---|---|
| k3s nodes | 3 × t3.small | $45.55 |
| DB VMs | 2 × t3.micro | $15.18 |
| ALB + LCU + 2 public IPv4 | 1 | $29.57 |
| NAT gateway + IPv4 | 1 | $36.50 |
| EBS gp3 | 116 GB | $9.28 |
| S3, snapshots, ECR, secrets, CloudWatch | | ~$5 |
| **Total 24/7** | | **≈ $141** |

Burn rate: $141 / 730 h ≈ **$0.19 per hour** with everything running.
Always-on (even when "off"): S3, snapshots, secrets ≈ $3 / month.

## Spending plan (must be ≤ monthly_budget_usd = $25)
- Build sessions of ~3 h, `terraform destroy` after each: ~30 h ≈ $6
- Load test + drills + demo video, full stack: ~10 h ≈ $2
- Always-on: ~$3
- **Planned total ≈ $11 ≤ $25**; budget alert at $25 (Learner Lab: the lab's spend counter)

## Optimizations (≥ 3)
1. ... 2. ... 3. ...
```

The example numbers are the example design (us-east-1 list prices from
the lecture). **Your** table comes from **your** calculator link. The
24/7 total is **above** every variant budget, and that's expected: the
budget limits what you **spend**, the plan shows how.

---

## Part 2 — The network in Terraform (both tracks)

The README's rule: your **variant values live in one
`infra/terraform.tfvars`** (no secrets in it). The template's
`.gitignore` ignores `*.tfvars`, so un-ignore yours, and ignore the
files that must never be committed:

```bash
cat >> .gitignore <<'EOF'
# Week 15 practice
!infra/terraform.tfvars
*_override.tf
.vault/
EOF
```

Create these files.

**`infra/versions.tf`**

```hcl
terraform {
  required_version = ">= 1.10"

  # State stays local today (never commit it: .gitignore has *.tfstate).
  # In the project you move it to S3 / GCS with a lock (week 13).

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.region

  # Every resource gets these tags: the variant prefix is visible everywhere
  default_tags {
    tags = {
      Project = var.resource_prefix
      Course  = "CSS4005"
    }
  }
}
```

**`infra/variables.tf`** — the whole variant, with **validation**: a
typo fails at `plan`, not at the defense.

```hcl
# Your personal variant. The values live in terraform.tfvars (no secrets there).
# A typo in a value fails at plan time, not at the defense.

variable "region" {
  type    = string
  default = "us-east-1"
}

variable "azs" {
  description = "Two availability zones (written out, so plan needs no API call)"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}

variable "theme" {
  type = string
}

variable "resource_prefix" {
  type = string
  validation {
    condition     = can(regex("^[a-z0-9-]{3,24}$", var.resource_prefix))
    error_message = "resource_prefix: lowercase letters, digits and '-', at most 24 characters (ALB names are limited to 32)."
  }
}

variable "vpc_cidr" {
  type = string
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0)) && endswith(var.vpc_cidr, "/16")
    error_message = "vpc_cidr must be a /16 such as 10.42.0.0/16."
  }
}

variable "traffic_pattern" {
  type = string
  validation {
    condition     = contains(["steady", "spiky", "scheduled-peaks"], var.traffic_pattern)
    error_message = "traffic_pattern: steady, spiky or scheduled-peaks."
  }
}

variable "rpo" {
  type = string
}

variable "rto" {
  type = string
}

variable "monthly_budget_usd" {
  type = number
}

variable "db_engine" {
  type = string
  validation {
    condition     = contains(["postgresql", "mysql"], var.db_engine)
    error_message = "db_engine: postgresql or mysql."
  }
}

variable "clickhouse_required" {
  type = bool
}

variable "cluster" {
  type = string
  validation {
    condition     = contains(["k3s-on-autoscaled-vms", "k3s-fixed-3-nodes", "managed-k8s-if-credits"], var.cluster)
    error_message = "cluster: k3s-on-autoscaled-vms, k3s-fixed-3-nodes or managed-k8s-if-credits."
  }
}

variable "hpa_cpu_target" {
  type = number
}

variable "password_min_length" {
  type = number
  validation {
    condition     = var.password_min_length >= 12 && var.password_min_length <= 16
    error_message = "password_min_length is between 12 and 16."
  }
}

variable "password_require_symbol" {
  type = bool
}

variable "alert_cpu_pct" {
  type = number
}

variable "alert_5xx_per_min" {
  type = number
}

variable "backup_retention_days" {
  type = number
}

# Not part of the variant: your own design choices
variable "enable_nat" {
  description = "One NAT gateway for both zones (see ADR 0001). It bills ~$0.045/h even when idle."
  type        = bool
  default     = true
}

variable "ingress_node_port" {
  description = "NodePort of the k3s ingress (Traefik) that the load balancer forwards to"
  type        = number
  default     = 30080
}
```

Most of these variables aren't used yet: the cluster, database and
monitoring modules will use them. Declaring them now means
`terraform.tfvars` is complete and checked from day one.

**`infra/terraform.tfvars`** — **your** values (`yes`/`no` become
`true`/`false`):

```hcl
# EXAMPLE variant: replace EVERY value with the one from your private Canvas comment.
theme                   = "library"
resource_prefix         = "ci-example-0042"
vpc_cidr                = "10.42.0.0/16"
traffic_pattern         = "spiky"
rpo                     = "1 h"
rto                     = "1 h"
monthly_budget_usd      = 25
db_engine               = "postgresql"
clickhouse_required     = false
cluster                 = "k3s-on-autoscaled-vms"
hpa_cpu_target          = 60
password_min_length     = 14
password_require_symbol = true
alert_cpu_pct           = 80
alert_5xx_per_min       = 10
backup_retention_days   = 14
```

**`infra/main.tf`**

```hcl
module "network" {
  source = "./modules/network"

  prefix            = var.resource_prefix
  vpc_cidr          = var.vpc_cidr
  azs               = var.azs
  db_engine         = var.db_engine
  enable_nat        = var.enable_nat
  ingress_node_port = var.ingress_node_port
}
```

**`infra/outputs.tf`**

```hcl
output "vpc_id" {
  value = module.network.vpc_id
}

output "subnets" {
  description = "tier-zone => CIDR: compare it with the subnet plan in docs/"
  value       = module.network.subnet_cidrs
}

output "security_groups" {
  value = module.network.security_group_ids
}
```

**`infra/modules/network/variables.tf`**

```hcl
variable "prefix" {
  type = string
}

variable "vpc_cidr" {
  type = string
}

variable "azs" {
  type = list(string)
}

variable "db_engine" {
  type = string
}

variable "enable_nat" {
  type = bool
}

variable "ingress_node_port" {
  type = number
}
```

**`infra/modules/network/outputs.tf`**

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_cidrs" {
  value = { for k, s in aws_subnet.this : k => s.cidr_block }
}

output "subnet_ids" {
  value = { for k, s in aws_subnet.this : k => s.id }
}

output "security_group_ids" {
  value = {
    lb  = aws_security_group.lb.id
    app = aws_security_group.app.id
    db  = aws_security_group.db.id
  }
}
```

**`infra/modules/network/main.tf`** — the module: six subnets carved
from your CIDR, one public and one private route table, one optional
NAT, and the security chain **internet → lb → app → db** by SG
reference. **You write the DB rule** (the TODO):

```hcl
# 3 tiers x 2 zones, carved from the variant CIDR:
#   public  10.X.1.0/24, 10.X.2.0/24    (load balancer, NAT)
#   app     10.X.11.0/24, 10.X.12.0/24  (Kubernetes nodes)
#   db      10.X.21.0/24, 10.X.22.0/24  (database VMs)
locals {
  tier_base = { public = 0, app = 10, db = 20 }
  subnets = merge([
    for tier, base in local.tier_base : {
      for i, az in var.azs : "${tier}-${az}" => {
        tier = tier
        az   = az
        cidr = cidrsubnet(var.vpc_cidr, 8, base + i + 1)
      }
    }
  ]...)
  db_port = var.db_engine == "postgresql" ? 5432 : 3306
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags                 = { Name = "${var.prefix}-vpc" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.prefix}-igw" }
}

resource "aws_subnet" "this" {
  for_each = local.subnets

  vpc_id                  = aws_vpc.main.id
  availability_zone       = each.value.az
  cidr_block              = each.value.cidr
  map_public_ip_on_launch = false # nothing gets a public IP by accident
  tags = {
    Name = "${var.prefix}-${each.key}"
    Tier = each.value.tier
  }
}

# ---- routing: public -> internet gateway; app + db -> one NAT ----

resource "aws_eip" "nat" {
  count  = var.enable_nat ? 1 : 0
  domain = "vpc"
  tags   = { Name = "${var.prefix}-nat-eip" }
}

resource "aws_nat_gateway" "main" {
  count         = var.enable_nat ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.this["public-${var.azs[0]}"].id
  tags          = { Name = "${var.prefix}-nat" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = { Name = "${var.prefix}-rt-public" }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  dynamic "route" {
    for_each = var.enable_nat ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.main[0].id
    }
  }
  tags = { Name = "${var.prefix}-rt-private" }
}

resource "aws_route_table_association" "this" {
  for_each       = local.subnets
  subnet_id      = aws_subnet.this[each.key].id
  route_table_id = each.value.tier == "public" ? aws_route_table.public.id : aws_route_table.private.id
}

# ---- security chaining: internet -> lb -> app -> db, by SG reference ----

resource "aws_security_group" "lb" {
  name        = "${var.prefix}-lb"
  description = "Load balancer: HTTP/HTTPS from the internet"
  vpc_id      = aws_vpc.main.id
}

resource "aws_security_group" "app" {
  name        = "${var.prefix}-app"
  description = "Kubernetes nodes: ingress NodePort from the load balancer only"
  vpc_id      = aws_vpc.main.id
}

resource "aws_security_group" "db" {
  name        = "${var.prefix}-db"
  description = "Database VMs: ${var.db_engine} from the app tier only"
  vpc_id      = aws_vpc.main.id
}

resource "aws_vpc_security_group_ingress_rule" "lb_web" {
  for_each          = toset(["80", "443"])
  security_group_id = aws_security_group.lb.id
  cidr_ipv4         = "0.0.0.0/0" # the ONLY 0.0.0.0/0 inbound rule in the project
  ip_protocol       = "tcp"
  from_port         = tonumber(each.key)
  to_port           = tonumber(each.key)
}

resource "aws_vpc_security_group_egress_rule" "lb_to_app" {
  security_group_id            = aws_security_group.lb.id
  referenced_security_group_id = aws_security_group.app.id
  ip_protocol                  = "tcp"
  from_port                    = var.ingress_node_port
  to_port                      = var.ingress_node_port
}

resource "aws_vpc_security_group_ingress_rule" "app_from_lb" {
  security_group_id            = aws_security_group.app.id
  referenced_security_group_id = aws_security_group.lb.id
  ip_protocol                  = "tcp"
  from_port                    = var.ingress_node_port
  to_port                      = var.ingress_node_port
}

# TODO: resource "aws_vpc_security_group_ingress_rule" "db_from_app"
#       The database accepts its port ONLY from the app tier.
#       Hints: copy app_from_lb above; which SG gets the rule, which SG is the
#       source, and which local value holds the port of YOUR db_engine?

# Nodes and DB VMs reach the internet (updates, registry) through the NAT
resource "aws_vpc_security_group_egress_rule" "all_out" {
  for_each          = { app = aws_security_group.app.id, db = aws_security_group.db.id }
  security_group_id = each.value
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}
```

Read the `locals` block once: for `10.42.0.0/16`, `cidrsubnet(…, 8, 11)`
adds 8 bits to the mask (/24) and takes network number 11:
`10.42.11.0/24`. That's the whole subnet plan in one expression.

### 2.1 Check it: fmt and validate (both tracks)

```bash
cd infra
terraform fmt -recursive          # fix the formatting...
terraform fmt -check -recursive   # ...then no output = formatted (CI runs exactly this)
terraform init
terraform validate                # Success! The configuration is valid.
```

No Terraform installed (Track B)? The same commands in a container:

```bash
tf() { docker run --rm -v "$PWD/..:/w" -w /w/infra hashicorp/terraform:1.17.2 "$@"; }
tf fmt -check -recursive && tf init && tf validate
```

Commit `.terraform.lock.hcl` (it pins the provider version for CI);
never `.terraform/` or `*.tfstate`.

### 2.2 The plan

**Track A** — with your Learner Lab session (**AWS Details → AWS CLI →
Show** → `~/.aws/credentials`, region `us-east-1`):

```bash
aws sts get-caller-identity       # an assumed-role/voclabs/... ARN
terraform plan
```

**Track B** — no account. Create `infra/offline_override.tf`. Terraform
merges every `*_override.tf` into the matching block, here the
provider, so the plan runs without any AWS call (`.gitignore` already
keeps the file out of Git):

```hcl
# Track B only: lets "terraform plan" run with NO AWS account.
# Terraform merges *_override.tf files into the provider block in versions.tf.
# Never use this file with a real account; delete it on Track A.
provider "aws" {
  access_key                  = "offline"
  secret_key                  = "offline"
  skip_credentials_validation = true
  skip_requesting_account_id  = true
  skip_metadata_api_check     = true
}
```

```bash
terraform plan                    # or: tf plan
```

(This only works because the module has **no data sources**: the zones
are written out in `var.azs`. A `data "aws_availability_zones"` would
call the API and fail offline.)

The end of the plan (example variant, after your TODO):

```text
Plan: 28 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + subnets         = {
      + app-us-east-1a    = "10.42.11.0/24"
      + app-us-east-1b    = "10.42.12.0/24"
      + db-us-east-1a     = "10.42.21.0/24"
      + db-us-east-1b     = "10.42.22.0/24"
      + public-us-east-1a = "10.42.1.0/24"
      + public-us-east-1b = "10.42.2.0/24"
    }
```

`27 to add` = your TODO is missing. Find your `db_from_app` rule in the
plan and check its `from_port` (5432 or 3306) and
`referenced_security_group_id`. Check that **your** prefix is in the
`tags_all` of every resource.

Try the validation once: `terraform plan -var traffic_pattern=spikey`:

```text
Error: Invalid value for variable
  │ var.traffic_pattern is "spikey"
traffic_pattern: steady, spiky or scheduled-peaks.
```

**Don't `apply`** today: a NAT gateway bills from the first minute, and
you'll apply the network in your next build session. Copy the
`Plan:` line and the subnet outputs into `docs/variant.md` as the
first evidence.

---

## Part 3 — CI green (both tracks)

The template's `.github/workflows/ci.yml` has one job, `test` (ruff +
pytest). Add **two jobs** at the end of the file, under `jobs:` (same
indentation as `test:`):

```yaml
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: "1.17.2"
      - run: terraform fmt -check -recursive
      - run: terraform init -input=false -backend=false # no cloud credentials needed
      - run: terraform validate

  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0 # the whole history, not only the last commit
      - name: Secret scan (every commit)
        run: |
          curl -sSL https://github.com/gitleaks/gitleaks/releases/download/v8.30.1/gitleaks_8.30.1_linux_x64.tar.gz | tar xz gitleaks
          ./gitleaks git . --redact --no-banner
```

- `init -backend=false`: CI checks the code without touching any state
  or cloud — no secrets needed for this job.
- `fetch-depth: 0`: gitleaks scans **every** commit. A password removed
  in the next commit is still found.

Check before you push (`git status` must not show `.env`, `.vault/`,
`*.tfstate` or `offline_override.tf`):

```bash
cd ..                              # repository root
git status
git add -A
git commit -m "Week 15: variant, design docs, network skeleton, CI"
git push
gh run watch                       # or the Actions tab
gh run list --limit 1              # completed  success  ci
```

All three jobs green. If `terraform` fails on `fmt -check`, run
`terraform fmt -recursive` and push again. If `test` fails on `ruff
format --check`, you edited a Python file: `make fmt`.

---

## Part 4, Track A — a secret through `LabInstanceProfile`

> **Using AWS Academy Learner Lab**
>
> 1. **Start Lab**, wait for the green dot, copy the credentials from
>    **AWS Details → AWS CLI → Show** into `~/.aws/credentials`. They
>    **expire with the session (about 4 hours)**.
> 2. Region **`us-east-1`**. You **can't create IAM roles**: the
>    instance uses the existing **`LabInstanceProfile`** (role
>    `LabRole`).
> 3. If Secrets Manager answers `AccessDenied`, use the SSM Parameter
>    Store commands (the default below). Write the gap into the report.
> 4. When you're done: A4.5, then **End Lab**.

> **[not run]** Track A was **not run** by the author (no Learner Lab
> session). `terraform validate` of the Track A code was run; the AWS
> CLI commands below follow the AWS documentation. If something behaves
> differently, write the command and the error down and ask.

### A4.1 Store the secret

The app's `SECRET_KEY` (≥ 32 characters). Generate it into a file only
you can read, store it, delete the file — it never appears on the
command line or in your shell history:

```bash
PREFIX=ci-example-0042                       # your resource_prefix
umask 077
python3 -c 'import secrets; print(secrets.token_urlsafe(48), end="")' > /tmp/sk
aws ssm put-parameter --name "/$PREFIX/app/secret_key" --type SecureString \
  --value file:///tmp/sk --tags Key=Project,Value=$PREFIX
rm /tmp/sk
aws ssm describe-parameters --parameter-filters "Key=Name,Values=/$PREFIX/app/secret_key" \
  --query 'Parameters[].[Name,Type,Version]' --output table
```

(Secrets Manager instead: `aws secretsmanager create-secret --name
"$PREFIX/app/secret_key" --secret-string file:///tmp/sk` — $0.40 per
secret per month from the lab credit. SSM standard parameters are
free; `SecureString` is encrypted with the `aws/ssm` KMS key.)

### A4.2 The boot script

What every node will do in your project: read the secret **with the
instance role** and write it to a file only the app user can read.
`$PREFIX` is filled in now; everything else runs on the instance:

```bash
{ echo '#!/bin/bash'; echo "PREFIX=$PREFIX"; cat <<'EOF'
# Runs once at boot, as root. The instance profile authorises the read: no keys here.
set -euo pipefail
useradd --system --uid 10001 app 2>/dev/null || true
install -d -m 0750 -o root -g app /run/campus
umask 077
aws ssm get-parameter --region us-east-1 --name "/$PREFIX/app/secret_key" --with-decryption \
  --query Parameter.Value --output text > /run/campus/secret_key
chown app: /run/campus/secret_key && chmod 0400 /run/campus/secret_key
# Prove it without printing it: mode, owner, length
echo "SECRET-CHECK $(stat -c '%a %U %s' /run/campus/secret_key)" > /dev/console
EOF
} > fetch-secret.sh
cat fetch-secret.sh              # no secret in it - only the parameter NAME
```

(`/run` is a tmpfs: the file disappears at shutdown and is fetched
again at the next boot — the Learner Lab stops instances between
sessions, so this is exactly what you want.)

### A4.3 A test instance with `LabInstanceProfile`

In the default VPC (your own network isn't applied yet):

```bash
IID=$(aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t3.micro --iam-instance-profile Name=LabInstanceProfile \
  --metadata-options HttpTokens=required \
  --user-data file://fetch-secret.sh \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$PREFIX-secret-test},{Key=Project,Value=$PREFIX}]" \
  --query 'Instances[0].InstanceId' --output text)
echo "$IID"
aws ec2 wait instance-status-ok --instance-ids "$IID"         # ~3 min
aws ec2 get-console-output --instance-id "$IID" --latest --output text | grep SECRET-CHECK
```

Expected:

```text
SECRET-CHECK 400 app 65
```

`400` = read-only for the owner, `app` = the owner, `65` = 64
characters + the newline from `--output text` (the app strips it).

### A4.4 Who else can read it?

With SSM Run Command (no SSH, no open port — week 13):

```bash
CID=$(aws ssm send-command --instance-ids "$IID" --document-name AWS-RunShellScript \
  --parameters 'commands=["stat -c \"%a %U\" /run/campus/secret_key","sudo -u ec2-user cat /run/campus/secret_key > /dev/null && echo ec2-user CAN read || echo ec2-user denied"]' \
  --query Command.CommandId --output text)
sleep 5
aws ssm get-command-invocation --command-id "$CID" --instance-id "$IID" \
  --query StandardOutputContent --output text
```

Expected: `400 app` and `ec2-user denied`. (If the command stays
`Pending`, the SSM agent hasn't registered yet: wait a minute. The
console output from A4.3 is enough evidence.)

Now the **gap**: `LabRole` can read **every** parameter in the account,
not just yours. Write the policy you **would** attach to the node role
in a real account into `docs/iam/node-secrets-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOnlyMyAppSecrets",
      "Effect": "Allow",
      "Action": ["ssm:GetParameter", "ssm:GetParameters"],
      "Resource": "arn:aws:ssm:us-east-1:<ACCOUNT_ID>:parameter/ci-example-0042/app/*"
    }
  ]
}
```

(With the AWS-managed `aws/ssm` key no `kms:Decrypt` statement is
needed; with your own KMS key, add `kms:Decrypt` on that key only.)

### A4.5 Cleanup (required)

```bash
aws ec2 terminate-instances --instance-ids "$IID" --query 'TerminatingInstances[0].CurrentState.Name'
aws ec2 describe-instances --filters "Name=tag:Project,Values=$PREFIX" \
  "Name=instance-state-name,Values=pending,running,stopping" --query 'Reservations[].Instances[].InstanceId'
```

Keep the parameter: it's free and your project uses it. (Secrets
Manager: keep it too, or `delete-secret --force-delete-without-recovery`
— a normal delete keeps the name reserved for 7–30 days, and
re-creating it fails.)
Delete `fetch-secret.sh` or move it to `scripts/` — it holds no secret.
Then **End Lab**.

---

## Part 4, Track B — a secret through Vault (no AWS)

A **Vault dev server** plays the cloud secret manager. A one-shot
`fetch-secrets` container plays the node's boot script: it logs in with
a token that may read **one path only** (the instance role), writes one
file per secret with mode `0400` for the user that needs it, and only
then do the database and the app start. The app reads
`SECRET_KEY_FILE` and `DB_PASSWORD_FILE` — exactly as in the cloud.

```text
 vault (dev, :8200)          fetch-secrets (runs once)            postgres         app
 secret/<prefix>/app  ──read with the app token──►  /out/app/* (uid 10001) ──► /run/secrets ──► SECRET_KEY_FILE
                                                    /out/db/*  (uid 999)  ──► /run/secrets ──► POSTGRES_PASSWORD_FILE
```

### B4.1 The compose file

In the repository root, `compose.vault.yaml` (Track B only; the
template's `compose.yaml` stays as it is):

```yaml
# Track B: a Vault dev server plays the cloud secret manager.
#   export RESOURCE_PREFIX=<your resource_prefix>
#   docker compose -f compose.vault.yaml up -d --build
name: ${RESOURCE_PREFIX:?export RESOURCE_PREFIX first}

services:
  # The "secret manager". Dev mode: in memory, unsealed, gone when the
  # container stops. The root token is fixed so the exercise is repeatable;
  # a real Vault never has a token in a file like this.
  vault:
    image: hashicorp/vault:2.2.2
    command: ["server", "-dev", "-dev-listen-address=0.0.0.0:8200"]
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: dev-only-root-token
    ports: ["127.0.0.1:8200:8200"]
    cap_add: [IPC_LOCK]
    healthcheck:
      test: ["CMD", "vault", "status", "-address=http://127.0.0.1:8200"]
      interval: 2s
      retries: 15

  # The "boot script": reads the secrets with the APP's limited token and
  # writes one file per secret, readable only by the user that needs it.
  fetch-secrets:
    image: hashicorp/vault:2.2.2
    user: "0" # root, only to hand each file to its owner (chown)
    depends_on:
      vault:
        condition: service_healthy
    environment:
      VAULT_ADDR: http://vault:8200
      SECRET_PATH: secret/${RESOURCE_PREFIX}/app
    volumes:
      - ./.vault/app-token:/run/app-token:ro
      - app-secrets:/out/app
      - db-secrets:/out/db
    entrypoint: ["/bin/sh", "-ec"]
    command:
      - |
        export VAULT_TOKEN="$$(cat /run/app-token)"
        umask 077
        vault kv get -field=secret_key  "$$SECRET_PATH" > /out/app/secret_key
        vault kv get -field=db_password "$$SECRET_PATH" > /out/app/db_password
        cp /out/app/db_password /out/db/db_password
        chown 10001 /out/app/* && chmod 0400 /out/app/*
        chown 999 /out/db/* && chmod 0400 /out/db/*
        echo "fetched $$SECRET_PATH -> app: $$(ls /out/app | xargs), db: $$(ls /out/db | xargs)"

  postgres:
    image: postgres:17.12
    depends_on:
      fetch-secrets:
        condition: service_completed_successfully
    environment:
      POSTGRES_DB: campus
      POSTGRES_USER: app_rw
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password # uid 999 = postgres
    volumes:
      - db-secrets:/run/secrets:ro
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_rw -d campus"]
      interval: 3s
      retries: 20

  app:
    build: .
    image: campus-app:dev
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      APP_NAME: ${THEME:-campus-app}
      APP_OWNER: ${APP_OWNER:-Your Name (student ID)}
      SECRET_KEY_FILE: /run/secrets/secret_key   # uid 10001 = the image's "app" user
      DB_PASSWORD_FILE: /run/secrets/db_password
      DB_URL: postgresql+psycopg://app_rw:$${DB_PASSWORD}@postgres:5432/campus
      STORAGE_BACKEND: local
      BUCKET: /app/data/uploads
      PASSWORD_MIN_LENGTH: ${PASSWORD_MIN_LENGTH:-12}
    volumes:
      - app-secrets:/run/secrets:ro
    ports: ["127.0.0.1:${APP_PORT:-8000}:8000"]

volumes:
  app-secrets:
  db-secrets:
  pgdata:
```

Read it for three things: `$$` is a literal `$` for the shell inside the
container (one `$` would be filled in by Compose on your laptop); the
`depends_on` chain (vault healthy → fetch completed → postgres healthy
→ app); and that the app and the database each get **only their own**
volume. Keep PostgreSQL today even if your variant is MySQL: the secret
flow is the same.

### B4.2 Start the secret manager, store the secrets

Set your variant in this terminal (every command below needs it):

```bash
export RESOURCE_PREFIX=ci-example-0042 THEME=library PASSWORD_MIN_LENGTH=14 \
       APP_OWNER="Your Name (student ID)"
docker compose -f compose.vault.yaml up -d vault
docker compose -f compose.vault.yaml ps vault         # (healthy)

# a shortcut: the vault CLI inside the container, as the ROOT token (the "admin")
V() { docker compose -f compose.vault.yaml exec -T -e VAULT_ADDR=http://127.0.0.1:8200 \
        -e VAULT_TOKEN=dev-only-root-token vault vault "$@"; }
```

Generate the values **inside** the container, so they never pass
through your terminal. The DB password gets an upper-case letter, a
digit and a symbol on top of the random part, so it meets the password
policy:

```bash
docker compose -f compose.vault.yaml exec -T -e VAULT_ADDR=http://127.0.0.1:8200 \
  -e VAULT_TOKEN=dev-only-root-token -e P="$RESOURCE_PREFIX" vault sh -c '
  vault kv put "secret/$P/app" \
    secret_key="$(head -c 48 /dev/urandom | base64 | tr -d "\n/+=")" \
    db_password="$(head -c 24 /dev/urandom | base64 | tr -d "\n/+=")-Aa1!"'
V kv metadata get "secret/$RESOURCE_PREFIX/app" | grep -E 'created_time|version'
```

### B4.3 Least privilege: a token for the app

A **policy** that allows reading one path, and a token with only that
policy — what an instance role is to the node:

```bash
V policy write campus-app - <<EOF
path "secret/data/$RESOURCE_PREFIX/app" {
  capabilities = ["read"]
}
EOF
mkdir -p .vault
V token create -policy=campus-app -ttl=8h -field=token > .vault/app-token   # .vault/ is in .gitignore
```

Prove it can't read anything else. Store "another team's" secret as
root, then try it with the app token:

```bash
V kv put secret/other-team/app api_key=not-yours
docker compose -f compose.vault.yaml exec -T -e VAULT_ADDR=http://127.0.0.1:8200 \
  -e VAULT_TOKEN="$(cat .vault/app-token)" vault vault kv get secret/other-team/app
```

```text
Code: 403. Errors:
* 1 error occurred:
	* permission denied
```

### B4.4 Boot the stack

```bash
docker compose -f compose.vault.yaml up -d --build
docker compose -f compose.vault.yaml logs fetch-secrets | tail -1
curl -s localhost:8000/readyz; echo
```

```text
fetch-secrets-1  | fetched secret/ci-example-0042/app -> app: db_password secret_key, db: db_password
{"ready":true,"checks":{"db":true,"storage":true}}
```

(Port 8000 taken? `export APP_PORT=8015` and run `up -d` again.) The
evidence, without printing a secret:

```bash
docker compose -f compose.vault.yaml exec app stat -c '%a %u %s %n' /run/secrets/secret_key /run/secrets/db_password
docker compose -f compose.vault.yaml exec postgres stat -c '%a %u %n' /run/secrets/db_password
docker compose -f compose.vault.yaml exec app env | grep -E 'SECRET|PASSWORD'
docker compose -f compose.vault.yaml exec -u 999 app cat /run/secrets/secret_key   # another user
curl -s localhost:8000/info; echo
```

```text
400 10001 61 /run/secrets/secret_key
400 10001 36 /run/secrets/db_password
400 999 /run/secrets/db_password
SECRET_KEY_FILE=/run/secrets/secret_key
DB_PASSWORD_FILE=/run/secrets/db_password
DB_URL=postgresql+psycopg://app_rw:${DB_PASSWORD}@postgres:5432/campus
cat: /run/secrets/secret_key: Permission denied
{"app":"library","owner":"Your Name (student ID)", ... "db_engine":"postgresql", ...}
```

The environment holds only **paths**; the value of `DB_URL` holds the
placeholder `${DB_PASSWORD}`, filled in by the app from the file. And
the generated password passes your policy (the app's own checker,
inside the container — nothing leaves it but `True`):

```bash
docker compose -f compose.vault.yaml exec app python -c \
  'from app.security import check_password; print(check_password(open("/run/secrets/db_password").read(), 14)["ok"])'
curl -s -X POST localhost:8000/api/admin/check-password -H 'content-type: application/json' \
  -d '{"password":"library2026"}'; echo                       # ok: false — which rules fail?
```

### B4.5 Cleanup (required)

```bash
docker compose -f compose.vault.yaml down -v        # vault is in memory: its secrets are gone too
rm -rf .vault
docker image rm campus-app:dev
docker ps -a --filter "name=$RESOURCE_PREFIX"       # nothing left
```

Keep `compose.vault.yaml` in your repository if you like: it is your
local development setup with secrets as files, and it contains no
secret (only a dev token for an in-memory server bound to
`127.0.0.1`).

### Stretch (optional, both tracks)

- **Rotate `SECRET_KEY`.** Track A: `aws ssm put-parameter --overwrite`
  with a new value (the version number goes up), then reboot the
  instance. Does the file change? (User data runs only at the **first**
  boot: move the fetch into a systemd `oneshot` unit that runs at every
  boot — that's what your nodes need anyway.) **[not run]**. Track B:
  `V kv put` again, then `docker compose -f compose.vault.yaml up -d
  --force-recreate fetch-secrets app`. What would rotating the **DB**
  password need in addition, and in which order? (The lab does it.)
- **The second network layer.** Add an `aws_network_acl` for the two db
  subnets: allow the DB port from the two app CIDRs, the ephemeral ports
  1024–65535 back in, everything out; plan again. Why do the ephemeral
  ports have to be there?
- **IaC scan in CI.** Add a step to the `terraform` job:
  `docker run --rm -v "$PWD:/src" aquasec/trivy:0.69.3 config --exit-code 1 --severity HIGH,CRITICAL /src`.
  What does it say about `0.0.0.0/0` on the load balancer, and why do
  you **accept** that finding (with a reason in the findings table)?

---

## Be ready to explain

1. Pick any three rows of your `docs/variant.md`: what decision does
   each value force, and how will you **prove** it is applied?
2. Your `rpo` and your backup plan: which mechanism protects against a
   dead disk, which against `DROP TABLE`, and how much data can you lose
   in each case?
3. Read your ADR 0001 aloud: what is the downside of your NAT choice,
   and what exactly stops working if that zone fails?
4. Your cost: the 24/7 total, the hourly burn rate, and why your
   **spend** stays within `monthly_budget_usd` even though the 24/7
   total doesn't.
5. `cidrsubnet(var.vpc_cidr, 8, 11)`: what does each argument do? Why
   does the DB SG reference the app **SG** and not `10.42.11.0/24`?
6. Why is `terraform.tfvars` committed but `*.tfstate` and
   `*_override.tf` ignored? What would be in the state once you add
   `random_password`?
7. What do `fetch-depth: 0` and `init -backend=false` do in CI, and why
   does the `terraform` job need no cloud credentials?
8. Track A: who authorised the instance to read the parameter, what
   could `LabRole` read that it shouldn't, and what does your policy
   JSON change? Track B: what plays the instance role, and how did you
   prove least privilege?
9. Why is the secret delivered as a **file** with mode `0400`, and not
   as an environment variable?

## Common mistakes

- **`terraform.tfvars` isn't in Git.** The template ignores `*.tfvars`.
  Add `!infra/terraform.tfvars` to `.gitignore` (Part 2) — the file
  holds your variant and no secrets; the examiner needs it.
- **`Plan: 27 to add`.** Your TODO (`db_from_app`) is missing or
  commented out.
- **`Error: Invalid value for variable`.** Read the message: a value in
  `terraform.tfvars` doesn't match the allowed ones (`yes` instead of
  `true`, `postgres` instead of `postgresql`, a CIDR that isn't /16).
- **`No valid credential sources found` (Track B).** The
  `offline_override.tf` file is missing or not in `infra/`.
- **`ExpiredToken` / `InvalidClientTokenId` (Track A).** The lab
  session ended: copy fresh credentials into `~/.aws/credentials`.
- **The `terraform` CI job fails on `fmt -check`.** Run `terraform fmt
  -recursive` locally and push. The plan worked on your laptop, but CI
  checks formatting first.
- **gitleaks fails in CI.** Something secret was committed — probably
  `.env`. Removing it is not enough: it's in the history. Change the
  value (it has leaked), then remove the file from Git (`git rm
  --cached .env`).
- **`SECRET-CHECK` never appears (Track A).** `get-console-output` lags
  a few minutes; run it again. If the line says `AccessDenied` instead,
  the instance has no `LabInstanceProfile`, or the parameter name/region
  is wrong.
- **`can't create /out/app/secret_key: Permission denied` (Track B).**
  The `user: "0"` line is missing from `fetch-secrets`: the Vault image
  runs as an unprivileged user by default.
- **`service "fetch-secrets" didn't complete successfully` (Track B).**
  `docker compose -f compose.vault.yaml logs fetch-secrets`: usually
  `.vault/app-token` is missing (B4.3), or Vault was restarted — a dev
  server forgets everything, so redo B4.2 and B4.3.
- **`export RESOURCE_PREFIX first` (Track B).** A new terminal: export
  the variables from B4.2 again.
- **The app restarts with `SECRET_KEY ... at least 32 characters`.**
  The secret file is empty: the path in Vault or SSM doesn't match the
  one the fetch step reads.
