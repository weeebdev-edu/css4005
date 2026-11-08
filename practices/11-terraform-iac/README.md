# Practice 11 — Weeks 4–9 in One Command (Terraform)

**Objective:** describe a VPC, security groups, a load balancer and a
self-healing web fleet **as code**, create it with `terraform apply`,
change it by editing a file, and delete all of it with
`terraform destroy`. These are the skills from today's lecture.

**Timebox:** in class: ~50–55 min on Track A (the ALB takes a few
minutes), ~35 min on Track B. If you've been stuck on one step for more
than 10 minutes, stop and ask. Don't push through alone.

> **How you're checked:** this practice is done in class and checked
> **orally**. There is no report. When you finish, keep your terminal
> open. The instructor will ask you to show the items in the
> *Definition of done* and to explain what you did.

## Pick your track

You don't need AWS access to pass. Pick **one** track:

| | Track A — real AWS | Track B — no AWS |
|---|---|---|
| Needs | AWS Academy Learner Lab (or your own account), Terraform ≥ 1.10 | Docker and Terraform ≥ 1.10 |
| Provider | `hashicorp/aws` 6.x | `kreuzwerker/docker` 4.x |
| What you build | VPC, 2 subnets, IGW, routes, 2 SGs, launch template, ASG, ALB | Network, N web containers, an nginx load balancer |
| Slow step | ALB + instances becoming healthy: 3–5 min | none |

The Terraform ideas are the same on both tracks:

| Concept | Track A | Track B |
|---|---|---|
| Provider | `aws` | `docker` |
| "Network" | `aws_vpc` + `aws_subnet` (for_each over 2 AZs) | `docker_network` |
| Identical servers | `aws_launch_template` + `aws_autoscaling_group` | `docker_container.web` with `count` |
| Load balancer | `aws_lb` + target group + listener | `docker_container.lb` running nginx |
| Template file | `user-data.sh.tftpl` → `templatefile()` | `lb.conf.tftpl` → `templatefile()` |
| Desired capacity | `var.desired_capacity` | `var.web_count` |
| Output | `alb_dns` | `lb_url` |

### Install Terraform

- macOS: `brew install hashicorp/tap/terraform`
- Ubuntu / WSL: follow the HashiCorp apt repository steps on
  <https://developer.hashicorp.com/terraform/install>
- Windows: `choco install terraform` or the zip from the same page
- **OpenTofu** works too: install `tofu` and type `tofu` wherever this
  page says `terraform`.

```bash
terraform version     # must be 1.10 or newer
```

## Definition of done

Be ready to **show** each of these:

- [ ] Your `.tf` files, and the line you wrote yourself (Track A: the
      web-sg rule and the output. Track B: the `web_count` change).
- [ ] A `terraform plan` output, and you can read its last line
      (`Plan: X to add, Y to change, Z to destroy`).
- [ ] The load balancer answering from **different servers**
      (`curl` several times).
- [ ] A change made **in code** (desired 2 → 3 / web_count 2 → 3) and the
      plan for it: what is created, changed or replaced, and why.
- [ ] `terraform state list` and one `terraform state show`.
- [ ] Track B: drift — a container removed by hand and put back by
      Terraform.
- [ ] `terraform destroy` finished, and nothing of yours left.

---

## Track A — real AWS (AWS Academy Learner Lab)

> **Using AWS Academy Learner Lab**
>
> 1. Open the **AWS Academy Learner Lab** course → **Modules → Launch AWS
>    Academy Learner Lab** → **Start Lab**. Wait until the dot next to
>    **AWS** turns green.
> 2. Copy the credentials from **AWS Details → AWS CLI → Show** into
>    `~/.aws/credentials` (section `[default]`, three lines including
>    `aws_session_token`). They **expire** with the session: when
>    Terraform says `ExpiredToken`, copy them again.
> 3. Region **`us-east-1`**. Per the Learner Lab Readme, only `us-east-1`
>    and `us-west-2` are allowed.
> 4. You **can't create IAM roles** in Learner Lab. The code reads the
>    existing **`LabInstanceProfile`** with a data source instead.
> 5. If `apply` fails with `AccessDenied` or `UnauthorizedOperation` on a
>    resource, the lab doesn't allow it. Switch to Track B.
> 6. When you're done: `terraform destroy`, then **End Lab**. A load
>    balancer bills every hour it exists.

Check that the credentials work:

```bash
aws sts get-caller-identity     # an assumed-role/voclabs/... ARN
```

### A1. Create the project

```bash
mkdir lab11 && cd lab11
```

Create these files. Put **your own name** in `prefix` (lowercase letters,
digits and dashes only; the ALB name must be ≤ 32 characters).

`versions.tf`:

```hcl
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
  region = var.region

  default_tags {
    tags = {
      Project = var.prefix
      Course  = "CSS4005"
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
  description = "Put your own name here, e.g. lab11-aidana"
  type        = string
  default     = "lab11"
}

variable "vpc_cidr" {
  type    = string
  default = "10.11.0.0/16"
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

# One public subnet per AZ: 10.11.0.0/24 in the first AZ, 10.11.1.0/24 in the second
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
```

`security.tf` — **you write one rule** (the TODO):

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

# The web servers
resource "aws_security_group" "web" {
  name   = "${var.prefix}-web-sg"
  vpc_id = aws_vpc.main.id
}

# TODO: allow TCP port 80 into web-sg ONLY from the ALB's security group.
# Hint: it looks like alb_http above, but instead of cidr_ipv4 it uses
#   referenced_security_group_id = <the id of the alb security group>
resource "aws_vpc_security_group_ingress_rule" "web_from_alb" {
  security_group_id = aws_security_group.web.id
  # ...
}

resource "aws_vpc_security_group_egress_rule" "web_all" {
  security_group_id = aws_security_group.web.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}
```

`user-data.sh.tftpl` (`${prefix}` is filled in by Terraform; `$IID` and
`$AZ` are shell variables on the instance):

```bash
#!/bin/bash
dnf install -y nginx
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
IID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "${prefix}: hello from $IID in $AZ" > /usr/share/nginx/html/index.html
systemctl enable --now nginx
```

`compute.tf`:

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
  user_data              = base64encode(templatefile("${path.module}/user-data.sh.tftpl", { prefix = var.prefix }))

  iam_instance_profile {
    name = data.aws_iam_instance_profile.lab.name
  }

  metadata_options {
    http_tokens = "required" # IMDSv2 only
  }
}

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

  health_check {
    path                = "/"
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
  name                      = "${var.prefix}-asg"
  min_size                  = 2
  max_size                  = 3
  desired_capacity          = var.desired_capacity
  vpc_zone_identifier       = [for s in aws_subnet.public : s.id]
  target_group_arns         = [aws_lb_target_group.web.arn]
  health_check_type         = "ELB"
  health_check_grace_period = 120

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "${var.prefix}-web"
    propagate_at_launch = true
  }
}
```

`outputs.tf` — **you write one output** (the TODO):

```hcl
# TODO: an output called "alb_dns" whose value is "http://" followed by
# the load balancer's DNS name. Hint: the attribute is aws_lb.web.dns_name
# and you can build the string with "http://${...}".

output "subnet_ids" {
  value = { for az, s in aws_subnet.public : az => s.id }
}
```

`.gitignore` (if you put this in Git — state never goes into Git):

```gitignore
.terraform/
*.tfstate
*.tfstate.*
tfplan
```

### A2. init, fmt, validate, plan

```bash
terraform init          # downloads the aws provider, creates .terraform.lock.hcl
terraform fmt           # formats your files (run it after every edit)
terraform validate      # Success! The configuration is valid.
terraform plan -out tfplan
```

Read the **last line** of the plan. You should see **18 to add** (the data sources are only read, not created).
Find your security group rule in the plan and check that it says
`referenced_security_group_id`.

### A3. Apply and test

```bash
terraform apply tfplan          # takes 2–4 minutes (the ALB is the slow part)
terraform output alb_dns
```

Give the instances 2–3 minutes to install nginx and pass the health
check, then:

```bash
ALB=$(terraform output -raw alb_dns)
for i in 1 2 3 4 5 6; do curl -s $ALB; done
```

The answers should alternate between **two instance IDs in two
different AZs**. A `502`/`503` in the first minutes just means the
targets aren't healthy yet.

### A4. Change it in code

Open `variables.tf`, change `desired_capacity`'s default from `2` to `3`,
then:

```bash
terraform plan
```

Expected: `~ desired_capacity = 2 -> 3` and **`Plan: 0 to add, 1 to
change, 0 to destroy`**. The ASG is updated **in place**, not replaced.
Apply it and watch a third instance join:

```bash
terraform apply
for i in $(seq 1 9); do curl -s $ALB; done     # after 2–3 min: 3 different instance IDs
```

Try an invalid value and read the error from your `validation` block:

```bash
terraform plan -var desired_capacity=10
```

### A5. Look at the state

```bash
terraform state list
terraform state show aws_lb.web
```

Find the ALB's `dns_name` and `arn` in the state. That's how Terraform
remembers which real ALB belongs to `aws_lb.web`.

### A6. Destroy (required)

```bash
terraform destroy
```

Type `yes`. Terraform deletes everything in the right order, which takes
a few minutes. Check that nothing is left, then **End Lab**:

```bash
terraform state list                                                    # empty
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerName"       # yours is gone
aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[].AutoScalingGroupName"
```

(Or look in the EC2 console → Load Balancers and Auto Scaling Groups.)

### Stretch (optional): scan your code

Run a configuration scanner (needs Docker):

```bash
docker run --rm -v "$PWD":/src aquasec/trivy:0.69.3 config --severity HIGH,CRITICAL /src
```

You'll see findings such as *the ALB doesn't drop invalid headers*, *the
listener doesn't use HTTPS* and *unrestricted egress*. Fix the first one
(add `drop_invalid_header_fields = true` to `aws_lb.web`), run the scan
again, and be ready to explain why the others are acceptable (or not) for
this lab.

---

## Track B — no AWS (Terraform + Docker)

You need Docker running and Terraform ≥ 1.10. Terraform talks to your
local Docker instead of AWS. Everything else — the workflow, state,
variables, templates, plans — is identical.

### B1. Create the project

```bash
mkdir lab11 && cd lab11
```

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.10"
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 4.6"
    }
  }
}

provider "docker" {}
```

`variables.tf`:

```hcl
variable "prefix" {
  description = "Name prefix for every resource (like the course resource prefix)"
  type        = string
  default     = "lab11"
}

variable "web_count" {
  description = "How many web containers (the 'desired capacity')"
  type        = number
  default     = 2

  validation {
    condition     = var.web_count >= 1 && var.web_count <= 5
    error_message = "web_count must be between 1 and 5."
  }
}

variable "lb_port" {
  description = "Port on your laptop where the load balancer listens"
  type        = number
  default     = 8080
}
```

`network.tf`:

```hcl
# The "VPC": a private Docker network only these containers share
resource "docker_network" "app" {
  name = "${var.prefix}-net"
}
```

`web.tf` — the "launch template + Auto Scaling group":

```hcl
resource "docker_image" "nginx" {
  name         = "nginx:1.27-alpine"
  keep_locally = true
}

# N identical web containers
resource "docker_container" "web" {
  count = var.web_count
  name  = "${var.prefix}-web-${count.index + 1}"
  image = docker_image.nginx.image_id

  networks_advanced {
    name = docker_network.app.id
  }

  upload {
    file    = "/etc/nginx/conf.d/default.conf"
    content = <<-CONF
      server {
        listen 80;
        default_type text/plain;
        location / { return 200 "Hello from ${var.prefix}-web-${count.index + 1} ($hostname)\n"; }
      }
    CONF
  }
}
```

`lb.conf.tftpl` — the load balancer's config, generated from the list of
web containers:

```nginx
upstream web_servers {
  zone web_servers 64k; # share round-robin state between nginx workers
%{ for s in servers ~}
  server ${s}:80;
%{ endfor ~}
}

server {
  listen 80;
  location / { proxy_pass http://web_servers; }
}
```

`lb.tf` — the "ALB":

```hcl
resource "docker_container" "lb" {
  name  = "${var.prefix}-lb"
  image = docker_image.nginx.image_id

  networks_advanced {
    name = docker_network.app.id
  }

  ports {
    internal = 80
    external = var.lb_port
  }

  upload {
    file = "/etc/nginx/conf.d/default.conf"
    content = templatefile("${path.module}/lb.conf.tftpl", {
      servers = docker_container.web[*].name
    })
  }
}
```

`outputs.tf`:

```hcl
output "lb_url" {
  description = "Open this in a browser or curl it"
  value       = "http://localhost:${var.lb_port}"
}

output "web_names" {
  value = docker_container.web[*].name
}
```

### B2. init, plan, apply

```bash
terraform init
terraform fmt
terraform validate
terraform plan -out tfplan        # Plan: 5 to add, 0 to change, 0 to destroy.
terraform apply tfplan
for i in 1 2 3 4; do curl -s $(terraform output -raw lb_url); done
```

```text
Hello from lab11-web-1 (a67300e598b5)
Hello from lab11-web-2 (336d4ba7de5d)
Hello from lab11-web-1 (a67300e598b5)
Hello from lab11-web-2 (336d4ba7de5d)
```

(If port 8080 is busy on your laptop, use `-var lb_port=8081` on every
command, or put `lb_port = 8081` in a `terraform.tfvars` file.)

### B3. Change it in code

Change `web_count`'s default to `3` in `variables.tf` (or run with
`-var web_count=3`):

```bash
terraform plan
```

Expected:

```text
  # docker_container.lb must be replaced
  # docker_container.web[2] will be created
Plan: 2 to add, 0 to change, 1 to destroy.
```

Be ready to explain **why the load balancer is replaced**: its config
file lists the web containers, the list changed, and a container's
uploaded file can't be changed in place. (On AWS the ALB isn't replaced:
the ASG registers new instances in the target group.)

```bash
terraform apply
for i in 1 2 3; do curl -s $(terraform output -raw lb_url); done   # web-1, web-2, web-3
terraform plan -var web_count=9                                    # your validation error
```

### B4. Look at the state

```bash
terraform state list
terraform state show 'docker_container.web[0]'
```

Open `terraform.tfstate` in an editor (read only!) and find the container
IDs. This file is why Terraform knows which real containers belong to
which block.

### B5. Drift: change reality by hand

Delete one web container **outside Terraform**:

```bash
docker rm -f lab11-web-2
terraform plan -refresh-only     # "Objects have changed outside of Terraform"
terraform plan                   # docker_container.web[1] will be created
terraform apply                  # Terraform puts it back
```

(`-refresh-only` may also list harmless attribute differences on the
other containers — Docker reports some empty settings that weren't in
the code. Focus on the deleted container.)

### B6. Destroy

```bash
terraform destroy
docker ps -a --filter name=lab11      # nothing left
```

`keep_locally = true` keeps the `nginx` image, so the next `apply` is
fast.

---

## Be ready to explain

1. What's the difference between `terraform plan` and `terraform apply`?
   What does the last line of a plan tell you?
2. What is the **state file**, and why must it never go into Git?
3. You changed desired capacity / web_count from 2 to 3. What did the plan
   create, change or replace, and why?
4. Where did your code create an **implicit dependency** (one resource
   using another's attribute)? What order did Terraform create things in?
5. `count` vs `for_each`: which did you use where, and why does `for_each`
   suit the subnets better?
6. What is **drift**? How does Terraform notice it, and what's the right
   way to fix it?
7. Track A: why is the instance profile a **data source** and not a
   resource? Track B: what in your code plays the role of the ALB and the
   ASG?
8. How would a team share this state and stop two people running `apply`
   at the same time?

## Common mistakes

- **`ExpiredToken` / `InvalidClientTokenId` (Track A).** The Learner Lab
  session ended or restarted. Copy the new credentials from AWS Details.
- **`Reference to undeclared resource` on `terraform validate`.** A typo
  in a resource address (e.g. `aws_security_group.lb` instead of `.alb`).
  The error shows the file and line.
- **`name` cannot be longer than 32 characters (Track A).** ALB and
  target group names are short. Use a shorter `prefix`.
- **Targets stay unhealthy / `502` (Track A).** Wait 2–3 minutes for user
  data to install nginx. If it persists, check your web-sg rule: it must
  allow port 80 **from alb-sg**.
- **You deleted resources in the console and now `destroy` fails or
  `plan` wants to recreate them.** Don't mix the console and Terraform.
  Run `terraform plan` to see what Terraform thinks exists.
- **`terraform.tfstate` deleted or committed.** Without state, Terraform
  forgets what it created. Keep it local for today, add it to
  `.gitignore`, and never edit it by hand.
- **`Error: Unable to create container ... port is already allocated`
  (Track B).** Another program uses 8080. Use `-var lb_port=8081`.
- **Every request goes to `web-1` (Track B).** The `zone web_servers 64k;`
  line is missing from `lb.conf.tftpl`: each nginx worker keeps its own
  round-robin counter without it.
- **`Cannot connect to the Docker daemon` (Track B).** Start Docker
  Desktop / OrbStack first.
