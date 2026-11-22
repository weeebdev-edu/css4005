# Practice 13 — Push to Deploy (CI/CD)

**Objective:** build a pipeline that takes a `git push` all the way to a
running deployment, with nobody touching the console: lint and test the
app, build an image tagged with the commit SHA, push it to a registry,
deploy it without downtime, smoke-test it, and **roll back by itself**
when a broken release gets through. These are the skills from today's
lecture.

**Timebox:** in class: ~60 min on Track A (most of it is waiting for
instances: start the next step's reading while a job runs), ~40 min on
Track B. If you've been stuck on one step for more than 10 minutes, stop
and ask. Don't push through alone.

> **How you're checked:** this practice is done in class and checked
> **orally**. There is no report. When you finish, keep your terminal and
> the pipeline page open. The instructor will ask you to show the items
> in the *Definition of done* and to explain what you did.

## Pick your track

You don't need AWS access to pass. Pick **one** track:

| | Track A — real AWS + GitHub | Track B — no AWS |
|---|---|---|
| Needs | AWS Academy Learner Lab, a GitHub account, `git`, GitHub CLI `gh` (logged in), Terraform ≥ 1.10, AWS CLI v2 | Docker with Compose v2, `git`, `curl`, [`act`](https://github.com/nektos/act) (macOS, Linux or WSL) |
| What you build | Week 11 fleet running the week 10 image from ECR; GitHub Actions: tests + `terraform plan` on PRs, build → scan → push → approval → rolling deploy → smoke test → rollback on `main` | The same pipeline run locally by `act`; a local registry; nginx switching between a **blue** and a **green** container |
| Slow step | First deploy: ~10 min. Each rolling deploy: ~6–8 min | Pulling `python:3.12-slim` the first time |

The CI/CD ideas are the same on both tracks:

| Concept | Track A (AWS + GitHub) | Track B (your laptop) |
|---|---|---|
| Pipeline definition | `.github/workflows/ci.yml` and `deploy.yml` | `.github/workflows/deploy.yml` (same syntax) |
| Runner | GitHub-hosted `ubuntu-latest` | `act` runs the jobs on your machine (`-P self-hosted=-self-hosted`) |
| Trigger | `git push` to GitHub, a pull request | `act push` (simulates a push event) |
| Lint + unit tests | ruff + pytest on the runner | ruff + pytest in a throwaway `python` container |
| Registry | Amazon ECR repository `lab13-web` | `registry:2` on `localhost:5001` |
| Image tag | Short commit SHA | Short commit SHA |
| Image scan | Trivy between build and push (+ ECR scan on push) | Stretch: the same Trivy command |
| CI credentials | Learner Lab session credentials in GitHub Secrets (a real account: OIDC) | None needed |
| Infrastructure in CI | `terraform plan` on PRs, `apply` on `main`, state in S3 with a lockfile | — (Compose starts the "infrastructure") |
| Approval gate | Environment `production`, you as the reviewer | `environment: local` — `act` doesn't pause |
| One deploy at a time | `concurrency: deploy-production` | `concurrency:` (`act` runs one at a time anyway) |
| Load balancer | ALB + target group | nginx container `lab13-lb` |
| Deploy strategy | **Rolling**: instance refresh (1 new instance up, then 1 old one down) | **Blue-green**: start the idle color, switch nginx |
| Health gate | Target group health check on `/healthz` | Docker `HEALTHCHECK` on `/healthz` |
| Smoke test | `verify.sh`: 20 requests, all from the new build | `deploy.sh`: 20 requests, all from the new build |
| Automatic rollback trigger | The 5xx alarm (week 12) goes to `ALARM` | Any failed smoke-test request |
| Rollback | `terraform apply` with the previous tag → another refresh (minutes) | nginx switched back to the old color (seconds) |
| Cleanup | `terraform destroy`, ECR repository, state bucket, secrets | `docker compose down -v` + the app containers |

## Definition of done

Be ready to **show** each of these:

- [ ] The part you wrote yourself (Track A: the rollback step's `if:`.
      Track B: the two `needs:` lines and the rollback line in
      `deploy.sh`).
- [ ] A pipeline run with its jobs in order — and what a failed test
      does to the jobs after it.
- [ ] Your images in the registry, tagged with commit SHAs, and which
      one is live right now (`curl` shows `build=<sha>`).
- [ ] Track A: a pull request with tests and the Terraform plan, and you
      can read its `Plan:` line.
- [ ] A deploy with **no failed requests** (Track A: a rolling instance
      refresh, old and new builds mixing; Track B: the blue → green
      switch).
- [ ] A broken release stopped automatically (Track A: 5xx alarm →
      rollback, red job, site back on the previous build. Track B: the
      health gate `BLOCKED`, and the smoke test `ROLLED BACK`).
- [ ] Where your pipeline's credentials come from and when they expire,
      and how OIDC would replace them (Track B: why `act` needed none).
- [ ] Cleanup done, nothing of yours left.

---

## The app (both tracks)

The week 10 app, with three changes: it prints its **build** (the image
tag, set at deploy time), it has a **`/healthz`** endpoint for health
checks, and it has **unit tests**. Create a folder for the project, and
in it a folder `app/`:

```bash
mkdir -p lab13/app && cd lab13
git init -b main
```

`app/app.py`:

```python
import os
import socket

from flask import Flask

app = Flask(__name__)
COLOR = os.environ.get("APP_COLOR", "blue")
VERSION = os.environ.get("APP_VERSION", "dev")  # the image tag, set at deploy time


@app.get("/")
def hello():
    return f"Hello from {socket.gethostname()} | color={COLOR} | v1 | build={VERSION}\n"


@app.get("/healthz")
def healthz():
    return "ok\n"


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", "8000")))
```

`app/test_app.py`:

```python
from app import app


def test_healthz():
    response = app.test_client().get("/healthz")
    assert response.status_code == 200


def test_index():
    response = app.test_client().get("/")
    assert response.status_code == 200
    assert b"Hello from" in response.data
```

`app/requirements.txt`:

```text
flask==3.1.3
```

`app/requirements-dev.txt` (only for the pipeline's lint and tests — not
in the image):

```text
pytest==9.2.0
ruff==0.17.6
```

`app/.dockerignore`:

```text
__pycache__/
*.pyc
.pytest_cache/
```

`app/Dockerfile` — week 10's, plus a `HEALTHCHECK`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
RUN useradd --create-home appuser
COPY app.py .
ENV APP_COLOR=blue
EXPOSE 8000
USER appuser
# Docker checks /healthz every 5 s; 3 failures in a row = "unhealthy"
HEALTHCHECK --interval=5s --timeout=2s --start-period=5s --retries=3 \
  CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/healthz', timeout=2)"]
CMD ["python", "app.py"]
```

`.gitignore` (in `lab13/`):

```gitignore
__pycache__/
.pytest_cache/
.terraform/
*.tfstate
*.tfstate.*
tfplan
```

Now go to your track.

---

## Track A — real AWS (AWS Academy Learner Lab) + GitHub Actions

> **Using AWS Academy Learner Lab from a pipeline**
>
> 1. Open the **AWS Academy Learner Lab** course → **Modules → Launch AWS
>    Academy Learner Lab** → **Start Lab**. Wait until the dot next to
>    **AWS** turns green.
> 2. Copy the credentials from **AWS Details → AWS CLI → Show** into
>    `~/.aws/credentials` (section `[default]`, three lines including
>    `aws_session_token`). They are **session** credentials: they
>    **expire when the session ends (about 4 hours)**.
> 3. You **can't create IAM users, roles or OIDC providers** in Learner
>    Lab, so the proper way (GitHub OIDC → an IAM role, see the lecture)
>    isn't possible. Instead you copy the **same session credentials
>    into GitHub Secrets** (A3). When a job fails with `ExpiredToken`,
>    copy fresh credentials into `~/.aws/credentials` and run the A3
>    secret loop again. The Stretch shows the other option: a
>    self-hosted runner using `LabInstanceProfile`.
> 4. Region **`us-east-1`**. The instances use the existing
>    **`LabInstanceProfile`** (role `LabRole`) to pull from ECR.
> 5. Your GitHub repository must be **public**: on GitHub Free, required
>    reviewers on an environment only work in public repositories. That's
>    fine — secrets stay secret, and pull requests from forks never get
>    them. Logs of a public repository are public: never print a secret.
> 6. If a step fails with `AccessDenied` on a service, the lab doesn't
>    allow it: note the action and error and ask (or switch to Track B).
> 7. When you're done: A7 (destroy, delete, disable), then **End Lab**. A
>    load balancer bills every hour it exists.

Check that the credentials and `gh` work:

```bash
aws sts get-caller-identity     # an assumed-role/voclabs/... ARN
gh auth status                  # Logged in to github.com
```

### A1. Bootstrap: a state bucket and a registry

Two things must exist **before** the pipeline runs: the S3 bucket for the
Terraform state (it can't hold its own state) and the ECR repository the
pipeline pushes to. Create them once, from your laptop:

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
BUCKET=lab13-tfstate-$ACCOUNT        # bucket names are global: the account ID makes it unique
echo "$BUCKET"

aws s3api create-bucket --bucket "$BUCKET"
aws s3api put-bucket-versioning --bucket "$BUCKET" --versioning-configuration Status=Enabled

aws ecr create-repository --repository-name lab13-web \
  --image-scanning-configuration scanOnPush=true --image-tag-mutability IMMUTABLE
```

(New buckets block public access and are encrypted by default.
Versioning lets you get an old state back if one is ever broken.
`IMMUTABLE`: a tag like `:3f9c2a1` can be pushed only once.)

Keep this terminal open: you need `$BUCKET` again.

### A2. The project

Inside `lab13/` (where `app/` already is):

```bash
mkdir -p infra scripts .github/workflows
cp ../lab11/*.tf infra/                                   # your finished week 11 files
cp ../lab11/.terraform.lock.hcl infra/ 2>/dev/null        # same provider version
```

(If you no longer have `lab11`, recreate it from **Practice 11, A1**,
with both TODOs done. Don't copy `terraform.tfstate` or `.terraform/`.)

You change five files and add two. In `infra/`:

**`versions.tf`** — replace it. The new part is the **S3 backend**: the
state lives in your bucket, shared by your laptop and the pipeline. The
bucket name is given at `init` time.

```hcl
terraform {
  required_version = ">= 1.10"

  # Remote state: one shared state for your laptop and the pipeline.
  # The bucket name is passed at init time: -backend-config="bucket=..."
  backend "s3" {
    key          = "lab13/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true # S3-native lock: two applies can't run at once
  }

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

**`variables.tf`** — change the `prefix` default to `"lab13"` (or
`lab13-<yourname>`; the ALB name must be ≤ 32 characters), then **add**
at the end:

```hcl
variable "ecr_repo" {
  description = "Name of the ECR repository with the app image (created in A1)"
  type        = string
  default     = "lab13-web"
}

variable "image_tag" {
  description = "Which image to run: the git SHA set by the pipeline"
  type        = string
}

variable "alert_5xx_per_min" {
  description = "5xx alarm threshold, errors per minute (final project variant: 5, 10 or 20)"
  type        = number
  default     = 5
}
```

**`user-data.sh.tftpl`** — replace it. Every instance now runs the
**image** (week 10, option 1) instead of installing nginx:

```bash
#!/bin/bash
# Week 10, option 1: every instance runs the same image from ECR.
# ${registry}, ${image} and ${image_tag} are filled in by Terraform.
dnf install -y docker
systemctl enable --now docker

# The instance profile (LabRole) allows the pull: no keys on the server
aws ecr get-login-password --region ${region} \
  | docker login --username AWS --password-stdin ${registry}

docker run -d --name web --restart unless-stopped -p 80:8000 \
  -e APP_VERSION=${image_tag} ${image}
```

**`compute.tf`** — replace it. Compared with week 11: the ECR repository
is read with a data source; the user data gets the image; the target
group checks `/healthz` and drains in 30 s instead of 300 s; and the ASG
uses a **real launch template version** plus an **`instance_refresh`**
block — that is the rolling deploy.

```hcl
# Latest Amazon Linux 2023 AMI, looked up at plan time
data "aws_ssm_parameter" "al2023" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

# Learner Lab can't create IAM roles: use the existing instance profile
data "aws_iam_instance_profile" "lab" {
  name = "LabInstanceProfile"
}

# The ECR repository the pipeline pushes to (created once, outside Terraform)
data "aws_ecr_repository" "app" {
  name = var.ecr_repo
}

resource "aws_launch_template" "web" {
  name_prefix            = "${var.prefix}-lt-"
  image_id               = data.aws_ssm_parameter.al2023.value
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.web.id]

  # A new image_tag = new user data = a new launch template VERSION
  user_data = base64encode(templatefile("${path.module}/user-data.sh.tftpl", {
    region    = var.region
    registry  = split("/", data.aws_ecr_repository.app.repository_url)[0]
    image     = "${data.aws_ecr_repository.app.repository_url}:${var.image_tag}"
    image_tag = var.image_tag
  }))

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
  name                 = "${var.prefix}-tg"
  port                 = 80
  protocol             = "HTTP"
  vpc_id               = aws_vpc.main.id
  deregistration_delay = 30 # default 300 s: every replaced instance would drain for 5 min

  health_check {
    path                = "/healthz"
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
    id = aws_launch_template.web.id
    # A real version number, not "$Latest": then Terraform SEES the new
    # version and starts the instance refresh below.
    version = aws_launch_template.web.latest_version
  }

  # Rolling deploy: when the launch template version changes, replace
  # the instances a few at a time.
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 100 # never go below 2 healthy instances...
      max_healthy_percentage = 150 # ...launch 1 extra first, then terminate 1 old
      instance_warmup        = 60  # seconds before a new instance counts as ready
    }
  }

  tag {
    key                 = "Name"
    value               = "${var.prefix}-web"
    propagate_at_launch = true
  }
}
```

**`outputs.tf`** — replace it (the first output is your week 11 TODO):

```hcl
output "alb_dns" {
  value = "http://${aws_lb.web.dns_name}"
}

output "subnet_ids" {
  value = { for az, s in aws_subnet.public : az => s.id }
}

# Used by the pipeline: what is live now, and what to watch
output "image_tag" {
  value = var.image_tag
}

output "asg_name" {
  value = aws_autoscaling_group.web.name
}

output "alarm_5xx" {
  value = aws_cloudwatch_metric_alarm.alb_5xx.alarm_name
}
```

**`alarms.tf`** — new: the 5xx alarm from week 12 (no SNS today, to keep
it short):

```hcl
# The 5xx alarm from week 12. Here it has a second job: the pipeline
# watches it during a deploy and rolls back if it fires.
resource "aws_cloudwatch_metric_alarm" "alb_5xx" {
  alarm_name          = "${var.prefix}-5xx"
  alarm_description   = "More than ${var.alert_5xx_per_min} target 5xx per minute"
  namespace           = "AWS/ApplicationELB"
  metric_name         = "HTTPCode_Target_5XX_Count"
  dimensions          = { LoadBalancer = aws_lb.web.arn_suffix }
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 1
  comparison_operator = "GreaterThanThreshold"
  threshold           = var.alert_5xx_per_min
  treat_missing_data  = "notBreaching" # no errors = no datapoint = fine
}
```

Check it **locally** before any pipeline exists (this also proves the
S3 backend works):

```bash
cd infra
terraform init -backend-config="bucket=$BUCKET"
terraform fmt -check -recursive        # no output = formatted
terraform validate                     # Success! The configuration is valid.
terraform plan -var image_tag=none     # Plan: 19 to add, 0 to change, 0 to destroy.
cd ..
```

19 = week 11's 18 + the alarm. Don't `apply`: from now on, **only the
pipeline applies**.

`scripts/verify.sh` — runs in the deploy job after `terraform apply`. It
waits for the instance refresh, sends traffic, checks that every answer
comes from the new build (the **smoke test**), and fails if the 5xx
alarm fires during the deploy:

```bash
#!/usr/bin/env bash
# Wait for the rolling deploy, smoke-test it and watch the 5xx alarm.
# Exit 0 = the new build answers every request. Exit 1 = roll back.
# Usage: scripts/verify.sh <asg-name> <alb-url> <expected-build> <alarm-name>
set -uo pipefail
ASG=$1 URL=$2 WANT=$3 ALARM=$4
START=$(date -u +%Y-%m-%dT%H:%M:%S)   # alarms that fired before this don't count
deadline=$(( $(date +%s) + 900 ))      # give up after 15 minutes

while [ "$(date +%s)" -lt "$deadline" ]; do
  refresh=$(aws autoscaling describe-instance-refreshes --auto-scaling-group-name "$ASG" \
    --max-records 1 --query 'InstanceRefreshes[0].[Status,PercentageComplete]' --output text)
  read -r alarm since <<< "$(aws cloudwatch describe-alarms --alarm-names "$ALARM" \
    --query 'MetricAlarms[0].[StateValue,StateUpdatedTimestamp]' --output text)"

  # 20 requests: real traffic for the alarm, and the smoke test
  new=0 old=0 errors=0
  for _ in $(seq 1 20); do
    answer=$(curl -s -m 3 -w ' HTTP%{http_code}' "$URL/" || true)
    case "$answer" in
      *"build=$WANT"*HTTP200) new=$((new + 1)) ;;
      *HTTP200)               old=$((old + 1)) ;;
      *)                      errors=$((errors + 1)) ;;
    esac
  done
  echo "$(date +%T)  refresh: $refresh  alarm: $alarm  answers: new=$new old=$old errors=$errors"

  if [ "$alarm" = ALARM ] && [[ "$since" > "$START" ]]; then
    echo "::error::$ALARM went to ALARM during the deploy"; exit 1
  fi
  case "$refresh" in
    Failed*|Cancelled*) echo "::error::Instance refresh: $refresh"; exit 1 ;;
  esac
  # Done when no refresh is running and all 20 answers came from the new build
  case "$refresh" in
    None*|Successful*) [ "$new" -eq 20 ] && { echo "Smoke test passed: $WANT is live"; exit 0; } ;;
  esac
  sleep 10
done
echo "::error::Timed out: $WANT was not live and healthy after 15 minutes"; exit 1
```

```bash
chmod +x scripts/verify.sh
```

`.github/workflows/ci.yml` — runs on every **pull request**: lint and
test the app, check the image builds, and show the Terraform plan.
Nothing is changed in AWS.

```yaml
name: ci

# Every pull request: test the app and show the Terraform plan.
# Nothing is changed in AWS here.
on:
  pull_request:
    branches: [main]

permissions:
  contents: read

defaults:
  run:
    shell: bash

jobs:
  app:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
      - name: Lint and unit tests
        working-directory: app
        run: |
          pip install -r requirements.txt -r requirements-dev.txt
          ruff check .
          python -m pytest -q
      - name: The image builds
        run: docker build -t narxoz-web:ci app

  plan:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra
    env:
      TF_VAR_ecr_repo: ${{ vars.ECR_REPO }}
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: "1.17.2"
          terraform_wrapper: false
      - name: AWS credentials (Learner Lab session, from repository secrets)
        uses: aws-actions/configure-aws-credentials@v6
        with:
          aws-region: us-east-1
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
      - run: terraform fmt -check -recursive
      - run: terraform init -input=false -backend-config="bucket=${{ vars.TF_STATE_BUCKET }}"
      - run: terraform validate
      - name: terraform plan (with the image that is live now)
        run: |
          TAG=$(terraform output -raw image_tag 2>/dev/null || echo none)
          terraform plan -input=false -lock-timeout=5m -no-color -var "image_tag=$TAG" | tee plan.txt
          {
            echo "### Terraform plan"
            echo '```text'
            grep -E '^  # |^Plan:|^No changes' plan.txt
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"
```

`.github/workflows/deploy.yml` — runs on every **push to `main`**:
test → build → scan → push → (approval) → deploy → verify. You add the
rollback step in A6.

```yaml
name: deploy

# Every push to main: test -> build -> scan -> push -> (approval) -> deploy -> verify
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

# One deploy at a time; a second push waits in the queue
concurrency:
  group: deploy-production
  cancel-in-progress: false

defaults:
  run:
    shell: bash

env:
  AWS_REGION: us-east-1

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
      - name: Lint and unit tests
        working-directory: app
        run: |
          pip install -r requirements.txt -r requirements-dev.txt
          ruff check .
          python -m pytest -q

  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.tag.outputs.tag }}
    steps:
      - uses: actions/checkout@v7
      - name: Image tag = short commit SHA
        id: tag
        run: echo "tag=${GITHUB_SHA:0:7}" >> "$GITHUB_OUTPUT"
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          aws-region: ${{ env.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
      - name: Log in to Amazon ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2
      - name: Build, scan, push
        env:
          IMAGE: ${{ steps.ecr.outputs.registry }}/${{ vars.ECR_REPO }}:${{ steps.tag.outputs.tag }}
        run: |
          docker build -t "$IMAGE" app
          # Stop here if the image has a CRITICAL vulnerability that has a fix
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:0.69.3 \
            image --exit-code 1 --severity CRITICAL --ignore-unfixed --no-progress --skip-version-check "$IMAGE"
          docker push "$IMAGE"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production # waits here until a reviewer approves
    defaults:
      run:
        working-directory: infra
    env:
      TF_VAR_ecr_repo: ${{ vars.ECR_REPO }}
      TAG: ${{ needs.build.outputs.tag }}
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: "1.17.2"
          terraform_wrapper: false
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          aws-region: ${{ env.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
      - run: terraform init -input=false -backend-config="bucket=${{ vars.TF_STATE_BUCKET }}"
      - name: Remember the version that is live now
        id: prev
        run: echo "tag=$(terraform output -raw image_tag 2>/dev/null || echo none)" >> "$GITHUB_OUTPUT"
      - name: terraform apply (new launch template version -> instance refresh)
        run: terraform apply -input=false -auto-approve -lock-timeout=5m -var "image_tag=$TAG"
      - name: Wait for the rollout, smoke test, watch the 5xx alarm
        run: |
          ../scripts/verify.sh "$(terraform output -raw asg_name)" "$(terraform output -raw alb_dns)" \
            "$TAG" "$(terraform output -raw alarm_5xx)"
```

Read `deploy.yml` once from top to bottom and find: the job order
(`needs`), where the tag is made, where the image is scanned, where the
job waits for approval, and where the previous tag is remembered.

### A3. GitHub: repository, secrets, variables, environment

Create a **public** repository from your folder (don't push yet: the
first push deploys, and the secrets must exist first):

```bash
gh repo create lab13-<yourname> --public --source . --remote origin
```

The pipeline's AWS credentials — the **lab session** credentials from
`~/.aws/credentials`. Run this again whenever the lab session restarts:

```bash
for k in aws_access_key_id aws_secret_access_key aws_session_token; do
  gh secret set "$(echo "$k" | tr a-z A-Z)" --body "$(aws configure get "$k")"
done
gh secret list        # AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN
```

Two **variables** (not secret, visible in logs):

```bash
gh variable set TF_STATE_BUCKET --body "$BUCKET"
gh variable set ECR_REPO --body lab13-web
gh variable list
```

The **`production` environment with you as a required reviewer** — this
is the approval gate (or in the browser: **Settings → Environments → New
environment → production → Required reviewers**):

```bash
gh api -X PUT "repos/{owner}/{repo}/environments/production" --input - <<EOF
{"reviewers": [{"type": "User", "id": $(gh api user -q .id)}]}
EOF
gh api "repos/{owner}/{repo}/environments/production" -q '.protection_rules[].type'   # required_reviewers
```

### A4. The first push = the first deploy

```bash
git add -A
git commit -m "Week 13: app, infrastructure and pipeline"
git push -u origin main
gh run list --limit 3
```

Open the run in the browser (`gh run view --web`, or the repository's
**Actions** tab). `test` and `build` run (~3 min: Trivy downloads its
database the first time), then `deploy` shows **Waiting for review**.
Click **Review deployments → production → Approve and deploy**.

The deploy job runs `terraform apply` (19 resources, ~3–4 min), then
`verify.sh` prints a line every ~15 s until the instances have started
Docker, pulled the image and passed the health check:

```text
10:41:12  refresh: None  alarm: INSUFFICIENT_DATA  answers: new=0 old=0 errors=20
...
10:44:05  refresh: None  alarm: INSUFFICIENT_DATA  answers: new=20 old=0 errors=0
Smoke test passed: 4d1e7a2 is live
```

(`refresh: None` — the first deploy creates the ASG; there is nothing to
refresh yet. The `errors` before that are the ALB answering `503` while
no target is healthy.)

Meanwhile, look at what the build pushed:

```bash
aws ecr describe-images --repository-name lab13-web \
  --query 'sort_by(imageDetails,&imagePushedAt)[].[imageTags[0],imagePushedAt]' --output table
git log --oneline -1          # the same 7 characters
```

When the run is green, set the ALB address in a **second terminal**
(you'll keep this one for watching):

```bash
cd lab13/infra
BUCKET=lab13-tfstate-$(aws sts get-caller-identity --query Account --output text)
terraform init -backend-config="bucket=$BUCKET" >/dev/null
ALB=$(terraform output -raw alb_dns)
curl -s $ALB/        # Hello from ip-10-11-... | color=blue | v1 | build=4d1e7a2
```

### A5. A pull request → a plan → a rolling deploy

Make a change on a branch, as a team would:

```bash
git switch -c v2
```

- In `app/app.py`, change `v1` to `v2` in the `return` line of `hello()`.
- In `infra/variables.tf`, set `alert_5xx_per_min`'s default to your
  final project variant's value (`10` if you don't have one yet).

```bash
git commit -am "v2, and the 5xx threshold from my variant"
git push -u origin v2
gh pr create --fill
gh pr checks --watch
```

Both `ci` jobs run. Open the PR → **Checks → ci → plan → Summary**: the
plan shows only the **infrastructure** change, because it plans with the
image that is live now:

```text
  # aws_cloudwatch_metric_alarm.alb_5xx will be updated in-place
Plan: 0 to add, 1 to change, 0 to destroy.
```

That's what a reviewer approves. Merge it:

```bash
gh pr merge --squash --delete-branch
git switch main && git pull
```

In the second terminal, start watching the site **before** you approve:

```bash
while true; do curl -s -m 2 $ALB/ || echo "--- no answer"; sleep 1; done
```

Approve the deploy. `terraform apply` creates launch template version 2
and starts an **instance refresh**. In a third terminal (or between
answers), follow it:

```bash
PREFIX=lab13                 # your var.prefix
aws autoscaling describe-instance-refreshes --auto-scaling-group-name "$PREFIX-asg" \
  --query 'InstanceRefreshes[].[StartTime,Status,PercentageComplete]' --output table
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names "$PREFIX-asg" \
  --query 'AutoScalingGroups[0].Instances[].[InstanceId,LaunchTemplate.Version,LifecycleState,HealthStatus]' --output table
```

What you should see over ~6–8 minutes: a **third** instance with
launch template version 2 starts; when it is healthy, the answers mix
`v1 | build=<old>` and `v2 | build=<new>`; one old instance is
terminated; the same again; then only `v2`. **No `--- no answer` and no
errors** — that's a rolling deploy. The deploy job ends with `Smoke test
passed`.

### A6. A broken release and the automatic rollback

A realistic bug: a new "message of the day" is read from a file. The
unit tests pass (they run in `app/`, where the file exists) — but the
Dockerfile doesn't copy the file into the image.

Create `app/motd.txt`:

```text
Exams start on December 15
```

In `app/app.py`, replace the `hello()` function with:

```python
@app.get("/")
def hello():
    with open("motd.txt") as f:  # message of the day
        motd = f.read().strip()
    return f"Hello from {socket.gethostname()} | color={COLOR} | v3 | build={VERSION} | {motd}\n"
```

Now the part **you write**. Add this step at the **end** of
`.github/workflows/deploy.yml` (same indentation as the step above it)
and replace the `TODO`:

```yaml
      - name: Automatic rollback to the previous image
        # TODO: run this step ONLY if an earlier step failed AND there was a
        #       previous version (on the very first deploy, prev is 'none').
        #       Hints: the status function failure(), the expression
        #       steps.prev.outputs.tag, the operators && and !=
        if: TODO
        env:
          PREV: ${{ steps.prev.outputs.tag }}
        run: |
          echo "::warning::Deploy of $TAG failed - rolling back to $PREV"
          terraform apply -input=false -auto-approve -lock-timeout=5m -var "image_tag=$PREV"
```

Ship it straight to `main` (in a team this would be a PR too):

```bash
git add -A
git commit -m "v3: message of the day"
git push
```

Keep the watch loop running and approve the deploy. What happens:

1. `test` passes — the bug isn't visible to unit tests.
2. The refresh starts a new instance. `/healthz` works, so it becomes
   **healthy** and gets traffic — and every request to it returns
   **`500 Internal Server Error`**. Your watch loop shows the error
   pages mixed with `v2` answers.
3. `verify.sh` keeps sending traffic; within ~2–4 minutes the target 5xx
   count crosses your threshold and the **5xx alarm** goes to `ALARM`:
   `::error::lab13-5xx went to ALARM during the deploy`.
4. Your rollback step runs `terraform apply` with the **previous tag**.
   Terraform cancels the running refresh and starts a new one back to
   `v2`. A few minutes later the watch loop shows only `v2` again.

The run is **red** — correct: the deploy failed, even though the site
recovered. The evidence:

```bash
aws cloudwatch describe-alarm-history --alarm-name "$PREFIX-5xx" --history-item-type StateUpdate \
  --query 'AlarmHistoryItems[].[Timestamp,HistorySummary]' --output table
aws autoscaling describe-instance-refreshes --auto-scaling-group-name "$PREFIX-asg" \
  --query 'InstanceRefreshes[].[StartTime,Status,StatusReason]' --output table
```

Be ready to explain why `/healthz` didn't catch this bug and what did.
(`main` still contains the broken commit: the Stretch fixes it forward.)

### A7. Destroy (required)

The pipeline created everything; your laptop shares its state, so it can
destroy it (wait until no deploy is running):

```bash
cd infra
terraform destroy -var image_tag=none      # type yes; 19 resources
terraform state list                       # empty
cd ..
```

Then remove what A1 and A3 created, and stop the pipeline from building
it all again on the next push:

```bash
aws ecr delete-repository --repository-name lab13-web --force

aws s3api delete-objects --bucket "$BUCKET" --delete "$(aws s3api list-object-versions \
  --bucket "$BUCKET" --query '{Objects: [Versions, DeleteMarkers][].{Key: Key, VersionId: VersionId}}' --output json)"
aws s3api delete-bucket --bucket "$BUCKET"

gh workflow disable deploy.yml
for s in AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN; do gh secret delete "$s"; done
```

(A versioned bucket keeps every old version of the state and a "delete
marker" for every removed lock file: all of them must go before the
bucket can be deleted.) Check, then **End Lab**:

```bash
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerName"
aws ecr describe-repositories --query "repositories[].repositoryName"
aws s3 ls | grep lab13
```

### Stretch (optional)

- **Fix forward.** Add `motd.txt` to the `COPY app.py .` line of the
  Dockerfile (`COPY app.py motd.txt ./`), push, approve, and watch `v3`
  roll out correctly. Why is "fix forward" often better than `git
  revert` here?
- **A self-hosted runner with `LabInstanceProfile`: no keys in GitHub.**
  Only for a **private** repository (a public repo would let strangers'
  pull requests run on your instance) — and on a private repo, GitHub
  Free won't enforce the approval. This wasn't run in class; adjust if
  something differs. From the `lab13/` folder:

  ```bash
  TOKEN=$(gh api -X POST "repos/{owner}/{repo}/actions/runners/registration-token" -q .token)
  REPO_URL=$(gh repo view --json url -q .url)
  cat > runner.sh <<EOF
  #!/bin/bash
  dnf install -y docker git libicu
  systemctl enable --now docker
  useradd -m runner && usermod -aG docker runner
  mkdir /home/runner/actions-runner && cd /home/runner/actions-runner
  curl -sL -o r.tgz https://github.com/actions/runner/releases/download/v2.338.2/actions-runner-linux-x64-2.338.2.tar.gz
  tar xzf r.tgz && chown -R runner: /home/runner
  sudo -u runner ./config.sh --url $REPO_URL --token $TOKEN --labels lab13 --unattended
  ./svc.sh install runner && ./svc.sh start
  EOF
  aws ec2 run-instances \
    --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
    --instance-type t3.small --iam-instance-profile Name=LabInstanceProfile \
    --user-data file://runner.sh \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab13-runner}]' \
    --query 'Instances[0].InstanceId' --output text
  ```

  After ~2 minutes the runner shows up as **Idle** under **Settings →
  Actions → Runners**. In `ci.yml` (`plan`) and `deploy.yml` (`build`,
  `deploy`) change `runs-on` to `[self-hosted, lab13]`, delete the
  `configure-aws-credentials` steps, add `AWS_REGION: us-east-1` to the
  `plan` job's `env`, and delete the three AWS secrets. Terraform, the
  AWS CLI and `amazon-ecr-login` now get short-lived credentials from
  the instance profile, like an OIDC role. Terminate the instance and
  remove the runner when you're done.

---

## Track B — no AWS (act, a local registry, blue-green)

You need Docker with Compose v2 and Git. On Windows, do everything inside
**WSL** (with Docker Desktop's WSL integration on). The pipeline is a
real GitHub Actions workflow, run on your laptop by **`act`**; nginx is
the "ALB", a `registry:2` container is the "ECR", and two app containers,
**blue** and **green**, take turns being live.

### B1. Install act and create the project

- macOS: `brew install act`
- Linux / WSL: `curl --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash -s -- -b /usr/local/bin`

```bash
act --version          # act version 0.2.x
```

You already have `lab13/` with `app/` and `.gitignore` from **The app**
section. In `lab13/`, create the rest.

`compose.yaml` — the "cloud": a registry and the load balancer. The app
containers are started by the deploy script, like an ASG launches
instances.

```yaml
name: lab13

services:
  # The "ECR": a private registry on your laptop
  registry:
    image: registry:2
    ports: ["5001:5000"]

  # The "ALB": nginx in front of the blue and the green container
  lb:
    image: nginx:1.27-alpine
    container_name: lab13-lb
    ports: ["${LB_PORT:-8080}:80"]
    volumes:
      - ./lb.conf:/etc/nginx/conf.d/default.conf:ro
    # Start with "no version live yet": a placeholder that answers 502
    command: ["sh", "-c", "echo 'server 127.0.0.1:9;' > /etc/nginx/active.inc && exec nginx -g 'daemon off;'"]

networks:
  default:
    name: lab13
```

`lb.conf` — the "listener": all traffic goes to whichever container the
one-line file `active.inc` names:

```nginx
# The "ALB listener": all traffic goes to whichever color active.inc names
upstream app {
  include /etc/nginx/active.inc;
}

server {
  listen 80;
  location / {
    proxy_pass http://app;
    add_header X-Upstream $upstream_addr always;   # which container answered
  }
}
```

`deploy.sh` — the blue-green deploy, with a health gate, a smoke test and
an automatic switch-back. **You write the rollback line** (the TODO):

```bash
#!/usr/bin/env bash
# Blue-green deploy of one image tag behind the lab13 nginx "load balancer".
# Usage: ./deploy.sh <image-tag>
set -euo pipefail

TAG=${1:?usage: ./deploy.sh <image-tag>}
IMAGE=localhost:5001/narxoz-web:$TAG
LB_URL=http://localhost:${LB_PORT:-8080}

switch_to() {   # point the load balancer at lab13-<color>
  docker exec lab13-lb sh -c "echo 'server lab13-$1:8000;' > /etc/nginx/active.inc && nginx -t -q && nginx -s reload"
  sleep 2       # a reload is not instant: old nginx workers finish first
}

# 1. Which color is live? The new version goes to the other one.
LIVE=$(docker exec lab13-lb cat /etc/nginx/active.inc | grep -oE 'blue|green' || echo none)
if [ "$LIVE" = blue ]; then IDLE=green; else IDLE=blue; fi
echo "live: $LIVE -> deploying $TAG to $IDLE"

# 2. Start the new version next to the old one. It gets no traffic yet.
docker rm -f "lab13-$IDLE" >/dev/null 2>&1 || true
docker run -d --name "lab13-$IDLE" --network lab13 \
  -e APP_COLOR="$IDLE" -e APP_VERSION="$TAG" "$IMAGE" >/dev/null

# 3. Health gate: wait up to 40 s for Docker's HEALTHCHECK to say "healthy"
state=starting
for _ in $(seq 1 20); do
  state=$(docker inspect -f '{{.State.Health.Status}}' "lab13-$IDLE")
  [ "$state" = starting ] || break
  sleep 2
done
if [ "$state" != healthy ]; then
  echo "BLOCKED: lab13-$IDLE is $state. Traffic stays on $LIVE."
  docker logs --tail 5 "lab13-$IDLE"
  docker rm -f "lab13-$IDLE" >/dev/null
  exit 1
fi

# 4. Switch all traffic to the new color
switch_to "$IDLE"
echo "switched: traffic -> $IDLE"

# 5. Smoke test through the load balancer: 20 requests, all must be
#    200 and come from the new build. One failure = roll back.
fails=0
for _ in $(seq 1 20); do
  answer=$(curl -s -m 2 -w ' HTTP%{http_code}' "$LB_URL/" | tr -d '\n' || true)
  case "$answer" in
    *"build=$TAG"*HTTP200) ;;
    *) fails=$((fails + 1)); echo "  bad answer: ...${answer: -40}" ;;
  esac
done

if [ "$fails" -gt 0 ]; then
  if [ "$LIVE" = none ]; then
    echo "SMOKE TEST FAILED ($fails/20) and there is no previous version to go back to."
    exit 1
  fi
  # TODO: roll back - send all traffic to the color that was live before.
  #       One line. Hint: the function at the top of this file, and $LIVE.

  echo "SMOKE TEST FAILED ($fails/20): ROLLED BACK to $LIVE."
  exit 1
fi
echo "OK: $TAG is live on $IDLE."
[ "$LIVE" = none ] || echo "The old version keeps running on $LIVE for an instant rollback."
```

```bash
chmod +x deploy.sh
```

`.github/workflows/deploy.yml` — the pipeline. `runs-on: self-hosted`
means "a machine I provide": with `act` that's your laptop. **You write
the job order** (the TODOs):

```yaml
name: deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

# Never run two deploys at the same time; queue the second one
concurrency:
  group: deploy-local
  cancel-in-progress: false

jobs:
  test:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v7
      - name: Lint and unit tests (in a throwaway Python container)
        run: >
          docker run --rm -v "$PWD/app:/src" -w /src python:3.12-slim
          sh -c "pip install -q --root-user-action=ignore -r requirements.txt -r requirements-dev.txt
          && ruff check . && python -m pytest -q -p no:cacheprovider"

  build:
    # TODO: build must wait for test (and be skipped if test fails)
    runs-on: self-hosted
    outputs:
      tag: ${{ steps.tag.outputs.tag }}
    steps:
      - uses: actions/checkout@v7
      - name: Image tag = short commit SHA
        id: tag
        run: echo "tag=${GITHUB_SHA:0:7}" >> "$GITHUB_OUTPUT"
      - name: Build and push to the local registry
        env:
          IMAGE: localhost:5001/narxoz-web:${{ steps.tag.outputs.tag }}
        run: |
          docker build -t "$IMAGE" app
          docker push "$IMAGE"

  deploy:
    # TODO: deploy must wait for build (it also needs build's output "tag")
    runs-on: self-hosted
    environment: local
    steps:
      - uses: actions/checkout@v7
      - name: Blue-green deploy with smoke test and automatic rollback
        env:
          TAG: ${{ needs.build.outputs.tag }}
        run: ./deploy.sh "$TAG"
```

### B2. Start the "cloud" and run the pipeline

```bash
docker compose up -d
docker compose ps                                    # registry and lb running
curl -s -o /dev/null -w '%{http_code}\n' localhost:8080/   # 502: nothing deployed yet
curl -s localhost:5001/v2/_catalog                   # {"repositories":[]}
```

(If port 8080 is busy: `LB_PORT=8081 docker compose up -d`, and
`export LB_PORT=8081` in every terminal you use below.)

The image tag is the **commit SHA**, so always **commit before you run
the pipeline**:

```bash
git add -A
git commit -m "Week 13: app, blue-green deploy and pipeline"
act push -P self-hosted=-self-hosted
```

`-P self-hosted=-self-hosted` tells `act` to run jobs labelled
`self-hosted` directly on your machine (no job container), so `docker`,
`localhost:5001` and `localhost:8080` are the ones you just started. If
`act` asks which default image to use, pick **Medium** — it doesn't
matter here.

The end of the output:

```text
[deploy/deploy]   | live: none -> deploying 79e9f5d to blue
[deploy/deploy]   | switched: traffic -> blue
[deploy/deploy]   | OK: 79e9f5d is live on blue.
[deploy/deploy]   ✅  Success - Main Blue-green deploy with smoke test and automatic rollback
[deploy/deploy] 🏁  Job succeeded
```

```bash
curl -s localhost:8080/                              # ... | color=blue | v1 | build=79e9f5d
curl -s localhost:5001/v2/narxoz-web/tags/list       # your tag
git log --oneline -1                                 # the same 7 characters
```

Try the TODO's purpose: put a syntax error in `app/app.py` (delete a
`)`), commit, `act push -P self-hosted=-self-hosted` — `test` fails and
`build` and `deploy` **don't run**. Then `git revert --no-edit HEAD`.

### B3. v2: a blue-green switch with zero errors

In a **second terminal**, watch the site (keep it running until B6):

```bash
while true; do curl -s -m 1 localhost:8080/ || echo "--- no answer"; sleep 0.2; done
```

In `app/app.py`, change `v1` to `v2` in the `return` line, then:

```bash
git commit -am "v2"
act push -P self-hosted=-self-hosted
```

The watch loop jumps from `color=blue | v1` to `color=green | v2` in one
step — no mix, no errors. Blue is **still running** (that's the instant
rollback):

```bash
docker ps --filter name=lab13- --format '{{.Names}}  {{.Image}}  {{.Status}}'
```

### B4. Broken release 1: the health gate

Someone "standardises" the port. In `app/app.py`, change the default in
the last line from `"8000"` to `"8080"`:

```bash
git commit -am "listen on 8080"
act push -P self-hosted=-self-hosted
```

`test` passes (the test client doesn't use a port), `build` passes, and
then:

```text
[deploy/deploy]   | live: green -> deploying 87a4e5e to blue
[deploy/deploy]   | BLOCKED: lab13-blue is unhealthy. Traffic stays on green.
[deploy/deploy]   |  * Running on http://127.0.0.1:8080
[deploy/deploy] 🏁  Job failed
```

The watch loop never noticed: the broken version got **no traffic**.
Undo the change:

```bash
git revert --no-edit HEAD
```

### B5. Broken release 2: the smoke test and the automatic switch-back

A bug the health check can't see: the new "message of the day" is read
from a file that the Dockerfile doesn't copy into the image. Create
`app/motd.txt`:

```text
Exams start on December 15
```

In `app/app.py`, replace the `hello()` function with:

```python
@app.get("/")
def hello():
    with open("motd.txt") as f:  # message of the day
        motd = f.read().strip()
    return f"Hello from {socket.gethostname()} | color={COLOR} | v3 | build={VERSION} | {motd}\n"
```

```bash
git add -A
git commit -m "v3: message of the day"
act push -P self-hosted=-self-hosted
```

The tests pass (they run in `app/`, where `motd.txt` exists), the health
gate passes (`/healthz` works) — and then:

```text
[deploy/deploy]   | switched: traffic -> blue
[deploy/deploy]   |   bad answer: ...an error in the application.</p> HTTP500
...
[deploy/deploy]   | SMOKE TEST FAILED (20/20): ROLLED BACK to green.
[deploy/deploy] 🏁  Job failed
```

The watch loop shows `500` pages for about two seconds, then `v2` again.
(If it keeps showing errors, your TODO in `deploy.sh` is missing.) Find
the cause:

```bash
docker logs lab13-blue 2>&1 | grep -m1 FileNotFoundError
```

Fix it: in `app/Dockerfile`, change `COPY app.py .` to
`COPY app.py motd.txt ./`, then:

```bash
git commit -am "fix: copy motd.txt into the image"
act push -P self-hosted=-self-hosted        # v3 goes live on blue
```

### B6. Instant rollback by hand, and redeploying an old tag

v3 is live on blue; green still runs v2. Roll back in one command, the
blue-green way:

```bash
docker exec lab13-lb sh -c "echo 'server lab13-green:8000;' > /etc/nginx/active.inc && nginx -s reload"
```

The watch loop is back on `v2` instantly. Now the rollback that always
works — redeploy any earlier tag from the registry:

```bash
curl -s localhost:5001/v2/narxoz-web/tags/list
./deploy.sh <one of the older tags>
```

### B7. Cleanup

```bash
docker rm -f lab13-blue lab13-green
docker compose down -v
docker image ls --format '{{.Repository}}:{{.Tag}}' localhost:5001/narxoz-web | xargs docker image rm
docker ps -a --filter name=lab13      # nothing left
```

### Stretch (optional)

- **Scan before push.** In the `build` job, between `docker build` and
  `docker push`, add the same scan as Track A:
  `docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:0.69.3 image --exit-code 1 --severity CRITICAL --ignore-unfixed --no-progress --skip-version-check "$IMAGE"`.
  Run the pipeline: how long does the scan add?
- **The same workflow on real GitHub.** Push `lab13` to a **private**
  repository, add your laptop as a runner (**Settings → Actions →
  Runners → New self-hosted runner**, follow the commands for your OS,
  then `./run.sh`), start Compose, and push: the jobs run on your laptop,
  started by GitHub. Never do this with a public repository. Stop
  `run.sh` and remove the runner when you're done.

---

## Be ready to explain

1. CI, continuous delivery, continuous deployment: which one did you
   build, and what single change would turn it into the other?
2. Why is the image tagged with the commit SHA and not `latest`? How do
   you find out which commit is live right now?
3. What does `needs:` do? What happened to `build` and `deploy` when a
   test failed?
4. Track A: follow a change from `git push` to a new instance: image tag
   → user data → launch template version → instance refresh. Why
   `latest_version` and not `"$Latest"`? Track B: how does `deploy.sh`
   pick the color?
5. Your broken release passed the unit tests and the health check. Why?
   What caught it?
6. Rolling (Track A) vs blue-green (Track B): how long does each
   rollback take, and how much extra capacity does each need?
7. Track A: where do the pipeline's AWS credentials come from, when do
   they expire, and what would OIDC change? Why is the repository public?
   Track B: why did `act` need no credentials?
8. What do `concurrency` (and, in Track A, the S3 lockfile) prevent?
   Track B: what plays the role of the ALB, ECR and the instance refresh?

## Common mistakes

- **`ExpiredToken` / `InvalidClientTokenId` in a GitHub job (Track A).**
  The lab session ended. Copy the new credentials into
  `~/.aws/credentials`, run the A3 secret loop again, then **Re-run
  failed jobs**.
- **`ImageTagAlreadyExistsException` (Track A).** You used **Re-run all
  jobs**: the build tried to push the same SHA tag again into an
  `IMMUTABLE` repository. Use **Re-run failed jobs**, or push a new
  commit.
- **The deploy job never starts (Track A).** It's *Waiting for review*:
  approve it under **Review deployments**. If it runs **without** waiting,
  the repository is private or the environment has no reviewer (A3).
- **`Unrecognized named-value: 'TODO'` (Track A) / `undefined variable
  "TODO"`.** The rollback step's `if:` is still `TODO`. Write the
  condition.
- **`RepositoryNotFoundException` for `lab13-web` (Track A).** The ECR
  repository from A1 doesn't exist, or the `ECR_REPO` variable has a
  different name.
- **`Error acquiring the state lock` (Track A).** A pipeline run (or your
  laptop) holds the lock. Wait for it; never `force-unlock` while a job
  is running.
- **`No value for required variable "image_tag"` (Track A).** On the
  laptop, pass `-var image_tag=...` (any value for `plan`/`destroy`).
- **`verify.sh` times out with `errors=20` (Track A).** The instances
  never got healthy. Look at one instance's boot log:
  `aws ec2 get-console-output --instance-id <id> --latest --output text | tail -30`
  — usually the image pull failed (wrong `ECR_REPO`, or the image isn't
  `amd64`).
- **`act: command not found` / `Cannot connect to the Docker daemon`
  (Track B).** Install `act` (B1); start Docker Desktop / OrbStack.
- **`network lab13 not found` or `No such container: lab13-lb` (Track
  B).** Run `docker compose up -d` first (B2).
- **`manifest unknown` when deploying (Track B).** The image isn't in the
  registry: the `build` job didn't run or the registry was restarted
  (`down -v` empties it). Run the pipeline again.
- **A new commit deploys old code (Track B).** You ran `act` without
  committing: the tag comes from the last commit. Commit, then run.
- **`ROLLED BACK` is printed but the site still shows errors (Track B).**
  The TODO line in `deploy.sh` is missing.
- **`port is already allocated` (Track B).** Something else uses 8080 or
  5001. Use `LB_PORT=8081` (B2) or stop the other program. (macOS uses
  port 5000 for AirPlay — that's why the registry is on 5001.)
