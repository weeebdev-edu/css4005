---
theme: seriph
title: "CSS4005 — Lecture 11: Infrastructure as Code with Terraform"
info: |
  CSS4005 — Cloud Infrastructure Construction · Narxoz University
  Lecture 11 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# CSS4005 — Cloud Infrastructure Construction

## Lecture 11: Infrastructure as Code with Terraform

<div class="pt-8 opacity-70">
Nauruzbayeva Farikha · Lesson 11
</div>

---
layout: default
---

# Recap — Lesson 10 (Docker and Containers)

<v-clicks>

- Image vs container? <span v-click class="opacity-60">(the image is the read-only recipe; a container is a running instance of it)</span>
- Why keep containers stateless? <span v-click class="opacity-60">(any container can be replaced; state lives in RDS, S3 or a volume)</span>
- How does an EC2 instance pull from ECR without access keys? <span v-click class="opacity-60">(its instance profile role allows it — `aws ecr get-login-password` uses the role)</span>
- ECS: task definition vs service? <span v-click class="opacity-60">(the recipe for a container vs the thing that keeps N copies running, e.g. behind an ALB)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] Why Infrastructure as Code
- [ ] Terraform basics: providers, resources, variables, outputs
- [ ] The workflow: init → plan → apply → destroy
- [ ] State: what it is and why it's sensitive
- [ ] Building the course network and web fleet as code
- [ ] Modules, environments, import and drift
- [ ] Security, CI and cost → today's practice

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Over weeks 4–9 you clicked and typed your way to a VPC, two subnets,
security groups, a load balancer and an Auto Scaling group. It took
hours, and nobody — including you — could repeat it exactly.

Now the team needs the same thing again for **staging**, and once more
for **production**.

</div>

<div v-click class="mt-8 text-xl font-bold">
Today: describe it once in files, review it like code, and create it —
identically — in two minutes.
</div>

---
layout: section
transition: slide-left
---

# Block 1
## Why Infrastructure as Code

---
---

# The problem with clicking

<v-clicks>

- **Not repeatable:** staging and prod end up different in small ways
  nobody wrote down.
- **Not reviewable:** a teammate can't comment on a console click before
  it happens.
- **No history:** "who opened port 22 to the world, and when?"
- **Drift:** someone fixes production by hand at 2 a.m., and the fix
  exists nowhere else.
- **Slow to rebuild:** after an incident, recreating everything from
  memory takes hours.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
Infrastructure as Code: the infrastructure is described in text files,
kept in Git, reviewed in pull requests, and created by a tool.
</div>

---
---

# Declarative vs imperative

| | Imperative (scripts, CLI) | Declarative (Terraform) |
|---|---|---|
| You write | The **steps**: create this, then that | The **end result**: this should exist |
| Running it twice | Creates things twice (or fails) | Changes nothing the second time |
| Changing something | Write new steps for the change | Edit the description, the tool works out the steps |
| Deleting | Write the delete steps yourself | Remove it from the file (or `destroy`) |

<div v-click class="mt-6 text-sm opacity-70">
Your <code>aws ec2 create-security-group</code> commands from week 9 were
imperative. Today you write <b>what</b> should exist and let Terraform
compute <b>how</b>.
</div>

---
---

# The tools

| Tool | Language | Clouds | Notes |
|---|---|---|---|
| **Terraform** (HashiCorp) | HCL | All major clouds + thousands of providers | The most widely used; BSL licence since 2023 |
| **OpenTofu** | HCL | Same as Terraform | Open-source, community-run fork of Terraform; `tofu` CLI |
| **CloudFormation** | YAML / JSON | AWS only | Built into AWS; no state file to manage |
| **Pulumi** | Python, TypeScript, Go… | Many | Infrastructure in a general-purpose language |

<div v-click class="mt-6 text-sm opacity-70">
We use <b>Terraform</b>. Everything today works the same in OpenTofu —
replace <code>terraform</code> with <code>tofu</code>.
</div>

---
layout: section
transition: slide-left
---

# Block 2
## Terraform basics

---
---

# Providers and version pinning

```hcl {1-9|11-13}
terraform {
  required_version = ">= 1.10"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

<v-clicks>

- A **provider** is a plugin that talks to one API (AWS, GCP, Docker,
  GitHub…). `terraform init` downloads it.
- `~> 6.0` means "any 6.x, never 7.0" — a new major version can break
  your code. The exact version goes into `.terraform.lock.hcl`
  (commit that file).

</v-clicks>

---
---

# Resources

```hcl {1|2-4|5}
resource "aws_vpc" "main" {
  cidr_block           = "10.11.0.0/16"
  enable_dns_hostnames = true
  tags                 = { Name = "lab11-vpc" }
}
```

<v-clicks>

- `resource "<TYPE>" "<NAME>"`: the type comes from the provider
  (`aws_vpc`), the name is **yours** — it's only used inside Terraform.
- Inside: the same settings you chose in the console (CIDR, DNS
  hostnames, tags).
- Other resources refer to it as `aws_vpc.main.id`.

</v-clicks>

---
---

# Variables and outputs

```hcl {1-10|12-15}
variable "desired_capacity" {
  description = "How many web servers the ASG keeps running"
  type        = number
  default     = 2

  validation {
    condition     = var.desired_capacity >= 1 && var.desired_capacity <= 3
    error_message = "desired_capacity must be between 1 and 3."
  }
}

output "alb_dns" {
  description = "Where to send your browser"
  value       = "http://${aws_lb.web.dns_name}"
}
```

<div v-click class="mt-4 text-sm opacity-70">
Set variables with <code>-var desired_capacity=3</code>, a
<code>terraform.tfvars</code> file or <code>TF_VAR_desired_capacity</code>.
Outputs print after <code>apply</code> and with <code>terraform output</code>.
</div>

---
---

# Data sources and locals

```hcl {1-3|5-7|9-11}
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_ssm_parameter" "al2023" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 2)
}
```

<v-clicks>

- A **data source** reads something that already exists — Terraform
  doesn't create or delete it (`data.aws_ssm_parameter.al2023.value`).
- **Locals** are named expressions, so you don't repeat yourself
  (`local.azs`).

</v-clicks>

---
---

# The workflow

```bash {1|2|3|4|5|6}
terraform init       # download providers, set up the backend
terraform fmt        # format the code
terraform validate   # check syntax and references
terraform plan       # show what WOULD change — changes nothing
terraform apply      # make the changes (asks "yes")
terraform destroy    # delete everything this config manages
```

<v-click>

| Plan symbol | Meaning |
|---|---|
| `+` | create |
| `~` | update in place |
| `-/+` | destroy and re-create ("must be replaced") |
| `-` | destroy |

</v-click>

---
---

# Reading a plan

```text
  # aws_autoscaling_group.web will be updated in-place
  ~ resource "aws_autoscaling_group" "web" {
      ~ desired_capacity = 2 -> 3
        name             = "lab11-asg"
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

<v-clicks>

- Always read the **last line** first: how many add / change / destroy?
- "**must be replaced**" means downtime for that resource — e.g. changing
  a subnet's CIDR recreates the subnet.
- A plan with an unexpected `destroy` is the moment to stop and ask.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You run <code>terraform apply</code> twice in a row without changing any
file. What happens the second time?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Nothing: "No changes. Your infrastructure matches the configuration."
Terraform is declarative — the second run finds nothing to do.
</div>

---
layout: section
transition: slide-left
---

# Block 3
## State

---
---

# What state is

<v-clicks>

- `terraform.tfstate` is Terraform's memory: which real resource (ID)
  belongs to which block in your code.
- On every `plan`, Terraform compares **code** ↔ **state** ↔ **reality**.
- Lose the state, and Terraform no longer knows it created your VPC —
  the next `apply` would try to create a second one.
- `terraform state list` and `terraform state show <address>` let you
  look inside.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
State contains resource attributes in <b>plain text</b> — sometimes
passwords and keys. Never commit <code>*.tfstate</code> to Git; add it to
<code>.gitignore</code>.
</div>

---
---

# Remote state for teams

```hcl
terraform {
  backend "s3" {
    bucket       = "narxoz-tfstate-lab11"
    key          = "network/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true # S3-native locking (Terraform 1.10+)
  }
}
```

<v-clicks>

- The state lives in a versioned, encrypted S3 bucket, so the whole team
  (and CI) share one state.
- **Locking** stops two people from running `apply` at the same time.
  Since Terraform 1.10 the S3 backend can lock with a lock file in the
  bucket; the older DynamoDB-table locking is deprecated.
- Today we keep state **local** — fine for one person.

</v-clicks>

---
---

# Dependencies and the graph

```hcl {1-4|6-9}
resource "aws_subnet" "a" {
  vpc_id     = aws_vpc.main.id          # implicit dependency
  cidr_block = "10.11.0.0/24"
}

resource "aws_instance" "app" {
  # ...
  depends_on = [aws_internet_gateway.igw] # explicit, rarely needed
}
```

<v-clicks>

- Referencing another resource (`aws_vpc.main.id`) creates the order
  automatically: VPC first, then the subnet.
- Independent resources are created **in parallel**.
- `destroy` walks the same graph backwards — that's why you no longer
  hit `DependencyViolation` when cleaning up.

</v-clicks>

---
---

# count and for_each

```hcl {1-4|6-12}
resource "aws_instance" "web" {
  count = 2
  # ... names: aws_instance.web[0], aws_instance.web[1]
}

resource "aws_subnet" "public" {
  for_each = { "us-east-1a" = 0, "us-east-1b" = 1 }

  vpc_id            = aws_vpc.main.id
  availability_zone = each.key
  cidr_block        = cidrsubnet("10.11.0.0/16", 8, each.value)
}
```

<v-clicks>

- `count`: N copies addressed by **index** — removing the first one
  shifts all the others.
- `for_each`: copies addressed by **key** (`aws_subnet.public["us-east-1a"]`)
  — removing one key touches only that one. Prefer it for named things.

</v-clicks>

---
---

# lifecycle

```hcl
resource "aws_launch_template" "web" {
  # ...
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_s3_bucket" "backups" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```

<v-clicks>

- `create_before_destroy`: build the replacement first, then delete the
  old one — no gap when something must be replaced.
- `prevent_destroy`: `plan` fails if a change would delete this resource
  — a seatbelt for data you can't lose.

</v-clicks>

---
layout: section
transition: slide-left
---

# Block 4
## The course network, as code

---
---

# Step 1 — VPC and public subnets in two AZs

```hcl {1-5|7-14}
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags                 = { Name = "${var.prefix}-vpc" }
}

resource "aws_subnet" "public" {
  for_each                = { for i, az in local.azs : az => i }
  vpc_id                  = aws_vpc.main.id
  availability_zone       = each.key
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, each.value)
  map_public_ip_on_launch = true
  tags                    = { Name = "${var.prefix}-public-${each.key}" }
}
```

<div v-click class="mt-4 text-sm opacity-70">
<code>cidrsubnet("10.11.0.0/16", 8, 1)</code> = <code>10.11.1.0/24</code>.
Private subnets are the same block with a different offset (e.g.
<code>each.value + 10</code>) and no public IPs.
</div>

---
---

# Step 2 — Internet gateway and routes

```hcl {1-3|5-11|13-17}
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "public" {
  for_each       = aws_subnet.public
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}
```

<div v-click class="mt-4 text-sm opacity-70">
Exactly week 8: a subnet is public because its route table sends
<code>0.0.0.0/0</code> to the internet gateway.
</div>

---
---

# Step 3 — Chained security groups

```hcl {1-4|6-12}
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
```

<v-clicks>

- One rule = one resource (`aws_vpc_security_group_ingress_rule`); the
  recommended style in the current AWS provider.
- `referenced_security_group_id` = "from alb-sg": weeks 4 and 9, now
  reviewable in a pull request.

</v-clicks>

---
---

# Step 4 — Launch template with templatefile

```hcl {1-5|6-8|9-11}
resource "aws_launch_template" "web" {
  name_prefix            = "${var.prefix}-lt-"
  image_id               = data.aws_ssm_parameter.al2023.value
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.web.id]
  user_data = base64encode(templatefile("${path.module}/user-data.sh.tftpl", {
    prefix = var.prefix
  }))
  iam_instance_profile {
    name = data.aws_iam_instance_profile.lab.name
  }
}
```

<div v-click class="mt-4 text-sm opacity-70">
<code>templatefile</code> fills <code>${prefix}</code> into your user data
script, so the same script works in every environment. Launch templates
need user data <b>base64-encoded</b>.
</div>

---
---

# Step 5 — ALB, target group, listener

```hcl {1-6|8-13|15-19}
resource "aws_lb" "web" {
  name               = "${var.prefix}-alb"
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = [for s in aws_subnet.public : s.id]
}

resource "aws_lb_target_group" "web" {
  name     = "${var.prefix}-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.web.arn
  port              = 80
  default_action { ... forward to aws_lb_target_group.web.arn }
}
```

---
---

# Step 6 — The Auto Scaling group

```hcl {1-5|6-8|9-12}
resource "aws_autoscaling_group" "web" {
  name                = "${var.prefix}-asg"
  min_size            = 2
  max_size            = 3
  desired_capacity    = var.desired_capacity
  vpc_zone_identifier = [for s in aws_subnet.public : s.id]
  target_group_arns   = [aws_lb_target_group.web.arn]
  health_check_type   = "ELB"
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }
}
```

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
Weeks 4–9 in about 150 lines. Change <code>desired_capacity</code> to 3,
run <code>plan</code>: one in-place update. That's today's practice.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You change the VPC's <code>cidr_block</code> from
<code>10.11.0.0/16</code> to <code>10.12.0.0/16</code> and run
<code>plan</code>. What do you expect to see?
</div>

<div v-click class="mt-8 text-lg opacity-70">
"must be replaced" for the VPC — and for everything inside it that
depends on it: subnets, security groups, the ALB. A VPC's CIDR can't be
changed in place, so read the plan before you type "yes".
</div>

---
layout: section
transition: slide-left
---

# Block 5
## Modules, environments, import and drift

---
---

# Modules

```hcl {1-7|9-11}
module "network" {
  source   = "./modules/network"
  prefix   = "staging"
  vpc_cidr = "10.20.0.0/16"
}

module "web" {
  source     = "./modules/web"
  subnet_ids = module.network.public_subnet_ids
}

output "alb" { value = module.web.alb_dns }
```

<v-clicks>

- A **module** is a folder of `.tf` files with variables (inputs) and
  outputs. You call it like a function.
- Ready-made modules live in the Terraform Registry, e.g.
  `terraform-aws-modules/vpc/aws` builds a full VPC with public, private
  and database subnets and NAT.

</v-clicks>

---
---

# Staging and production

```text
infra/
├── modules/
│   ├── network/
│   └── web/
└── envs/
    ├── staging/   main.tf  terraform.tfvars  (desired = 1, t3.micro)
    └── prod/      main.tf  terraform.tfvars  (desired = 3, t3.small)
```

<v-clicks>

- Each environment folder calls the **same modules** with different
  values and has its **own state**.
- Terraform **workspaces** (`terraform workspace new staging`) are
  another option: one folder, several states. Folders make the
  differences easier to see in a review.

</v-clicks>

---
---

# Bringing existing resources under Terraform

```hcl
import {
  to = aws_s3_bucket.reports
  id = "narxoz-reports-2026"
}

resource "aws_s3_bucket" "reports" {
  bucket = "narxoz-reports-2026"
}
```

<v-clicks>

- The `import` block adopts a resource you created by hand, without
  recreating it. `plan` shows it as "will be imported".
- `terraform plan -generate-config-out=generated.tf` can even write the
  resource block for you to clean up.

</v-clicks>

---
---

# Drift

<v-clicks>

- **Drift:** someone changed the real infrastructure outside Terraform
  (console, CLI), so reality no longer matches the code.
- `terraform plan` notices it and proposes to put things back the way
  the code says.
- `terraform plan -refresh-only` only **reports** what changed outside
  Terraform ("Objects have changed outside of Terraform"), without
  proposing fixes.
- The fix: change the **code**, not the console. If the manual change was
  right, write it into the code; otherwise `apply` undoes it.

</v-clicks>

---
layout: section
transition: slide-left
---

# Block 6
## Security, CI and cost

---
---

# Secrets and Terraform

<v-clicks>

- Never write passwords in `.tf` or `terraform.tfvars` that you commit.
- Mark sensitive values: `variable "db_password" { sensitive = true }` —
  Terraform hides them in plan output.
- **But** state still stores them in plain text. Keep state in an
  encrypted, access-controlled backend.
- Better: keep the secret in **Secrets Manager** and let the instance
  read it at boot through its role — Terraform only references the
  secret's ARN.

</v-clicks>

```hcl
data "aws_secretsmanager_secret" "db" {
  name = "lab11/db-password"   # only the ARN goes into the launch template
}
```

---
---

# Scanning your Terraform

```bash
trivy config .          # or: checkov -d .   /   tfsec .
```

```text
AWS-0054 (CRITICAL): Listener for application load balancer does not use HTTPS.
AWS-0052 (HIGH):     Application load balancer is not set to drop invalid headers.
AWS-0104 (CRITICAL): Security group rule allows unrestricted egress to any IP address.
```

<v-clicks>

- Scanners catch misconfigurations **before** `apply`: public buckets,
  open security groups, missing encryption.
- Not every finding is a bug: a public ALB is the point of a website. Fix
  what's wrong, and document why you accept the rest.

</v-clicks>

---
---

# Terraform in CI (concept)

<v-clicks>

- On every pull request: `fmt -check`, `validate`, a scanner, and
  `terraform plan` posted as a comment for review.
- On merge to `main`: `terraform apply` from the pipeline — nobody runs
  `apply` from a laptop against production.
- CI needs cloud credentials: short-lived ones (OIDC roles) instead of
  long-lived access keys. Week 13 builds the pipeline.

</v-clicks>

---
---

# Terraform in AWS Academy Learner Lab

<v-clicks>

- Credentials come from **AWS Details** and **expire** when the session
  ends: copy them again into `~/.aws/credentials` before each `plan`.
- Region **`us-east-1`** (per the Learner Lab Readme, `us-east-1` and
  `us-west-2` only).
- You **can't create IAM roles**: reference the existing
  `LabInstanceProfile` with a data source
  (`data "aws_iam_instance_profile" "lab"`).
- If a resource is refused (`AccessDenied`), the lab doesn't allow it —
  check the Readme, or use Track B.

</v-clicks>

---
---

# Cost hygiene

<v-clicks>

- Everything Terraform created, `terraform destroy` removes — in the
  right order, in one command.
- An ALB and running instances bill **every hour** they exist. After
  class: `destroy`, then **End Lab**.
- Tags (`default_tags` in the provider block) mark every resource with
  your project name, so you can find stray ones in the console.
- `terraform plan` before every `apply` is also a cost check: count what
  you're about to create.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You deleted <code>terraform.tfstate</code> by accident, but everything is
still running in AWS. What does the next <code>terraform apply</code> try
to do, and how do you avoid it?
</div>

<div v-click class="mt-8 text-lg opacity-70">
It tries to create everything again (names may clash and fail). Bring
the resources back into state with <code>import</code> blocks — and use a
versioned remote backend so it can't happen again.
</div>

---
layout: section
transition: slide-left
---

# Block 7
## Today's practice

---
---

# Today's practice — weeks 4–9 in one command

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Track A — Learner Lab**

1. A Terraform project: VPC, 2 public subnets in 2 AZs, IGW, routes
2. alb-sg and web-sg (80 only from alb-sg) — you write one rule
3. Launch template, ASG 2/2/3, ALB, target group, listener
4. `plan` → `apply` → `curl` alternates between servers
5. desired 2 → 3 in code: an in-place update
6. `terraform destroy`

</div>
<div>

**Track B — no AWS**

The same workflow with the **Docker provider**: a network, N web
containers, an nginx load balancer. Plan, apply, scale with a
variable, cause drift by hand, destroy.

<div v-click class="mt-4 opacity-70">
Checked orally: show your plan output, the load balancer answering,
the scale change, the state, and the destroy.
</div>

</div>
</div>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Explain why infrastructure should be code, and declarative vs imperative
- [ ] Write providers, resources, variables, outputs and data sources
- [ ] Run init → plan → apply → destroy and read a plan
- [ ] Explain what state is, why it's sensitive and where teams keep it
- [ ] Use for_each, implicit dependencies and lifecycle rules
- [ ] Describe modules, environments, import and drift
- [ ] Keep secrets out of code and scan Terraform for misconfigurations

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Finish today's practice and run **`terraform destroy`** (Track A),
      then **End Lab**
- [ ] Add a second pair of **private** subnets to your code (no public IPs)
      and look at the plan — don't apply it
- [ ] Think about what you'd want to **see** on a dashboard if your ASG
      was under load at 2 a.m.

<div class="mt-8 text-sm opacity-60">
Next week: metrics, logs, dashboards and alarms for the infrastructure
you just built.
</div>

---
layout: end
---

# Next lecture

Monitoring and Logging with AWS CloudWatch — seeing what your
infrastructure is doing.
