# Module 7: Lambda (Fundamentals + Performance + Deployment + Monitoring)

> **Covered**: May 4, 5
>
> 中文版本：[`zh/07-lambda.md`](../zh/07-lambda.md)

---

## 1. Lambda Part 1 — Fundamentals, Invocation, Triggers, Layers, ENV

### What Lambda Is

**= AWS's serverless compute service.** You only worry about code; AWS handles everything underneath (servers, OS, scaling, patching). You pay nothing while code isn't running.

**Core traits**: ✅ fully serverless; ✅ pay-per-use (request count + execution duration × memory); ✅ auto-scales (0 to thousands of concurrent executions); ✅ supports many languages: Node.js, Python, Java, Go, Ruby, .NET, custom runtimes; ✅ integrates with 200+ AWS services as trigger sources.

**Good fit**: ✅ short tasks (< 15 min), ✅ event-driven work, ✅ unpredictable/bursty traffic, ✅ intermittent tasks (scheduled reports, cron), ✅ microservice backends.

**Poor fit**: ❌ long-running tasks (> 15 min), ❌ sustained 24/7 heavy load (EC2/Fargate is cheaper), ❌ heavy compute (model training, large-file video transcoding), ❌ long-lived WebSocket connections, ❌ GPU workloads.

### Key Lambda Limits ⭐⭐⭐

| Item | Limit |
|---|---|
| **Timeout** | 1 sec – **900 sec (15 min)**, defaults to 3 sec |
| **Memory** | **128 MB – 10,240 MB (10 GB)**, in 1 MB increments |
| **vCPU ratio** | Roughly 1 vCPU at 1,769 MB (proportional allocation) |
| **Ephemeral storage /tmp** | **512 MB – 10,240 MB (10 GB)**, defaults to 512 MB |
| **Synchronous invocation payload** | **6 MB** (request and response) |
| **Asynchronous invocation payload** | Recently raised to **1 MB** from an earlier 256 KB ⭐ recent change |
| **Total environment variable size** | **4 KB** |
| **Deployment package (direct zip upload)** | **50 MB** (zipped) |
| **Deployment package (via S3)** | **250 MB** (unzipped, including layers) |
| **Deployment package (container image)** | **10 GB** |
| **Layers per function** | **5** |
| **Total unzipped size** | function + all layers ≤ **250 MB** |
| **Concurrent executions (default)** | **1,000 per region per account** (shared) |

**Mnemonic**: "15 / 10 / 6 / 1 / 4" = 15 minutes / 10GB / 6MB sync / 1MB async / 4KB env.

⚠️ **Note**: the async payload increase is a recent change — exam question banks may still reflect the old 256 KB figure; if 1 MB isn't among the answer choices, go with 256 KB. Verify all these numbers against current AWS docs.

### Execution Model ⭐⭐⭐

**Three invocation modes**:

```
1. Synchronous (RequestResponse) — caller waits for a response; caller handles retries
2. Asynchronous (Event) — Lambda queues the event and returns success immediately; Lambda retries automatically
3. Event Source Mapping (polling) — the Lambda service actively polls a stream/queue and processes in batches
```

**1. Synchronous — InvocationType=RequestResponse**

Typical sources: ✅ API Gateway, ✅ Application Load Balancer, ✅ CloudFront (Lambda@Edge), ✅ Cognito triggers, ✅ direct CLI/SDK invoke, ✅ Lex/Alexa/Kinesis Firehose transforms, ✅ Step Functions (synchronous by default).

Traits: payload capped at **6 MB**; error handling is **the caller's responsibility** (Lambda doesn't retry automatically); the caller sees the full response.

**2. Asynchronous — InvocationType=Event**

Typical sources: ✅ S3, ✅ SNS, ✅ EventBridge, ✅ CloudWatch Events/Logs, ✅ SES, ✅ CodeCommit.

Traits: payload capped at **1 MB** (previously 256 KB); **Lambda retries automatically** — 2 retries by default, 3 executions total (waits 1 minute after the first failure, 2 minutes after the second); `MaximumRetryAttempts` is configurable (0-2); `MaximumEventAgeInSeconds` is configurable (60 sec to 6 hours); once retries are exhausted, the event goes to a **DLQ** or an **on-failure Destination**; the caller only ever sees "queued successfully."

**3. Event Source Mapping (Polling)**

Typical sources (all streams/queues): ✅ SQS (standard + FIFO), ✅ Kinesis Data Streams, ✅ DynamoDB Streams, ✅ Amazon MQ, ✅ self-managed Kafka / Amazon MSK.

Traits: the Lambda service polls internally — this doesn't consume your own concurrency; **batch processing** (configured via `BatchSize`); a batch window (waiting N seconds to accumulate a batch); at-least-once delivery (duplicates possible); error handling can retry, drop, or route the whole batch to a DLQ. SQS specifics: a failed message isn't deleted — it becomes visible again once the visibility timeout expires.

**Quick reference**:

|  | Sync | Async | Event Source Mapping |
|---|---|---|---|
| Invocation | Waits for a response | Returns immediately after queuing | Lambda pulls the data |
| Payload | 6 MB | 1 MB | Depends on the service |
| Retries | **Caller's job** | **Automatic (Lambda)** | **Batch retry (Lambda)** |
| Typical sources | API GW / ALB | S3 / SNS / EventBridge | SQS / Kinesis / DDB Streams |

**Judgment calls**: "S3 upload triggers Lambda — what happens on failure?" → S3 is async, Lambda retries twice, then DLQ/Destination; "API Gateway calling Lambda fails" → synchronous, the client retries; "process messages from SQS" → Event Source Mapping (not a Trigger).

### Triggers

**Two mechanisms**: **Push model** (S3, SNS, API Gateway, ALB, Cognito, CloudFront, etc. — the trigger is configured on the source service, which automatically adds a resource-based policy to Lambda); **Pull model = Event Source Mapping** (SQS, Kinesis, DynamoDB Streams, Kafka, MSK — the mapping is configured on Lambda itself, and Lambda's execution role needs matching permissions).

### IAM Integration ⭐

**Execution Role**: the role Lambda assumes at runtime, determining which AWS services it can call.

Trust policy (who can assume it):

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "lambda.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

Common managed policies: `AWSLambdaBasicExecutionRole` (write CloudWatch Logs), `AWSLambdaVPCAccessExecutionRole` (VPC access), `AWSLambdaSQSQueueExecutionRole`, `AWSLambdaDynamoDBExecutionRole`, `AWSLambdaKinesisExecutionRole`, `AWSXRayDaemonWriteAccess`.

Best practices: ❌ don't grant `:*`; ✅ use AWS managed policies plus a minimal custom inline policy; ✅ give each Lambda its own role.

**Resource-based Policy**: controls who can invoke this function.

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "s3.amazonaws.com" },
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:...",
  "Condition": {
    "ArnLike": { "AWS:SourceArn": "arn:aws:s3:::my-bucket" },
    "StringEquals": { "AWS:SourceAccount": "111122223333" }
  }
}
```

Used for: cross-account Lambda invocation; AWS services calling Lambda (added automatically when creating a trigger from the console). **Trap**: always add SourceArn/SourceAccount conditions to guard against a confused deputy.

### Environment Variables

**Key limits**: total size capped at **4 KB**; keys can only use letters/numbers/underscore, and must start with a letter.

**Encryption by default**: every environment variable is always KMS-encrypted at rest, using an AWS-managed key (`aws/lambda`, free) by default — a customer-managed key can be substituted.

**Note**: ⚠️ environment variables display in plaintext in the console by default (only at-rest encryption applies); for in-transit encryption, enable "Encryption helpers"; for highly sensitive data (passwords, API keys), use Secrets Manager or SSM Parameter Store instead of raw env vars.

**Reserved environment variables** (set automatically by Lambda — you can't override them): `AWS_REGION`, `AWS_LAMBDA_FUNCTION_NAME`, `AWS_LAMBDA_FUNCTION_VERSION`, `AWS_LAMBDA_FUNCTION_MEMORY_SIZE`, `AWS_EXECUTION_ENV`, `_HANDLER`, `_X_AMZN_TRACE_ID`, `LAMBDA_TASK_ROOT`, `LAMBDA_RUNTIME_DIR`, `TZ`.

### Lambda Layers ⭐⭐

**= packaging shared code/dependencies into a reusable layer.** Solves: every Lambda bundling the same dependencies (bloated, slow-to-upload zips); multiple Lambdas sharing a utility library requiring every function to be updated on a change.

**Limits**: up to **5 layers** per function; function + all layers unzipped **≤ 250 MB**; a layer's zipped upload caps at 50 MB.

**Directory layout**: layers unpack into `/opt` — Node.js: `/opt/nodejs/node_modules`; Python: `/opt/python`; Java: `/opt/java/lib`; Ruby: `/opt/ruby/gems/...`; all runtimes: `/opt/bin` for executables.

Standard flow for a Python layer:

```bash
mkdir -p python
pip install requests -t python/
zip -r layer.zip python/
aws lambda publish-layer-version --layer-name my-layer --zip-file fileb://layer.zip
```

**Versioning**: layers are **immutable** once published — a change means publishing a new version; Lambda references a specific version (`my-layer:3`); deleting a layer version doesn't affect functions already referencing it.

**Sharing across accounts**: `aws lambda add-layer-version-permission --layer-name my-layer --version-number 3 --principal 111122223333 --action lambda:GetLayerVersion`.

---

## 2. Versions and Aliases ⭐⭐⭐

### Versions

**= an immutable snapshot of a function's code + configuration.** `$LATEST` is the mutable draft; a published version (v1, v2, ...) is immutable — code, env vars, memory, timeout, and layer config are all frozen. A published version gets a version-specific ARN: `arn:aws:lambda:us-east-1:123:function:my-fn:3`. Version numbers only increment, never reused.

### Aliases

**= a named pointer to a specific version.** Common pattern: `prod → version 5`, `staging → version 6`, `dev → $LATEST`. Aliases are mutable (can be repointed); they have their own ARN; API Gateway/event sources should point at an **alias**, not directly at a version; deploying a new version just means updating the alias pointer for an instant cutover.

### Weighted Aliases (Canary / Blue-Green)

**= a single alias pointing at two versions simultaneously, split by weight.**

Limits: points at a max of **2 versions**; both must already be **published** (not `$LATEST`); both must share the same execution role and DLQ configuration.

Typical flow: deploy new code → publish version 6 → configure the alias `90% → v5, 10% → v6` → monitor v6's metrics/error rate → gradually ramp to 100% if healthy → immediately roll back to 100% v5 if not.

CodeDeploy's predefined deployment configs: `Canary10Percent5Minutes`, `Canary10Percent10Minutes`, `Linear10PercentEvery1Minute`, `Linear10PercentEvery2/3/10Minutes`, `AllAtOnce`.

---

## 3. Concurrency ⭐⭐⭐

**Default account-level limit**: 1,000 concurrent executions per region per account (shared across all functions); a quota increase can be requested.

**Sizing formula**: required concurrency ≈ requests/second × average execution duration (seconds).

**Three types**:

**1. Unreserved** (default, shared pool): every function without reserved concurrency shares whatever's left after reservations; AWS always keeps at least 100 for the unreserved pool.

**2. Reserved Concurrency**: **both a ceiling and a floor** — the function always has N units available (unusable by other functions), and it can never exceed N (going over throttles it); free of charge; used for protecting critical functions, capping connections to a downstream system, or setting it to **0** to fully disable a function.

**3. Provisioned Concurrency**: pre-initializes N execution environments in a "hyper-ready" state; **eliminates cold starts** (double-digit millisecond response times); billed by provisioned duration regardless of usage; must be configured on an alias or a specific version (not `$LATEST`); good fit for latency-sensitive production traffic and predictable traffic spikes.

**Reserved vs Provisioned**:

|  | Reserved | Provisioned |
|---|---|---|
| Ceiling/floor | Both | Just a "warm" count |
| Cold starts | Still happens (not pre-warmed) | **None** (pre-warmed) |
| Cost | Free | Billed by provisioned duration |
| Use case | Throttling/reserving capacity | Performance/low latency |
| Configured at | Function level | Alias/version level |

**Judgment calls**: "API Gateway calls Lambda and the first request is slow" → **Provisioned Concurrency**; "worried this critical Lambda gets crowded out by a traffic surge" → **Reserved Concurrency**; "Lambda overwhelms RDS with connections" → Reserved Concurrency to throttle it (or add RDS Proxy); "need to urgently disable a function" → set Reserved Concurrency to **0**.

**Throttling**: triggered by exceeding the account-level concurrency cap or a function's reserved/provisioned concurrency. Synchronous calls return `429 TooManyRequestsException`; asynchronous calls are retried automatically by Lambda; Event Source Mapping slows down its polling rate.

---

## 4. Lambda Destinations ⭐

**= routing the outcome of an async invocation (success or failure) to another service.** Four destination types: SQS queue, SNS topic, another Lambda function (chaining!), EventBridge event bus. Configured via `OnSuccess` / `OnFailure`.

**vs DLQ**:

|  | DLQ | Destinations |
|---|---|---|
| Triggered by | Failure only, after retries exhaust | Both success **and** failure |
| Supported targets | SQS / SNS | SQS / SNS / Lambda / EventBridge |
| Metadata | Minimal | Full (request/response/error detail) |
| Status | Legacy | **Recommended** |

⚠️ Destinations only apply to asynchronous invocations and (partially) Event Source Mapping — synchronous calls have no Destination.

---

## 5. Lambda + VPC ⭐

**Default behavior**: Lambda runs in an AWS-managed VPC by default, giving it public internet access, but **no** access to resources in your own VPC (like a private RDS instance).

**Configuring Lambda inside your VPC**: the Lambda service provisions a **Hyperplane ENI** in your VPC (a shared ENI pool, with cold starts already optimized); the ENI draws from your VPC's IP address pool.

**Traps**: ⚠️ a VPC-attached Lambda has no public internet access by default — a NAT Gateway (in a public subnet) is required; ⚠️ the execution role must have `AWSLambdaVPCAccessExecutionRole`; ⚠️ don't put Lambda in a public subnet — it has no public IP, so it's pointless.

**Connection to IAM**: for reaching S3 etc., prefer a VPC Endpoint (stays on the AWS network, avoids NAT charges).

---

## 6. Cold Start

**Cold start phases**: ① Init phase (downloading code/container image, starting the runtime, running init code); ② Invoke phase (actually calling the handler).

**Typical cold-start latency (rough order of magnitude)**: Python ~100-500 ms; Node.js ~100-300 ms; Java/.NET noticeably slower (seconds — JVM/CLR startup is heavy).

**Reducing cold starts**: ① Provisioned Concurrency (pre-warming); ② SnapStart (Java/Python/.NET); ③ smaller deployment packages; ④ avoiding heavy dependencies; ⑤ lazy initialization; ⑥ keeping functions warm (not an officially recommended technique); ⑦ choosing a lighter runtime.

**Init code vs handler code**:

```python
# init code — runs once per container reuse
import boto3
db_client = boto3.client('dynamodb')   # global, created once

def lambda_handler(event, context):    # runs on every request
    response = db_client.get_item(...)
    return response
```

Best practices: ✅ initialize the SDK client globally; ✅ cache config/static data globally; ❌ never `import` or create a client inside the handler.

---

## 7. Lambda Monitoring (Basics)

**CloudWatch Metrics** (automatic, free): `Invocations`, `Errors`, `Duration`, `Throttles`, `ConcurrentExecutions`, `ProvisionedConcurrencyUtilization`.

**CloudWatch Logs**: each Lambda gets a log group at `/aws/lambda/<function-name>`; `console.log`/`print` output is written automatically; the execution role needs `logs:CreateLogStream` + `logs:PutLogEvents`.

**X-Ray**: enabling Active Tracing makes Lambda send traces automatically; the execution role needs `AWSXRayDaemonWriteAccess`; lets you see the call waterfall across Lambda → DynamoDB/S3/external APIs.

---

## 8. Lambda Part 2 — Performance Deep Dive, Deployment, Advanced Monitoring

### The Full Cold Start Sequence

```
Steps 1-4 = "Init Duration" (cold-start only, runs once per new container)
Step 5    = "Duration" (happens on every invocation)
```

**Container reuse / warm starts**: after a Lambda finishes executing, the container isn't destroyed immediately — it stays around for a while so the next invocation can reuse it (a warm start, skipping the init phase entirely). Reuse isn't guaranteed (AWS can tear it down anytime), and how long a container stays warm is undocumented.

### SnapStart ⭐⭐

**= snapshotting the initialized execution environment and restoring from it on subsequent invocations, skipping the init phase.** Built on Firecracker MicroVM's snapshot capability.

**Supported**: ✅ Java 11+, ✅ Python 3.12+, ✅ .NET 8+. **Not supported**: ❌ Node.js/Ruby/Go/custom runtimes, ❌ container images.

**Limits**: ❌ mutually exclusive with Provisioned Concurrency; ❌ no EFS support; ❌ no ephemeral storage beyond 512 MB; ❌ only usable on a published version or an alias pointing to one (never `$LATEST`).

**Pricing**: free for Java; Python/.NET incur two charges (caching cost + restoration cost) — delete unused versions to avoid ongoing caching costs.

**The uniqueness trap**: restoring multiple execution environments from the same snapshot means they all start in identical state — if the init code generated a UUID, a random seed, or anything meant to be unique, every restored instance shares the same value! Fix: use runtime hooks (`afterRestore` / `beforeCheckpoint`) to regenerate unique values after restoration.

**SnapStart vs Provisioned Concurrency**:

|  | SnapStart | Provisioned Concurrency |
|---|---|---|
| Mechanism | Skips init, restores from a snapshot | Pre-initializes N environments |
| Cold start | Still technically a cold start, but fast | **No cold start at all** |
| Pricing | Free for Java, paid for others | Paid (by provisioned duration) |
| Runtime support | Java/Python/.NET | All runtimes |
| Container image support | ❌ | ✅ |
| Mutual exclusivity | ✅ Mutually exclusive | ✅ Mutually exclusive |

**Choosing**: Java applications with bursty traffic → SnapStart; genuinely low-latency production apps → Provisioned Concurrency; Node.js/Go or container images → Provisioned Concurrency is the only option.

### Container Image Support ⭐

**= deploying Lambda as a Docker image.** Why: zip deployments cap at 250 MB, and large dependencies (ML models) blow past that easily; container images allow up to **10 GB** — a big jump — and let you define the environment with a Dockerfile, integrating smoothly with existing CI/CD.

**Base images**: AWS provides Lambda base images per language (Python, Node.js, Java, .NET, Ruby, custom runtimes); a custom base image needs to implement the Lambda Runtime API.

**Image vs zip**:

|  | Zip package | Container image |
|---|---|---|
| Size limit | 250 MB (unzipped) | **10 GB** |
| Deployment source | S3 / direct upload | **ECR required** |
| Layer support | ✅ | ❌ (baked into the image) |
| SnapStart support | ✅ (Java/Python/.NET) | ❌ |
| Fits existing CI/CD | Weak | **Strong** (Docker tooling) |

### Function URLs ⭐⭐

**= giving Lambda a direct HTTPS endpoint without API Gateway.** URL format: `https://<url-id>.lambda-url.<region>.on.aws/`.

**Traits**: ✅ HTTPS only; ✅ built-in CORS support; ✅ IAM auth or public access; ✅ can attach to an alias/`$LATEST` (not to a published version); ✅ supports streaming responses; ✅ synchronous invocation (still 6 MB payload cap).

**Two auth types**: **NONE** (public — requires an explicit allow in the resource policy for `Principal: "*"` + `lambda:InvokeFunctionUrl`); **AWS_IAM** (SigV4-signed requests, caller needs `lambda:InvokeFunctionUrl` — fits internal APIs/cross-account use).

**Function URL vs API Gateway**:

|  | Function URL | API Gateway |
|---|---|---|
| Complexity | Minimal | Complex |
| Price | **Free** (just Lambda charges) | Billed per request |
| Rate limiting | ❌ | ✅ |
| Custom domain | ❌ | ✅ |
| WebSocket | ❌ | ✅ |
| Streaming response | ✅ | Partial |

**Choosing**: internal tools, prototypes, simple webhooks → Function URL; production APIs needing complex auth/throttling/routing → API Gateway.

⚠️ **Trap**: even with Function URL CORS configured, the Lambda function's own response headers must also return CORS headers, or the browser will still error out.

### Lambda@Edge ⭐⭐

**= Lambda functions that run at CloudFront edge locations.** Deployment location: must be deployed in **us-east-1**, but executes globally at CloudFront edge locations.

**The four trigger points**: **viewer-request** (every request), **origin-request** (cache misses only), **origin-response** (cache misses only), **viewer-response** (every request).

**Limits differ from standard Lambda**: viewer triggers cap at 5-second timeout and 1 MB function size; origin triggers cap at 30-second timeout and 50 MB function size. General restrictions: ❌ no environment variables, layers, X-Ray, reserved/provisioned concurrency, ARM64, VPC, or DLQ; ❌ **Node.js and Python only**; must use a published version.

**Typical use cases**: A/B testing, JWT/custom authorization, URL rewriting, geo-based content, response header injection (security headers), dynamic image processing, multi-origin routing.

### CloudFront Functions ⭐

**= lightweight JavaScript functions running at CloudFront's edge**, a lighter-weight alternative to Lambda@Edge.

|  | CloudFront Functions | Lambda@Edge |
|---|---|---|
| Language | **JavaScript only** | Node.js + Python |
| Execution locations | **All edge locations** (hundreds) | Fewer regional edge caches |
| Trigger points | **viewer-request / viewer-response** | All 4 |
| Max duration | Sub-millisecond | Seconds |
| Network access | ❌ None | ✅ Yes |
| Pricing | **Very cheap** | More expensive than regular Lambda |

**Rule of thumb**: CloudFront Functions = minimal, extremely fast, extremely cheap (viewer-only triggers, no network access); Lambda@Edge = powerful but slower and pricier (all 4 trigger points, can call external APIs).

### Lambda Insights (Enhanced Monitoring)

**= a CloudWatch extension for enhanced Lambda monitoring.** Extra metrics: CPU utilization, detailed memory utilization, network traffic, disk I/O. Enabled by attaching AWS's `LambdaInsightsExtension` layer, plus adding `CloudWatchLambdaInsightsExecutionRolePolicy` to the execution role. Billed per metric.

### X-Ray ⭐⭐⭐

**= AWS's distributed tracing service.**

**Core concepts**: a Trace (the end-to-end record of one request) → Segments (one service's handling of that request) → Subsegments (finer-grained operations within a segment). A Service Map is a visual topology auto-generated from traces.

**Lambda + X-Ray integration**: **Active Tracing** (Lambda sends traces automatically, even without configuring the X-Ray SDK); **PassThrough** (no active tracing, just forwards the tracing header). Enable via Configuration → Monitoring → Active tracing; the execution role needs `AWSXRayDaemonWriteAccess`.

With Active Tracing on, X-Ray shows 2 segments: `AWS::Lambda` (the Lambda service's own overhead) and `AWS::Lambda::Function` (the function code's actual execution).

**Fine-grained tracing with the X-Ray SDK** (Python example):

```python
from aws_xray_sdk.core import xray_recorder, patch_all

patch_all()  # auto-traces boto3 / requests / sqlite3, etc.

@xray_recorder.capture('my_function')
def process_order(order_id):
    pass

def lambda_handler(event, context):
    subsegment = xray_recorder.begin_subsegment('custom-work')
    subsegment.put_annotation('user_id', '12345')
    subsegment.put_metadata('event', event)
    xray_recorder.end_subsegment()
```

**Annotations vs Metadata** ⭐:

|  | Annotations | Metadata |
|---|---|---|
| Data types | string/number/boolean | Any serializable value |
| Indexed | ✅ Yes (searchable via filter expressions) | ❌ No |
| Purpose | **Searching/filtering traces** | **Storing detailed context** |
| Limits | Up to 50 per trace | No strict limit (single segment doc < 64 KB) |

**A Lambda-specific trap**: ⚠️ the parent segment is managed by the Lambda service itself — you can't add annotations/metadata to it directly; a subsegment must be created first.

**Sampling**: the default rule is 100% for the first request in the first second, then 5% thereafter; custom rules can be defined based on URL, HTTP method, or service name.

> **Note**: AWS's X-Ray SDK/Daemon has officially entered maintenance mode, with OpenTelemetry as the recommended migration path — but the DVA exam question bank still assumes the X-Ray SDK. Check current AWS documentation for the latest status.

### CloudWatch Logs Insights ⭐⭐

**= a SQL-like query language for searching/analyzing logs.**

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 25
```

Key commands: `fields`, `filter`, `sort`, `limit`, `stats` (aggregation), `parse`, `bin(N)` (time bucketing).

Fields Lambda auto-discovers: `@timestamp`, `@message`, `@requestId`, `@duration`, `@billedDuration`, `@maxMemoryUsed`, `@memorySize`, `@type`, `@initDuration`.

Example query (finding the cold-start rate):

```sql
filter @type = "REPORT"
| stats sum(strcontains(@message, "Init Duration")) / count(*) * 100 as coldStartPct,
        avg(@duration)
        by bin(5m)
```

### DLQ vs Destinations, Revisited

|  | DLQ | Destinations |
|---|---|---|
| Trigger | Async failure after retries exhaust | Both success and failure |
| Targets | SQS/SNS | SQS/SNS/Lambda/EventBridge |
| Status | Legacy | **Preferred** |

### Troubleshooting Quick Reference

| Symptom | Likely cause | How to investigate |
|---|---|---|
| First call slow, then fast | Cold start | Check `Init Duration` in logs; enable Provisioned Concurrency / SnapStart |
| Frequent timeouts | Function doesn't finish before the timeout | Extend timeout / optimize code / raise memory |
| Frequent throttling | Exceeding concurrency limits | Check the `Throttles` metric; add reserved concurrency or request a quota increase |
| OOM | Insufficient memory | Check `Max Memory Used` in logs; raise memory size |
| VPC Lambda can't reach the internet | No NAT Gateway configured | Private subnet + NAT Gateway |
| Lambda exhausts RDS connections | A new connection per invocation | RDS Proxy + Reserved Concurrency |
| DynamoDB throttling despite unused capacity | A hot partition | Check CloudWatch's per-partition distribution |
| Async events lost | Retries exhausted, no DLQ configured | Configure a DLQ or Destinations |
| API Gateway → Lambda returns 502 | Internal Lambda error/timeout | Check Lambda logs |
