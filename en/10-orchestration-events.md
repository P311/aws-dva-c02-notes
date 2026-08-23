# Module 10: Orchestration & Events (Step Functions / EventBridge)

> **Covered**: May 10
>
> 中文版本：[`zh/10-orchestration-events.md`](../zh/10-orchestration-events.md)

---

## 1. What Step Functions Is ⭐⭐⭐

**= AWS's fully managed workflow orchestration service.** Core value: orchestrating calls across multiple AWS services (Lambda, ECS, DynamoDB, SNS, SQS, etc.) into an ordered, visualized, fault-tolerant workflow.

Without Step Functions: Lambda A calls Lambda B calls Lambda C — error handling scattered everywhere, state management a mess. With Step Functions: the flow is defined as a **state machine**, and AWS manages state, errors, and retries for you.

**Key concepts**: a State Machine (the workflow definition, in JSON/ASL); an Execution (one run of the state machine); a State (one step in the workflow); ASL (Amazon States Language, the JSON DSL describing the state machine).

### Standard vs. Express Workflows ⭐⭐⭐

**Must be chosen when creating the state machine — cannot be changed afterward!**

| Dimension | **Standard** | **Express** |
|---|---|---|
| **Max execution duration** | **1 year** | **5 minutes** |
| **Execution semantics** | **Exactly-once** (unless explicit retries) | **At-least-once** (duplicates possible) |
| **Throughput** | Lower (thousands per second) | **Much higher** (hundreds of thousands per second) |
| **Execution history** | Queryable for 90 days | **Not retained** (CloudWatch Logs only) |
| **Visual history** | ✅ Every execution viewable in the console | ❌ (view via CloudWatch Logs) |
| **Billing** | By state transitions | By requests + duration + memory |
| **Cost at high throughput** | Notably more expensive | Cheaper |
| **Supports `.waitForTaskToken`** | ✅ (can pause up to 1 year) | ❌ |
| **Supports `.sync`** | ✅ | ❌ |
| **Sync/async invocation** | Async only | **Both sync and async** (two subtypes) |
| **Typical use** | Long workflows, order fulfillment, ETL, human approval | High-frequency events, IoT ingestion, API backends, stream processing |

**Two Express subtypes**: Asynchronous Express (`StartExecution`, returns a confirmation); Synchronous Express (`StartSyncExecution`, waits for the result — up to 5 minutes — fitting synchronous call scenarios like API Gateway/Lambda).

**Choosing**: pick Standard for long workflows (> 5 min), exactly-once requirements, detailed execution history, human approval steps, complex business processes. Pick Express for short workflows (< 5 min), high throughput, idempotent business logic, microservice orchestration, or minimizing cost.

**Nesting pattern** (advanced): a Standard workflow invoking an Express workflow — the parent Standard handles "long-running + human approval + detailed history," while the child Express handles "high-frequency idempotent operations," balancing reliability and cost.

### States ⭐⭐⭐

ASL defines 8 state types:

**1. Task**: calls an AWS service or Lambda. Integration patterns: a standard Lambda invocation (async); an optimized Lambda invoke; `.waitForTaskToken` (pause pending a callback); `.sync` (wait synchronously for completion).

**2. Choice**: like if-else, branching based on input data.

**3. Wait**: waits for a duration or until a timestamp. ⚠️ Wait doesn't consume state transitions/time billing in Standard mode — good for "waiting on a downstream system."

**4. Parallel**: multiple branches execute simultaneously; the next state only starts once all complete. Typical use: calling several APIs concurrently for parallel identity verification.

**5. Map** (loop/batch processing): runs the same workflow against every element of an array. Two modes ⭐: **Inline Map** (each iteration runs within the parent execution, sharing execution history, with a limited concurrency ceiling); **Distributed Map** (a newer feature — each iteration is an independent child execution, supporting much higher concurrency, good for processing large volumes of data like one iteration per line of a large S3 file).

**6. Pass** (pass-through/data transformation): does nothing but pass or transform the input — used for reshaping data, debugging, or as a placeholder.

**7. Succeed**: terminates the execution and marks it successful.

**8. Fail**: terminates the execution and marks it failed, with an error/cause.

### Error Handling ⭐⭐⭐

**Retry** (automatic retries): configured on any Task/Parallel state:

```json
{
  "Type": "Task",
  "Resource": "arn:aws:lambda:...",
  "Retry": [
    {
      "ErrorEquals": ["States.Timeout", "Lambda.ServiceException"],
      "IntervalSeconds": 2,
      "MaxAttempts": 3,
      "BackoffRate": 2.0,
      "MaxDelaySeconds": 60,
      "JitterStrategy": "FULL"
    },
    { "ErrorEquals": ["States.ALL"], "MaxAttempts": 1 }
  ]
}
```

Key parameters: `ErrorEquals` (which errors trigger a retry — `States.ALL` matches everything), `IntervalSeconds` (initial wait), `MaxAttempts` (max retry count), `BackoffRate` (exponential backoff multiplier), `MaxDelaySeconds` (cap on any single wait), `JitterStrategy` (adds random jitter to avoid thundering herds).

Predefined errors: `States.ALL`, `States.Timeout`, `States.TaskFailed`, `States.Permissions`, `States.HeartbeatTimeout`, `Lambda.ServiceException`.

**Catch** (catching errors and branching): similar to try-catch — jumps to a specified state once retries are exhausted.

```json
{
  "Type": "Task",
  "Resource": "arn:...",
  "Retry": [...],
  "Catch": [
    { "ErrorEquals": ["Lambda.TooManyRequestsException"], "Next": "ThrottledHandler", "ResultPath": "$.error" },
    { "ErrorEquals": ["States.ALL"], "Next": "GenericErrorHandler" }
  ]
}
```

`ResultPath` determines which field of the input the error info gets placed into. **Order**: Retry happens before Catch — Catch only kicks in once retries are exhausted.

### Service Integrations ⭐⭐

Step Functions integrates with AWS services directly, without a Lambda in between. **Three integration patterns**:

| Pattern | Suffix | Behavior |
|---|---|---|
| **Request Response** (default) | None | Calls the API and returns immediately (doesn't wait for the result) |
| **Run a Job** | `.sync` | Calls the API and waits for the task to complete before advancing |
| **Wait for Callback** | `.waitForTaskToken` | Pauses the execution, waiting for an external `SendTaskSuccess` callback |

A `.sync` example (waiting synchronously for an ECS RunTask): fits ECS/Batch jobs, Glue jobs, SageMaker training, and other long-running tasks.

A `.waitForTaskToken` example (human approval): the workflow pauses (up to 1 year), waiting for an external call to `send-task-success --task-token <token>`. Fits human approval steps and waiting on an external API's async callback.

⚠️ **Express does not support `.sync` or `.waitForTaskToken`!**

Over 200 AWS services can be integrated; frequently tested ones include Lambda, ECS, Batch, Glue, EMR, DynamoDB, S3, SNS, SQS, Kinesis, SageMaker, API Gateway, EventBridge, and even Step Functions itself (nested calls!).

### Other Important Features

**Activity** (legacy worker integration): the workflow pauses while an external worker polls an API for the task (`GetActivityTask` → process → `SendTaskSuccess`). Fits tasks that must run on self-managed EC2/on-prem infrastructure — modern scenarios generally use `.waitForTaskToken` instead, which is more flexible.

**Input/Output Processing**: each state can configure fields to filter data — `InputPath`, `Parameters`, `ResultSelector`, `ResultPath`, `OutputPath`. ⚠️ the DVA rarely tests exact syntax here — knowing these fields control data flow is enough.

**Limits**: max input/output/data payload **256 KB** (larger data should go through S3 + a reference); state machine definition size capped at 1 MB.

---

## 2. What EventBridge Is ⭐⭐⭐

**= AWS's fully managed serverless event bus** (publish/subscribe + routing).

```
        Event Sources                  Rules                Targets
        ┌──────────────┐              ┌─────┐              ┌──────────┐
        │ AWS Service  │              │     │              │ Lambda   │
        │ (S3/EC2/...) │ ──events──→  │ Bus │ ──filter──→  │ SQS      │
        │ Custom App   │              │     │              │ SNS      │
        │ SaaS Partner │              │     │              │ Step Fn  │
        └──────────────┘              └─────┘              │ 30+ more │
                                                           └──────────┘
```

**vs. SNS**: SNS is simple fanout at very high throughput (millions of TPS); EventBridge supports complex routing + filtering + transformation + SaaS integration, but at comparatively lower throughput.

**Predecessor**: CloudWatch Events (EventBridge is CW Events' evolution, API-compatible).

### The Three Event Bus Types ⭐⭐

**1. Default Event Bus**: one automatically exists per account per region; automatically receives events from all AWS services; can't be deleted, and no additional ones can be created. Used for things like an S3 upload triggering Lambda, EC2 state changes, CloudTrail events.

**2. Custom Event Bus**: created by you, for your own application's events, published via `PutEvents`. Fits decoupling between applications and custom event routing.

**3. Partner Event Bus**: receives events from SaaS partners (e.g. Datadog, Auth0, Zendesk, PagerDuty, Salesforce, Shopify, Stripe). Must be bound to a matching partner event source at creation. ⚠️ you cannot attach a resource policy to a partner event bus — only the partner can write to it.

**Cross-account event routing**: attach a resource policy to the target bus allowing the source account, routing Account A's default bus into Account B's custom bus.

### EventBridge Rules ⭐⭐⭐

**Two ways to trigger a rule**:

**1. Event Pattern** (pattern matching):

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": { "name": ["my-bucket"] },
    "object": { "key": [{ "prefix": "uploads/" }] }
  }
}
```

**2. Schedule** (time-based): a cron expression (`cron(0 12 * * ? *)`) or a rate expression (`rate(5 minutes)`).

⚠️ AWS now recommends **EventBridge Scheduler** (a standalone, more capable service) over EventBridge's scheduled rules, though scheduled rules remain usable.

**Pattern match types**: exact value, prefix, suffix, exclusion (anything-but), numeric comparison, existence (exists), IP CIDR, wildcard (a newer addition).

**Input Transformer** (frequently tested) ⭐: modifies event data before it reaches the target, using `InputPathsMap` + `InputTemplate` to adapt the event to the target's expected input format (so the target doesn't need the whole event).

### EventBridge Targets ⭐⭐

Over 30 target types; common ones include: Lambda functions (async trigger), Step Functions (start a workflow), SQS/SNS (forwarding), Kinesis Data Streams/Firehose (streaming archive), ECS tasks, CodePipeline/CodeBuild (triggering CI/CD), API Gateway/API Destinations (calling external HTTP APIs), another EventBridge event bus (cross-account/region forwarding), Systems Manager Run Command.

**API Destinations** ⭐: lets EventBridge call any external HTTP/HTTPS API as a target. Mechanism: create a Connection (configuring auth — Basic/API Key/OAuth) → create an API Destination (pointing at the HTTP endpoint + connection) → a rule routes events to it. Use case: pushing AWS events to Datadog/Slack/PagerDuty/third-party webhooks.

### EventBridge Pipes ⭐⭐

**= a one-to-one, point-to-point integration** (a newer feature).

```
Source ──→ [Filter] ──→ [Enrichment] ──→ Target
(SQS/DDB Stream/Kinesis/MSK)       (Lambda/Step Fn/Kinesis/SQS/SNS/EventBridge bus, etc.)
```

**vs. Event Bus**: an Event Bus is many-to-many (multiple sources, multiple targets); Pipes is strictly **one-to-one** (one source, one target). Pipes' sources are limited to SQS/DDB Streams/Kinesis/MSK/self-managed Kafka/ActiveMQ/RabbitMQ. Pipes has a unique **Enrichment** step (using Lambda/Step Functions/API Destination/API Gateway to fill in additional data).

Typical scenarios: DDB Streams → enrich (look up another table) → Kinesis; SQS → Lambda enrich → Step Functions. Benefit: eliminates glue code (what used to require a Lambda stitching things together can now be pure configuration).

### EventBridge Schema Registry (Brief)

**= automatically discovering/managing event schemas.** EventBridge auto-discovers schemas from events passing through and generates SDK code bindings (Java, Python, TypeScript, etc.). Two schema sources: AWS schemas (pre-built, for all AWS service events) and custom schemas. Not heavily tested on the DVA.

### EventBridge vs. SNS vs. SQS vs. Kinesis ⭐⭐⭐

|  | **EventBridge** | **SNS** | **SQS** | **Kinesis** |
|---|---|---|---|---|
| Model | Event bus + routing | Pub/Sub | Queue | Stream |
| Primary use | **Rule-based routing / AWS events / SaaS integration** | **Large-scale fanout** | **Decouple + buffer** | **Stream processing + replay** |
| Throughput | Lower (adjustable) | Very high | Nearly unlimited | Per shard |
| Targets | 30+ AWS services + HTTP | Multiple endpoint types | Single logical consumer (workers share the work) | Multiple consumers |
| Filtering | **Rich pattern matching** | Message filtering (simple) | ❌ | Consumer-side |
| Replay | ✅ (Event Bus archive) | ❌ | ❌ | ✅ |
| Schema | ✅ Schema Registry | ❌ | ❌ | ❌ |
| SaaS integration | ✅ (native) | ❌ | ❌ | ❌ |

**Judgment calls**: "an AWS service event should trigger Lambda" → EventBridge default bus; "run Lambda on a schedule" → an EventBridge schedule rule or EventBridge Scheduler; "route Datadog alerts into AWS" → EventBridge Partner Event Bus; "complex routing/filtering between microservices" → EventBridge custom bus; "fan one event out to a massive number of subscribers" → SNS (EventBridge's throughput isn't built for that scale); "DDB Streams → Lambda enrich → Step Functions" → EventBridge Pipes.

### Event Replay (Advanced EventBridge)

**= replaying archived historical events back onto an event bus.** Flow: enable Archive on the event bus (specifying a retention period) → matching events archive automatically → when needed, create a replay (specifying a time range + filter pattern) → EventBridge resends the archived events onto the bus, running through the current rules. Use cases: reprocessing historical events after a bug fix; testing a new rule against historical data. ⚠️ replayed events also trigger targets — be careful not to create duplicate downstream effects.
