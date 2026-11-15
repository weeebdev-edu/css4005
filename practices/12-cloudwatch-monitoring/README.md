# Practice 12 — See It, Query It, Get Paged (CloudWatch)

**Objective:** add monitoring to the web fleet from week 11 **as code**:
memory and disk metrics from an agent, the nginx access log in a central
place, a log query, alarms that send you an email, and a dashboard. Then
break the site on purpose and watch the alarm go to `ALARM` and back to
`OK`. These are the skills from today's lecture.

**Timebox:** in class: ~55–60 min on Track A (the ALB and the first
metrics take a few minutes), ~40 min on Track B. If you've been stuck on
one step for more than 10 minutes, stop and ask. Don't push through
alone.

> **How you're checked:** this practice is done in class and checked
> **orally**. There is no report. When you finish, keep your terminal
> open. The instructor will ask you to show the items in the
> *Definition of done* and to explain what you did.

## Pick your track

You don't need AWS access to pass. Pick **one** track:

| | Track A — real AWS | Track B — no AWS |
|---|---|---|
| Needs | AWS Academy Learner Lab (or your own account), Terraform ≥ 1.10, AWS CLI v2, an email inbox | Docker with Compose v2, `curl` |
| What you build | Week 11 fleet + CloudWatch agent, log group, metric filter, SNS, 3 alarms, scaling policy, dashboard | nginx + Prometheus, node-exporter, Alloy, Loki, Alertmanager, Grafana |
| Slow step | ALB + instances healthy: 3–5 min; first agent metrics: ~5 min | Pulling the images the first time (~1 GB) |

The monitoring ideas are the same on both tracks:

| Concept | Track A (CloudWatch) | Track B (open-source stack) |
|---|---|---|
| Metric store | CloudWatch metrics | Prometheus |
| Built-in metrics | `AWS/EC2`, `AWS/ApplicationELB` | Prometheus scrapes `node-exporter` and `alloy` |
| Memory and disk | CloudWatch agent → `CWAgent` namespace | node-exporter (`node_memory_*`, `node_filesystem_*`) |
| Log shipper | CloudWatch agent `collect_list` | Grafana Alloy `loki.source.file` |
| Log store | Log group `/<prefix>/nginx/access` | Loki stream `{job="nginx"}` |
| Log query | Logs Insights | LogQL in Grafana **Explore** |
| Log line → metric | `aws_cloudwatch_log_metric_filter` | Alloy `stage.metrics` counter |
| Alarm | `aws_cloudwatch_metric_alarm` | Prometheus alert rule in `alerts.yml` |
| M of N / "how long" | `evaluation_periods`, `datapoints_to_alarm` | `for: 1m` (state `pending` → `firing`) |
| Missing data | `treat_missing_data` | No series → the rule doesn't fire |
| Notification | SNS topic → email | Alertmanager → webhook (`notify` container) |
| Back to normal | `ok_actions` | `send_resolved: true` |
| Dashboard | `aws_cloudwatch_dashboard` | Grafana dashboard |
| Retention | `retention_in_days` | Loki / Prometheus retention settings |

## Definition of done

Be ready to **show** each of these:

- [ ] The part you wrote yourself (Track A: the 5xx alarm and your Logs
      Insights query. Track B: the `High5xx` alert expression).
- [ ] A metric with its namespace/name, dimensions (or labels),
      statistic and period — and you can say why that statistic.
- [ ] Memory or disk metrics that come from the agent / node-exporter.
- [ ] A log query that finds the `500` lines, and one request found by
      its `request_id`.
- [ ] An alarm/alert going **ALARM → OK** (firing → resolved), with the
      notification for both.
- [ ] A dashboard with requests, errors, CPU/memory and a log panel.
- [ ] Cleanup: `terraform destroy` (Track A) or `docker compose down -v`
      (Track B), and nothing of yours left.

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
>    Terraform or the CLI says `ExpiredToken`, copy them again.
> 3. Region **`us-east-1`**. Per the Learner Lab Readme, only `us-east-1`
>    and `us-west-2` are allowed.
> 4. You **can't create IAM roles** in Learner Lab. The instances use the
>    existing **`LabInstanceProfile`** (role `LabRole`), and the
>    CloudWatch agent gets only the permissions `LabRole` has. In a real
>    account you'd attach the managed policy
>    `CloudWatchAgentServerPolicy` to the instance role.
> 5. CloudWatch, CloudWatch Logs and SNS are used today. If `apply` fails
>    with `AccessDenied` on one of them, the lab doesn't allow it: note
>    the resource and error, remove that block, and continue (or switch
>    to Track B).
> 6. When you're done: `terraform destroy`, then **End Lab**. A load
>    balancer bills every hour it exists; detailed monitoring and the
>    dashboard use your credits too.

Check that the credentials work:

```bash
aws sts get-caller-identity     # an assumed-role/voclabs/... ARN
```

### A1. Start from your week 11 project

You need your finished week 11 Terraform project (both TODOs done). If
you no longer have it, recreate the files from **Practice 11, A1**.

```bash
mkdir lab12
cp lab11/*.tf lab11/*.tftpl lab12/
cp lab11/.terraform.lock.hcl lab12/ 2>/dev/null   # same provider version as last week
cd lab12
```

(Don't copy `terraform.tfstate` or `.terraform/`: this is a new
project with its own state.)

In `variables.tf`, change the `prefix` default to `"lab12"` (or
`lab12-<yourname>`; keep it short — the ALB name must be ≤ 32
characters), then **add** these variables at the end:

```hcl
variable "alert_email" {
  description = "Where alarm notifications go (you confirm the subscription by email)"
  type        = string
}

variable "alert_cpu_pct" {
  description = "CPU alarm threshold, % (final project variant: 60, 70 or 80)"
  type        = number
  default     = 70
}

variable "alert_5xx_per_min" {
  description = "5xx alarm threshold, errors per minute (variant: 5, 10 or 20)"
  type        = number
  default     = 5
}

variable "log_retention_days" {
  description = "How long CloudWatch Logs keeps the access log"
  type        = number
  default     = 7

  validation {
    condition     = contains([1, 3, 5, 7, 14, 30, 60, 90], var.log_retention_days)
    error_message = "Use one of the retention values CloudWatch allows: 1, 3, 5, 7, 14, 30, 60, 90."
  }
}
```

Create `terraform.tfvars` with **your** email (and your final project
variant's thresholds, if you know them):

```hcl
alert_email = "you@example.com"
```

### A2. Logs and metrics from inside the instance

Replace `user-data.sh.tftpl` with this version. It does three things:
the same page as week 11; a **JSON access log** plus a broken page
`/boom` that always returns `500`; and the **CloudWatch agent** with
memory, disk and the access log. (`${prefix}` and `${log_group}` are
filled in by Terraform; `$${aws:...}` becomes `${aws:...}` for the
agent; everything else is left for the shell.)

```bash
#!/bin/bash
dnf install -y nginx amazon-cloudwatch-agent

# 1. The page (same as week 11)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
IID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "${prefix}: hello from $IID in $AZ" > /usr/share/nginx/html/index.html

# 2. A JSON access log (one object per request) and a broken page for the alarm test
cat > /etc/nginx/conf.d/json-log.conf <<'CONF'
log_format json escape=json '{"time":"$time_iso8601","request_id":"$request_id",'
                            '"trace_id":"$http_x_amzn_trace_id","method":"$request_method",'
                            '"path":"$uri","status":$status,"rt":$request_time}';
access_log /var/log/nginx/access.json json;
CONF
cat > /etc/nginx/default.d/boom.conf <<'CONF'
location = /boom { return 500 "boom\n"; }
CONF
systemctl enable --now nginx

# 3. CloudWatch agent: memory + disk metrics, and ship the JSON log
cat > /opt/aws/amazon-cloudwatch-agent/etc/lab12.json <<'JSON'
{
  "agent": { "metrics_collection_interval": 60 },
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": {
      "AutoScalingGroupName": "$${aws:AutoScalingGroupName}",
      "InstanceId": "$${aws:InstanceId}"
    },
    "aggregation_dimensions": [["AutoScalingGroupName"]],
    "metrics_collected": {
      "mem":  { "measurement": ["mem_used_percent"] },
      "disk": { "measurement": ["used_percent"], "resources": ["/"] }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/nginx/access.json",
            "log_group_name": "${log_group}",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
JSON
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/lab12.json
```

In `compute.tf`, inside `resource "aws_launch_template" "web"`,
**replace** the `user_data = ...` line with the block below and add
`monitoring` (detailed monitoring: EC2 metrics every minute):

```hcl
  user_data = base64encode(templatefile("${path.module}/user-data.sh.tftpl", {
    prefix    = var.prefix
    log_group = aws_cloudwatch_log_group.nginx.name
  }))

  monitoring {
    enabled = true # detailed monitoring: EC2 metrics every 1 min instead of 5
  }
```

In the same file, inside `resource "aws_autoscaling_group" "web"`, add
one line under `health_check_grace_period` (ASG group metrics are off
until you enable them):

```hcl
  enabled_metrics           = ["GroupDesiredCapacity", "GroupInServiceInstances"]
```

### A3. Log group, alarms and dashboard

`logs.tf`:

```hcl
# The log group the CloudWatch agent writes to. Created here (not by the
# agent) so that retention is in code and `destroy` deletes it.
resource "aws_cloudwatch_log_group" "nginx" {
  name              = "/${var.prefix}/nginx/access"
  retention_in_days = var.log_retention_days
}

# Metric filter: every log line with status >= 500 adds 1 to Lab12/Nginx5xx
resource "aws_cloudwatch_log_metric_filter" "nginx_5xx" {
  name           = "${var.prefix}-nginx-5xx"
  log_group_name = aws_cloudwatch_log_group.nginx.name
  pattern        = "{ $.status >= 500 }"

  metric_transformation {
    namespace = "Lab12"
    name      = "Nginx5xx"
    value     = "1"
    unit      = "Count"
  }
}
```

`alarms.tf` — **you complete the 5xx alarm** (the TODO):

```hcl
# Where notifications go: an SNS topic with your email subscribed
resource "aws_sns_topic" "alerts" {
  name = "${var.prefix}-alerts"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}

# 1. CPU of the whole fleet: 2 of the last 3 minutes above the threshold
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${var.prefix}-cpu-high"
  alarm_description   = "Average CPU of the web fleet above ${var.alert_cpu_pct}%"
  namespace           = "AWS/EC2"
  metric_name         = "CPUUtilization"
  dimensions          = { AutoScalingGroupName = aws_autoscaling_group.web.name }
  statistic           = "Average"
  period              = 60
  evaluation_periods  = 3
  datapoints_to_alarm = 2
  comparison_operator = "GreaterThanThreshold"
  threshold           = var.alert_cpu_pct
  treat_missing_data  = "missing"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]
}

# 2. Server errors returned by the web servers, per minute
resource "aws_cloudwatch_metric_alarm" "alb_5xx" {
  alarm_name        = "${var.prefix}-5xx"
  alarm_description = "More than ${var.alert_5xx_per_min} target 5xx per minute"
  namespace         = "AWS/ApplicationELB"
  # TODO: three lines.
  #   metric_name: the ALB metric that counts 5xx returned by the TARGETS
  #                (look at the "metrics of our stack" slide)
  #   dimensions:  { LoadBalancer = <the ALB's ARN suffix> }   hint: aws_lb.web.arn_suffix
  #   statistic:   which statistic do you use for counts?

  period              = 60
  evaluation_periods  = 1
  comparison_operator = "GreaterThanThreshold"
  threshold           = var.alert_5xx_per_min
  treat_missing_data  = "notBreaching" # no errors = no datapoint = fine
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]
}

# 3. Any unhealthy target for 2 minutes in a row
resource "aws_cloudwatch_metric_alarm" "unhealthy" {
  alarm_name        = "${var.prefix}-unhealthy-targets"
  alarm_description = "At least one web server fails the ALB health check"
  namespace         = "AWS/ApplicationELB"
  metric_name       = "UnHealthyHostCount"
  dimensions = {
    LoadBalancer = aws_lb.web.arn_suffix
    TargetGroup  = aws_lb_target_group.web.arn_suffix
  }
  statistic           = "Maximum"
  period              = 60
  evaluation_periods  = 2
  comparison_operator = "GreaterThanThreshold"
  threshold           = 0
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]
}

# Scaling: keep average CPU near 50% (AWS creates and manages its own alarms)
resource "aws_autoscaling_policy" "cpu_target" {
  name                   = "${var.prefix}-cpu-target"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 50
  }
}
```

`dashboard.tf`:

```hcl
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "${var.prefix}-dashboard"

  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric", x = 0, y = 0, width = 12, height = 6
        properties = {
          title  = "ALB: requests and target 5xx per minute"
          region = var.region
          stat   = "Sum"
          period = 60
          metrics = [
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", aws_lb.web.arn_suffix],
            [".", "HTTPCode_Target_5XX_Count", ".", "."],
          ]
        }
      },
      {
        type = "metric", x = 12, y = 0, width = 12, height = 6
        properties = {
          title   = "ALB: p95 response time (seconds)"
          region  = var.region
          stat    = "p95"
          period  = 60
          metrics = [["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", aws_lb.web.arn_suffix]]
        }
      },
      {
        type = "metric", x = 0, y = 6, width = 12, height = 6
        properties = {
          title  = "Fleet: CPU and memory (%)"
          region = var.region
          stat   = "Average"
          period = 60
          metrics = [
            ["AWS/EC2", "CPUUtilization", "AutoScalingGroupName", aws_autoscaling_group.web.name],
            ["CWAgent", "mem_used_percent", "AutoScalingGroupName", aws_autoscaling_group.web.name],
          ]
        }
      },
      {
        type = "log", x = 12, y = 6, width = 12, height = 6
        properties = {
          title  = "Latest server errors"
          region = var.region
          view   = "table"
          query  = "SOURCE '${aws_cloudwatch_log_group.nginx.name}' | fields @timestamp, status, path, request_id | filter status >= 500 | sort @timestamp desc | limit 20"
        }
      },
      {
        type = "alarm", x = 0, y = 12, width = 24, height = 3
        properties = {
          title = "Alarms"
          alarms = [
            aws_cloudwatch_metric_alarm.cpu_high.arn,
            aws_cloudwatch_metric_alarm.alb_5xx.arn,
            aws_cloudwatch_metric_alarm.unhealthy.arn,
          ]
        }
      },
    ]
  })
}
```

Add to `outputs.tf`:

```hcl
output "alb_arn_suffix" {
  value = aws_lb.web.arn_suffix
}

output "log_group" {
  value = aws_cloudwatch_log_group.nginx.name
}

output "sns_topic_arn" {
  value = aws_sns_topic.alerts.arn
}

output "dashboard_url" {
  value = "https://${var.region}.console.aws.amazon.com/cloudwatch/home?region=${var.region}#dashboards:name=${aws_cloudwatch_dashboard.main.dashboard_name}"
}
```

### A4. init, plan, apply — and confirm the email

```bash
terraform init
terraform fmt
terraform validate      # Success! The configuration is valid.
terraform plan -out tfplan
```

The last line should say **`Plan: 27 to add, 0 to change, 0 to
destroy.`** (week 11's 18 + 9 monitoring resources). In the plan, find
your 5xx alarm and check its `metric_name`, `dimensions` and
`statistic`.

```bash
terraform apply tfplan          # 3–5 minutes (the ALB is the slow part)
```

Within a minute you get an email **"AWS Notification - Subscription
Confirmation"**. Click **Confirm subscription** (check the spam folder).
Then:

```bash
aws sns list-subscriptions-by-topic --topic-arn $(terraform output -raw sns_topic_arn) \
  --query 'Subscriptions[].[Protocol,Endpoint,SubscriptionArn]' --output table
```

`SubscriptionArn` must be a real ARN, not `PendingConfirmation`. Until
you confirm, **no alarm email arrives**.

Set a few shell variables for the next steps (put **your** prefix):

```bash
PREFIX=lab12                                  # the value of var.prefix
ALB=$(terraform output -raw alb_dns)
LB=$(terraform output -raw alb_arn_suffix)    # app/<name>/<id>
LG=$(terraform output -raw log_group)
```

### A5. Metrics

Wait 2–3 minutes for the instances to be healthy, then send traffic:

```bash
for i in $(seq 1 100); do curl -s -o /dev/null $ALB; done
```

Which metrics does the ALB have, and how many requests per minute did it
count? (ALB metrics arrive about a minute late.)

```bash
aws cloudwatch list-metrics --namespace AWS/ApplicationELB \
  --dimensions Name=LoadBalancer,Value=$LB --query 'Metrics[].MetricName' --output text

aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB \
  --metric-name RequestCount --statistics Sum --period 60 \
  --dimensions Name=LoadBalancer,Value=$LB \
  --start-time $(( $(date +%s) - 900 )) --end-time $(date +%s) \
  --query 'sort_by(Datapoints,&Timestamp)[].[Timestamp,Sum]' --output table
```

Now the agent's metrics (they appear ~5 minutes after the instances
start):

```bash
aws cloudwatch list-metrics --namespace CWAgent \
  --query 'Metrics[].[MetricName,Dimensions[0].Name,Dimensions[0].Value]' --output table
```

You should see `mem_used_percent` and `disk_used_percent`, per instance
**and** per `AutoScalingGroupName` (the aggregated series). Be ready to
explain why EC2's own metrics don't include memory.

> **No `CWAgent` metrics after 10 minutes?** The agent probably lacks a
> permission in `LabRole`. Connect to an instance (EC2 console →
> instance → **Connect → Session Manager**, if the lab allows it) and
> read `/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`.
> Look for `AccessDenied`. Write down which action was denied and
> continue: the rest of the practice doesn't depend on it.

### A6. Logs and Logs Insights

The agent creates one **log stream per instance**:

```bash
aws logs describe-log-streams --log-group-name $LG \
  --query 'logStreams[].logStreamName' --output text
aws logs tail $LG --since 10m --format short | tail -5
```

Each line is one JSON object. Notice the ALB's health checks in the log
(`"path":"/"` every few seconds) — they are part of your log bill.

Run a Logs Insights query from the CLI:

```bash
QID=$(aws logs start-query --log-group-name $LG \
  --start-time $(( $(date +%s) - 900 )) --end-time $(date +%s) \
  --query-string 'stats count(*) as requests by status' \
  --query queryId --output text)
sleep 5
aws logs get-query-results --query-id $QID
```

**TODO — your query.** In the console (**CloudWatch → Logs → Logs
Insights**, select your log group, last 30 minutes), write a query that
lists the **last 20 lines with status ≥ 500**, newest first, showing
`@timestamp`, `status`, `path` and `request_id`. (You'll have such
lines after A7 — come back then.) Then copy one `request_id` and write a
second query that finds **only that request**. Save the first one with
**Save** — that's a *saved query*, as the final project asks for.

### A7. Break it: the 5xx alarm fires and recovers

First, test the notification path without breaking anything:

```bash
aws cloudwatch set-alarm-state --alarm-name "$PREFIX-cpu-high" \
  --state-value ALARM --state-reason "Testing the SNS notification"
```

You should get an **ALARM** email within a minute. At the next
evaluation the alarm goes back to its real state (`OK`), and you get an
**OK** email too, because of `ok_actions`.

Now the real thing — about 60 errors per minute for 3 minutes:

```bash
for i in $(seq 1 180); do curl -s -o /dev/null $ALB/boom; sleep 1; done
```

In a second terminal, watch the alarms (refresh every minute):

```bash
aws cloudwatch describe-alarms --alarm-name-prefix "$PREFIX" \
  --query 'MetricAlarms[].[AlarmName,StateValue]' --output table
```

`<prefix>-5xx` goes to **ALARM** after 1–3 minutes, and you get an
email. When the loop ends, it returns to **OK** within a few minutes
(no errors → no datapoints → `notBreaching`), and you get the OK email.
The history is your evidence:

```bash
aws cloudwatch describe-alarm-history --alarm-name "$PREFIX-5xx" \
  --history-item-type StateUpdate \
  --query 'AlarmHistoryItems[].[Timestamp,HistorySummary]' --output table
```

Look at the metric filter's metric too: **CloudWatch → Metrics → All
metrics → Lab12 → Metrics with no dimensions → Nginx5xx**. It counted
the same errors, from the logs instead of from the ALB.

### A8. The dashboard and the scaling policy

```bash
terraform output dashboard_url
```

Open the link (you must be signed in to the lab's console: **AWS** link
in the Learner Lab page first). You should see requests and the 5xx
spike, p95 latency, CPU and memory, your errors from the logs, and the
three alarms.

Target tracking created **its own** alarms — look at them, don't edit
them:

```bash
aws cloudwatch describe-alarms --alarm-name-prefix "TargetTracking-$PREFIX-asg" \
  --query 'MetricAlarms[].[AlarmName,StateValue,Threshold]' --output table
```

Be ready to explain why the `AlarmLow` one is usually in `ALARM` state
in an idle lab, and why that's fine.

### A9. Destroy (required)

```bash
terraform destroy
```

Type `yes`. This removes the fleet **and** the alarms, dashboard, log
group and SNS topic. Check, then **End Lab**:

```bash
terraform state list                                                    # empty
aws cloudwatch describe-alarms --alarm-name-prefix "$PREFIX" --query 'MetricAlarms[].AlarmName'
aws logs describe-log-groups --log-group-name-prefix "/$PREFIX" --query 'logGroups[].logGroupName'
aws cloudwatch list-dashboards --query 'DashboardEntries[].DashboardName' --output text
```

(Metrics themselves can't be deleted; they simply stop receiving data
and expire.)

### Stretch (optional)

- **A fourth alarm for the final project:** p95 latency. Use
  `metric_name = "TargetResponseTime"` with `extended_statistic = "p95"`
  instead of `statistic`, threshold `1` (second). Run `plan`: how many
  resources change?
- **Scale out on CPU:** burn CPU on one instance (Session Manager, if
  available: `yes > /dev/null & yes > /dev/null &`). After a few
  minutes: the CPU alarm, the target tracking `AlarmHigh`, and a third
  instance in `GroupInServiceInstances` on the dashboard. `pkill yes`
  when done.

---

## Track B — no AWS (Prometheus, Grafana, Loki in Docker)

You need Docker with Compose v2 (`docker compose version`). One nginx
"web server" writes a JSON access log; the rest of the stack collects,
stores, queries, alerts and draws it. Everything runs on your laptop.

### B1. Create the project

```bash
mkdir lab12 && cd lab12
```

`compose.yaml`:

```yaml
name: lab12

services:
  # The app: one nginx "web server" that writes JSON access logs
  web:
    image: nginx:1.27-alpine
    ports: ["${WEB_PORT:-8080}:80"]   # change with WEB_PORT=8081 if 8080 is busy
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - nginx-logs:/var/log/lab12

  # Host metrics: CPU, memory, disk (what the CloudWatch agent adds on EC2)
  node-exporter:
    image: prom/node-exporter:v1.12.1

  # Log shipper: tails the access log, sends it to Loki, counts 5xx
  alloy:
    image: grafana/alloy:v1.22.0
    command: run --server.http.listen-addr=0.0.0.0:12345 --storage.path=/var/lib/alloy/data /etc/alloy/config.alloy
    volumes:
      - ./config.alloy:/etc/alloy/config.alloy:ro
      - nginx-logs:/var/log/lab12:ro

  # Log store and query engine (CloudWatch Logs + Logs Insights)
  loki:
    image: grafana/loki:3.7.10

  # Metric store + alert rules (CloudWatch metrics + alarms)
  prometheus:
    image: prom/prometheus:v3.16.0
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alerts.yml:/etc/prometheus/alerts.yml:ro

  # Routes firing alerts to receivers (SNS)
  alertmanager:
    image: prom/alertmanager:v0.35.0
    ports: ["9093:9093"]
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro

  # The "email inbox": prints every notification it receives
  notify:
    image: mendhak/http-https-echo:43

  # Dashboards (CloudWatch dashboards)
  grafana:
    image: grafana/grafana:13.3.1
    ports: ["${GRAFANA_PORT:-3000}:3000"]
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
    volumes:
      - ./grafana-datasources.yml:/etc/grafana/provisioning/datasources/lab12.yml:ro

volumes:
  nginx-logs:
```

`nginx.conf` — the same JSON log format as Track A, and the broken page:

```nginx
# One JSON object per request: easy to query, like structured logs on EC2
log_format json escape=json '{"time":"$time_iso8601","request_id":"$request_id",'
                            '"method":"$request_method","path":"$uri",'
                            '"status":$status,"rt":$request_time}';

server {
  listen 80;
  access_log /var/log/lab12/access.log json;
  default_type text/plain;

  location = /boom { return 500 "boom\n"; }        # a broken endpoint
  location /       { return 200 "hello from $hostname\n"; }
}
```

`config.alloy` — the "CloudWatch agent" for logs, plus the "metric
filter":

```alloy
// 1. Which file to collect (the CloudWatch agent's "collect_list")
local.file_match "nginx" {
  path_targets = [{"__path__" = "/var/log/lab12/access.log", "job" = "nginx"}]
}

// 2. Tail the file
loki.source.file "nginx" {
  targets    = local.file_match.nginx.targets
  forward_to = [loki.process.nginx.receiver]
}

// 3. Parse the JSON and count requests by status (the "metric filter")
loki.process "nginx" {
  stage.json {
    expressions = { status = "" }
  }

  stage.labels {
    values = { status = "" }
  }

  stage.metrics {
    metric.counter {
      name        = "nginx_requests_total"
      description = "Requests seen in the nginx access log"
      match_all   = true
      action      = "inc"
    }
  }

  forward_to = [loki.write.local.receiver]
}

// 4. Ship the lines to Loki (CloudWatch Logs)
loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

The counter is published by Alloy as
`loki_process_custom_nginx_requests_total{status="..."}`, and Prometheus
scrapes it.

`prometheus.yml`:

```yaml
global:
  scrape_interval: 15s      # how often metrics are collected (the "period")
  evaluation_interval: 15s  # how often alert rules are checked

rule_files:
  - alerts.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: node          # CPU, memory, disk
    static_configs:
      - targets: ["node-exporter:9100"]
  - job_name: alloy         # the counters made from the access log
    static_configs:
      - targets: ["alloy:12345"]
```

`alerts.yml` — **you write the `High5xx` expression** (the TODO):

```yaml
groups:
  - name: lab12
    rules:
      - alert: HighCPU
        expr: 100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[1m]))) > 70
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Average CPU above 70% for 2 minutes"

      - alert: High5xx
        # TODO: fire when there were more than 5 responses with a 5xx status
        # in the last minute. Hints:
        #   - the metric is loki_process_custom_nginx_requests_total
        #   - select 5xx statuses with the label matcher {status=~"5.."}
        #   - increase(<metric>[1m]) = how much a counter grew in 1 minute
        #   - sum(...) adds up all the series
        expr: ""   # TODO: your expression here
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "More than 5 server errors per minute"
```

`alertmanager.yml`:

```yaml
route:
  receiver: notify
  group_wait: 10s        # wait a little to batch alerts that fire together
  group_interval: 1m
  repeat_interval: 1h    # remind every hour while still firing

receivers:
  - name: notify
    webhook_configs:
      - url: http://notify:8080/
        send_resolved: true   # also tell us when it's OK again (like ok_actions)
```

`grafana-datasources.yml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus
    url: http://prometheus:9090
    isDefault: true
  - name: Loki
    type: loki
    uid: loki
    url: http://loki:3100
```

### B2. Start it and check the targets

Check your alert rule first — `promtool` ships inside the Prometheus
image:

```bash
docker run --rm -v "$PWD":/w --entrypoint promtool prom/prometheus:v3.16.0 check rules /w/alerts.yml
# SUCCESS: 2 rules found
```

Then start everything:

```bash
docker compose up -d
docker compose ps                      # 8 services, all "running"
for i in $(seq 1 50); do curl -s -o /dev/null localhost:8080/; done
curl -s localhost:8080/
curl -s -o /dev/null -w '%{http_code}\n' localhost:8080/boom      # 500
```

(If port 8080 or 3000 is busy on your laptop: `WEB_PORT=8081
GRAFANA_PORT=3001 docker compose up -d`, and use those ports below.)

Open <http://localhost:9090/targets>: both `node` and `alloy` must be
**UP**. That's the list of "what is being monitored".

### B3. Metrics (Prometheus)

Open <http://localhost:9090/query> and run these one by one (switch to
the **Graph** tab):

```promql
sum by (status) (loki_process_custom_nginx_requests_total)
sum(rate(loki_process_custom_nginx_requests_total[1m])) * 60
100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[1m])))
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
max(100 * (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes))
```

1. Requests counted from the log, per status (a **counter**: it only goes up).
2. Requests per minute — `rate` turns a counter into "per second".
3. CPU %, 4. memory %, 5. disk %. On Docker Desktop / OrbStack these are
   the numbers of the **Docker VM**, not your whole laptop.

Be ready to map each one to its CloudWatch equivalent (namespace,
statistic, period).

### B4. Logs (Loki in Grafana Explore)

Open Grafana at <http://localhost:3000> (no login needed) → **Explore**
→ data source **Loki** → switch the query editor to **Code** and run:

```logql
{job="nginx"}
{job="nginx"} | json | status >= 500
sum by (status) (count_over_time({job="nginx"} | json [1m]))
```

1. All lines (a log "stream" ≈ a log stream in CloudWatch).
2. Only server errors (≈ `filter status >= 500` in Logs Insights).
3. Lines per status per minute — a metric computed from logs at query
   time.

Open one `500` line, copy its `request_id`, and find **only that
request**:

```logql
{job="nginx"} |= "<paste the request_id>"
```

### B5. Break it: the alert fires and resolves

About 60 errors per minute for 2½ minutes:

```bash
for i in $(seq 1 150); do curl -s -o /dev/null localhost:8080/boom; sleep 1; done
```

Meanwhile, watch <http://localhost:9090/alerts>: `High5xx` goes
**Inactive → Pending** (the condition is true, but not yet for `1m`)
**→ Firing**. Alertmanager (<http://localhost:9093>) shows it, and the
`notify` container receives the notification:

```bash
docker compose logs notify | grep -E '"status": "(firing|resolved)"|"alertname"'
```

When the loop ends, the errors stop, `increase(...[1m])` drops to 0, and
after 1–3 minutes the alert is **resolved** — the `notify` log shows a
second message with `"status": "resolved"`.

### B6. A dashboard (Grafana)

In Grafana: **Dashboards → New → New dashboard → Add visualization**,
pick the data source, and build at least these four panels (**Run
queries**, set a **Title**, then **Back to dashboard**):

| Panel | Data source | Query |
|---|---|---|
| Requests per minute by status | Prometheus | `sum by (status) (rate(loki_process_custom_nginx_requests_total[1m])) * 60` |
| CPU and memory (%) | Prometheus | the CPU and memory queries from B3 (two queries in one panel) |
| 5xx per minute | Prometheus | `sum(increase(loki_process_custom_nginx_requests_total{status=~"5.."}[1m]))` |
| Latest errors | Loki (visualization **Logs**) | `{job="nginx"} \| json \| status >= 500` |

Save the dashboard as `lab12`. Then **Export → Export as JSON**: that
JSON is your dashboard **as code**, like `dashboard_body` in Track A.

### B7. Cleanup

```bash
docker compose down -v             # -v also deletes the log volume
docker ps -a --filter name=lab12   # nothing left
```

### Stretch (optional): the CPU alert

Burn every CPU of the Docker VM for ~3 minutes and watch `HighCPU` go
Pending → Firing (it needs `for: 2m`):

```bash
docker run --rm -d --name lab12-burn alpine:3 sh -c 'for i in $(seq $(nproc)); do yes > /dev/null & done; wait'
# ... watch http://localhost:9090/alerts ...
docker rm -f lab12-burn
```

---

## Be ready to explain

1. Metrics vs logs: which one fired your alarm, and which one told you
   **which** request failed?
2. Why `Sum` for the 5xx count, `Average` for CPU and `p95` for latency?
3. Why doesn't EC2 report memory, and what did you use to get it?
4. Your 5xx alarm: what are its period, `evaluation_periods` and
   threshold (Track B: the `for` and the range `[1m]`)? Roughly how long
   from the first error to the notification?
5. Why `treat_missing_data = "notBreaching"` for the 5xx alarm? What
   state would it be in overnight with the default?
6. What does a metric filter (Track B: `stage.metrics`) do, and why
   does it only count errors from the moment it exists?
7. Track A: what's the difference between your `cpu-high` alarm and the
   `TargetTracking-…` alarms? Track B: what's the difference between
   **Pending** and **Firing**?
8. What in your setup costs money as traffic grows, and how did you
   limit it (retention, what you log)?

## Common mistakes

- **`ExpiredToken` / `InvalidClientTokenId` (Track A).** The Learner Lab
  session ended or restarted. Copy the new credentials from AWS Details.
- **No alarm emails (Track A).** The SNS subscription is still
  `PendingConfirmation`: find the confirmation email (spam folder) and
  click the link. `set-alarm-state` is the quickest test.
- **The 5xx alarm stays `INSUFFICIENT_DATA` (Track A).** Missing
  `treat_missing_data = "notBreaching"`, or a wrong dimension: the
  `LoadBalancer` value is the ARN **suffix** (`app/<name>/<id>`), not
  the DNS name or the full ARN.
- **The 5xx alarm never fires (Track A).** Check `metric_name`
  (`HTTPCode_Target_5XX_Count`, case matters) and `statistic = "Sum"`:
  `Average` of a count is always about 1.
- **`Invalid template interpolation value` / `${aws:...}` errors on
  `plan` (Track A).** In a `.tftpl` file the agent's `${aws:...}` must
  be written `$${aws:...}`.
- **No `CWAgent` metrics or no log streams (Track A).** The agent needs
  ~5 minutes; if still nothing, its log on the instance will say
  `AccessDenied` (a `LabRole` limit). Record it and continue.
- **`Error: ... port is already allocated` (Track B).** Another program
  uses 8080 or 3000: `WEB_PORT=8081 GRAFANA_PORT=3001 docker compose up -d`.
- **Grafana opens some other website on :3000 (Track B).** Another dev
  server owns port 3000 on `localhost`. Use `GRAFANA_PORT=3001`.
- **`alloy` target DOWN or no `loki_process_custom_…` metric (Track
  B).** No requests yet (the counter appears with the first log line), or
  a typo in `config.alloy`: `docker compose logs alloy`.
- **`promtool` says `field 'expr' must be set` (Track B).** The TODO in
  `alerts.yml` is still empty. Other errors are usually YAML indentation:
  `promtool check rules` points to the line.
- **node-exporter logs `Failed to open directory, disabling udev device
  properties` (Track B).** Harmless inside a container.
- **`Cannot connect to the Docker daemon` (Track B).** Start Docker
  Desktop / OrbStack first.
