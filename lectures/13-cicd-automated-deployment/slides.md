---
theme: seriph
title: "CSS4005 — Lecture 13: CI/CD and Automated Deployment"
info: |
  CSS4005 — Cloud Infrastructure Construction · Narxoz University
  Lecture 13 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# CSS4005 — Cloud Infrastructure Construction

## Lecture 13: CI/CD and Automated Deployment

<div class="pt-8 opacity-70">
Nauruzbayeva Farikha · Lesson 13
</div>

---
layout: default
---

# Recap — Lesson 12 (CloudWatch)

<v-clicks>

- Why `treat_missing_data = "notBreaching"` on the 5xx alarm? <span v-click class="opacity-60">(no errors = no datapoints; with the default it sits in INSUFFICIENT_DATA)</span>
- Which tells you *that* requests fail, and which tells you *which* one? <span v-click class="opacity-60">(the metric alarm says that; the logs with a `request_id` say which)</span>
- Why alarm on p95/p99 latency, not the Average? <span v-click class="opacity-60">(the average hides the few users who wait 8 seconds)</span>
- Your `cpu-high` alarm vs the `TargetTracking-…` alarms? <span v-click class="opacity-60">(yours tells a human; AWS's scale the fleet — don't edit them)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] CI, continuous delivery and continuous deployment
- [ ] GitHub Actions: workflows, jobs, runners, secrets, environments
- [ ] Images in the pipeline: Amazon ECR and GHCR
- [ ] Deploy strategies on AWS: rolling, blue-green, canary, rollback
- [ ] Terraform in CI, and AWS credentials: OIDC vs access keys
- [ ] Smoke tests, alarm-based rollback, AWS-native tools, cost → today's practice

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Five people now work on the fleet from weeks 11–12. On Monday, Daniyar
runs `terraform apply` from an old branch on his laptop — Aidana's new
alarm silently disappears. On Tuesday, someone builds `:latest` on a Mac,
pushes it, and half the instances crash with `exec format error`.

Which version is running now? Nobody knows.

</div>

<div v-click class="mt-8 text-xl font-bold">
Today: every change goes through Git, and a pipeline tests, builds,
deploys, checks — and rolls back on its own.
</div>

---
layout: section
transition: slide-left
---

# Block 1
## CI, CD and CD

---
---

# Deploying from a laptop

<v-clicks>

- **"Works on my machine":** a different Terraform, Docker or CPU
  architecture on every laptop.
- **No record:** who deployed what, when, from which commit?
- **Two at once:** two people run `apply` in the same minute.
- **Secrets everywhere:** every laptop holds production credentials.
- **Steps skipped in a hurry:** "I'll run the tests after."

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
A <b>pipeline</b> runs the same steps, in the same order, on a clean
machine, every time — and its log is the record of what happened.
</div>

---
---

# CI, continuous delivery, continuous deployment

<div class="text-sm">

| | Continuous Integration | Continuous Delivery | Continuous Deployment |
|---|---|---|---|
| Idea | Merge small changes often; every push is built and tested | Every change that passes is **ready** to release | Every change that passes **is** released |
| Ends with | A tested artifact (an image) | A deploy waiting for one click | Production — no human |
| Gate | Lint, tests, scans | A manual **approval** | Automatic checks only: smoke tests, alarms |

</div>

<v-clicks>

- "CD" means either of the last two. The difference is **one switch**:
  the approval before production.
- Today's practice has an approval (delivery). Remove it, and you have
  continuous deployment — then your automatic checks must be good.

</v-clicks>

---
---

# The pipeline

```text
push / PR ─▶ lint ─▶ test ─▶ build image ─▶ scan ─▶ push to registry  (tag = git SHA)
                                                          │
pull request:  + terraform plan, shown for review         │
                                                          ▼
main:          approval ─▶ deploy (terraform apply) ─▶ smoke test ─▶ watch alarms
                                                                         │ fail
                                                                         ▼
                                                                 automatic rollback
```

<v-clicks>

- **Fail fast:** cheap checks first (lint: seconds), slow ones later
  (deploy: minutes).
- Each stage **gates** the next: a failed test means no image; a failed
  scan means nothing reaches the registry.
- The same file runs for every person and every change.

</v-clicks>

---
---

# Build once, deploy that artifact

<v-clicks>

- Build the image **once**, tag it with the **commit SHA**
  (`narxoz-web:3f9c2a1`), and deploy *that* image to staging, then to
  production.
- Never rebuild per environment: a rebuild can pull a newer base image
  or package — production would run something nobody tested.
- `:latest` is a moving label: you can't tell what's running, and you
  can't roll back to it.
- Configuration comes from the environment (week 10): the same image
  with different `APP_*` variables.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
Ask at any moment: <b>which commit runs in production?</b> With SHA tags
the answer is in the launch template — or in the page itself:
<code>curl $ALB/ → … | build=3f9c2a1</code>.
</div>

---
layout: section
transition: slide-left
---

# Block 2
## GitHub Actions

---
---

# Anatomy of a workflow

```yaml {1|2-6|7-9|10-14}
name: ci                                  # a workflow: one file in .github/workflows/
on:                                       # the events that start it
  push:
    branches: [main]
  pull_request:

jobs:
  test:                                   # a job: runs on one runner
    runs-on: ubuntu-latest                # the runner: a fresh VM from GitHub
    steps:                                # steps run in order, on the same machine
      - uses: actions/checkout@v7         # an action: reusable code from a repo
      - run: pip install -r app/requirements.txt pytest
      - run: python -m pytest -q app      # a shell command
```

<v-clicks>

- A workflow has **jobs**; jobs run **in parallel** unless you say otherwise.
- A step is either `uses:` (an action) or `run:` (shell). A non-zero exit
  code fails the step — and the job.

</v-clicks>

---
---

# Jobs, needs and outputs

```yaml {2-3|4-10|11-15}
jobs:
  test:
    runs-on: ubuntu-latest           # ... steps ...
  build:
    needs: test                      # wait for test; skipped if it failed
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.tag.outputs.tag }}
    steps:
      - id: tag
        run: echo "tag=${GITHUB_SHA:0:7}" >> "$GITHUB_OUTPUT"
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploying ${{ needs.build.outputs.tag }}"
```

<v-clicks>

- Every job gets a **new machine**: files don't carry over. Pass small
  values as **outputs**, files as artifacts, images through a registry.
- `if: failure()` runs a step only when an earlier one failed — that's
  where a rollback goes. `if: always()` for cleanup.

</v-clicks>

---
---

# Runners: GitHub-hosted vs self-hosted

<div class="text-sm">

| | GitHub-hosted | Self-hosted |
|---|---|---|
| What | A fresh VM per job (`ubuntu-latest`, Windows, macOS) | Your laptop or an EC2 instance running the runner agent |
| Cost | Free on public repos; a monthly quota of minutes on private ones (2,000 on the Free plan) | You pay for the machine |
| Clean? | Yes, thrown away after the job | No — leftovers stay between jobs |
| Network | The internet only | Can reach private subnets in your VPC |
| AWS credentials | Secrets or OIDC | The **instance profile** — no keys at all |

</div>

<div v-click class="mt-4 p-4 rounded bg-red-500/10 text-sm">
Whoever can run a workflow runs code on a self-hosted runner: never
attach one to a <b>public</b> repository. Jobs pick it with
<code>runs-on: self-hosted</code>.
</div>

---
---

# Secrets and variables

```yaml
env:
  TF_VAR_ecr_repo: ${{ vars.ECR_REPO }}             # a variable: not secret
steps:
  - run: ./deploy.sh
    env:
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}        # a secret: masked as *** in logs
```

<v-clicks>

- Secrets are encrypted, can't be read back in the UI, and are masked in
  logs. Levels: repository, **environment**, organization.
- They are **not** given to workflows started by pull requests from
  **forks**.
- A secret is still a password: anyone who can edit a workflow on a
  branch can print it (`| base64` defeats the masking). Fewer secrets is
  better — see OIDC in Block 5.
- From the terminal: `gh secret set NAME`, `gh variable set NAME`.

</v-clicks>

---
---

# Environments and approvals

```yaml
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production                       # protection rules apply here
      url: http://lab13-alb-123.us-east-1.elb.amazonaws.com
```

<v-clicks>

- An **environment** (`staging`, `production`) has its own secrets and
  **protection rules**: required reviewers, a wait timer, which branches
  may deploy.
- The job **pauses** until a reviewer clicks **Approve and deploy** —
  that's continuous **delivery**.
- GitHub records every deployment: which commit, when, who approved.

</v-clicks>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
Plan limit: on GitHub Free, Pro and Team, required reviewers work only in
<b>public</b> repositories (private ones need GitHub Enterprise). The
final project repository is public.
</div>

---
---

# Concurrency and permissions

```yaml
permissions:
  contents: read              # what the automatic GITHUB_TOKEN may do
  id-token: write             # only when you use OIDC (Block 5)

concurrency:
  group: deploy-production
  cancel-in-progress: false   # queue the next deploy; never kill a running one
```

<v-clicks>

- Two pushes a minute apart → two deploys at once: two `terraform apply`
  fight over the state lock, or the **older** one finishes last.
- `concurrency` allows one run per group at a time.
- For pull request checks `cancel-in-progress: true` saves minutes; for
  deploys keep it `false` — a half-finished apply is worse than a queue.
- `GITHUB_TOKEN` is created for every run. Give it only what it needs.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Someone forks your public repo and opens a pull request that changes
<code>ci.yml</code> to run
<code v-pre>echo ${{ secrets.AWS_SECRET_ACCESS_KEY }} | base64</code>.
What leaks?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Nothing: workflows started by a fork's pull request get no secrets (and
a first-time contributor's run waits for your approval). That's also why
<code>pull_request_target</code>, which does get secrets, must never run
the PR's code.
</div>

---
layout: section
transition: slide-left
---

# Block 3
## Images: Amazon ECR and GHCR

---
---

# Build and push to ECR in a workflow

```yaml {1-4|5-7|8-12}
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          aws-region: us-east-1
          role-to-assume: arn:aws:iam::123456789012:role/lab13-deploy   # OIDC, Block 5
      - name: Log in to Amazon ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2
      - env:
          IMAGE: ${{ steps.ecr.outputs.registry }}/narxoz-web:${{ github.sha }}
        run: |
          docker build -t "$IMAGE" app
          docker push "$IMAGE"
```

<v-clicks>

- `amazon-ecr-login` does what you typed in week 10:
  `aws ecr get-login-password | docker login …`.
- The runner is **amd64**, like `t3` instances: no `exec format error`.
- `docker/build-push-action` adds layer caching and multi-arch builds.

</v-clicks>

---
---

# Tags you can trust

```hcl
resource "aws_ecr_repository" "app" {
  name                 = "narxoz-web"
  image_tag_mutability = "IMMUTABLE"          # :3f9c2a1 can be pushed only once
  image_scanning_configuration { scan_on_push = true }
}

resource "aws_ecr_lifecycle_policy" "keep_last_20" {
  repository = aws_ecr_repository.app.name
  policy = jsonencode({ rules = [{
    rulePriority = 1, description = "keep the last 20 images"
    selection    = { tagStatus = "any", countType = "imageCountMoreThan", countNumber = 20 }
    action       = { type = "expire" }
  }] })
}
```

<v-clicks>

- **Immutable tags:** nobody can silently replace a released image — and
  `latest` can't be moved, so use only SHA tags.
- **Lifecycle policy:** storage costs $0.10/GB-month. But keep enough old
  images to **roll back** to.

</v-clicks>

---
---

# GHCR: GitHub's registry

```yaml
permissions:
  contents: read
  packages: write                     # lets GITHUB_TOKEN push images
steps:
  - uses: docker/login-action@v4
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}
  - run: |
      docker build -t ghcr.io/aidana/narxoz-web:${GITHUB_SHA:0:7} app
      docker push ghcr.io/aidana/narxoz-web:${GITHUB_SHA:0:7}
```

<v-clicks>

- No cloud credentials needed to push; free for **public** images. Names
  must be lowercase.
- But a private image on EC2 needs a **token on the instance** (kept in
  Secrets Manager). ECR uses the instance role instead — the natural fit
  on AWS.

</v-clicks>

---
---

# Scan before you push

```bash
trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed "$IMAGE"
```

<v-clicks>

- Scan **between build and push**: an image with a known critical hole
  never reaches the registry.
- `--ignore-unfixed`: fail only when a fixed version exists — something
  you can actually do (update the base image or package).
- AWS side: ECR **scan on push** (basic), or Amazon Inspector (enhanced,
  rescans when new CVEs appear).
- Scan the rest too: **gitleaks** for secrets in Git history,
  `trivy config` for your Terraform (week 11).

</v-clicks>

<div v-click class="mt-4 text-sm opacity-70">
The final project requires all of these in CI, with a findings table in
the report.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your instances run <code>narxoz-web:latest</code>. A bad release goes
out and you want to go back. What do you deploy?
</div>

<div v-click class="mt-8 text-lg opacity-70">
You can't name it: <code>latest</code> now points at the bad image, and
the good one is untagged (or deleted by the lifecycle policy). With SHA
tags you redeploy <code>:3f9c2a1</code> — the one that worked.
</div>

---
layout: section
transition: slide-left
---

# Block 4
## Deploy strategies on AWS

---
---

# Four ways to replace a version

<div class="text-sm">

| Strategy | How | Downtime | Extra capacity | Rollback |
|---|---|---|---|---|
| **Recreate** | Stop all old, start all new | Yes | None | Deploy the old one again (slow) |
| **Rolling** | Replace a few instances at a time | No (a bit less capacity) | 0 – a few instances | Roll again with the old version |
| **Blue-green** | A full new fleet; switch all traffic at once | No | 2× during the deploy | Switch back: seconds |
| **Canary** | 5–10% of traffic to the new version, watch, then 100% | No | Small | Weight back to 0 |

</div>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
In rolling and canary deploys, old and new versions serve users <b>at
the same time</b>: the app and the database schema must work with both.
</div>

---
---

# Rolling on AWS: instance refresh

```text
launch template:  v7 (image :3f9c2a1)  ──▶  v8 (image :9be41d0)

ASG (desired 2, max 3)
  t0   [v7] [v7]
  t1   [v7] [v7] [v8 starting]     max healthy 150%: one extra first
  t2   [v7] [v8]                   v8 healthy + warmup → one v7 terminated
  t3   [v7] [v8] [v8 starting]
  t4   [v8] [v8]                   status: Successful
```

<v-clicks>

- **Instance refresh** is the ASG's built-in rolling replacement.
- `min_healthy_percentage` / `max_healthy_percentage`: speed vs capacity.
  100 / 150 = launch first, then terminate — never below desired.
- **Checkpoints** (`checkpoint_percentages = [50]`) pause half-way: a
  canary-like step to watch the alarms.

</v-clicks>

---
---

# Instance refresh in Terraform

```hcl {3-6|8-15}
resource "aws_autoscaling_group" "web" {
  # ... same as week 11 ...
  launch_template {
    id      = aws_launch_template.web.id
    version = aws_launch_template.web.latest_version   # not "$Latest"
  }

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 100
      max_healthy_percentage = 150
      instance_warmup        = 60
    }
  }
}
```

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
With <code>"$Latest"</code> the ASG's setting never changes, so Terraform
sees nothing to do: running instances keep the old image until they die.
A real version number makes the change visible — and starts the refresh.
</div>

---
---

# Blue-green with two target groups

```text
                                 ┌──▶ TG blue  ──▶ ASG blue   (v7)   live: 100%
 users ──▶ ALB ── listener :80 ──┤
                                 └──▶ TG green ──▶ ASG green  (v8)   new: 0%

 test:      a second listener (:8080) or a header rule sends only YOU to green
 switch:    the :80 listener forwards to green — seconds, for everyone
 rollback:  forward to blue again (the old fleet is still running)
```

<v-clicks>

- Two ASGs, two target groups — one listener decides who gets traffic.
- Users never see a mix of versions, and you test green on real
  infrastructure first.
- Keep blue for a while (an hour, a day), then scale it to 0.
- Cost: **twice the fleet** during the deploy.

</v-clicks>

---
---

# Weighted target groups: canary and blue-green

```hcl {6-15}
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.web.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type = "forward"
    forward {
      target_group {
        arn    = aws_lb_target_group.blue.arn
        weight = var.blue_weight     # 90
      }
      target_group {
        arn    = aws_lb_target_group.green.arn
        weight = var.green_weight    # 10 = canary
      }
    }
  }
}
```

<v-clicks>

- **Canary:** 90/10 → watch the alarms for 10 minutes → 50/50 → 0/100.
- **Blue-green:** jump 100/0 → 0/100. **Rollback:** weights back to 100/0.

</v-clicks>

---
---

# Rollback: what it really means

<div class="text-sm">

| Strategy | Rollback | How fast |
|---|---|---|
| Rolling (instance refresh) | Apply the previous image tag → another refresh; or the refresh's own `auto_rollback` | Minutes |
| Blue-green | Point the listener back at blue | Seconds |
| Canary | Set the new weight to 0 | Seconds |
| ECS / Kubernetes | Previous task definition / `kubectl rollout undo` | About a minute |

</div>

<v-clicks>

- You roll back **code, not data**. A migration that dropped a column
  can't be undone by switching traffic: change schemas in steps (add the
  new column → deploy → remove the old one later).
- After an automatic rollback, also **`git revert`** — or the next push
  deploys the bad commit again.

</v-clicks>

---
---

# Replace instances, or update them in place?

<div class="text-sm">

| | SSH from CI | SSM Run Command |
|---|---|---|
| Network | Port 22 open to the runner; public IPs or a bastion | **No inbound ports**: the SSM agent connects out |
| Credentials | A private key in GitHub Secrets | IAM permission `ssm:SendCommand` |
| Targets | One host at a time, by IP | By **tag**, many at once, with rate limits |
| Audit | Your terminal history | CloudTrail + saved command output |
| Instance needs | sshd + your key | SSM agent (built into AL2023) + `AmazonSSMManagedInstanceCore` |

</div>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
Prefer <b>replacing</b> instances (instance refresh, blue-green): every
server is built the same way. In-place updates are faster, but servers
slowly drift apart. If you update in place, use <b>SSM, not SSH</b>.
</div>

---
---

# SSM Run Command: a deploy without SSH

```bash {1|2-6|7-8}
IMG=123456789012.dkr.ecr.us-east-1.amazonaws.com/narxoz-web:9be41d0
CID=$(aws ssm send-command --document-name AWS-RunShellScript \
  --targets Key=tag:Name,Values=lab13-web \
  --max-concurrency 50% --max-errors 0 --comment "deploy 9be41d0" \
  --parameters "commands=docker pull $IMG,docker rm -f web,docker run -d --name web -p 80:8000 $IMG" \
  --query Command.CommandId --output text)
aws ssm list-command-invocations --command-id "$CID" --details \
  --query 'CommandInvocations[].[InstanceId,Status]' --output table
```

<v-clicks>

- `--targets` by tag: every instance of the fleet, no IP addresses.
- `--max-concurrency 50%`: half the fleet at a time (a rolling update);
  `--max-errors 0`: stop at the first failure.
- New instances from the ASG still start the **old** image — update the
  launch template too. That's why replacing is simpler.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Blue-green needs twice the instances during a deploy; a rolling deploy
needs only one extra. Why would you still choose blue-green?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Rollback in seconds (the old fleet is still running), no mix of versions
for users, and you can test the new fleet on real infrastructure before
anyone sees it.
</div>

---
layout: section
transition: slide-left
---

# Block 5
## Terraform in CI and AWS credentials

---
---

# Terraform in CI: plan on PR, apply on main

```text
feature branch ──PR──▶  ci.yml:  fmt -check · validate · scan · plan  ──▶  plan shown in the PR
                                                                                │ review, merge
main ─────────────────▶ deploy.yml:  approval ──▶ apply  (the code that was reviewed)
```

<v-clicks>

- Reviewers read the **plan**, not only the code:
  `Plan: 0 to add, 1 to change, 0 to destroy`.
- Nobody runs `apply` from a laptop; the pipeline is the only writer.
- `-input=false -auto-approve` only in CI, **after** a human approved the
  environment.
- The plan may be stale by merge time, so `apply` plans again. Stricter:
  save it (`plan -out`) and apply exactly that file.

</v-clicks>

---
---

# Remote state, shared by laptop and pipeline

```hcl
terraform {
  backend "s3" {
    key          = "lab13/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true    # the lock: a .tflock object next to the state
  }
}
```

```bash
terraform init -backend-config="bucket=$TF_STATE_BUCKET"   # bucket name from a CI variable
```

<v-clicks>

- The state bucket: **versioning** on (undo a broken state), public
  access blocked, encrypted. Created **once**, by hand or a small
  bootstrap stack — it can't hold its own state.
- A second `apply` waits (`-lock-timeout=5m`) or fails with
  `Error acquiring the state lock` — never two at once.

</v-clicks>

---
---

# The plan job

```yaml
  plan:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: infra } }
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
        with: { terraform_version: "1.17.2", terraform_wrapper: false }
      - uses: aws-actions/configure-aws-credentials@v6
        with: { aws-region: us-east-1, role-to-assume: "${{ vars.PLAN_ROLE_ARN }}" }
      - run: terraform fmt -check -recursive
      - run: terraform init -input=false -backend-config="bucket=${{ vars.TF_STATE_BUCKET }}"
      - run: terraform validate
      - run: terraform plan -input=false -no-color | tee plan.txt
      - run: grep -E '^Plan:|^No changes' plan.txt >> "$GITHUB_STEP_SUMMARY"
```

<div v-click class="mt-2 text-sm opacity-70">
The plan role can be <b>read-only</b>: a pull request can look at
everything and change nothing.
</div>

---
---

# The deploy job

```yaml {2|5-6|7|8|9-11}
  deploy:
    environment: production              # ⏸ waits for approval
    runs-on: ubuntu-latest
    steps:                               # checkout, setup-terraform, credentials, init ...
      - id: prev
        run: echo "tag=$(terraform output -raw image_tag)" >> "$GITHUB_OUTPUT"
      - run: terraform apply -input=false -auto-approve -var "image_tag=$TAG"
      - run: ../scripts/verify.sh ...    # wait, smoke test, watch the 5xx alarm
      - if: failure()
        env: { PREV: "${{ steps.prev.outputs.tag }}" }
        run: terraform apply -input=false -auto-approve -var "image_tag=$PREV"
```

<v-clicks>

- The deploy **is** a Terraform change: new `image_tag` → new launch
  template version → instance refresh.
- The rollback is the same `apply` with the **previous** tag — read from
  the state before the deploy changed it.

</v-clicks>

---
---

# AWS credentials in CI: two ways

<div class="text-sm">

| | Access keys in GitHub Secrets | OIDC + an IAM role |
|---|---|---|
| GitHub stores | `AWS_ACCESS_KEY_ID` + secret key | Nothing secret — a role ARN |
| Lifetime | Until someone rotates them (months, years) | About **1 hour**, one per job |
| If they leak | Work from anywhere until noticed | Expire soon; only GitHub can get new ones |
| Who may use them | Anyone holding them | The trust policy: **this repo, this branch or environment** |
| Rotation | Manual | Not needed |

</div>

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
Leaked long-lived access keys are one of the most common ways cloud
accounts get breached through pipelines. In a real account: <b>OIDC</b>,
and no IAM users for CI at all.
</div>

---
---

# How OIDC works

```text
 job ──(1) "give me an ID token" ────────────────────────▶ GitHub's OIDC provider
     ◀── signed token: repo:aidana/lab13, ref main, environment production

 job ──(2) AssumeRoleWithWebIdentity(token, role ARN) ───▶ AWS STS
           STS checks:  signed by token.actions.githubusercontent.com?
                        aud = sts.amazonaws.com?   sub allowed by the role's trust policy?
     ◀── temporary credentials, ~1 hour

 job ──(3) docker push / terraform apply with them ──────▶ AWS
```

<v-clicks>

- The workflow needs `permissions: id-token: write`.
- One-time setup in the account: an **OIDC identity provider** for
  `token.actions.githubusercontent.com` and a **role** that trusts it.
- `configure-aws-credentials` does steps 1–2 for you.

</v-clicks>

---
---

# OIDC: the trust policy

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": { "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "repo:aidana/lab13:environment:production"
  } }
}
```

<div class="grid grid-cols-2 gap-6 mt-2">
<div>

```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v6
    with:
      aws-region: us-east-1
      role-to-assume: ${{ vars.DEPLOY_ROLE_ARN }}
```

</div>
<div>

<div v-click class="p-4 rounded bg-red-500/10 text-sm">
The <code>sub</code> condition is the security: only <b>this repository</b>,
and only jobs in the <b>production</b> environment (after approval). Without
it, <b>any</b> repository on GitHub could assume your role.
</div>

</div>
</div>

---
---

# AWS Academy Learner Lab: what we do instead

<v-clicks>

- You **can't** create IAM users, roles or OIDC providers: no access keys
  of your own, no OIDC.
- What you have: the **session credentials** from AWS Details (key ID,
  secret key, **session token**) — they **expire with the session
  (~4 h)** — plus `LabRole` / `LabInstanceProfile`. Region `us-east-1`.
- **Option 1 (today):** paste the session credentials into GitHub
  Secrets at the start of each session. `ExpiredToken` in a job → paste
  them again.
- **Option 2:** a **self-hosted runner** on an EC2 instance with
  `LabInstanceProfile`: no keys in GitHub at all — the closest thing to
  OIDC. Private repositories only.
- Final project: write down which one you used, why, and how you would
  do it with OIDC in a real account.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your deploy role's trust policy uses <code>StringLike</code> with
<code>sub = repo:aidana/lab13:*</code>. What can a workflow on a pull
request branch do?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Everything the deploy role can: its token's <code>sub</code> is
<code>repo:aidana/lab13:pull_request</code>, which matches <code>*</code>.
Pin the deploy role to <code>environment:production</code> (approval
needed) and give pull requests a read-only plan role.
</div>

---
layout: section
transition: slide-left
---

# Block 6
## Verify, roll back, AWS-native tools, cost

---
---

# Smoke tests

<v-clicks>

- A **smoke test** = a few requests right after the deploy, **through the
  real load balancer**: does it answer `200`, is it the **new** build, does
  the main page work?
- Not the full test suite: seconds, not minutes. Unit tests already ran
  before the build.

</v-clicks>

```bash
for i in $(seq 1 20); do
  curl -s -m 3 "$ALB/" | grep -q "build=$TAG" || exit 1    # one bad answer = fail
done
```

<v-clicks>

- A health check is **not** a smoke test: `/healthz` says "the process
  runs"; the smoke test says "users get the right page". Today's practice
  ships a bug that passes the first and fails the second.

</v-clicks>

---
---

# Alarm-based automatic rollback

```text
deploy ──▶ for N minutes: send traffic, watch the week 12 alarms (5xx, p95 latency)
             all OK  ──▶ done
             ALARM   ──▶ roll back automatically + notify (SNS)
```

<div class="text-sm">

| Who rolls back | How |
|---|---|
| **Your pipeline** (today) | Poll `describe-alarms` during the rollout; on `ALARM` → apply the previous tag |
| **ASG instance refresh** | `alarm_specification` + `auto_rollback = true`: the refresh fails and reverts itself |
| **CodeDeploy** | Alarms on the deployment group + automatic rollback when one fires |

</div>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
Pick alarms that a <b>healthy</b> deploy doesn't trip: a new instance
fails health checks while it boots, so <code>UnHealthyHostCount</code> above
0 for a minute is normal during a rollout. The 5xx count is a better signal.
</div>

---
---

# The AWS-native pipeline

<div class="text-sm">

| Service | Job | In GitHub Actions terms |
|---|---|---|
| **CodePipeline** | Orders the stages: source → build → approval → deploy | The workflow |
| **CodeBuild** | Runs your commands in a container (`buildspec.yml`), billed per minute | A job on a runner |
| **CodeDeploy** | Deploys to EC2/ASG, ECS or Lambda: in place or blue-green, with alarms and rollback | Your deploy script |
| **CodeConnections** | Links the pipeline to a GitHub repository | `on: push` |

</div>

<v-clicks>

- Everything is IAM: **service roles**, no keys leave AWS, builds can run
  inside your VPC.
- Choose it when a company standardises on AWS or needs private-network
  builds. GitHub Actions is the more common choice, and what the final
  project uses.
- Learner Lab: check the Readme; where these services work, they use
  `LabRole` as the service role.

</v-clicks>

---
---

# CodeDeploy in one slide

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

`appspec.yml` (EC2 / ASG):

```yaml
version: 0.0
os: linux
hooks:
  ApplicationStop:
    - location: scripts/stop.sh
  ApplicationStart:
    - location: scripts/start.sh
  ValidateService:
    - location: scripts/smoke.sh
```

</div>
<div>

<v-clicks>

- An **agent** on each instance runs the hooks in order. A failing
  `ValidateService` (your smoke test) fails the deployment.
- **Deployment configs:** `CodeDeployDefault.OneAtATime`, `HalfAtATime`,
  `AllAtOnce` (EC2); `…Canary10Percent5Minutes`,
  `…Linear10PercentEvery1Minute` (ECS, Lambda).
- **Blue-green** on EC2: copies the ASG, shifts the ALB, then removes
  the old instances.

</v-clicks>

</div>
</div>

---
---

# What CI/CD costs

<div class="text-sm">

| Item | Price (us-east-1) | Free |
|---|---|---|
| GitHub Actions, GitHub-hosted runners | Per minute beyond the quota (private repos) | Public repos; 2,000 min/month private (Free plan) |
| Amazon ECR storage | $0.10 / GB-month | Pulls by EC2 in the same region: no transfer cost |
| CodeBuild (small Linux) | ≈ $0.005 / build minute | 100 build minutes / month |
| CodePipeline (V2) | $0.002 / action-execution minute | 100 minutes / month |
| CodeDeploy to EC2 / ASG | Free | — |
| Blue-green | A **second fleet** for the length of the deploy | — |

</div>

<div v-click class="mt-4 text-sm opacity-70">
Prices change — check the pricing pages. The pipeline itself is cheap;
what it <b>creates</b> (ALB, instances, NAT) is the bill.
</div>

---
---

# Cleanup: pipelines bring things back

<v-clicks>

- `terraform destroy` against the **same remote state** (`init
  -backend-config=…`), then delete the ECR repository
  (`delete-repository --force`) and empty and delete the state bucket.
- Delete the lab's GitHub secrets, and **disable or delete** the deploy
  workflow of an old lab.
- Self-hosted runner: terminate the instance and remove the runner from
  the repository.
- Then **End Lab**.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
A repository with a working deploy workflow recreates the whole stack on
the next push — even months later, even by accident.
</div>

---
---

# Your final project

<div class="text-sm">

| Requirement (§5.9) | What it means |
|---|---|
| lint → test → build → **scan** → push, on every push/PR | Immutable tag = git SHA; Trivy fails on CRITICAL |
| Deploy on merge to `main` | `kubectl set image` / `helm upgrade`, then `kubectl rollout status`; the job fails if readiness fails |
| **Rollback**, shown once | `kubectl rollout undo` or redeploy the previous tag |
| CI credentials | OIDC if your account allows; Learner Lab: session creds in Secrets or a self-hosted runner with `LabInstanceProfile` — document why |

</div>

<v-clicks>

- Also in CI: gitleaks, image and IaC scans, the password-policy test
  (§5.10), and `terraform fmt`/`validate` (§5.11). That's milestone **M5**.
- Your app runs on **Kubernetes**, not on an ASG: the pipeline has the
  same shape — only the deploy step changes.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your pipeline deploys every merge, watches the 5xx alarm and rolls back
automatically. What is the one thing it still can't undo?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Data. A migration that dropped or rewrote data stays done after the code
rolls back. Make schema changes backward compatible, and back up before
risky ones.
</div>

---
layout: section
transition: slide-left
---

# Block 7
## Today's practice

---
---

# Today's practice — push to deploy

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Track A — Learner Lab**

1. Bootstrap: S3 state bucket + ECR repository
2. Week 11 fleet runs the **image**; instance refresh
3. GitHub: lab credentials as secrets, `production` environment with you as reviewer
4. Pull request → tests + `terraform plan`
5. Merge → build, scan, push, approve, **rolling deploy**, smoke test
6. A broken release → 5xx alarm → **automatic rollback** (you write the condition)
7. Destroy everything

</div>
<div>

**Track B — no AWS**

The same pipeline with **`act`** on your laptop, a local registry, and
nginx switching between **blue** and **green**: a health gate, a smoke
test and an automatic switch-back.

<div v-click class="mt-4 opacity-70">
Report with screenshots on Canvas: the pipeline run, the plan or deploy log, a stopped bad
release, the rollback, and the cleanup.
</div>

</div>
</div>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Explain CI vs continuous delivery vs continuous deployment
- [ ] Read and write a GitHub Actions workflow: jobs, `needs`, outputs, secrets, environments, concurrency
- [ ] Build, scan and push an image with an immutable SHA tag to ECR (or GHCR)
- [ ] Choose between rolling, blue-green and canary, and say how each rolls back
- [ ] Run Terraform from CI: plan on PR, apply on main, remote state with a lock
- [ ] Explain why OIDC beats access keys — and what Learner Lab allows instead
- [ ] Add a smoke test and an alarm-based automatic rollback to a deploy

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Finish today's practice: **`terraform destroy`**, delete the ECR
      repository and the state bucket, then **End Lab** (Track A) — or
      the Track B cleanup
- [ ] In your final project repository: extend the template's CI workflow
      to lint → test → build → scan, and decide how CI gets cloud
      credentials
- [ ] Think about this: if one availability zone of your platform goes
      down right now, what keeps working — and how would you prove it?

<div class="mt-8 text-sm opacity-60">
Next week: surviving failures, a security review and the bill.
</div>

---
layout: end
---

# Next lecture

High Availability, Security and Cost Optimization — zone failures,
hardening the platform, and what it all costs.
