---
theme: seriph
title: "CSS4005 — Lecture 12: Monitoring and Logging with AWS CloudWatch"
info: |
  CSS4005 — Cloud Infrastructure Construction · Narxoz University
  Lecture 12 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# CSS4005 — Cloud Infrastructure Construction

## Lecture 12: Monitoring and Logging with AWS CloudWatch

<div class="pt-8 opacity-70">
Nauruzbayeva Farikha · Lesson 12
</div>

---
layout: default
---

# Recap — Lesson 11 (Terraform)

<v-clicks>

- `plan` vs `apply`? <span v-click class="opacity-60">(plan shows what would change and changes nothing; apply makes the changes)</span>
- What is the state file, and why not in Git? <span v-click class="opacity-60">(Terraform's memory of which real resource belongs to which block; it can hold secrets in plain text)</span>
- `count` vs `for_each`? <span v-click class="opacity-60">(copies by index vs by key — removing the first `count` item shifts all the others)</span>
- What is drift, and how do you fix it? <span v-click class="opacity-60">(reality changed outside Terraform; `plan` notices it; fix the code, not the console)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] Why observability: metrics, logs, traces
- [ ] CloudWatch metrics: namespaces, dimensions, statistics, periods
- [ ] The CloudWatch agent and CloudWatch Logs
- [ ] Alarms, SNS notifications and alarms that scale
- [ ] Dashboards, EventBridge and the cost of monitoring
- [ ] Monitoring as code with Terraform → today's practice

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Friday, 23:40. The fleet you built with Terraform last week is running.
A message arrives: **"the site is slow, sometimes it shows an error."**

Slow for whom? Since when? Which server? You SSH into three instances
and `grep` the logs. By the time you find something, it has either fixed
itself — or got worse.

</div>

<div v-click class="mt-8 text-xl font-bold">
Today: metrics tell you <i>what</i> is wrong, logs tell you <i>why</i>,
alarms tell you <i>before</i> your users do.
</div>

---
layout: section
transition: slide-left
---

# Block 1
## Why observability

---
---

# You can't fix what you can't see

<v-clicks>

- **Monitoring:** watching known numbers and being told when one goes
  wrong ("CPU above 80%", "5xx above 10 per minute").
- **Observability:** having enough data to answer questions you didn't
  plan for ("why are only requests to `/login` slow, only in us-east-1b?").
- In the cloud you **can't walk to the server**: instances come and go
  (week 9), and an ASG may already have replaced the one that failed.
- So the data must leave the server **as it happens** — to a place that
  outlives the instance.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
On AWS that place is <b>Amazon CloudWatch</b>: metrics, logs, alarms and
dashboards for almost every service, in every region.
</div>

---
---

# Three kinds of telemetry

| | Metrics | Logs | Traces |
|---|---|---|---|
| What | Numbers over time | Text records of events | One request's path through services |
| Example | `CPUUtilization = 82%` at 10:31 | `GET /boom 500 0.002s req=4f1c…` | ALB → web → DB, 340 ms of which 300 ms in DB |
| Good for | Trends, alarms, dashboards | The **why**: errors, details | Where the time goes |
| Cost | Cheap, small | Grows with traffic | Usually sampled |
| On AWS | CloudWatch metrics | CloudWatch Logs | AWS X-Ray / Application Signals |

<div v-click class="mt-6 text-sm opacity-70">
They work together: an alarm on a <b>metric</b> wakes you up, the
<b>logs</b> around that minute explain it, and a <b>request ID</b> lets
you follow one failing request.
</div>

---
---

# What to measure: three checklists

| Method | For | Signals | In our stack |
|---|---|---|---|
| **Golden signals** (Google SRE) | Any user-facing service | Latency, Traffic, Errors, Saturation | ALB `TargetResponseTime`, `RequestCount`, 5xx, fleet CPU |
| **RED** | Request-driven services | Rate, Errors, Duration | Requests/min, 5xx/min, p95 latency |
| **USE** | Resources (CPU, disk, DB) | Utilization, Saturation, Errors | EC2 CPU, memory, disk; RDS connections, free storage |

<v-clicks>

- RED tells you **users are hurting**; USE tells you **which resource**
  is the bottleneck.
- Start with these few numbers. A dashboard of 60 graphs is as useless
  as none.

</v-clicks>

---
---

# Alert on symptoms, not causes

<v-clicks>

- **Symptom:** "users get errors", "p95 latency above 1 s" — something a
  user would notice. Wake someone up for these.
- **Cause:** "CPU at 85%" — maybe a problem, maybe the ASG is about to
  scale out and fix it. Put it on the dashboard; alert only if it lasts.
- A good alert is **real** (not noise), **urgent** (can't wait until
  Monday) and **actionable** (someone knows what to do).
- Too many alerts → people mute them → the real one is missed.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
Every alarm you create should have an answer to "who gets it, and what
do they do?" If nobody would act on it, make it a graph instead.
</div>

---
layout: section
transition: slide-left
---

# Block 2
## CloudWatch metrics

---
---

# Anatomy of a metric

```text {1|2|3|4}
Namespace:   AWS/ApplicationELB
MetricName:  HTTPCode_Target_5XX_Count
Dimensions:  LoadBalancer = app/lab12-alb/50dc6c495c0c9188
Datapoint:   2026-11-16T10:31:00Z   Sum = 7   Unit = Count
```

<v-clicks>

- **Namespace:** a folder of metrics. AWS services use `AWS/<service>`
  (`AWS/EC2`, `AWS/RDS`); yours are anything else (`CWAgent`, `Lab12`).
- **Dimensions:** name/value pairs that say **which** thing (which ALB,
  which instance, which ASG). Up to 30 per metric.
- Namespace + name + dimensions = **one metric**. Change one dimension
  value and it's a different metric — and custom ones are billed each.

</v-clicks>

---
---

# Statistics and periods

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

| Statistic | Use it for |
|---|---|
| `Sum` | Counts: requests, errors |
| `Average` | CPU, memory |
| `Maximum` / `Minimum` | Worst case: unhealthy hosts, free storage |
| `SampleCount` | How many datapoints arrived |
| `p95`, `p99` | Latency — what slow users feel |

</div>
<div>

**Period:** the time bucket a statistic is computed over (60 s, 300 s…).

**Retention** — older data is rolled up:

| Resolution | Kept for |
|---|---|
| < 60 s (high-res) | 3 hours |
| 1 minute | 15 days |
| 5 minutes | 63 days |
| 1 hour | 15 months |

</div>
</div>

<div v-click class="mt-4 text-sm opacity-70">
An <b>average</b> latency of 200 ms can hide 2% of users waiting 8 s.
Alarm on <b>p95/p99</b> for latency.
</div>

---
---

# EC2: basic vs detailed monitoring

| | Basic (default) | Detailed |
|---|---|---|
| Period | Every **5 minutes** | Every **1 minute** |
| Cost | Free | Paid, like custom metrics |
| Turn on | — | `monitoring { enabled = true }` in the launch template |

<v-clicks>

- What EC2 reports from **outside** the VM: `CPUUtilization`,
  `NetworkIn/Out`, `StatusCheckFailed`, `CPUCreditBalance` (t3).
- What it **can't** see — it's inside the OS: **memory** used and
  **disk space** used. For those you need an agent.
- With 5-minute data, a 1-minute alarm can't work, and a 3-minute spike
  disappears into the average.

</v-clicks>

---
---

# The metrics of our stack

<div class="text-sm">

| Service | Namespace | Key metrics | Dimensions |
|---|---|---|---|
| ALB | `AWS/ApplicationELB` | `RequestCount`, `TargetResponseTime`, `HTTPCode_Target_5XX_Count`, `HTTPCode_ELB_5XX_Count` | `LoadBalancer` |
| Target group | `AWS/ApplicationELB` | `HealthyHostCount`, `UnHealthyHostCount` | `TargetGroup` + `LoadBalancer` |
| ASG | `AWS/AutoScaling` | `GroupDesiredCapacity`, `GroupInServiceInstances` | `AutoScalingGroupName` |
| EC2 | `AWS/EC2` | `CPUUtilization`, `StatusCheckFailed` | `InstanceId` or `AutoScalingGroupName` |
| RDS | `AWS/RDS` | `CPUUtilization`, `FreeStorageSpace`, `DatabaseConnections`, `ReplicaLag` | `DBInstanceIdentifier` |

</div>

<v-clicks>

- `HTTPCode_*` metrics are sent **only when the count isn't zero** (no
  errors = no datapoints). ASG group metrics need `enabled_metrics` (free).

</v-clicks>

---
---

# Reading metrics from the CLI

```bash {1-3|5-9}
aws cloudwatch list-metrics --namespace AWS/ApplicationELB \
  --dimensions Name=LoadBalancer,Value=app/lab12-alb/50dc6c495c0c9188 \
  --query 'Metrics[].MetricName' --output text

aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB \
  --metric-name RequestCount --statistics Sum --period 60 \
  --dimensions Name=LoadBalancer,Value=app/lab12-alb/50dc6c495c0c9188 \
  --start-time $(( $(date +%s) - 900 )) --end-time $(date +%s) \
  --query 'sort_by(Datapoints,&Timestamp)[].[Timestamp,Sum]' --output table
```

<v-clicks>

- The `LoadBalancer` dimension is the **end of the ALB's ARN**
  (`app/<name>/<id>`) — Terraform calls it `aws_lb.web.arn_suffix`.
- The console (**CloudWatch → Metrics → All metrics**) shows the same
  data as graphs.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
In one minute the ALB served 1,000 requests: 980 took 50 ms and 20 took
8 seconds. What does the <b>Average</b> of <code>TargetResponseTime</code>
show, and what does <b>p99</b> show?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Average ≈ 0.21 s — looks fine. p99 = 8 s — 2 in 100 users wait eight
seconds. That's why latency alarms use percentiles.
</div>

---
layout: section
transition: slide-left
---

# Block 3
## The CloudWatch agent and CloudWatch Logs

---
---

# The CloudWatch agent

<v-clicks>

- A program on the instance that sends what AWS can't see from outside:
  **memory**, **disk**, processes — and **log files**.
- Install on Amazon Linux 2023: `dnf install -y amazon-cloudwatch-agent`,
  then start it with a JSON config (in user data, so every ASG instance
  gets it).
- It can also scrape **Prometheus** `/metrics` endpoints (useful for the
  final project's app metrics).
- It needs **permissions** (`cloudwatch:PutMetricData`,
  `logs:PutLogEvents`, `ec2:DescribeTags`…): normally the managed policy
  `CloudWatchAgentServerPolicy` on the instance role.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
Learner Lab: you can't create or change roles, so the agent uses whatever
<b>LabRole</b> (via <code>LabInstanceProfile</code>) allows. If a metric
never appears, read the agent's own log — today's practice shows where.
</div>

---
---

# The agent config

```json {2-9|10-14}
{
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": { "AutoScalingGroupName": "${aws:AutoScalingGroupName}" },
    "aggregation_dimensions": [["AutoScalingGroupName"]],
    "metrics_collected": {
      "mem":  { "measurement": ["mem_used_percent"] },
      "disk": { "measurement": ["used_percent"], "resources": ["/"] } }
  },
  "logs": { "logs_collected": { "files": { "collect_list": [{
    "file_path":       "/var/log/nginx/access.json",
    "log_group_name":  "/lab12/nginx/access",
    "log_stream_name": "{instance_id}" }] } } }
}
```

<div v-click class="mt-4 text-sm opacity-70">
<code>aggregation_dimensions</code> adds a fleet-wide series per ASG, so
your alarm doesn't break when the ASG replaces an instance (a new
<code>InstanceId</code> is a new metric).
</div>

---
---

# CloudWatch Logs: the model

| Level | What | Example |
|---|---|---|
| **Log group** | One kind of log; settings live here (retention, encryption, filters) | `/lab12/nginx/access` |
| **Log stream** | One source writing into the group | `i-0abc123…` (one per instance) |
| **Log event** | One line, with a timestamp | `{"status":500,"path":"/boom",…}` |

<v-clicks>

- **Retention:** by default a log group keeps data **forever** ("Never
  expire") — and you pay for it every month. Always set it (1 day … 10 years).
- Logs can also be **exported** to S3 or streamed out with **subscription
  filters** (to Lambda, Kinesis, OpenSearch).
- Many services write here directly: Lambda, ECS (`awslogs` driver),
  RDS engine logs, VPC Flow Logs.

</v-clicks>

---
---

# Structured logs

```text
10.11.0.47 - - [16/Nov/2026:10:31:02 +0000] "GET /boom HTTP/1.1" 500 5 "-" "curl/8.5"
```

```json
{"time":"2026-11-16T10:31:02+00:00","request_id":"4f1c9a…","method":"GET",
 "path":"/boom","status":500,"rt":0.002}
```

<v-clicks>

- The first line needs a `parse` pattern to pull out the status code.
  The second is **already fields**: `status`, `path`, `rt`.
- Logs Insights and metric filters understand JSON automatically.
- Log a **request ID** in every line (nginx: `$request_id`; the ALB adds
  an `X-Amzn-Trace-Id` header). Then you can follow one request across
  every service.

</v-clicks>

---
---

# CloudWatch Logs Insights

```text {1-4|6|8|10}
fields @timestamp, status, path, request_id
| filter status >= 500
| sort @timestamp desc
| limit 20

filter status >= 500 | stats count(*) as errors by bin(1m)

stats count(*) as requests by status

stats pct(rt, 95) as p95_seconds by bin(5m)

fields @timestamp, @message | filter request_id = "4f1c9a…"
```

<v-clicks>

- A query language over one or more log groups: `fields`, `filter`,
  `stats`, `sort`, `limit`, `parse`.
- You pay per **GB scanned**: pick a short time range and the right log
  group.

</v-clicks>

---
---

# Metric filters: from log lines to a metric

```text
Log group:   /lab12/nginx/access
Pattern:     { $.status >= 500 }
Metric:      Lab12 / Nginx5xx     value 1 per matching line
```

<v-clicks>

- A **metric filter** watches new log events and turns matches into a
  CloudWatch metric — which you can graph and alarm on.
- Patterns: plain words (`ERROR`), JSON (`{ $.status >= 500 }`), or
  space-delimited fields (`[ip, id, user, ts, req, status=5*, size]`).
- Only counts events from the moment it exists — it doesn't go back in time.
- Alternative: write metrics **inside** the log line with the Embedded
  Metric Format (EMF); CloudWatch extracts them.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your app writes 2 GB of logs a day into a log group with the default
retention. What happens to the bill over a year, and what do you change?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Ingestion is about $1/day (~$0.50/GB), and storage grows every month
because nothing is ever deleted. Set retention (7–30 days), log at INFO
not DEBUG, and don't log every health check.
</div>

---
layout: section
transition: slide-left
---

# Block 4
## Alarms, notifications and scaling

---
---

# Anatomy of an alarm

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**What it watches**

- one metric (or a math expression)
- a statistic and a period: `Sum` over `60 s`
- a comparison and threshold: `> 5`
- how many periods: `evaluation_periods`

</div>
<div>

**Its three states**

| State | Meaning |
|---|---|
| `OK` | Within the threshold |
| `ALARM` | Breaching |
| `INSUFFICIENT_DATA` | Not enough data to decide |

</div>
</div>

<v-clicks>

- **Actions** run on a **state change**: `alarm_actions` (→ ALARM),
  `ok_actions` (→ OK), `insufficient_data_actions`.
- An alarm doesn't fire again every minute while it stays in ALARM —
  only when the state changes.

</v-clicks>

---
---

# M out of N: evaluation periods

```text
period = 60 s   threshold = 70 %   evaluation_periods (N) = 3   datapoints_to_alarm (M) = 2

minute:      1     2     3     4     5
CPU %:      45    82    65    91    88
breaching:   ·     X     ·     X     X

window 1–3:  1 of 3 breaching  → OK
window 2–4:  2 of 3 breaching  → ALARM   (notification sent once)
window 3–5:  2 of 3 breaching  → stays ALARM
```

<v-clicks>

- **N** (`evaluation_periods`): how many recent periods to look at.
- **M** (`datapoints_to_alarm`): how many of them must breach.
- 1 of 1 is fast but noisy; 2 of 3 ignores one-minute spikes; 5 of 5
  is calm but slow. Time to alarm ≈ M × period + a minute of delay.

</v-clicks>

---
---

# When there is no data

| `treat_missing_data` | Missing datapoint counts as | Use it for |
|---|---|---|
| `missing` (default) | Ignored; with no data at all → `INSUFFICIENT_DATA` | Metrics that should always exist |
| `notBreaching` | Fine | Error counts that are absent when zero (`HTTPCode_*`) |
| `breaching` | A breach | Heartbeats: "no data" means it's dead |
| `ignore` | Keep the current state | Rare, irregular metrics |

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
A 5xx alarm with the default <code>missing</code> sits in
<code>INSUFFICIENT_DATA</code> all night when there are no errors — grey on
the dashboard, and no <code>ok_actions</code> after a real incident ends.
Use <code>notBreaching</code>.
</div>

---
---

# Notifications with SNS

```text
CloudWatch alarm ──(state change)──▶ SNS topic "lab12-alerts" ──▶ email: you@narxoz.kz
                                                              ├─▶ HTTPS webhook (Slack, Telegram bot)
                                                              ├─▶ SQS queue
                                                              └─▶ Lambda function
```

<v-clicks>

- **SNS** (Simple Notification Service): publish once to a **topic**,
  every **subscription** receives it.
- An **email** subscription stays *PendingConfirmation* until someone
  clicks the link in the confirmation email. Until then, nothing arrives.
- One topic per audience ("ops", "db team") — alarms just point at the
  topic ARN.
- Test the whole chain without breaking anything:
  `aws cloudwatch set-alarm-state --state-value ALARM …`

</v-clicks>

---
---

# Smarter alarms

<v-clicks>

- **Composite alarm:** `ALARM` only when a rule over other alarms is true,
  e.g. `ALARM(5xx) AND ALARM(unhealthy-targets)` — one page instead of
  five during an outage.
- **Anomaly detection:** CloudWatch learns the metric's normal daily and
  weekly pattern and alarms outside the band — for traffic that's busy at
  noon and quiet at night.
- **Metric math:** alarm on an expression, e.g. error **rate**
  `100 * errors / requests` instead of a raw count.
- **Other actions:** reboot, stop or recover an EC2 instance; run an
  Auto Scaling policy.

</v-clicks>

---
---

# Alarms that scale the fleet

<div class="text-sm">

| | Target tracking | Step scaling |
|---|---|---|
| You set | A target: "keep average CPU at 50%" | Your own alarm + steps: "+1 at 70%, +2 at 90%" |
| Alarms | **Created and managed by AWS** (`TargetTracking-…-AlarmHigh/Low`) | You create them and point `alarm_actions` at the policy |
| Scale in | Automatic, careful | You write the low-threshold alarm too |
| Use when | Almost always (week 9) | Uneven reactions, custom metrics |

</div>

<v-clicks>

- Don't edit or delete the `TargetTracking-…` alarms — AWS recreates or
  breaks them. Change the policy instead.
- Your **CPU alarm** and your **scaling target** are different jobs:
  scaling at 50% handles normal load; an alarm at 70–80% tells a human
  that scaling isn't keeping up (e.g. the ASG is at max).

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
A 5xx alarm has period 60 s, 1 of 1 datapoints, threshold 5. A bad
deploy causes 3 errors per minute for an hour. Does it fire? What
would you change?
</div>

<div v-click class="mt-8 text-lg opacity-70">
No — 3 is never above 5, so a steady trickle of errors is invisible.
Add an alarm on the error <b>rate</b> (metric math), or a lower threshold
over a longer window (e.g. Sum over 15 min).
</div>

---
layout: section
transition: slide-left
---

# Block 5
## Dashboards, EventBridge and cost

---
---

# Dashboards

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

```text
┌──────────────────────┬──────────────────────┐
│ ALB requests + 5xx   │ p95 response time    │
│ (Sum, 1 min)         │ (p95, 1 min)         │
├──────────────────────┼──────────────────────┤
│ Fleet CPU + memory   │ Latest 5xx log lines │
│ (Average, by ASG)    │ (Logs Insights)      │
├──────────────────────┴──────────────────────┤
│ Alarm status: cpu · 5xx · unhealthy         │
└─────────────────────────────────────────────┘
```

</div>
<div>

<v-clicks>

- Widgets: line / number / gauge graphs, **Logs Insights** tables,
  **alarm status**, text with runbook links.
- Users' view at the top (golden signals), resources below.
- **Automatic dashboards** per service exist in the console — a good
  starting point.
- A dashboard is JSON, so it belongs in **Terraform** with everything else.

</v-clicks>

</div>
</div>

---
---

# EventBridge: when something *happens*

```json
{
  "source": ["aws.autoscaling"],
  "detail-type": ["EC2 Instance Launch Unsuccessful"],
  "detail": { "AutoScalingGroupName": ["lab12-asg"] }
}
```

<v-clicks>

- Metrics are **numbers over time**; events are **one-off facts**: an
  instance changed state, an ASG launch failed, an alarm changed state.
- An **EventBridge rule** matches events with a pattern like the one
  above and sends them to a **target**: SNS, Lambda, SQS, Step Functions.
- Rules can also run on a **schedule** (`cron(0 2 * * ? *)`) — e.g. a
  nightly backup script.

</v-clicks>

---
---

# What monitoring costs (us-east-1)

<div class="text-sm">

| Item | Price | Free tier (always free) |
|---|---|---|
| Custom metric (incl. agent, detailed EC2) | $0.30 / metric / month | 10 metrics |
| Detailed EC2 monitoring | ~7 metrics → ≈ $2.10 / instance / month | — |
| Alarm (standard, per metric) | $0.10 / month | 10 alarms |
| Dashboard | $3 / month | 3 dashboards |
| Logs ingestion | $0.50 / GB | 5 GB logs in total |
| Logs storage | $0.03 / GB-month (compressed) | (same 5 GB) |
| Logs Insights | $0.005 / GB scanned | (same 5 GB) |

</div>

<div v-click class="mt-4 text-sm opacity-70">
Prices change — check the CloudWatch pricing page (linked in the final
project's free tier guide). The <b>logs</b> line is usually the one that
surprises people.
</div>

---
---

# Keeping the monitoring bill small

<v-clicks>

- **Retention** on every log group (`retention_in_days`), short in labs.
- **Log level** INFO in production; no request bodies; health checks
  sampled or not logged.
- **Dimensions** with few values: per-ASG, per-endpoint. Never per-user
  or per-request ID — every value is a new billed metric.
- **Aggregate** in the agent (`aggregation_dimensions`) instead of
  alarming on every instance.
- Logs Insights: short **time range**, the right **log group**.
- Delete test dashboards and alarms — or let `terraform destroy` do it.

</v-clicks>

---
---

# CloudWatch in AWS Academy Learner Lab

<v-clicks>

- CloudWatch metrics, alarms, dashboards and Logs work in the lab
  (region **`us-east-1`**). Check the Learner Lab Readme for the current
  list of services, including **SNS** and **EventBridge**.
- You **can't create IAM roles**: instances use `LabInstanceProfile`, so
  the agent has whatever **LabRole** allows. If memory metrics or logs
  never appear, check the agent's log and record it.
- SNS **email** needs you to confirm the subscription; if email is
  blocked, use the alarm **history** as your evidence.
- **Budgets** aren't available in the lab — watch the lab's spend
  counter instead. Detailed monitoring and dashboards use your credits.

</v-clicks>

---
layout: section
transition: slide-left
---

# Block 6
## Monitoring as code

---
---

# Why alarms belong in Terraform

<v-clicks>

- **Same alarms everywhere:** staging and prod get identical alarms with
  different thresholds (`terraform.tfvars`).
- **Reviewed:** a teammate sees "threshold 70 → 95" in the pull request
  before it silently stops paging anyone.
- **Wired by reference:** the alarm uses `aws_lb.web.arn_suffix` — no
  copying IDs from the console, no alarm pointing at a deleted ALB.
- **Cleaned up:** `terraform destroy` removes alarms, dashboards and log
  groups together with the fleet.

</v-clicks>

---
---

# An alarm in Terraform

```hcl {1-3|5-12|13-17}
resource "aws_sns_topic" "alerts" {
  name = "${var.prefix}-alerts"
}

resource "aws_cloudwatch_metric_alarm" "alb_5xx" {
  alarm_name          = "${var.prefix}-5xx"
  namespace           = "AWS/ApplicationELB"
  metric_name         = "HTTPCode_Target_5XX_Count"
  dimensions          = { LoadBalancer = aws_lb.web.arn_suffix }
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 1
  comparison_operator = "GreaterThanThreshold"
  threshold           = var.alert_5xx_per_min
  treat_missing_data  = "notBreaching"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]
}
```

---
---

# Log group and metric filter in Terraform

```hcl {1-4|6-16}
resource "aws_cloudwatch_log_group" "nginx" {
  name              = "/${var.prefix}/nginx/access"
  retention_in_days = 7
}

resource "aws_cloudwatch_log_metric_filter" "nginx_5xx" {
  name           = "${var.prefix}-nginx-5xx"
  log_group_name = aws_cloudwatch_log_group.nginx.name
  pattern        = "{ $.status >= 500 }"

  metric_transformation {
    namespace = "Lab12"
    name      = "Nginx5xx"
    value     = "1"
  }
}
```

<div v-click class="mt-4 text-sm opacity-70">
Create the log group <b>in Terraform</b>, not by the agent: then the
retention is in code, and <code>destroy</code> deletes it.
</div>

---
---

# A dashboard in Terraform

```hcl {1-3|4-15}
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "${var.prefix}-dashboard"
  dashboard_body = jsonencode({
    widgets = [{
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
    }]
  })
}
```

<div v-click class="mt-2 text-sm opacity-70">
Tip: build a widget in the console, then <b>Actions → View/edit source</b>
and copy the JSON into <code>jsonencode</code>.
</div>

---
---

# Your final project

| Variant field | Where it goes |
|---|---|
| `alert_cpu_pct` (60 / 70 / 80) | CPU alarm threshold — with a scaling target **below** it |
| `alert_5xx_per_min` (5 / 10 / 20) | 5xx alarm: `Sum` over 60 s `>` your value |
| `backup_retention_days` (7 / 14 / 30) | Snapshot schedule and backup lifecycle; pick log retention on purpose too |

<v-clicks>

- **≥ 4 alerts** (CPU and 5xx from your variant + two of: unhealthy
  targets, p95 latency, replication lag, disk) — show one **firing and
  resolving**.
- **Dashboards** (LB, app RED, nodes/pods, DB), **centralized logs**, and
  a **saved query** that follows one `request_id`.
- Your app runs on Kubernetes: CloudWatch agent / Fluent Bit **or**
  Prometheus + Grafana + Loki in the cluster — today's Track B.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You change <code>alert_cpu_pct</code> from 70 to 80 in
<code>terraform.tfvars</code> and run <code>terraform plan</code>. What do
you expect to see?
</div>

<div v-click class="mt-8 text-lg opacity-70">
<code>Plan: 0 to add, 1 to change, 0 to destroy</code> — the CPU alarm is
updated in place (<code>threshold = 70 -> 80</code>). Nothing else
references that variable.
</div>

---
layout: section
transition: slide-left
---

# Block 7
## Today's practice

---
---

# Today's practice — see it, query it, get paged

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Track A — Learner Lab**

1. Your week 11 fleet + **monitoring as code**
2. Agent: memory, disk and JSON nginx logs
3. SNS topic with your email; alarms: CPU, **5xx** (you write it), unhealthy targets
4. Logs Insights query; a metric filter
5. Break `/boom` → alarm → email → OK
6. Dashboard, then `terraform destroy`

</div>
<div>

**Track B — no AWS**

The same ideas with **Prometheus, Grafana, Loki and Alertmanager** in
Docker Compose: metrics, a log query, an alert you write that fires and
resolves, and a dashboard.

<div v-click class="mt-4 opacity-70">
Checked orally: show your alarm or alert rule, a query result, the
notification for ALARM and OK, the dashboard, and the cleanup.
</div>

</div>
</div>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Explain metrics, logs and traces, and pick signals with golden signals / RED / USE
- [ ] Read a CloudWatch metric: namespace, dimensions, statistic, period
- [ ] Say what the CloudWatch agent adds and how logs reach CloudWatch Logs
- [ ] Query logs with Logs Insights and turn log lines into a metric
- [ ] Design an alarm: M of N, missing data, SNS actions
- [ ] Explain target tracking vs step scaling and what monitoring costs
- [ ] Define alarms, log groups and dashboards in Terraform

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Finish today's practice and run **`terraform destroy`** (Track A),
      then **End Lab** — or `docker compose down -v` (Track B)
- [ ] Write down your final project variant's `alert_cpu_pct`,
      `alert_5xx_per_min` and `backup_retention_days`, and which four
      alerts you'll build
- [ ] Think about this: every change so far was `terraform apply` from
      **your laptop**. What could go wrong when five people do that?

<div class="mt-8 text-sm opacity-60">
Next week: a pipeline that tests, builds and deploys on every push.
</div>

---
layout: end
---

# Next lecture

CI/CD and Automated Deployment — from a push to a running
deployment, without touching the console.
