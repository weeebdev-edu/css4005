# Practice 14 — Game Day: High Availability, Security and Cost

**Objective:** run a fleet that survives losing an availability zone,
then prove it with a **game day**: write down what you expect, break a
zone and the database on purpose, and measure what users actually saw.
Along the way you close the usual security holes (an open database, a
password in the user data, unencrypted disks) and work out what the
whole thing costs per month. These are the skills from today's lecture.

**Timebox:** in class: ~70 min on Track A (RDS takes ~10 min to create
and ~15 min to become Multi-AZ: read the next step while you wait),
~45 min on Track B. If you've been stuck on one step for more than 10
minutes, stop and ask. Don't push through alone.

> **How you're checked:** this practice is done in class and checked
> **orally**. There is no report. When you finish, keep your terminals
> and your **game-day table** open. The instructor will ask you to show
> the items in the *Definition of done* and to explain what you did.

> [!WARNING]
> **Test status of this handout.** Track B was run end to end by the
> author (macOS, Docker Desktop, Compose v2); the numbers marked
> *"author's run"* come from that run. **Track A was never applied to
> AWS**: the Terraform code was written for the Learner Lab and its
> providers were initialised (`hashicorp/aws` 6.73.0, `random` 3.9.1,
> `tls` 4.4.2), but nothing was created in an account, so every AWS
> output below is what the AWS documentation says to expect, not a
> measurement. Steps marked **[not run]** were never executed. If
> something differs, write down what you saw: that's a game-day result
> too.

## Pick your track

You don't need AWS access to pass. Pick **one** track:

| | Track A — real AWS (Learner Lab) | Track B — no AWS |
|---|---|---|
| Needs | AWS Academy Learner Lab, Terraform ≥ 1.10, AWS CLI v2, `curl` | Docker with Compose v2, `curl`, `openssl` (macOS, Linux or WSL) |
| What you build | VPC over 2 AZs, ALB + ASG, RDS PostgreSQL in private DB subnets, the password in SSM Parameter Store | HAProxy as the "ALB", two app "zones", a PostgreSQL primary + streaming replica, secrets as files |
| Slow step | RDS create (~10 min) and the switch to Multi-AZ (~15 min) | Pulling `postgres:17` and `python:3.12-slim` the first time |

The ideas are the same on both tracks:

| Concept | Track A (AWS) | Track B (your laptop) |
|---|---|---|
| Availability zones | `us-east-1a`, `us-east-1b` (subnets) | Containers named `-a` and `-b` |
| Load balancer + health check | ALB, target group on `/healthz` | HAProxy, `option httpchk GET /healthz` |
| Replacing a dead server | ASG (`health_check_type = "ELB"`) | `restart: unless-stopped` |
| Losing a zone | `-var az_down=<az>`: the ASG may no longer use that subnet | `docker compose kill web-a` |
| Database copy in the other zone | RDS **Multi-AZ** standby (synchronous) | Streaming **replica** (asynchronous) |
| Database failover | Automatic, same endpoint (`--force-failover`) | By hand: `pg_promote()` + repoint |
| Where the app finds the DB | SSM parameter `/<prefix>/db/host` | File `config/db_host` |
| The DB password | SSM `SecureString` (KMS), read by `LabRole` | Compose **secret** (a file in `/run/secrets`) |
| DB reachable from | The web servers' security group only | The Compose network only (no published port) |
| Backups | RDS automated backups + point-in-time restore | `pg_dump -Fc` to `backups/` |
| Cost | Tags on everything, an estimate, a budget | The same estimate, on paper |
| Cleanup | `terraform destroy` | `docker compose down -v` |

## Definition of done

Be ready to **show** each of these:

- [ ] Your **game-day table** (below): what you expected before each
      drill, and what you measured.
- [ ] The part you wrote yourself (Track A: the DB security group rule
      and the `az_down` filter. Track B: the HAProxy retry lines and the
      failover commands).
- [ ] The zone drill: how many requests failed, for how long, and why
      not zero (or why zero).
- [ ] The database failover: how long the database was unavailable,
      how many rows were lost (your RPO), and how the app found the new
      database.
- [ ] The security checks from the review, each with its proof
      (command output).
- [ ] The monthly cost estimate of the Track A architecture: the biggest
      item, one hidden cost, and one optimization.
- [ ] Cleanup done, nothing of yours left.

## Before you start: the game-day table

Copy this into a file (`gameday.md`) and fill in the **Expected**
column **before** each drill. Engineers who skip this step learn nothing
from the drill: you can only be surprised if you wrote down what you
expected.

| Drill | Expected: failed requests | Expected: seconds until normal | Measured: failed requests | Measured: seconds | Data lost |
|---|---|---|---|---|---|
| 1. Lose a zone (web tier) | | | | | — |
| 2. (Track B) the same, with retries | | | | | — |
| 3. (Track B) a frozen zone | | | | | — |
| 4. Database failover | | | | | rows: |
| 5. Restore a backup | — | | — | | rows: |

---

## Track A — real AWS (AWS Academy Learner Lab)

> **AWS Academy Learner Lab limits**
>
> 1. **Start Lab**, wait for the green dot, copy **AWS Details → AWS
>    CLI** into `~/.aws/credentials`. The credentials expire with the
>    session (~4 hours): if Terraform says `ExpiredToken`, copy them
>    again and re-run the same command.
> 2. Region **`us-east-1`**. You **can't create IAM roles**: the
>    instances use the existing **`LabInstanceProfile`** (role
>    `LabRole`), which may read SSM parameters.
> 3. **AWS Budgets** is not available in Learner Lab: leave
>    `budget_email` empty and watch the lab's spend counter instead.
> 4. RDS in Learner Lab allows small classes (`db.t3.micro`). If a step
>    fails with `AccessDenied`, the lab doesn't allow it: write down the
>    action and the error and continue with the next step.
> 5. The ALB, the instances and the database bill every hour they exist.
>    Finish with **A8 (destroy)**, then **End Lab**.

```bash
aws sts get-caller-identity     # an assumed-role/voclabs/... ARN
terraform version               # >= 1.10
```

### A1. The project

This is the week 11 fleet with three new parts: **private DB subnets**
in both AZs, an **RDS PostgreSQL** instance, and the DB password in
**SSM Parameter Store**. Every instance reads the DB address and the
password from SSM every 15 seconds and shows on its page whether the
database answers.

```text
                             internet
                                │ :80
                         ALB  (both public subnets)          health check: GET /healthz
                     ┌──────────┴───────────┐
         us-east-1a  ▼                      ▼  us-east-1b
    ┌──────────────────────┐   ┌──────────────────────┐
    │ public 10.14.0.0/24  │   │ public 10.14.1.0/24  │   ASG min 1 / desired 2 / max 3
    │   web (nginx)        │   │   web (nginx)        │   IMDSv2, encrypted gp3, LabRole
    ├──────────────────────┤   ├──────────────────────┤
    │ db 10.14.10.0/24     │   │ db 10.14.11.0/24     │   only the local route
    │   RDS primary        │──▶│   (standby: A5)      │   5432 from web-sg only
    └──────────────────────┘   └──────────────────────┘
                 SSM: /<prefix>/db/host (String), /<prefix>/db/password (SecureString)
```

```bash
mkdir lab14 && cd lab14
```

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.10"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.7"
    }
  }
}

provider "aws" {
  region = var.region

  # Every resource Terraform creates gets these tags: Cost Explorer can
  # group the bill by them, and you can list "everything of lab14" later.
  default_tags {
    tags = {
      Project     = var.prefix
      Course      = "CSS4005"
      Owner       = var.owner
      Environment = "lab"
    }
  }
}
```

`variables.tf`:

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}

variable "prefix" {
  description = "Put your own name here, e.g. lab14-aidana"
  type        = string
  default     = "lab14"
}

variable "owner" {
  description = "Who pays for this: your name (a tag on every resource)"
  type        = string
  default     = "student"
}

variable "vpc_cidr" {
  type    = string
  default = "10.14.0.0/16"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "desired_capacity" {
  description = "How many web servers the ASG keeps running"
  type        = number
  default     = 2

  validation {
    condition     = var.desired_capacity >= 1 && var.desired_capacity <= 3
    error_message = "desired_capacity must be between 1 and 3 (Learner Lab limits)."
  }
}

variable "az_down" {
  description = "Simulate losing an AZ: its subnet is taken out of the ASG (\"\" = all AZs in use)"
  type        = string
  default     = ""
}

variable "db_instance_class" {
  type    = string
  default = "db.t3.micro"
}

variable "db_multi_az" {
  description = "A synchronous standby in the second AZ (doubles the DB cost)"
  type        = bool
  default     = false
}

variable "backup_retention_days" {
  description = "Automated backups (point-in-time restore) kept for N days. Final project variant: 7, 14 or 30"
  type        = number
  default     = 7
}

variable "monthly_budget_usd" {
  description = "Final project variant: 15, 25 or 40"
  type        = number
  default     = 15
}

variable "budget_email" {
  description = "Budget alerts go here. Leave empty in Learner Lab (AWS Budgets is not available there)"
  type        = string
  default     = ""
}
```

`network.tf`:

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 2)
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags                 = { Name = "${var.prefix}-vpc" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.prefix}-igw" }
}

# One public subnet per AZ: 10.14.0.0/24 and 10.14.1.0/24
resource "aws_subnet" "public" {
  for_each = { for i, az in local.azs : az => i }

  vpc_id                  = aws_vpc.main.id
  availability_zone       = each.key
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, each.value)
  map_public_ip_on_launch = true
  tags                    = { Name = "${var.prefix}-public-${each.key}" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = { Name = "${var.prefix}-public-rt" }
}

resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public

  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

# NEW: one private DB subnet per AZ: 10.14.10.0/24 and 10.14.11.0/24
resource "aws_subnet" "db" {
  for_each = { for i, az in local.azs : az => i }

  vpc_id            = aws_vpc.main.id
  availability_zone = each.key
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, 10 + each.value)
  tags              = { Name = "${var.prefix}-db-${each.key}" }
}

# A route table with ONLY the local route: nothing in the DB subnets can
# reach the internet, and the internet can't reach them
resource "aws_route_table" "db" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.prefix}-db-rt" }
}

resource "aws_route_table_association" "db" {
  for_each = aws_subnet.db

  subnet_id      = each.value.id
  route_table_id = aws_route_table.db.id
}
```

`security.tf` — **TODO 2** is yours:

```hcl
# The load balancer accepts HTTP from anywhere
resource "aws_security_group" "alb" {
  name   = "${var.prefix}-alb-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_vpc_security_group_ingress_rule" "alb_http" {
  security_group_id = aws_security_group.alb.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "tcp"
  from_port         = 80
  to_port           = 80
}

resource "aws_vpc_security_group_egress_rule" "alb_all" {
  security_group_id = aws_security_group.alb.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}

# The web servers: port 80 only from the ALB (your week 11 TODO)
resource "aws_security_group" "web" {
  name   = "${var.prefix}-web-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_vpc_security_group_ingress_rule" "web_from_alb" {
  security_group_id            = aws_security_group.web.id
  referenced_security_group_id = aws_security_group.alb.id
  ip_protocol                  = "tcp"
  from_port                    = 80
  to_port                      = 80
}

resource "aws_vpc_security_group_egress_rule" "web_all" {
  security_group_id = aws_security_group.web.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}

# NEW: the database. No egress rules at all: it never starts a connection.
resource "aws_security_group" "db" {
  name   = "${var.prefix}-db-sg"
  vpc_id = aws_vpc.main.id
}

# TODO 2: allow PostgreSQL (TCP 5432) into db-sg ONLY from the web servers'
# security group. Not from a CIDR, not from the ALB, not from 0.0.0.0/0.
# Hint: it looks like web_from_alb above.
resource "aws_vpc_security_group_ingress_rule" "db_from_web" {
  security_group_id = aws_security_group.db.id
  # ...
}
```

`database.tf`:

```hcl
# The password is generated by Terraform: nobody types it, it's not in Git.
# It IS in the state file - that's why the state lives in an encrypted,
# private bucket (week 13).
resource "random_password" "db" {
  length  = 24
  special = false # RDS refuses / @ " and spaces; 24 letters and digits are plenty
}

# The secret store: SSM Parameter Store, SecureString = encrypted with KMS
# (the AWS-managed key alias/aws/ssm). Standard parameters cost nothing.
resource "aws_ssm_parameter" "db_password" {
  name  = "/${var.prefix}/db/password"
  type  = "SecureString"
  value = random_password.db.result
}

# Where the app finds the DB. Not secret. Changing it = repointing the fleet.
resource "aws_ssm_parameter" "db_host" {
  name  = "/${var.prefix}/db/host"
  type  = "String"
  value = aws_db_instance.main.address
}

resource "aws_db_subnet_group" "db" {
  name       = "${var.prefix}-db"
  subnet_ids = [for s in aws_subnet.db : s.id]
}

resource "aws_db_instance" "main" {
  identifier        = "${var.prefix}-db"
  engine            = "postgres"
  engine_version    = "17"
  instance_class    = var.db_instance_class
  allocated_storage = 20
  storage_type      = "gp3"
  storage_encrypted = true # KMS, key alias/aws/rds. Only possible at creation time!

  db_name  = "app"
  username = "dbadmin"
  password = random_password.db.result

  db_subnet_group_name   = aws_db_subnet_group.db.name
  vpc_security_group_ids = [aws_security_group.db.id]
  publicly_accessible    = false
  multi_az               = var.db_multi_az

  backup_retention_period = var.backup_retention_days # automated backups + point-in-time restore
  backup_window           = "03:00-03:30"             # UTC
  copy_tags_to_snapshot   = true

  apply_immediately   = true
  deletion_protection = false # lab only: in production, true
  skip_final_snapshot = true  # lab only: in production, take a final snapshot
}
```

`user-data.sh.tftpl` (week 11's page, plus the DB check):

```bash
#!/bin/bash
# Week 11's page, plus: every 15 s the instance reads the DB address and the
# password from SSM Parameter Store (with its role, LabRole - no keys here)
# and checks that the database answers. ${prefix}, ${region} and the
# parameter names are filled in by Terraform; $VARS are shell variables.
dnf install -y nginx postgresql17 || dnf install -y nginx postgresql16
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
IID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "IID=$IID AZ=$AZ" > /etc/instance-info
echo ok > /usr/share/nginx/html/healthz

cat > /usr/local/bin/dbcheck.sh <<'SCRIPT'
#!/bin/bash
source /etc/instance-info
# Both parameters are read on EVERY run: change /${prefix}/db/host and the fleet follows
HOST=$(aws ssm get-parameter --region ${region} --name ${host_param} --query Parameter.Value --output text)
export PGPASSWORD=$(aws ssm get-parameter --region ${region} --name ${password_param} \
  --with-decryption --query Parameter.Value --output text)
# sslmode=require: encrypted in transit (RDS for PostgreSQL 15+ refuses plain connections anyway)
if psql "host=$HOST dbname=app user=dbadmin sslmode=require connect_timeout=3" -tAc "SELECT 1" >/dev/null 2>&1; then
  DB="db=ok ($HOST)"
else
  DB="db=DOWN ($HOST)"
fi
echo "${prefix}: $IID in $AZ | $DB | checked $(date +%T)" > /usr/share/nginx/html/index.new
mv /usr/share/nginx/html/index.new /usr/share/nginx/html/index.html
SCRIPT
chmod 755 /usr/local/bin/dbcheck.sh

cat > /etc/systemd/system/dbcheck.service <<'UNIT'
[Service]
Type=oneshot
ExecStart=/usr/local/bin/dbcheck.sh
UNIT
cat > /etc/systemd/system/dbcheck.timer <<'UNIT'
[Timer]
OnBootSec=5
OnUnitActiveSec=15
AccuracySec=1
[Install]
WantedBy=timers.target
UNIT

/usr/local/bin/dbcheck.sh
systemctl daemon-reload
systemctl enable --now nginx dbcheck.timer
```

`compute.tf` — **TODO 1** is yours:

```hcl
# Latest Amazon Linux 2023 AMI, looked up at plan time
data "aws_ssm_parameter" "al2023" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

# Learner Lab can't create IAM roles: use the existing instance profile
data "aws_iam_instance_profile" "lab" {
  name = "LabInstanceProfile"
}

resource "aws_launch_template" "web" {
  name_prefix            = "${var.prefix}-lt-"
  image_id               = data.aws_ssm_parameter.al2023.value
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.web.id]

  # The user data gets parameter NAMES, never the password itself
  user_data = base64encode(templatefile("${path.module}/user-data.sh.tftpl", {
    prefix         = var.prefix
    region         = var.region
    host_param     = aws_ssm_parameter.db_host.name
    password_param = aws_ssm_parameter.db_password.name
  }))

  iam_instance_profile {
    name = data.aws_iam_instance_profile.lab.name
  }

  metadata_options {
    http_tokens = "required" # IMDSv2 only
  }

  # NEW: an encrypted gp3 root disk (gp3: ~20% cheaper than gp2)
  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  # NEW: default_tags do NOT reach instances the ASG launches. Tag them
  # (and their disks) here, or they are missing from the cost report.
  tag_specifications {
    resource_type = "instance"
    tags          = { Name = "${var.prefix}-web", Project = var.prefix, Owner = var.owner }
  }
  tag_specifications {
    resource_type = "volume"
    tags          = { Name = "${var.prefix}-web", Project = var.prefix, Owner = var.owner }
  }
}

resource "aws_lb" "web" {
  name               = "${var.prefix}-alb"
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = [for s in aws_subnet.public : s.id] # the ALB stays in BOTH AZs
}

resource "aws_lb_target_group" "web" {
  name                 = "${var.prefix}-tg"
  port                 = 80
  protocol             = "HTTP"
  vpc_id               = aws_vpc.main.id
  deregistration_delay = 30

  health_check {
    path                = "/healthz" # liveness: a DB problem must not take the fleet out
    interval            = 10
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.web.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

resource "aws_autoscaling_group" "web" {
  name             = "${var.prefix}-asg"
  min_size         = 1
  max_size         = 3
  desired_capacity = var.desired_capacity

  # TODO 1: the public subnets the ASG may launch into - all of them,
  # EXCEPT the one in var.az_down (when it's "", none is excluded).
  # Hint: a for expression over aws_subnet.public (key = AZ name, value =
  # the subnet) with an "if" at the end:  [for az, s in ... : s.id if ...]
  vpc_zone_identifier = [for s in aws_subnet.public : s.id]

  target_group_arns         = [aws_lb_target_group.web.arn]
  health_check_type         = "ELB"
  health_check_grace_period = 180

  launch_template {
    id      = aws_launch_template.web.id
    version = aws_launch_template.web.latest_version
  }

  tag {
    key                 = "Name"
    value               = "${var.prefix}-web"
    propagate_at_launch = true
  }
}
```

`budget.tf`:

```hcl
# A monthly cost budget with two email alerts. Created only when
# budget_email is set: AWS Budgets is NOT available in Learner Lab, so
# there you watch the lab's spend counter instead.
resource "aws_budgets_budget" "monthly" {
  count = var.budget_email == "" ? 0 : 1

  name         = "${var.prefix}-monthly"
  budget_type  = "COST"
  limit_amount = tostring(var.monthly_budget_usd)
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  # Early warning: the month's FORECAST will pass 80% of the budget
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = [var.budget_email]
  }

  # The money is actually spent
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.budget_email]
  }
}
```

`outputs.tf`:

```hcl
output "alb_dns" {
  value = "http://${aws_lb.web.dns_name}"
}

output "subnet_ids" {
  value = { for az, s in aws_subnet.public : az => s.id }
}

output "db_id" {
  value = aws_db_instance.main.identifier
}

output "db_sg_id" {
  value = aws_security_group.db.id
}

output "db_host_param" {
  value = aws_ssm_parameter.db_host.name
}
```

`terraform.tfvars` — your name, so your resources don't collide with
your neighbour's (they share the account only in your own head, but the
names also appear in the bill):

```hcl
prefix = "lab14-aidana"   # your name
owner  = "aidana"
```

And the request counter you will use in every drill, `watch.sh`:

```bash
#!/usr/bin/env bash
# Sends ~4 requests per second and counts the failures. Ctrl-C prints the totals.
# Usage: ./watch.sh <url>
URL=${1:?usage: ./watch.sh <url>}
ok=0 err=0
trap 'echo; echo "TOTAL: ok=$ok errors=$err"; exit 0' INT TERM
while true; do
  if answer=$(curl -fsS -m 2 "$URL" 2>&1); then
    ok=$((ok + 1)); echo "$(date +%T) $answer"
  else
    err=$((err + 1)); echo "$(date +%T) --- ERROR #$err: ${answer:0:60}"
  fi
  sleep 0.25
done
```

```bash
chmod +x watch.sh
```

### A2. The two TODOs, then apply

**TODO 2** (`security.tf`): PostgreSQL into `db-sg` only from instances
that carry `web-sg`. Copy the shape of `web_from_alb`: the same four
lines, with the right group and port.

**TODO 1** (`compute.tf`): the subnets for the ASG are all public
subnets **except** the one in `var.az_down`. The map `aws_subnet.public`
has the AZ name as its key, so `for az, s in aws_subnet.public` gives you
both.

```bash
terraform init
terraform fmt
terraform validate
terraform plan        # Plan: 30 to add (counted from the files, not from a real run)
terraform apply       # ~10-12 min: most of it is RDS
```

While it runs, read `database.tf` and `user-data.sh.tftpl` again and
answer: where does the password exist? (Terraform state, SSM, the RDS
instance, the instance's memory for a moment. Where does it **not**
exist?)

When it's done:

```bash
ALB=$(terraform output -raw alb_dns)
curl -s "$ALB/"
# lab14-aidana: i-0a1b2c3d4e5f60718 in us-east-1a | db=ok (lab14-aidana-db.xxxx.us-east-1.rds.amazonaws.com) | checked 10:41:07
curl -s "$ALB/"       # the other instance, the other AZ
```

If a page says `db=DOWN`, TODO 2 is missing or wrong (the instance can't
reach port 5432). The first minutes after boot can show nginx's default
page: user data is still installing.

### A3. The security review

Prove each line of the lecture's checklist with a command. Paste the
output into `gameday.md`.

```bash
P=$(terraform output -raw db_id | sed 's/-db$//')     # your prefix

# 1. The DB: not public, encrypted, backups on
aws rds describe-db-instances --db-instance-identifier "$P-db" --query \
  'DBInstances[0].{Public:PubliclyAccessible,Encrypted:StorageEncrypted,MultiAZ:MultiAZ,Backups:BackupRetentionPeriod,AZ:AvailabilityZone}'

# 2. Who may reach 5432: one rule, a security group, no CIDR
aws ec2 describe-security-group-rules \
  --filters Name=group-id,Values=$(terraform output -raw db_sg_id) \
  --query 'SecurityGroupRules[].{In:!IsEgress,Port:FromPort,FromSG:ReferencedGroupInfo.GroupId,CIDR:CidrIpv4}'

# 3. From your laptop the DB doesn't answer (private IP, no route, SG)
nc -vz -w 5 "$(aws rds describe-db-instances --db-instance-identifier "$P-db" \
  --query 'DBInstances[0].Endpoint.Address' --output text)" 5432       # times out

# 4. The password is a SecureString; without --with-decryption you see no password
aws ssm get-parameter --name "/$P/db/password" --query 'Parameter.[Type,Value]'

# 5. The user data has parameter NAMES, never the password
LT=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names "$P-asg" \
  --query 'AutoScalingGroups[0].LaunchTemplate.LaunchTemplateId' --output text)
aws ec2 describe-launch-template-versions --launch-template-id "$LT" --versions '$Latest' \
  --query 'LaunchTemplateVersions[0].LaunchTemplateData.UserData' --output text \
  | base64 -d | grep -n -i -E 'password|PGPASS'

# 6. Instances: IMDSv2 required, in two AZs; disks encrypted gp3
aws ec2 describe-instances --filters Name=tag:Project,Values="$P" Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].[InstanceId,Placement.AvailabilityZone,MetadataOptions.HttpTokens]' --output table
aws ec2 describe-volumes --filters Name=tag:Project,Values="$P" \
  --query 'Volumes[].[VolumeId,VolumeType,Encrypted]' --output table
```

What you should see: `Public: false`, `Encrypted: true`; one inbound rule
on 5432 with `FromSG` set and `CIDR: null`; `nc` times out; the
parameter is `SecureString` with an encrypted blob as its value; the
user data contains only the parameter **names** (`/lab14-aidana/db/password`)
and `PGPASSWORD=$(aws ssm get-parameter ...)`; `required` on every instance, two different AZs;
every volume `gp3` and `True`.

(Volumes carry `Project` only because the launch template tags them:
`default_tags` don't reach what the ASG launches. Remove the
`tag_specifications` blocks and they would be missing from the cost
report.)

### A4. Game day 1: lose an availability zone

Fill in the **Expected** columns for drill 1 first. Then, in **terminal
1**:

```bash
./watch.sh "$ALB/"
```

In **terminal 2**, see where the instances are, and take one AZ away:

```bash
terraform output subnet_ids      # { "us-east-1a" = "subnet-...", "us-east-1b" = "subnet-..." }
terraform apply -var az_down=us-east-1a      # use the AZ of one of your instances
```

TODO 1 removes that subnet from the ASG. The ASG must not keep an
instance in a subnet it may no longer use: it terminates it and launches
a replacement in the remaining AZ. Watch it:

```bash
TG=$(aws elbv2 describe-target-groups --names "$P-tg" --query 'TargetGroups[0].TargetGroupArn' --output text)
while true; do date +%T; aws elbv2 describe-target-health --target-group-arn "$TG" \
  --query 'TargetHealthDescriptions[].[Target.Id,TargetHealth.State]' --output text; sleep 5; done
aws autoscaling describe-scaling-activities --auto-scaling-group-name "$P-asg" --max-items 4 \
  --query 'Activities[].[StartTime,StatusCode,Description]' --output text
```

**[not run]** Expected (from how the ASG and the ALB work, not
measured): the replacement starts in the other AZ, the old target goes
`draining` for up to 30 s (`deregistration_delay`), and terminal 1 shows
few or no errors — the ALB stops sending new requests to a target before
it's terminated. A real AZ failure is harsher: the instance just stops,
and requests fail until two health checks (2 × 10 s) mark it unhealthy.
For that version of the drill, in terminal 2:

```bash
aws ec2 stop-instances --instance-ids <id-of-one-instance>   # like a power loss
```

Stop terminal 1 with **Ctrl-C** after each drill and copy the `TOTAL`
line into your table. Then give the AZ back:

```bash
terraform apply                  # az_down = "" again
```

The ASG now has 2 instances in one AZ and a subnet with none: watch it
**rebalance** (launch one in the empty AZ, then terminate one in the
full one) in `describe-scaling-activities`.

### A5. Game day 2: RDS Multi-AZ failover

Make the database Multi-AZ. It runs as a modification of the live
instance (no downtime), and takes ~10–20 min: do A6 meanwhile. Put it in
`terraform.tfvars`, so later applies keep it:

```bash
echo 'db_multi_az = true' >> terraform.tfvars
terraform apply
aws rds describe-db-instances --db-instance-identifier "$P-db" \
  --query 'DBInstances[0].[MultiAZ,AvailabilityZone,SecondaryAvailabilityZone,DBInstanceStatus]'
# [ true, "us-east-1a", "us-east-1b", "available" ]
```

Fill in the **Expected** column for drill 4. Then, in terminal 1,
watch the **database** status the instances report (they check every
15 s, so you see the outage with a 15 s resolution):

```bash
while true; do echo "$(date +%T) $(curl -s -m 3 "$ALB/" | grep -o 'db=[A-Za-z]*')"; sleep 2; done
```

In terminal 2, fail it over:

```bash
aws rds reboot-db-instance --db-instance-identifier "$P-db" --force-failover
aws rds describe-events --source-type db-instance --source-identifier "$P-db" --duration 30 \
  --query 'Events[].[Date,Message]' --output text
aws rds describe-db-instances --db-instance-identifier "$P-db" \
  --query 'DBInstances[0].[AvailabilityZone,SecondaryAvailabilityZone]'   # swapped
```

**[not run]** Expected (AWS documentation): the failover usually
completes in **60–120 seconds**; the events show `Multi-AZ instance
failover started` and `... completed`; the endpoint name doesn't change,
so the instances follow it without any change — they just reconnect on
their next check. Committed data is not lost: the standby was
synchronous. Write down the **measured** downtime from terminal 1, and
compare it with the events' timestamps.

Note what kept working during the failover: the web pages (they don't
need the DB to answer `/healthz`). If the target group checked the
database, the ASG would have started replacing healthy instances.

### A6. Cost: tags and the estimate

Everything this project made carries your `Project` tag:

```bash
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values="$P" \
  --query 'ResourceTagMappingList[].ResourceARN' --output text | tr '\t' '\n'
```

Now fill in the monthly estimate (us-east-1, on-demand, 730 hours). The
prices are the lecture's list prices at the time of writing — check two
of them in the [AWS Pricing Calculator](https://calculator.aws/):

| Resource | Quantity | Price | Per month, single-AZ DB | Per month, Multi-AZ DB |
|---|---|---|---|---|
| EC2 `t3.micro` | 2 | $0.0104/h | | |
| EBS gp3 root disks | 2 × 8 GB | $0.08/GB-month | | |
| ALB | 1 (+ ~1 LCU) | $0.0225/h + $0.008/LCU-h | | |
| Public IPv4 addresses | 2 instances + 2 ALB nodes | $0.005/h each | | |
| RDS `db.t3.micro` PostgreSQL | 1 (×2 Multi-AZ) | $0.018/h | | |
| RDS gp3 storage | 20 GB (×2 Multi-AZ) | $0.115/GB-month | | |
| SSM parameters (standard), backups ≤ DB size | | free | $0 | $0 |
| **Total** | | | | |

(The lecture's answer: ≈ $69 and ≈ $84.) Then answer in `gameday.md`:
which item surprised you; what one change would save the most for a
**dev** copy of this environment; and what a NAT gateway in each AZ
would add if the instances moved to private subnets.

Learner Lab has no AWS Budgets. In your own account you would set
`budget_email` and `monthly_budget_usd`: `budget.tf` creates a budget
with a **forecast** alert at 80% and an **actual** alert at 100%.

### A7. (Optional) Repoint the fleet by hand

The instances read `/<prefix>/db/host` on every check. Point it at
something wrong and watch every page go `db=DOWN` within 15 s; then
`terraform apply` puts the right value back (Terraform sees the drift):

```bash
aws ssm put-parameter --name "/$P/db/host" --type String --overwrite --value wrong.example.internal
terraform plan        # ~ aws_ssm_parameter.db_host: value will be updated in-place
terraform apply
```

This is how you would repoint the fleet to a **restored** database
(a point-in-time restore creates a new instance with a new endpoint).

### A8. Destroy (required)

```bash
terraform destroy     # ~5-10 min: RDS deletion is the slow part
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values="$P" \
  --query 'ResourceTagMappingList[].ResourceARN'        # [] (deleted resources can linger here a few minutes)
aws rds describe-db-instances --query 'DBInstances[].DBInstanceIdentifier'
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'
```

(`skip_final_snapshot = true` and `deletion_protection = false` are
lab-only settings: in production you'd keep both safety nets.) Then
**End Lab**.

### Stretch (optional) [not run]

- **HTTPS on the ALB.** Add `tls` to `versions.tf` and a file
  `https.tf`; change the HTTP listener's `default_action` to a redirect;
  then `terraform init -upgrade && terraform apply`.

  ```hcl
  # versions.tf, in required_providers
  tls = {
    source  = "hashicorp/tls"
    version = "~> 4.1"
  }
  ```

  ```hcl
  # Stretch: HTTPS on the ALB with a self-signed certificate imported into ACM.
  # (A real certificate needs a domain you own: ACM validates it by DNS.)
  resource "tls_private_key" "alb" {
    algorithm = "RSA"
    rsa_bits  = 2048
  }

  resource "tls_self_signed_cert" "alb" {
    private_key_pem       = tls_private_key.alb.private_key_pem
    dns_names             = [aws_lb.web.dns_name]
    validity_period_hours = 24
    allowed_uses          = ["key_encipherment", "digital_signature", "server_auth"]

    subject {
      common_name = "${var.prefix}.lab"
    }
  }

  resource "aws_acm_certificate" "alb" {
    private_key      = tls_private_key.alb.private_key_pem
    certificate_body = tls_self_signed_cert.alb.cert_pem
  }

  resource "aws_vpc_security_group_ingress_rule" "alb_https" {
    security_group_id = aws_security_group.alb.id
    cidr_ipv4         = "0.0.0.0/0"
    ip_protocol       = "tcp"
    from_port         = 443
    to_port           = 443
  }

  resource "aws_lb_listener" "https" {
    load_balancer_arn = aws_lb.web.arn
    port              = 443
    protocol          = "HTTPS"
    ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
    certificate_arn   = aws_acm_certificate.alb.arn

    default_action {
      type             = "forward"
      target_group_arn = aws_lb_target_group.web.arn
    }
  }
  ```

  ```hcl
  # compute.tf: the HTTP listener now only redirects
  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
  ```

  `curl -sI "$ALB/"` → `301`, `curl -sk "https://${ALB#http://}/"` → the
  page. Why does a browser warn, and what would a real certificate need?

- **Scale in at night.** `schedule.tf`:

  ```hcl
  # Stretch: scale in at night, scale out in the morning (Almaty time)
  resource "aws_autoscaling_schedule" "night" {
    scheduled_action_name  = "${var.prefix}-night"
    autoscaling_group_name = aws_autoscaling_group.web.name
    recurrence             = "0 20 * * *" # every day at 20:00
    time_zone              = "Asia/Almaty"
    min_size               = 1
    max_size               = 1
    desired_capacity       = 1
  }

  resource "aws_autoscaling_schedule" "morning" {
    scheduled_action_name  = "${var.prefix}-morning"
    autoscaling_group_name = aws_autoscaling_group.web.name
    recurrence             = "0 8 * * MON-FRI" # weekdays at 08:00
    time_zone              = "Asia/Almaty"
    min_size               = 1
    max_size               = 3
    desired_capacity       = 2
  }
  ```

  `aws autoscaling describe-scheduled-actions --auto-scaling-group-name "$P-asg"`.
  How many instance-hours a month does it save? Which **drill** would
  this schedule break at 21:00?

---

## Track B — no AWS (Docker Compose game day)

You need Docker with Compose v2. On Windows, do everything inside
**WSL**. Every AWS part has a stand-in:

```text
                          localhost:8080           stats: localhost:8404
                                │
                        lb (HAProxy) "the ALB"     GET /healthz every 2 s
                     ┌──────────┴──────────┐
          "zone a"   ▼                     ▼   "zone b"
              ┌─────────────┐      ┌─────────────┐
              │   web-a     │      │   web-b     │   restart: unless-stopped
              ├─────────────┤      ├─────────────┤
              │ db-primary  │─────▶│ db-replica  │   streaming replication (async)
              └─────────────┘      └─────────────┘
       writer: one INSERT every 0.5 s into whatever config/db_host names
       secrets/*.txt → /run/secrets/... (files, never environment variables)
```

### B1. The project

```bash
mkdir -p lab14ha/app lab14ha/db lab14ha/config lab14ha/secrets lab14ha/backups && cd lab14ha
```

`app/app.py` — the week 10 app, plus a DB status line. It reads the DB
host and the password from **files** on every request:

```python
import os
import socket

import psycopg
from flask import Flask

app = Flask(__name__)
ZONE = os.environ.get("ZONE", "?")  # which "availability zone" this replica runs in


def db_settings():
    # Read on EVERY request: repointing the app = changing one file (like an SSM parameter)
    with open(os.environ["DB_HOST_FILE"]) as f:
        host = f.read().strip()
    with open(os.environ["DB_PASSWORD_FILE"]) as f:  # a secret file, never an env var
        password = f.read().strip()
    return host, password


def db_status():
    host, password = db_settings()
    try:
        with psycopg.connect(host=host, dbname="app", user="app", password=password,
                             connect_timeout=2) as conn:
            notes = conn.execute("SELECT count(*) FROM notes").fetchone()[0]
        return f"db=ok ({host}, {notes} notes)"
    except Exception as e:  # show the error class, not the details
        return f"db=DOWN ({host}: {type(e).__name__})"


@app.get("/")
def hello():
    return f"Hello from {socket.gethostname()} | zone={ZONE} | {db_status()}\n"


@app.get("/healthz")  # liveness: is the process alive? (no DB check on purpose)
def healthz():
    return "ok\n"


@app.get("/crash")  # simulate a process crash: the restart policy must bring it back
def crash():
    os._exit(1)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

`app/requirements.txt`:

```text
flask==3.1.3
psycopg[binary]==3.3.9
```

`app/Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
RUN useradd --create-home appuser
COPY app.py .
EXPOSE 8000
USER appuser
HEALTHCHECK --interval=5s --timeout=2s --start-period=5s --retries=3 \
  CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/healthz', timeout=2)"]
CMD ["python", "app.py"]
```

`db/init-primary.sh` — runs once, when the primary's data directory is
empty: the app's role, the database, the replication role:

```bash
#!/bin/bash
# Runs once, when the primary's data directory is empty.
set -euo pipefail
APP_PW=$(cat /run/secrets/app_db_password)
REPL_PW=$(cat /run/secrets/repl_password)

psql -v ON_ERROR_STOP=1 -U postgres <<SQL
CREATE ROLE app LOGIN PASSWORD '$APP_PW';
CREATE DATABASE app OWNER app;
CREATE ROLE replicator REPLICATION LOGIN PASSWORD '$REPL_PW';
SQL
psql -v ON_ERROR_STOP=1 -U app -d app -c \
  "CREATE TABLE notes (id bigserial PRIMARY KEY, body text NOT NULL)"

# Allow the replica to stream the WAL (the default rules don't cover replication)
echo "host replication replicator all scram-sha-256" >> "$PGDATA/pg_hba.conf"
```

`db/start-replica.sh` — the replica copies the primary once, then
follows it:

```bash
#!/bin/bash
# The replica: on first start, copy the primary (pg_basebackup), then follow it.
set -euo pipefail
if [ ! -s "$PGDATA/PG_VERSION" ]; then
  echo "replica: copying the primary..."
  mkdir -p "$PGDATA" && chown postgres "$PGDATA" && chmod 700 "$PGDATA"
  # -R writes the replication settings; -C -S creates a slot so the primary keeps WAL for us
  gosu postgres env PGPASSWORD="$(cat /run/secrets/repl_password)" \
    pg_basebackup -h db-primary -U replicator -D "$PGDATA" -R -X stream -C -S replica1
fi
exec gosu postgres postgres
```

`compose.yaml`:

```yaml
name: lab14ha

# Two "availability zones" (a, b), each with one app replica, behind a load
# balancer with health checks; a PostgreSQL primary in zone a and a streaming
# replica in zone b. Secrets come from files, never from environment variables.

x-web: &web
  build: ./app
  image: lab14ha-web
  restart: unless-stopped # the "ASG": a crashed process is started again
  volumes:
    - ./config:/config:ro
  secrets: [app_db_password]

services:
  # The "ALB": HAProxy with active health checks on /healthz
  lb:
    image: haproxy:3.2-alpine
    ports:
      - "${LB_PORT:-8080}:80"
      - "${STATS_PORT:-8404}:8404" # the "target group" page: which servers are UP
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro

  web-a:
    <<: *web
    environment:
      ZONE: a
      DB_HOST_FILE: /config/db_host # which DB to use: read on every request
      DB_PASSWORD_FILE: /run/secrets/app_db_password # a file, not the password

  web-b:
    <<: *web
    environment:
      ZONE: b
      DB_HOST_FILE: /config/db_host
      DB_PASSWORD_FILE: /run/secrets/app_db_password

  # The primary database, "zone a"
  db-primary:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/admin_db_password
    volumes:
      - primary-data:/var/lib/postgresql/data
      - ./db/init-primary.sh:/docker-entrypoint-initdb.d/init-primary.sh:ro
    secrets: [admin_db_password, app_db_password, repl_password]
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 2s
      retries: 30

  # The streaming replica, "zone b": a copy of the primary, kept up to date
  db-replica:
    image: postgres:17
    depends_on:
      db-primary:
        condition: service_healthy
    volumes:
      - replica-data:/var/lib/postgresql/data
      - ./db/start-replica.sh:/start-replica.sh:ro
    secrets: [repl_password]
    command: ["bash", "/start-replica.sh"]

  # Writes one row every 0.5 s to whichever DB config/db_host names: the
  # "users" whose data you must not lose (RPO)
  writer:
    image: postgres:17
    depends_on:
      db-primary:
        condition: service_healthy
    volumes:
      - ./config:/config:ro
    secrets: [app_db_password]
    entrypoint: ["bash", "-c"]
    command:
      - |
        export PGPASSWORD=$$(cat /run/secrets/app_db_password)
        while true; do
          id=$$(psql -h "$$(cat /config/db_host)" -U app -d app -qtA -c "INSERT INTO notes(body) VALUES ('written at ' || now()) RETURNING id" 2>&1 | head -1)
          echo "$$(date +%T.%N | cut -c1-12) $$id"
          sleep 0.5
        done

secrets:
  admin_db_password:
    file: ./secrets/admin_db_password.txt
  app_db_password:
    file: ./secrets/app_db_password.txt
  repl_password:
    file: ./secrets/repl_password.txt

volumes:
  primary-data:
  replica-data:
```

`haproxy.cfg` — the **first** version, without retries:

```text
# The "ALB": round robin over two zones, active health checks
defaults
  mode http
  timeout connect 5s
  timeout client 10s
  timeout server 10s
  # TODO (B4): retries

# Docker's DNS: a stopped container disappears from it
resolvers docker
  nameserver dns 127.0.0.11:53
  hold valid 1s

frontend web
  bind :80
  default_backend app

backend app
  balance roundrobin
  option httpchk GET /healthz    # like the target group health check
  # every 2 s; 2 failures = DOWN, 2 successes = UP again
  default-server inter 2s fall 2 rise 2 resolvers docker init-addr last,libc,none
  server web-a web-a:8000 check
  server web-b web-b:8000 check

# The "target group" page: http://localhost:8404
frontend stats
  bind :8404
  stats enable
  stats uri /
  stats refresh 2s
```

`watch.sh` — sends ~4 requests per second and counts the failures:

```bash
#!/usr/bin/env bash
# Sends ~4 requests per second and counts the failures. Ctrl-C prints the totals.
# Usage: ./watch.sh <url>
URL=${1:?usage: ./watch.sh <url>}
ok=0 err=0
trap 'echo; echo "TOTAL: ok=$ok errors=$err"; exit 0' INT TERM
while true; do
  if answer=$(curl -fsS -m 2 "$URL" 2>&1); then
    ok=$((ok + 1)); echo "$(date +%T) $answer"
  else
    err=$((err + 1)); echo "$(date +%T) --- ERROR #$err: ${answer:0:60}"
  fi
  sleep 0.25
done
```

The secrets and the DB address:

```bash
chmod +x watch.sh db/*.sh
for s in admin_db_password app_db_password repl_password; do
  openssl rand -hex 16 > "secrets/$s.txt"
done
echo db-primary > config/db_host
printf 'secrets/\nbackups/\n' > .gitignore     # never commit them
```

### B2. Start the "region"

```bash
docker compose up -d --build
docker compose ps          # lb, web-a, web-b, db-primary, db-replica, writer: running
curl -s localhost:8080/
# Hello from 48ffa6c89b34 | zone=a | db=ok (db-primary, 140 notes)
curl -s localhost:8080/
# Hello from 4ac41d3b0b1c | zone=b | db=ok (db-primary, 141 notes)
```

(If 8080 or 8404 is busy: `LB_PORT=8081 STATS_PORT=8405 docker compose up -d`,
and use that port below.) Open <http://localhost:8404>: both servers
**UP** — this is your "target group".

Is the replica really following the primary?

```bash
docker compose exec db-primary psql -U postgres -c \
  "SELECT application_name, state, sync_state FROM pg_stat_replication"
#  application_name | state     | sync_state
#  walreceiver      | streaming | async
docker compose exec db-replica psql -U postgres -d app -tAc "SELECT count(*) FROM notes"   # grows
docker compose exec db-replica psql -U postgres -d app -c "INSERT INTO notes(body) VALUES ('x')"
# ERROR:  cannot execute INSERT in a read-only transaction
```

### B3. Game day 1: lose a zone (no retries)

Fill in the **Expected** columns for drill 1. Then, in **terminal 1**:

```bash
./watch.sh http://localhost:8080/
```

In **terminal 2**, zone a loses power:

```bash
docker compose kill web-a
```

Watch terminal 1 and the stats page. When only `zone=b` answers, bring
zone a back, wait until both zones answer again, and stop terminal 1
with **Ctrl-C**:

```bash
docker compose start web-a
```

Author's run: **4 failed requests in ~7 s** (`curl: (28) Operation timed
out`), then only zone b, then both again. Why timeouts, and not "connection
refused"? The container is gone, so nothing answers at its old IP at all;
HAProxy waits `timeout connect 5s`, and `watch.sh` gives up after 2 s.
The requests stop failing once two health checks in a row (2 × 2 s) mark
web-a **DOWN**.

(Note: the author's first run used an earlier draft of this
configuration; the "without retries" file above is that draft minus the
retry lines. Your numbers may differ by a request or two.)

### B4. Game day 1 again: with retries

A dead server should cost the user nothing. In `haproxy.cfg`, replace
the `timeout connect` line and the TODO with:

```text
  timeout connect 500ms          # a dead server is given up on quickly...
  retries 2                      # ...the request is retried...
  option redispatch              # ...on the OTHER server
  retry-on conn-failure empty-response response-timeout 502 503
```

```bash
docker compose restart lb
```

Repeat B3 (drill 2 in your table): `./watch.sh`, `docker compose kill
web-a`, wait, `docker compose start web-a`, Ctrl-C.

Author's run: **0 failed requests** in ~1,000, over several zone
outages. A failed connection now costs 0.5 s, and the last retry
goes to the other zone (`redispatch`): the user waits a moment longer
and never sees an error.

The AWS ALB **doesn't** do this for you: it stops sending to a target
after the health checks fail, but a request already sent to a dying
target fails. Retries belong in the client (and are only safe for
requests that can be repeated, like `GET`).

### B5. Game day: a frozen zone

The worst failure isn't a dead server but one that accepts connections
and never answers (a full disk, a deadlock, a stuck GC). Fill in drill 3
first. Then:

```bash
./watch.sh http://localhost:8080/          # terminal 1
docker compose pause web-a                 # terminal 2: the process is frozen
# wait ~15 s
docker compose unpause web-a
```

Author's run: **3 failed requests in ~5 s**, even with retries. The
connection succeeds, so `conn-failure` doesn't apply, and
`response-timeout` would only fire after `timeout server 10s` — longer
than the user waits. Only the health check saves you: 2 failed checks
(2 × 2 s) and web-a is **DOWN**. After `unpause`, 2 good checks and it's
back. That's why detection time (interval × threshold) matters on AWS
too.

### B6. A crashed process

`restart: unless-stopped` plays the ASG: a process that dies is started
again. Crash web-a from **inside** (not through the load balancer — think
about why: what would HAProxy's `retry-on empty-response` do with a
request that crashes the server?):

```bash
docker compose exec web-a python -c \
  "import urllib.request as u; u.urlopen('http://localhost:8000/crash')"
docker compose ps web-a           # Up 3 seconds: started again
```

The command prints an error (the server died in the middle of the
request). **[not run separately]** Expected: no failed requests in
terminal 1 (the retries cover the restart), and the **same** container
ID in `Hello from` afterwards: the container is restarted, not
replaced. On AWS the ASG **replaces** the instance instead: a new ID, a
new IP, a fresh disk.

### B7. Game day 2: the database fails over

Fill in drill 4 first. The writer inserts a row every 0.5 s; watch it in
**terminal 2**, and keep `./watch.sh` in terminal 1:

```bash
docker compose logs -f --since 5s writer      # 00:13:00.512 556  (time, new row id)
```

In **terminal 3**, the primary's zone dies:

```bash
docker compose kill db-primary
```

The pages now say `db=DOWN (db-primary: OperationalError)` and the writer
prints errors. Nothing moves by itself: an asynchronous replica must be
**promoted**, and the app must be **repointed**:

```bash
docker compose exec db-replica psql -U postgres -c "SELECT pg_promote()"    # t
echo db-replica > config/db_host                                            # the "SSM parameter"
```

Author's run: `db=DOWN` for **~2 s**
(`00:13:01` → `00:13:03`), **0 failed HTTP requests** (the pages kept
answering; only the DB status changed), and the note count went on from
556 to 557: **no row lost** — at 2 writes per second, the replica was
fully caught up.

Check your RPO yourself:

```bash
docker compose logs writer | grep -v ERROR | tail -3    # the last ids before and after
docker compose exec db-replica psql -U postgres -d app -tAc "SELECT count(*), max(id) FROM notes"
```

(If ids **jump** after the promotion — e.g. 556, then 590 — that is not
lost data: PostgreSQL logs sequences 32 values ahead, and the replica
continues from the logged value. Count the rows, don't subtract ids.)

Your RTO here was **you**: how fast you typed two commands. With RDS
Multi-AZ (Track A) the same happens automatically in 1–2 minutes; with
an asynchronous replica you can also lose the last few writes that
hadn't reached it yet. Which of the two would you choose for grades?

The old primary must **not** come back as a primary: two primaries =
"split brain", and writes go to both. Leave it stopped (rebuilding it as
a replica of the new primary is the Stretch).

### B8. Backup, and the restore test

A backup taken from the new primary (`-T`: no terminal, so the binary
dump isn't mangled):

```bash
docker compose exec -T db-replica pg_dump -U postgres -Fc app > "backups/app-$(date +%Y%m%d-%H%M).dump"
ls -lh backups/
```

Author's run: with ~300,000 rows in `notes`, the custom-format dump was
**1.4 MB**.

**[not run]** The restore test — fill in drill 5 first, then time it:

```bash
F=$(ls -t backups/*.dump | head -1)
time (docker compose exec -T db-replica createdb -U postgres restore_test &&
      docker compose exec -T db-replica pg_restore -U postgres -d restore_test --no-owner < "$F")
docker compose exec db-replica psql -U postgres -d restore_test -tAc "SELECT count(*) FROM notes"
```

The restore time is your **real** RTO for "someone deleted the table";
the age of the dump is your RPO for it. A nightly dump = up to 24 hours
of data.

### B9. The security checks

**[not run as a separate step]** The design is meant to pass all of
these; run them and paste the output into `gameday.md`:

```bash
docker compose exec web-a env | grep -i -E 'pass|secret'      # only DB_PASSWORD_FILE=/run/secrets/...
docker inspect lab14ha-web-a-1 --format '{{json .Config.Env}}' | grep -c -i password_file   # 1: the file name, not the value
docker compose exec web-a whoami                              # appuser, not root
docker compose ps --format '{{.Service}}: {{.Ports}}'         # only lb publishes ports
git check-ignore secrets/app_db_password.txt                  # needs git init; prints the path
```

Compare with the AWS checklist from the lecture: which line of it does
each command prove? Which items have **no** Track B equivalent
(encryption at rest, IMDSv2, CloudTrail)?

### B10. Cost

Track B costs nothing, but the architecture it imitates doesn't. Fill in
the cost table from **A6** (the prices are in it), and answer the same
three questions there.

### B11. Cleanup

```bash
docker compose down -v          # containers, network, both DB volumes
docker image rm lab14ha-web
docker ps -a --filter name=lab14ha      # nothing left
```

`backups/` and `secrets/` stay on your disk: delete them too if you
don't need them.

### Stretch (optional) [not run]

- **Rebuild the old primary as a replica.** Its data is from the old
  timeline. Remove its volume, point `start-replica.sh` at the **new**
  primary (`-h db-replica`, a new slot name), and start it with that
  script. Now you can fail back.
- **Scan the image.** `docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:0.69.3 image --severity HIGH,CRITICAL --ignore-unfixed lab14ha-web`.
  Which findings come from the base image, and which from your
  `requirements.txt`?

---

## Be ready to explain

1. What were your single points of failure before this practice, and
   which ones are still left (Track A: the region, one NAT-less design,
   the Terraform state; Track B: the laptop itself, the writer)?
2. Your zone drill: expected vs measured. Why weren't the errors zero
   (or why were they)? What decides how long users see errors?
3. Track B: why did a **frozen** zone cause errors when a **dead** one
   didn't? Track A: why must the target group check `/healthz` and not
   the database?
4. Multi-AZ standby vs read replica: synchronous or not, automatic or
   not, readable or not. What was your measured RTO and RPO?
5. Where does the DB password exist, and where does it **not**? Why is
   it still in the Terraform state, and what protects the state?
6. Why does `db-sg` reference `web-sg` instead of the subnet CIDR? What
   would still protect the database if its password leaked?
7. The monthly estimate: the biggest line, one hidden cost, and what you
   would change for a dev copy that runs only in class hours.
8. "Static stability": with 2 AZs and 1 instance each, how much capacity
   is left after losing a zone? How many instances would you need to
   carry full traffic without launching anything?

## Common mistakes

- **`Error: creating EC2 Security Group Rule ... missing required field`
  (Track A).** TODO 2 is still `# ...`: write the four lines.
- **Every page says `db=DOWN` (Track A).** The DB SG rule is wrong (port,
  or it references the ALB SG), or RDS is still `creating`/`modifying`.
  `aws rds describe-db-instances --query 'DBInstances[0].DBInstanceStatus'`.
- **`az_down` has no effect (Track A).** TODO 1 still lists every subnet,
  or you typed an AZ name that isn't in `terraform output subnet_ids`.
- **The drill applied, then the next `apply` "undid" it (Track A).**
  `-var` values aren't remembered: every `terraform apply` without
  `-var az_down=...` puts all AZs back. That's intended — but say so in
  your table.
- **`InvalidParameterCombination` on `--force-failover` (Track A).** The
  instance isn't Multi-AZ yet (A5 still `modifying`). Wait for
  `available`.
- **`ExpiredToken` (Track A).** The lab session ended: new credentials into
  `~/.aws/credentials`, then the same command again. Terraform state is
  local, nothing is lost.
- **`permission denied` reading `/run/secrets/...` (Track B, Linux).**
  Compose (without Swarm) bind-mounts the secret files with your
  permissions: the container users (`appuser`, `postgres`) must be able to
  read them: `chmod 644 secrets/*.txt`. (This handout was tested on
  macOS only.)
- **The replica exits with `could not connect to the primary server`
  (Track B).** The primary was created before `init-primary.sh` existed
  (an old volume): `docker compose down -v`, then `up -d` again.
- **After `kill db-primary` and `start db-primary` nothing works as
  expected (Track B).** You now have two primaries if you promoted the
  replica. Don't restart the old primary (B7).
- **`pg_restore: error: input file does not appear to be a valid archive`
  (Track B).** The dump was taken without `-T`: the terminal mangled the
  binary output. Take it again with `exec -T`.
- **`port is already allocated` (Track B).** Use `LB_PORT=8081
  STATS_PORT=8405` (B2).
