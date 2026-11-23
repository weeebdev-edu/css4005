# Lab Work 13 — CI/CD and Automated Deployment

**Submission: a report with screenshots is required.** Upload one PDF to
Canvas with your repository link and, for **every** task you did,
screenshots of the commands, their output and the result, each with a
one-line caption. A lab without a screenshot report is not accepted.

In the practice you built a pipeline step by step. In this lab you are the
engineer who is called **after** something went wrong: a release that broke
the site at night, a pipeline that leaks keys, a release that only one
person knows how to do, a rollback nobody has ever tried. Each task is an
incident. You find the cause, change the pipeline so it can't happen again,
and prove it with a pipeline run.

You can choose 1 or 2 variant:

- **AWS Implementation** — AWS Academy Learner Lab + GitHub Actions: ECR,
  one EC2 instance, deploys through **SSM Run Command** (no SSH).
- **Local Alternative (no AWS)** — the same pipeline run on your laptop by
  [`act`](https://github.com/nektos/act), a local registry and the same
  nginx + blue/green containers.

Both variants use the **same** app, the same `deploy.sh` and the same
workflow ideas. Only the registry and the place where `deploy.sh` runs
differ.

```
   git push ──► test ──► build ──► push ──► deploy ──► site
                                    │          │
                AWS:               ECR     SSM Run Command → EC2 → deploy.sh
                Local:      localhost:5001   act on your laptop → deploy.sh
```

> **AWS Academy Learner Lab limits**
>
> - You **can't create IAM users, roles or OIDC providers**. The pipeline
>   uses the lab's **session credentials** stored in GitHub Secrets; the
>   instance uses the existing **`LabInstanceProfile`**.
> - Session credentials **expire with the session (about 4 hours)**. A job
>   that fails with `ExpiredToken` needs fresh credentials in the secrets,
>   then **Re-run failed jobs**.
> - Region **`us-east-1`**. The repository must be **public** (required
>   reviewers on environments need it on GitHub Free), so its logs are public
>   too: never print a secret.
> - If a call fails with `AccessDenied`, the lab doesn't allow it: write the
>   action and the error in your report and do that task on the Local
>   variant.
> - When you are done: **Cleanup**, then **End Lab**.

---

## Starter repository (both variants)

The Narxoz Shop: a small Flask app behind nginx. Two app containers,
**blue** and **green**; nginx sends all traffic to one of them.

```
                         site :80 (AWS) / :8080 (laptop)
                                    │
                                    v
                             lab13-lb (nginx)
                     active.inc: set $backend lab13-blue;
                          /                    \
                         v                      v
                   lab13-blue              lab13-green
                   build=4d1e7a2           build=9a0c3f1     (idle, kept for rollback)

                    network: lab13       deploy log: /var/tmp/lab13-deploys.log
```

Create a **public** repository `lab13-shop` on GitHub and clone it:

```bash
git clone git@github.com:<you>/lab13-shop.git && cd lab13-shop
mkdir -p app deploy .github/workflows
```

`app/app.py`

```python
import os
import socket

from flask import Flask, jsonify

app = Flask(__name__)
COLOR = os.environ.get("APP_COLOR", "blue")
VERSION = os.environ.get("APP_VERSION", "dev")  # the image tag, set at deploy time


@app.get("/")
def index():
    return f"Narxoz Shop | {socket.gethostname()} | color={COLOR} | build={VERSION}\n"


@app.get("/healthz")
def healthz():
    return "ok\n"


@app.get("/api/status")
def status():
    # The mobile app reads this. Its fields are a contract: don't rename them.
    return jsonify(status="ok", build=VERSION)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", "8000")))
```

`app/test_app.py`

```python
from app import app


def test_healthz():
    assert app.test_client().get("/healthz").status_code == 200


def test_status_contract():
    data = app.test_client().get("/api/status").get_json()
    assert data["status"] == "ok"
    assert "build" in data
```

`app/requirements.txt`

```text
flask==3.1.3
```

`app/requirements-dev.txt` (only for tests in the pipeline, not in the image)

```text
pytest==9.2.0
ruff==0.17.6
requests-mock==1.12.1
```

`app/Dockerfile`

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

`deploy/bootstrap.sh` — prepares a machine once: the network and the load
balancer (and, on the laptop, the registry)

```bash
#!/usr/bin/env bash
# The "server": a Docker network and an nginx load balancer in front of
# the blue / green app containers. Run once per machine (laptop or EC2).
# Usage: LB_PORT=8080 LOCAL_REGISTRY=1 ./deploy/bootstrap.sh
set -euo pipefail
LB_PORT=${LB_PORT:-8080}

docker network inspect lab13 >/dev/null 2>&1 || docker network create lab13 >/dev/null

docker rm -f lab13-lb >/dev/null 2>&1 || true
docker run -d --name lab13-lb --restart unless-stopped --network lab13 \
  -p "$LB_PORT:80" nginx:1.27-alpine >/dev/null
sleep 2

# Which color is live = one line in /etc/nginx/active.inc. Nothing yet.
docker exec lab13-lb sh -c 'echo "set \$backend lab13-none;" > /etc/nginx/active.inc'
docker exec -i lab13-lb sh -c 'cat > /etc/nginx/conf.d/default.conf' <<'NGINX'
resolver 127.0.0.11 valid=5s;            # Docker's DNS: container names -> IPs
server {
  listen 80;
  location / {
    include /etc/nginx/active.inc;       # set $backend lab13-blue;
    proxy_pass http://$backend:8000;
    add_header X-Backend $backend always;
  }
}
NGINX
docker exec lab13-lb sh -c 'nginx -t -q && nginx -s reload'

# Laptop only: a private registry on localhost:5001 (the "ECR")
if [ "${LOCAL_REGISTRY:-0}" = 1 ] && ! docker inspect lab13-registry >/dev/null 2>&1; then
  docker run -d --name lab13-registry --restart unless-stopped \
    -p 127.0.0.1:5001:5000 registry:2 >/dev/null
fi
echo "bootstrap done: load balancer on port $LB_PORT"
```

`deploy/deploy.sh` — the deploy that was used at night. Read it carefully:
it is part of Task 1.

```bash
#!/usr/bin/env bash
# Blue-green deploy of one image tag behind lab13-lb.
# Usage: REGISTRY=<registry> ./deploy/deploy.sh <image-tag>
#   REGISTRY defaults to localhost:5001 (laptop); on EC2 it is your ECR registry.
set -euo pipefail
TAG=${1:?usage: deploy.sh <image-tag>}
REGISTRY=${REGISTRY:-localhost:5001}
IMAGE=$REGISTRY/lab13-shop:$TAG
LOG=/var/tmp/lab13-deploys.log

if [[ $REGISTRY == *.amazonaws.com ]]; then   # EC2: the instance profile allows the pull
  aws ecr get-login-password --region "${AWS_REGION:-us-east-1}" \
    | docker login --username AWS --password-stdin "$REGISTRY" >/dev/null
fi

# 1. Which color is live? The new version goes to the other one.
LIVE=$(docker exec lab13-lb cat /etc/nginx/active.inc | grep -oE 'blue|green' || echo none)
if [ "$LIVE" = blue ]; then IDLE=green; else IDLE=blue; fi
echo "live: $LIVE -> deploying $TAG to $IDLE"

# 2. Start the new version next to the old one
docker pull -q "$IMAGE" >/dev/null
docker rm -f "lab13-$IDLE" >/dev/null 2>&1 || true
docker run -d --name "lab13-$IDLE" --network lab13 --restart unless-stopped \
  -e APP_COLOR="$IDLE" -e APP_VERSION="$TAG" "$IMAGE" >/dev/null

# 3. Give it a moment, then switch all traffic to it
sleep 3   # "should be enough"
docker exec lab13-lb sh -c "echo 'set \$backend lab13-$IDLE;' > /etc/nginx/active.inc && nginx -s reload"

echo "$(date -u +%FT%TZ) $TAG $IDLE" >> "$LOG"
echo "done: $TAG is live on $IDLE"
```

`.gitignore`

```gitignore
__pycache__/
.pytest_cache/
*.log
```

```bash
chmod +x deploy/*.sh
git add -A && git commit -m "Narxoz Shop starter" && git push
```

### Setup — Local Alternative

Docker, Git, `curl`, `jq`, and `act`
(`brew install act`, or the install script on the act README for Linux /
WSL). Windows: everything in **WSL2**.

```bash
LOCAL_REGISTRY=1 ./deploy/bootstrap.sh
curl -s -o /dev/null -w '%{http_code}\n' localhost:8080/     # 502: nothing deployed yet
curl -s localhost:5001/v2/_catalog                           # {"repositories":[]}
```

`.github/workflows/deploy.yml` — the pipeline as it was on the night of
the incident:

```yaml
name: deploy

# Every push to main: test -> build -> push -> deploy. Nobody watches it.
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: deploy-production
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
    needs: test
    runs-on: self-hosted
    outputs:
      tag: ${{ steps.tag.outputs.tag }}
    steps:
      - uses: actions/checkout@v7
      - name: Image tag = short commit SHA
        id: tag
        run: echo "tag=${GITHUB_SHA:0:7}" >> "$GITHUB_OUTPUT"
      - name: Build and push
        env:
          IMAGE: localhost:5001/lab13-shop:${{ steps.tag.outputs.tag }}
        run: |
          docker build -t "$IMAGE" app
          docker push "$IMAGE"

  deploy:
    needs: build
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v7
      - name: Blue-green deploy
        env:
          TAG: ${{ needs.build.outputs.tag }}
        run: ./deploy/deploy.sh "$TAG"
```

Run the pipeline (always **commit first**: the tag is the commit SHA):

```bash
git add -A && git commit -m "pipeline" && git push
act push -P self-hosted=-self-hosted
curl -s localhost:8080/          # Narxoz Shop | ... | color=blue | build=<sha>
```

### Setup — AWS Implementation

AWS CLI v2 with the Learner Lab credentials in `~/.aws/credentials`, and
GitHub CLI `gh` logged in.

```bash
aws sts get-caller-identity                   # assumed-role/voclabs/...
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)

# the registry
aws ecr create-repository --repository-name lab13-shop \
  --image-tag-mutability IMMUTABLE --image-scanning-configuration scanOnPush=true

# a security group: HTTP from anywhere (the GitHub runner smoke-tests the site)
SG=$(aws ec2 create-security-group --group-name lab13-shop-web \
  --description "lab13 shop HTTP" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id "$SG" --protocol tcp --port 80 --cidr 0.0.0.0/0

# one instance: Docker + the bootstrap from YOUR repository (pushed above)
cat > user-data.sh <<EOF
#!/bin/bash
dnf install -y docker
systemctl enable --now docker
curl -fsSL https://raw.githubusercontent.com/$REPO/main/deploy/bootstrap.sh | LB_PORT=80 bash
EOF
INSTANCE=$(aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t3.micro --iam-instance-profile Name=LabInstanceProfile \
  --security-group-ids "$SG" --user-data file://user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab13-shop}]' \
  --query 'Instances[0].InstanceId' --output text)
rm user-data.sh
aws ec2 wait instance-status-ok --instance-ids "$INSTANCE"
aws ssm describe-instance-information --filters "Key=InstanceIds,Values=$INSTANCE" \
  --query 'InstanceInformationList[0].PingStatus' --output text      # Online
IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
curl -s -o /dev/null -w '%{http_code}\n' "http://$IP/"              # 502: nothing deployed yet
```

(No `Online` after 5 minutes: SSM isn't allowed for your lab — do the
Local Alternative.)

Secrets and variables for the pipeline (run the secret loop again every time
the lab session restarts):

```bash
for k in aws_access_key_id aws_secret_access_key aws_session_token; do
  gh secret set "$(echo "$k" | tr a-z A-Z)" --body "$(aws configure get "$k")"
done
gh variable set INSTANCE_ID --body "$INSTANCE"
gh variable set SITE_URL --body "http://$IP"
```

`.github/workflows/deploy.yml` — the pipeline as it was on the night of the
incident:

```yaml
name: deploy

# Every push to main: test -> build -> push to ECR -> deploy on EC2 via SSM.
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: deploy-production
  cancel-in-progress: false

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
      - name: Build and push
        env:
          IMAGE: ${{ steps.ecr.outputs.registry }}/lab13-shop:${{ steps.tag.outputs.tag }}
        run: |
          docker build -t "$IMAGE" app
          docker push "$IMAGE"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    env:
      TAG: ${{ needs.build.outputs.tag }}
      INSTANCE_ID: ${{ vars.INSTANCE_ID }}
    steps:
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          aws-region: ${{ env.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
      - name: Blue-green deploy on the instance (SSM Run Command, no SSH)
        run: |
          REGISTRY="$(aws sts get-caller-identity --query Account --output text).dkr.ecr.$AWS_REGION.amazonaws.com"
          SCRIPT="https://raw.githubusercontent.com/$GITHUB_REPOSITORY/$GITHUB_SHA/deploy/deploy.sh"
          CMD="curl -fsSL $SCRIPT -o /tmp/deploy.sh && REGISTRY=$REGISTRY bash /tmp/deploy.sh $TAG"
          ID=$(aws ssm send-command --instance-ids "$INSTANCE_ID" \
            --document-name AWS-RunShellScript --comment "deploy $TAG" \
            --parameters "$(jq -nc --arg c "$CMD" '{commands: [$c]}')" \
            --query Command.CommandId --output text)
          aws ssm wait command-executed --command-id "$ID" --instance-id "$INSTANCE_ID" || true
          aws ssm get-command-invocation --command-id "$ID" --instance-id "$INSTANCE_ID" \
            --query '[Status, StandardOutputContent, StandardErrorContent]' --output text | tee result.txt
          head -1 result.txt | grep -q '^Success'
```

```bash
git add -A && git commit -m "pipeline" && git push
gh run watch
curl -s "http://$IP/"            # Narxoz Shop | ... | color=blue | build=<sha>
```

---

## Task 1 — The Night Release

Thursday, 23:40. A developer merged "Show prices in tenge". The pipeline
was green. At 23:43 the site started answering **502**, and stayed down
until the morning. Nobody was paged; customers wrote on Instagram.

```
  23:40 merge ──► test ✓ ──► build ✓ ──► push ✓ ──► deploy ✓  (green!)
                                                       │
                                                       v
                                         nginx → lab13-green  ✗ crashing
                                         lab13-blue (old, healthy) gets nothing
                                                       │
                                         07:30  "the site is down"
```

The release (apply it exactly like this):

In `app/app.py`, add `import requests` under `import socket`, and add this
endpoint above `if __name__ == "__main__":`

```python
@app.get("/api/rate")
def rate():
    # USD -> KZT for the price tags; 503 if the rates API is down
    try:
        r = requests.get("https://open.er-api.com/v6/latest/USD", timeout=2)
        return jsonify(kzt=r.json()["rates"]["KZT"])
    except Exception:
        return jsonify(kzt=None), 503
```

Add to `app/test_app.py`:

```python
def test_rate(requests_mock):
    requests_mock.get("https://open.er-api.com/v6/latest/USD", json={"rates": {"KZT": 512.3}})
    assert app.test_client().get("/api/rate").get_json() == {"kzt": 512.3}
```

```bash
git add -A && git commit -m "Show prices in tenge" && git push
# Local: act push -P self-hosted=-self-hosted
```

Requirements

- Reproduce the outage: the pipeline is **green**, the site answers `502`.
- Find the root cause with evidence (container state, its logs, which
  file is missing what). Explain why the **tests passed** although the
  image is broken.
- Restore the site **by hand** first (the old color still runs).
- Add a gate to the pipeline that would have caught it **before** the
  image reached the registry: a job or step that **runs the built image**,
  waits until Docker says `healthy`, and checks the contract
  (`/healthz` is 200, `/api/status` has `"status": "ok"`). If it fails,
  nothing is pushed and nothing is deployed.
- Add a second gate in `deploy.sh`: don't switch traffic until the new
  container is `healthy`; if it never gets healthy, remove it, keep
  traffic where it was, and **exit 1** (a red job).
- Show both gates working on the broken release (for the second gate, run
  `deploy.sh` with the broken tag by hand). Then fix the release and show it
  deployed.

Useful commands:

```bash
docker ps -a --filter name=lab13- --format '{{.Names}}\t{{.Status}}'
docker logs --tail 20 lab13-green
docker inspect -f '{{.State.Health.Status}}' lab13-green
docker exec lab13-lb cat /etc/nginx/active.inc
docker exec lab13-lb sh -c "echo 'set \$backend lab13-blue;' > /etc/nginx/active.inc && nginx -s reload"

# a gate in CI: run the image you just built
docker run -d --name smoke -p 127.0.0.1:18000:8000 "$IMAGE"
for i in $(seq 1 20); do docker inspect -f '{{.State.Health.Status}}' smoke; sleep 2; done
curl -fsS 127.0.0.1:18000/api/status | jq -e '.status == "ok"'
docker rm -f smoke
```

AWS Implementation: the gate runs on the GitHub runner in the `build` job,
**between** `docker build` and `docker push`. For the second gate, the
`deploy` job prints the SSM output; show the `BLOCKED` line from it. Find
the crashing container through SSM, not SSH:

```bash
aws ssm send-command --instance-ids "$INSTANCE" --document-name AWS-RunShellScript \
  --parameters 'commands=["docker ps -a","docker logs --tail 20 lab13-green"]' \
  --query Command.CommandId --output text
aws ssm get-command-invocation --command-id <ID> --instance-id "$INSTANCE" \
  --query StandardOutputContent --output text
```

Local Alternative: the gate is a step in `build` on your laptop (act).

Explain: Why is "the pipeline is green" not the same as "the release
works"? Why did the unit tests pass? Why is a `sleep 3` not a health check?
What does each of your two gates protect against that the other one
doesn't?

---

## Task 2 — The Pipeline Leaks the Keys

A student from another group sent you a screenshot: your AWS secret key, in
base64, in the public log of your repository. A new teammate added this
"debug" workflow last week.

`.github/workflows/debug-deploy.yml`

```yaml
name: debug-deploy

on:
  pull_request_target: # "so that PRs from forks can deploy previews too"
  push:
    branches: [main]

permissions: write-all

env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  AWS_SESSION_TOKEN: ${{ secrets.AWS_SESSION_TOKEN }}
  AWS_REGION: us-east-1

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          ref: ${{ github.event.pull_request.head.sha || github.sha }}

      - name: Debug - why does the deploy fail?
        run: |
          set -x
          env | sort
          echo "key check: $(echo "$AWS_SECRET_ACCESS_KEY" | base64)"
          echo "Deploying: ${{ github.event.head_commit.message }}"

      - name: Save credentials for the next job
        run: |
          mkdir -p creds
          printf '[default]\naws_access_key_id=%s\naws_secret_access_key=%s\naws_session_token=%s\n' \
            "$AWS_ACCESS_KEY_ID" "$AWS_SECRET_ACCESS_KEY" "$AWS_SESSION_TOKEN" > creds/credentials

      - uses: actions/upload-artifact@v7
        with:
          name: aws-credentials
          path: creds/

      - name: Notify the team
        uses: some-random-user/telegram-notify@main # found on a blog
        with:
          message: "deploy ${{ github.sha }} started"
```

```
     public repository
        │
        ├── Actions log      ← key in base64, full env, commit message as code
        ├── Artifact         ← creds/credentials  (anyone can download it)
        ├── Fork PR          ← pull_request_target + checkout of the fork's code
        │                      runs with your secrets and a write token
        └── @main action     ← someone else's code, gets every secret
```

Requirements

- Find **at least 6** problems in this file. For each one: the line, what
  can leak or be abused, who could do it (anyone on the internet / a fork /
  the action's author).
- Fix the workflow: least-privilege `permissions:`, secrets only in the
  step that needs them, no debug output of secrets, no script injection, no
  credentials in artifacts, a third-party action pinned to a full commit
  SHA (or removed), and no fork code running with secrets.
- Run `actionlint` on the old and the new file and include the output.
- **Incident response:** the keys were public. Write and do the steps in
  order: stop the leak, remove the evidence from GitHub (run logs and
  artifacts), make the leaked credentials useless, put new ones into the
  secrets, check that nothing was created with them.
- Prove the fixed workflow doesn't leak: run it with a **fake** secret and
  search the whole log for the fake value **and** its base64.

Useful commands:

```bash
actionlint .github/workflows/*.yml
gh run list --workflow debug-deploy.yml
gh run delete <run-id>
gh api "repos/{owner}/{repo}/actions/artifacts" -q '.artifacts[] | [.id, .name] | @tsv'
gh api -X DELETE "repos/{owner}/{repo}/actions/artifacts/<id>"
git ls-remote https://github.com/<owner>/<action>.git refs/heads/main    # a SHA to pin to
```

AWS Implementation: in a real account you would **deactivate the access key
at once** (IAM → Security credentials). In Learner Lab you can't: **End
Lab**, **Start Lab** again, check that `aws configure get aws_access_key_id`
is different, and run the secret loop. Then look for anything you didn't
create:

```bash
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,State.Name,LaunchTime,Tags]' --output table
aws ecr describe-images --repository-name lab13-shop --query 'imageDetails[].[imageTags[0],imagePushedAt]' --output table
```

Local Alternative: run the old and the new workflow with act and fake
values, and save both logs:

```bash
act push -W .github/workflows/debug-deploy.yml -P self-hosted=-self-hosted \
  -s AWS_ACCESS_KEY_ID=AKIAFAKEFAKEFAKE0000 -s AWS_SECRET_ACCESS_KEY=fake-secret-12345 \
  -s AWS_SESSION_TOKEN=fake-token 2>&1 | tee leak-before.log
grep -c 'fake-secret-12345' leak-before.log
grep -c "$(printf 'fake-secret-12345' | base64)" leak-before.log
```

(For act, change `runs-on` to `self-hosted` and skip the
`telegram-notify` and `upload-artifact` steps with `if: ${{ !env.ACT }}` —
or explain from reading the file what they would do.)

Explain: GitHub masks secrets in logs — so why did the key leak anyway?
Why is `pull_request_target` with a checkout of the PR's code one of the
most dangerous patterns in GitHub Actions? Why does OIDC make this whole
incident smaller (what would there be to steal)?

---

## Task 3 — The Runbook Only Aidar Knows

Releases are done by hand by one engineer, from his laptop. Aidar is on
vacation next week, and a security fix must go out. This is the whole
documentation:

`RUNBOOK.md`

```text
# How to release (Aidar does it, Friday ~19:00)

1. ssh ec2-user@<server-ip>               (the key is on Aidar's laptop)
2. docker login to ECR with Aidar's own keys
3. docker pull <registry>/lab13-shop:<tag>    (tag from the chat)
4. cat active.inc in lab13-lb to see which color is live
5. docker run the new tag as the other color
6. wait "a bit", edit active.inc, nginx -s reload
7. open the site in a browser — "looks fine"
8. write "released <tag>" in the team Telegram chat
If it breaks: call Aidar.
```

Convert it into a **manual release workflow**: anyone in the team can start
it, but it only deploys after a second person approves it.

```
   "Run workflow"  tag = 9a0c3f1
          │
          v
   check: tag looks like a SHA, image exists in the registry
          │
          v
   ┌──────────────────────────┐
   │ environment: production  │  ← waits for a reviewer
   └──────────────────────────┘
          │ approved
          v
   deploy.sh 9a0c3f1  ──► smoke test ──► summary: who, what, when, from → to
```

Create `.github/workflows/release.yml`.

Requirements

- `on: workflow_dispatch` with an input `tag` (required, string).
- Reject a `tag` that is not exactly 7 hex characters, and a tag that is
  **not in the registry**, before anything is deployed.
- Deploy only after **approval** (see your variant below).
- `concurrency: deploy-production` shared with `deploy.yml`, never cancel a
  running deploy.
- After the deploy: a smoke test through the load balancer (20 requests,
  all `200` and all `build=<tag>`); red job if not.
- A job summary (`$GITHUB_STEP_SUMMARY`): who started it, who approved,
  which tag was live before and which is live now, time.
- A table in your report: every RUNBOOK step → what replaced it (or why it's
  gone).

Useful commands:

```bash
echo "$TAG" | grep -qE '^[0-9a-f]{7}$' || { echo "::error::bad tag: $TAG"; exit 1; }
echo "started by: $GITHUB_ACTOR" >> "$GITHUB_STEP_SUMMARY"
curl -s -m 2 "$SITE_URL/" | grep -c "build=$TAG"
```

AWS Implementation: check the tag with
`aws ecr describe-images --repository-name lab13-shop --image-ids imageTag="$TAG"`.
Create the environment with yourself as a **required reviewer** (Settings →
Environments → `production` → Required reviewers), and put
`environment: production` on the deploy job. Deploy through SSM like
`deploy.yml`. Start it:

```bash
gh workflow run release.yml -f tag=<an older tag>
gh run watch                # Waiting for review → approve in the browser
```

Local Alternative: check the tag with
`curl -s localhost:5001/v2/lab13-shop/tags/list | jq -e --arg t "$TAG" '.tags | index($t)'`.
act does **not** pause for environment reviewers, so make the approval an
input: a second input `confirm` that must be **exactly** the tag, typed by
the "second person" (and say in your report why this is weaker than a real
reviewer). Start it:

```bash
act workflow_dispatch -W .github/workflows/release.yml -P self-hosted=-self-hosted \
  --input tag=<an older tag> --input confirm=<the same tag>
```

Explain: What can't happen anymore that could happen with the runbook
(name at least four things)? Why is the approval on the **environment**
and not just "be careful"? Why must the image already exist — why doesn't
the release workflow build it?

---

## Task 4 — The Rollback Drill

Rollback is the thing everybody "has" and nobody has tried. Today you try
it, with a timer. A release that passes **every** gate — healthy, contract
fine — but is wrong for the business:

In `app/app.py`, change the `return` line of `index()` to:

```python
    return f"Narxoz Shop | {socket.gethostname()} | color={COLOR} | build={VERSION} | SALE -90% ON EVERYTHING\n"
```

```bash
git commit -am "Black Friday banner" && git push      # Local: act push ...
```

Marketing calls: "There is no sale. Take it down **now**."

```
  T0  bad build live           ─┐
  T1  noticed (the call)        │  time to detect
  T2  decision: roll back       │
  T3  site on the good build   ─┘  time to recover   (MTTR = T3 - T1)

  Option 1: switch nginx back to the other color      seconds
  Option 2: redeploy the last good tag (release.yml)  minutes
  Option 3: git revert + full pipeline                minutes + review
```

Requirements

- Before you start: write down which build is live, which color it is on,
  and which tag is the **last known good** — from the deploy log and the
  registry, not from memory.
- Do **all three** options one after another (redeploy the bad release
  between them), and time each one from the decision to the site serving
  the good build. Keep a watch loop running and count failed requests.
- Write a `rollback.yml` (`workflow_dispatch`, no inputs): it switches the
  load balancer to the **other** color only if that container is running
  and `healthy`, and fails otherwise. Use it for Option 1.
- After the drill, `main` must not contain the bad commit's banner
  anymore: explain why that matters for the **next** push.
- A timeline table T0–T3 for each option and the number of failed
  requests.

Useful commands:

```bash
while true; do curl -s -m 1 "$SITE_URL/" || echo "--- no answer"; sleep 0.5; done
# SITE_URL: Local http://localhost:8080, AWS http://$IP
cat /var/tmp/lab13-deploys.log            # Local; AWS: through SSM
docker inspect -f '{{.Name}} {{.State.Health.Status}} {{.Config.Image}}' lab13-blue lab13-green
git revert --no-edit <bad-sha>
date +%T
```

AWS Implementation: `rollback.yml` runs its checks and the switch on the
instance through SSM, like `deploy.yml`. Read the deploy log the same way:
`commands=["tail -5 /var/tmp/lab13-deploys.log"]`.

Local Alternative: `rollback.yml` runs on `self-hosted` with act and calls
`docker exec lab13-lb ...` directly.

Explain: Why is Option 1 so fast, and when is it **not** available
(after which kind of deploy, or which kind of change)? Why can a release
with a **database migration** make every option above dangerous? Rollback,
revert, fix forward: when would you choose each?

---

## Task 5 — Final Challenge: The Post-mortem

The company wants proof that the night from Task 1 can't happen again, and
a written post-mortem.

The final pipeline must look like this:

```
   pull request ──► test ──► build ──► image smoke gate ──► (no push on PRs)

   push to main ──► test ──► build ──► image smoke gate ──► push
                                                             │
                                                             v
                                         deploy.sh: health gate ──► switch
                                                             │
                                                             v
                                         smoke test through the LB
                                            │              │
                                           ok           failed
                                            │              │
                                          done      switch back to the old color
                                                     + red job

   release.yml  (manual, approval)      rollback.yml  (one click)
```

Requirements

- `deploy.yml` also runs on `pull_request` (test, build and the image
  gate only — **no** push, **no** deploy, **no** secrets).
- After the switch, `deploy.sh` runs a smoke test through the load
  balancer and **switches back** automatically if any request fails.
- Show in one place (screenshots or logs): the Task 1 release **blocked**,
  a good release **deployed**, a bad-but-healthy release (make one: e.g.
  `/` returns `500`) **switched back** automatically.
- `POSTMORTEM.md` (blameless — about the system, not the person):
  summary, impact (how long, what users saw), timeline, root cause, why it
  wasn't detected, what you changed (link the commits), action items that
  are still open.

Useful commands:

```bash
gh run list --limit 10                     # AWS
act pull_request -P self-hosted=-self-hosted   # Local
git log --oneline --graph
```

AWS Implementation, extra question for the post-mortem: the session
credentials expire after about 4 hours. What state is the system in if they
expire **in the middle** of the `deploy` job, and how does your pipeline
recover?

Explain: Which of your gates would have caught the night release, which
would have caught the "SALE" release, and which would catch neither? What
does that tell you about what pipelines can and can't protect?

---

## Cleanup

AWS Implementation:

```bash
aws ec2 terminate-instances --instance-ids "$INSTANCE"
aws ec2 wait instance-terminated --instance-ids "$INSTANCE"
aws ec2 delete-security-group --group-id "$SG"
aws ecr delete-repository --repository-name lab13-shop --force
for s in AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN; do gh secret delete "$s"; done
gh workflow disable deploy.yml          # the next push would deploy to nothing
```

Then **End Lab**.

Local Alternative:

```bash
docker rm -f lab13-lb lab13-blue lab13-green lab13-registry smoke 2>/dev/null
docker network rm lab13
docker image ls --format '{{.Repository}}:{{.Tag}}' | grep lab13-shop | xargs docker image rm
rm -f /var/tmp/lab13-deploys.log
```

---

## Final Report (you can do it in free form)

Submit the GitHub repository:

```
GitHub Repository  lab13-shop
│
├── app/
├── deploy/
│   ├── bootstrap.sh
│   └── deploy.sh             (with the health gate, smoke test, switch-back)
├── .github/
│   └── workflows/
│       ├── deploy.yml
│       ├── debug-deploy.yml  (fixed)
│       ├── release.yml
│       └── rollback.yml
├── RUNBOOK.md                (now: "use release.yml", and the step table)
├── POSTMORTEM.md
└── REPORT.md
```

Just an example of how you can show your report:

- Which variant you did (AWS or Local), and anything the Learner Lab didn't
  allow (action + error)
- For every task: the incident, the evidence (logs, `docker` / `aws`
  output, screenshots of pipeline runs), what you changed and why
- The list of problems from Task 2 and the incident response steps
- The rollback timeline table from Task 4
- The answers to the **Explain** questions
- Problems you ran into and how you solved them
