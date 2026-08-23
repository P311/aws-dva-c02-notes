# Module 18: Monitoring & Observability (X-Ray / CloudWatch / CloudTrail)

> **Covered**: May 23
>
> 中文版本：[`zh/18-monitoring.md`](../zh/18-monitoring.md)

---

# 1. X-Ray — Distributed Tracing ⭐⭐⭐

## What X-Ray Is

**= a distributed application tracing service**, showing how a request moves through microservices. Core value: visualizing service topology (Service Map), finding performance bottlenecks, root-causing errors, seeing the complete request path.

## Core Concepts ⭐⭐⭐

| Concept | Meaning |
|---|---|
| **Trace** | The full end-to-end record of one request as it crosses every service |
| **Segment** | One service's data for handling that request |
| **Subsegment** | A finer breakdown within a service: a DB call, an external HTTP call |
| **Annotations** | Indexed key-value pairs, searchable via filter expressions (up to 50/trace) |
| **Metadata** | Unindexed data of any type, viewable only in trace details |

### Annotations vs. Metadata ⭐⭐⭐

|  | Annotations | Metadata |
|---|---|---|
| Indexed | ✅ Yes | ❌ No |
| Searchable | ✅ (filter expressions) | ❌ |
| Value types | string/number/boolean | Anything (including objects/arrays) |
| Limits | Up to **50** per trace | Size-limited (a single segment caps at 64 KB) |

**Judgment call**: wanting to "search traces by user_id" → Annotation; wanting to "log the full request body without needing to search it" → Metadata.

## Service Map ⭐⭐

**= a "service dependency graph" X-Ray generates automatically**:

```
[Client]──→[API Gateway]──→[Lambda]──→[DynamoDB]
                            │
                            └──→[SNS]──→[Lambda 2]
```

Each node shows average latency, error rate (4xx/5xx), and request count. The natural first stop for debugging.

## Sampling Rules ⭐⭐⭐

X-Ray doesn't trace every single request (for cost and performance reasons) — it samples according to rules.

**Default Sampling Rule**: a reservoir (a fixed count per second) + a rate (a percentage of everything beyond that) — reservoir size 1 request/second (guaranteeing at least 1 traced per second); fixed rate 5% (once the reservoir is used, 5% of the remainder is traced).

**Custom Sampling Rules** example: `/checkout` → reservoir 5/s + rate 100% (trace every critical-flow request); `/health` → reservoir 0/s + rate 0% (never trace health checks).

**Adaptive Sampling** (a newer feature): automatically raises the sample rate during anomalies and lowers it during normal operation, managed by AWS.

## The X-Ray Daemon ⭐⭐

**= a local agent that aggregates trace data and batches it to the X-Ray API.** Listens on UDP port **2000**.

| Environment | Daemon |
|---|---|
| **EC2/on-prem** | Install the X-Ray daemon manually |
| **ECS/Fargate** | Add an X-Ray sidecar container to the task definition |
| **Lambda** | Managed automatically by AWS (enable Active Tracing) |
| **Elastic Beanstalk** | Built into the platform (just enable it) |

**IAM permissions**: requires `xray:PutTraceSegments` + `xray:PutTelemetryRecords`. Managed policy: **`AWSXRayDaemonWriteAccess`** ⭐.

## The X-Ray SDK

Supported languages: Node.js, Python, Java, .NET, Ruby, Go.

Lambda uses environment variables to communicate with the X-Ray daemon: `_X_AMZN_TRACE_ID` (the tracing header — sampling decision, trace ID, parent segment ID); `AWS_XRAY_CONTEXT_MISSING` (behavior when there's no tracing header, defaulting to `LOG_ERROR` in Lambda); `AWS_XRAY_DAEMON_ADDRESS` (the daemon's IP:PORT); `AWS_XRAY_TRACING_NAME`; `AWS_XRAY_DEBUG_MODE`.

Manual instrumentation (Python example):

```python
from aws_xray_sdk.core import xray_recorder

@xray_recorder.capture("process_order")
def process_order(order):
    xray_recorder.put_annotation("order_id", order.id)   # indexed, searchable
    xray_recorder.put_metadata("full_request", order.raw)  # not indexed
    return result
```

## The Tracing Header ⭐

```
X-Amzn-Trace-Id: Root=1-5759e988-...; Parent=53995c3f...; Sampled=1
```

**Root**: the ID for the whole trace (unchanged across every service it touches); **Parent**: the ID of the upstream segment; **Sampled**: whether this request was sampled.

⭐ if API Gateway doesn't forward a trace header, Lambda starts a new trace. For true end-to-end tracing, Active Tracing must be enabled on both API Gateway and Lambda.

## Filter Expressions

```
service("my-service") AND duration > 5
annotation.user_id = "12345"
fault = true
```

⭐ only `annotation.xxx` is searchable — metadata is not.

## AWS Distro for OpenTelemetry (Supplementary)

A more broadly applicable tracing option. ADOT fits when you need to send traces to multiple different tracing backends without re-instrumenting code, need a wide range of community-maintained library instrumentations, or need a fully managed Lambda layer for Java/Python/Node.js. The X-Ray SDK fits when you want a tightly integrated single-vendor solution, or need full integration with X-Ray's centralized sampling rules (Node.js/Python/Ruby/.NET).

---

# 2. CloudWatch ⭐⭐⭐

## Metrics Basics ⭐⭐

**Standard vs. high resolution**: Standard resolution is 60 seconds (default, `StorageResolution=60`); high resolution is 1/5/10/30 seconds (`StorageResolution=1`).

**Metric retention**: 1-minute data retained for **15 days**; 5-minute data for **63 days**; 1-hour data for **15 months (roughly 455 days)**. Older data is deleted automatically.

**EC2 default monitoring** ⭐: Basic monitoring (free) at 5-minute granularity; Detailed monitoring (billed) at 1-minute granularity. ⚠️ **EC2 does not monitor memory or disk by default** — the CloudWatch Agent is required for that.

## Custom Metrics ⭐⭐

```bash
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "OrdersProcessed" \
  --value 42 \
  --dimensions "Environment=prod" \
  --unit Count \
  --storage-resolution 1
```

Three core components: Namespace, Metric Name, Dimensions (up to roughly 30).

## The CloudWatch Agent ⭐

Data EC2 doesn't collect by default, but the Agent does: ✅ memory utilization, ✅ disk used %, ✅ swap, ✅ process metrics, ✅ custom log files → CloudWatch Logs.

## CloudWatch Logs ⭐⭐⭐

**Hierarchy**: Log Group → Log Stream → Log Event.

**Retention** ⭐: defaults to Never Expire (permanent — and billed!) — production log groups must have retention set.

**Subscription Filters** ⭐⭐: streams logs in real time to another service (Kinesis/Lambda/Firehose/OpenSearch). ⚠️ a log group can have at most 2 subscription filters.

**Metric Filters** ⭐: extracts numbers from logs to generate a metric. Pattern syntax: plain text (`"ERROR"`); JSON (`{ $.statusCode = 500 }`).

**CloudWatch Logs Insights** ⭐⭐: a powerful SQL-like log query language:

```sql
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100
```

A more complex example (extracting a field and aggregating):

```sql
fields @timestamp, @message
| parse @message "duration=*ms" as duration
| stats avg(duration), max(duration), pct(duration, 99) by bin(5m)
```

Limits: a single query scans roughly 10 GB max; queries time out after 60 minutes; roughly 20 log groups max per query.

## Embedded Metric Format (EMF) ⭐⭐

**= embedding a metric inside a log entry, which CloudWatch extracts automatically.** The traditional approach requires the application to call `put-metric-data` synchronously (slow); EMF only requires writing a log line (asynchronous) — CloudWatch extracts the metric automatically.

```json
{
  "_aws": {
    "Timestamp": 1700000000000,
    "CloudWatchMetrics": [{
      "Namespace": "MyApp",
      "Dimensions": [["Environment"]],
      "Metrics": [{ "Name": "OrderDuration", "Unit": "Milliseconds", "StorageResolution": 1 }]
    }]
  },
  "Environment": "prod",
  "OrderDuration": 523,
  "OrderId": "abc-123"
}
```

**EMF's advantages**: ✅ asynchronous (just write a log, no blocking); ✅ supports high resolution (1 second); ✅ retains high-cardinality context (e.g. OrderId stays in the log); ✅ a single EMF log supports roughly 100 metrics + 30 dimensions.

## CloudWatch Alarms ⭐⭐⭐

**States**: **OK** (the metric is within threshold); **ALARM** (the threshold has been breached); **INSUFFICIENT_DATA** (not enough data to judge).

**Three key settings**: **Period** (the time granularity used to evaluate the metric, in seconds — e.g. a 1-minute period gives one data point per minute); **Evaluation Period** (how many of the most recent data points to consider); **Datapoints to Alarm** (how many of those data points within the evaluation window need to breach the threshold to trigger ALARM — the breaching points don't need to be consecutive, just within the last N points where N is the Evaluation Period).

**Actions**: **SNS** (notifications — email/SMS/Lambda); **EC2 Auto Recovery** (automatically restarts the instance); **Auto Scaling** (triggers ASG scaling); **Lambda invoke** (a newer feature — calling Lambda directly).

**Composite Alarms** ⭐: a logical combination of multiple alarms, e.g. `(HighCPU AND HighLatency) OR LowDiskSpace`. Reduces alarm noise and expresses complex conditions.

**Anomaly Detection**: machine learning learns a metric's baseline, triggering an alarm when the value falls outside the expected band. Fits scenarios with no fixed threshold.

## CloudWatch Synthetics ⭐

**= "canaries"** — scheduled simulations of user behavior. Types: Heartbeat (a simple HTTP check), API Canary, Broken Link, GUI Workflow (Puppeteer-based), Visual Monitoring. Results: metrics + logs + screenshots + a HAR file; can run as often as roughly once a minute.

⭐ "find out a site is down immediately, without waiting on a user report" → Synthetics.

## Other Tools

**Container Insights**: monitoring for the ECS/EKS container layer (auto-collects metrics at the container, task, pod, and cluster levels). **Application Insights**: automatically monitors an application stack (Java, .NET, SQL Server). **ServiceLens**: a unified X-Ray + CloudWatch view (Service Map + metrics + logs + alarms together).

---

# 3. CloudTrail ⭐⭐

## What CloudTrail Is

**= records "what API calls happened in an AWS account."** Used for auditing/compliance (SOC, PCI), security investigations, and operational troubleshooting.

⭐ CloudTrail is enabled by default in every account (recording the last 90 days of management events to Event History, for free).

## The Four Event Types ⭐⭐⭐

**1. Management Events** (on by default): **"control-plane" operations** (resource configuration). Examples: `iam:CreateRole`, `ec2:RunInstances`, `s3:CreateBucket`. Subcategories: write events (modifying resources), read events (read-only queries). The first management-events trail is free.

**2. Data Events** (must be enabled manually): **"data-plane" operations** (acting on a resource's contents). Examples: `s3:GetObject`, `lambda:Invoke`, `dynamodb:GetItem`. ⚠️ data event volume is huge — off by default.

**3. Insights Events** ⭐: AWS's automatic detection of "unusual API patterns." It learns a baseline automatically and generates an Insight event on anomalies (a sudden spike in deletes, a surge in IAM operations).

**4. Network Activity Events** (a newer feature): analyzes API traffic over VPC endpoints.

## CloudTrail Event History ⭐

**= the built-in 90-day event browser.** ✅ free, enabled by default in every account; ✅ retains 90 days; ❌ management events only; ❌ single account, single region.

## Trails ⭐⭐

Single-region trail (one region); **Multi-region trail** (recommended — every region); **Organization trail** (the entire AWS Organization).

**Where trail logs go**: an S3 bucket (required — the primary store, uploaded in periodic batches); CloudWatch Logs (optional — real-time-ish analysis, seconds); EventBridge (optional — sub-second, for triggering automated responses).

## Log File Integrity Validation ⭐

**= prevents CloudTrail logs from being tampered with.** A digest file (a SHA-256 hash) is generated hourly — mandatory for compliance scenarios: `aws cloudtrail validate-logs --trail-arn ... --start-time ...`.

## CloudTrail Lake

**= a managed "event data lake."** Query trail data with SQL; aggregates events across multiple accounts and regions; immutable, write-once storage; a configurable retention period (e.g. 1 year/7 years).

vs. S3+Athena: Lake is fully managed; S3+Athena is self-managed but more flexible.

## EventBridge Integration ⭐

```
A CloudTrail event (e.g. someone created a root user)
  ↓ an EventBridge rule
A Lambda function (auto-revoking access / notifying the security team)
```

---

# 4. Where the Three Services Fit

| Service | Purpose |
|---|---|
| **Tracing + distributed call chains + service topology** | X-Ray |
| **Numeric monitoring + logs + alerting** | CloudWatch |
| **API call auditing + compliance** | CloudTrail |

**Common combined scenarios**: an application is slow, find the bottleneck service → X-Ray Service Map; Lambda erroring frequently, need automatic notification → CloudWatch Logs filter → SNS/Alarm; a user secretly created a bucket → a CloudTrail event + EventBridge → Lambda; one-stop end-to-end troubleshooting → ServiceLens; end-to-end tracing across API Gateway + Lambda → enable Active Tracing on both; high-frequency custom metric reporting → EMF (asynchronous); container-layer monitoring → Container Insights; simulating users for 24/7 monitoring → a Synthetics canary.
