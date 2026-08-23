# Module 9: Messaging & Streaming (SQS / SNS / Kinesis)

> **Covered**: May 8, 9
>
> 中文版本：[`zh/09-messaging-streaming.md`](../zh/09-messaging-streaming.md)

---

## 1. SQS Fundamentals ⭐⭐⭐

**= AWS's fully managed message queue service** (point-to-point). Producers send messages to a queue; consumers actively poll and process them.

**Key traits**: ✅ fully managed; ✅ near-unlimited scale; ✅ durable storage (replicated across AZs); ✅ decouples producers and consumers (asynchronous); ✅ no cap on the number of messages within the retention window.

### The Two SQS Queue Types ⭐⭐⭐

|  | **Standard Queue** | **FIFO Queue** |
|---|---|---|
| Ordering | Best-effort — **can be out of order** | **Strict FIFO** (per message group) |
| Duplicates | **At-least-once delivery** — duplicates possible | **Exactly-once delivery** (5-minute dedup window) |
| Throughput | Nearly unlimited | Lower unbatched, higher when batched (check current docs for exact figures) |
| Name suffix | Any | **Must end in `.fifo`** |
| Message Group ID | Not needed | **Required** (strict FIFO within a group, groups run in parallel) |
| Deduplication ID | Not needed | **Required** (or enable ContentBasedDeduplication) |

**Choosing**: ✅ Standard — most scenarios, high throughput needed, the business tolerates duplicates/out-of-order delivery (handle it idempotently); ✅ FIFO — strict ordering is required (banking transactions, order state machines), duplicates aren't acceptable.

### Key Universal Limits ⭐⭐⭐

| Item | Value |
|---|---|
| **Message size cap** | **256 KB** (the whole message) |
| **Larger messages** | Use S3 + the Extended Client Library (body in S3, SQS stores a pointer) |
| **Message retention** | 60 sec – **14 days**, **defaults to 4 days** |
| **Visibility Timeout** | 0 sec – **12 hours**, **defaults to 30 seconds** |
| **Delay queue / per-message delay** | 0 sec – **15 minutes** |
| **Max long-polling wait** | **20 seconds** |
| **Batch size** | Up to **10** per send/receive/delete batch |
| **Message attributes** | Up to **10** per message |

**Encryption**: in-transit is always HTTPS; at-rest SSE is optional — **SSE-SQS** (AWS-managed key, free, enabled by default) or **SSE-KMS** (customer-managed key).

### Visibility Timeout ⭐⭐⭐

**= the window during which a received message is hidden from other consumers.**

```
T=0    Consumer A polls → receives message X, which goes "in-flight" and the countdown starts (default 30s)
T=0-30s    Consumer B polls → can't see X (it's locked)
T=30s+ (if Consumer A hasn't deleted it within 30s)
       X becomes visible again → Consumer B can now receive it
```

**Three outcomes**: ① the consumer deletes it within the timeout → permanently removed; ② the consumer doesn't (crash or slow processing) → it becomes visible again → possible duplicate processing → the ReceiveCount climbs toward maxReceiveCount → routed to a DLQ; ③ the consumer proactively calls `ChangeMessageVisibility` to extend the timeout.

**Sizing strategy**: too short (< processing time) → gets reprocessed by other consumers; too long (hours) → after a crash, the message takes a long time to become visible again, raising latency; the ideal is slightly longer than average processing time, with critical tasks sending periodic heartbeats to extend it.

**Judgment call**: "want a failed message to become visible immediately" → call `ChangeMessageVisibility(0)`.

### Polling ⭐⭐⭐

**Short Polling** (default, but not recommended): `WaitTimeSeconds=0`, returns immediately, and can return empty even when messages exist (it only checks a subset of backend servers). Downsides: lots of empty responses, more API calls, higher cost.

**Long Polling** (recommended) ⭐: `WaitTimeSeconds=1-20`, waits until a message arrives or the timeout hits, checking all SQS backend servers (no false empties). Benefits: far fewer API calls, lower latency, almost no empty responses. ⚠️ AWS strongly recommends long polling (20 seconds).

**Trap**: when Lambda is triggered by SQS (Event Source Mapping), the Lambda service uses long polling automatically — no manual config needed. If you're polling manually at the application layer, you need to enable it explicitly.

### Dead Letter Queue (DLQ) ⭐⭐

**= a special queue that "catches" messages that have repeatedly failed processing.**

Redrive policy:

```json
{
  "deadLetterTargetArn": "arn:aws:sqs:...:my-dlq",
  "maxReceiveCount": 5
}
```

If a message has been received 5 times without being successfully processed (never deleted), it's automatically moved to the DLQ (keeping its original message ID).

**Key constraints**: ✅ the DLQ must match the source queue's type (Standard→Standard, FIFO→FIFO); ✅ the DLQ must be in the same account and region; ✅ one DLQ can serve multiple source queues; ⚠️ never set `maxReceiveCount` to 1 (no chance to retry).

**Uses**: diagnosing failures; isolating poison messages; manual retry — using **redrive to source** to batch-send DLQ messages back to the source queue for reprocessing.

### Delay Queue & Per-Message Delay

**Delay Queue** (queue-level): newly enqueued messages are delayed N seconds before becoming visible, 0-15 minutes, good for "delayed notification" scenarios. Vs. Visibility Timeout: a delay queue hides a message *on arrival*; visibility timeout hides it *after a consumer receives it*.

**Per-Message Delay**: set `DelaySeconds` (0-15 min) on `SendMessage` for a per-message delay. ⚠️ FIFO queues don't support per-message delay — only queue-level.

### Lambda + SQS ⭐⭐

The Lambda service actively polls SQS (Event Source Mapping, using your execution role), receives a batch of messages → invokes the Lambda function → deletes on success, or leaves it to reappear after the visibility timeout on failure → routed to a DLQ once maxReceiveCount is hit.

**Key configuration**: `BatchSize` (messages per Lambda invocation), `MaximumBatchingWindow` (0-300 sec), `ReportBatchItemFailures` (partial batch failure support).

**Trap 1**: Visibility Timeout should be ≥ 6 × the Lambda timeout (AWS's recommendation) — otherwise SQS releases the message to another consumer before Lambda finishes, causing duplicate processing.

**Trap 2**: by default, a Standard queue treats one failure as a failure of the entire batch — if 1 of 10 messages fails, all 10 get retried. **Fix**: use `ReportBatchItemFailures` so the Lambda response specifies which items failed, retrying only those:

```json
{
  "batchItemFailures": [
    {"itemIdentifier": "msg-id-3"},
    {"itemIdentifier": "msg-id-7"}
  ]
}
```

**Trap 3**: FIFO + Lambda concurrency — messages within the same message group are processed serially (a single Lambda instance handles that group), while different groups run in parallel; for high-throughput FIFO, use multiple message groups.

### SQS Auto Scaling Integration

Key CloudWatch metrics: `ApproximateNumberOfMessagesVisible` (messages waiting to be processed — the most commonly used), `ApproximateNumberOfMessagesNotVisible` (in-flight count), `ApproximateAgeOfOldestMessage` (a latency signal).

**A better "backlog per instance" metric**: periodically compute `messages_visible / current_instance_count` in a Lambda, push it to CloudWatch as a custom metric, and use it for target-tracking auto scaling.

---

## 2. SNS Fundamentals ⭐⭐⭐

**= AWS's fully managed pub/sub messaging service.** Core value: **fanout** — one message delivered to multiple subscribers in parallel.

### SNS Topic Types

|  | **Standard Topic** | **FIFO Topic** |
|---|---|---|
| Ordering | Best-effort | **Strict FIFO** |
| Delivery | At-least-once | **Exactly-once** (5-minute dedup window) |
| Throughput | Nearly unlimited | Higher but with a defined ceiling (check current docs) |
| Subscribers | Very large scale per topic | Much lower cap per topic |
| Supported subscription types | SQS, Lambda, HTTP/S, Email, SMS, Mobile push, Firehose | **SQS, HTTP/S, Lambda** (Lambda support is a relatively recent addition) |
| Name suffix | Any | **Must end in `.fifo`** |
| Archive/replay | ❌ | ✅ (a newer feature: archive policy + replay) |

**Key limits**: message size caps at **256 KB**; larger messages need the SNS Extended Client Library (body to S3).

### SNS Subscriptions ⭐⭐

**Supported subscription types** (Standard Topic): SQS (most common), Lambda, HTTP/HTTPS (webhook), Email/Email-JSON, SMS, Mobile Push, Kinesis Data Firehose.

**Confirmation flow** (HTTP/Email/SMS): once a subscription is created, SNS sends a confirmation message to the endpoint → the endpoint must click a confirmation link (Email) or call `ConfirmSubscription` (HTTP) → delivery starts only after confirmation. Same-account Lambda and SQS subscriptions confirm automatically (IAM handles identity verification).

**Delivery Retry Policy**: for Lambda/SQS/Firehose/SMS/Mobile, AWS retries internally with no configuration available; for HTTP/S endpoints, retry policy **is** customizable (attempt count, delay, backoff function).

**DLQ for SNS**: routes messages that fail delivery to an SQS DLQ; configured at the subscription level (not the topic level); each subscription can have its own DLQ; the DLQ must be an SQS queue.

### Message Filtering ⭐⭐⭐

**= letting subscribers receive only messages matching a condition.** Configure a filter policy on the subscription; SNS filters before delivery.

```json
// Subscription 1's filter policy
{ "event_type": ["order_placed"] }

// Subscription 2's filter policy
{ "event_type": ["order_cancelled"], "amount": [{"numeric": [">", 100]}] }
```

**Filter Policy Scope** ⭐: **`MessageAttributes`** (default, matches message attributes) or **`MessageBody`** (matches the JSON body — fits legacy applications that can't add attributes and only send a JSON body).

Filter match types: exact match, numeric comparison, prefix match, suffix match, `exists`, `anything-but`.

### The SNS + SQS Fanout Pattern ⭐⭐⭐

**= a classic architecture: SNS as the fanout entry point, with multiple SQS queues subscribing to one topic.**

```
                  ┌── SQS Queue A ─→ Service A (order processing)
                  │
Publisher → SNS Topic ── SQS Queue B ─→ Service B (inventory update)
                  │
                  └── SQS Queue C ─→ Service C (email notification)
```

**Why not just SNS → Service directly**:

|  | Direct SNS → Service | SNS → SQS → Service |
|---|---|---|
| Service temporarily down | Message may be lost | Message **stays safe in SQS** |
| Durability | SNS doesn't persist | **SQS persists, 14 days** |
| Replay ability | None | Can redrive |
| Monitoring | SNS metrics | SQS queue depth metrics → auto scaling |

**Judgment call**: any "fan an event out to multiple downstream services" + "the downstream might be slow/down" + "delivery must be reliable" → **SNS + SQS Fanout**.

### SQS vs SNS vs EventBridge vs Kinesis ⭐⭐⭐

|  | **SQS** | **SNS** | **EventBridge** | **Kinesis Data Streams** |
|---|---|---|---|---|
| Model | Point-to-point queue | Pub/Sub topic | Event bus + routing | Streaming |
| Persistence | ✅ 14 days | ❌ (not stored, delivered directly) | Short-term | 1 day default, up to 365 days |
| Ordering | Standard unordered, FIFO ordered | Standard unordered, FIFO ordered | Guaranteed | Guaranteed per partition key |
| Consumer count | 1 logical consumer (multiple instances sharing) | Multiple subscriptions (broadcast) | Multiple rules (broadcast) | Multiple consumers, each with a cursor |
| Replay | Not possible (deleted on consume) | ❌ (FIFO can archive) | ❌ | ✅ (back to N days ago) |

**Rule of thumb**: decouple + buffer + process-once → **SQS**; broadcast to multiple consumers → **SNS**; rule-based routing / AWS service events → **EventBridge**; stream processing / multiple consumers each replaying → **Kinesis**.

---

## 3. Kinesis — Streaming Data Processing

### The Kinesis Family Overview

| Service | Purpose |
|---|---|
| **Kinesis Data Streams (KDS)** | Real-time data streams (low-level, you build the consumer) |
| **Kinesis Data Firehose (KDF)** | Streaming ETL + delivery to S3/Redshift/OpenSearch (fully managed) |
| **Kinesis Data Analytics / Managed Apache Flink** | Real-time stream analysis with SQL or Flink |
| **Kinesis Video Streams** | Video streaming |

> ⚠️ AWS has renamed Kinesis Data Firehose to "Amazon Data Firehose" and Kinesis Data Analytics to "Managed Service for Apache Flink," but exam question banks and much documentation still widely use the old names. Check current AWS naming.

### Kinesis Data Streams ⭐⭐⭐

**= a real-time data streaming service** — producers write, consumers read and process in real time.

**Shards** (the unit of capacity) ⭐⭐⭐:

|  | Write | Read |
|---|---|---|
| Data volume | **1 MB/s** | **2 MB/s** |
| Record count | **1,000 records/s** | / |
| GetRecords API | / | **5 TPS** per shard |

**Record size limit**: a single record maxes out at **1 MB**.

**How partition keys work**: `PutRecord(stream, data, partitionKey)` → hashing the partition key determines which shard the record lands on → records within a shard are strictly ordered by sequence number. All records with the same partition key land on the same shard; ordering isn't guaranteed across shards; a hot partition key overwhelms its shard while other records elsewhere get throttled.

**Partition key design**: ✅ high cardinality (user_id, device_id, UUID); ❌ low cardinality (country, status); ❌ time-based (causes today's writes to pile onto one shard).

**Stream Capacity Modes** ⭐⭐:

**1. Provisioned mode**: you specify the shard count, billed hourly per shard, fits predictable traffic, scaled manually via SplitShard/MergeShards/UpdateShardCount.

**2. On-Demand mode**: shards scale automatically, billed by GB read/written, fits unpredictable traffic or when you'd rather not manage capacity. Switching limit: at most 2 mode switches within 24 hours.

**Retention** ⭐⭐: default retention **24 hours**; maximum retention **365 days** (a relatively recent increase from 7 days). KDS's key value: replay (re-reading from any point in the retention window); independent cursors per consumer; data can't be deleted or modified until retention expires.

**Producers**: ① AWS SDK/CLI — `PutRecord` (single) and `PutRecords` (batch); ② **Kinesis Producer Library (KPL)** (recommended for high-throughput scenarios, Java/C++, automatic batching/aggregation/retries); ③ **Kinesis Agent** (for log files — a standalone daemon).

**Consumers — Two Fan-Out Models** ⭐⭐⭐:

**1. Classic Fan-Out** (shared, default, cheap): multiple consumers share a shard's 2 MB/s read throughput, using the `GetRecords` API (pull model), latency around 200ms, **free**.

**2. Enhanced Fan-Out** (billed, high performance): each consumer gets its own dedicated 2 MB/s read throughput per shard, using `SubscribeToShard` (push model, HTTP/2), latency around 70ms (notably faster), a max of 20 enhanced fan-out consumers per stream, billed per consumer-shard hour + GB retrieved.

**Choosing**: 1-2 consumers, latency doesn't matter much → Classic Fan-Out; 3+ consumers or low latency required → Enhanced Fan-Out.

**Kinesis Client Library (KCL)**: a library that builds the consumer for you. Handles cross-shard load balancing, checkpointing (progress stored in a **DynamoDB table**), fault recovery, and adapting to resharding automatically. ⚠️ KCL creates a DynamoDB table for checkpointing automatically — it needs DDB permissions and incurs extra cost.

**Resharding** ⭐: **Split** (one shard becomes two, raising capacity); **Merge** (two shards combine into one, lowering capacity). Trap: ⚠️ after resharding, the old shard doesn't vanish immediately — it goes CLOSED, but its data remains until retention expires; a consumer must finish the old shard before starting the new one (KCL handles this automatically).

### Kinesis Data Firehose ⭐⭐⭐

**= a streaming ETL and delivery service** (fully managed, serverless).

**Key traits**: fully managed, auto-scaling; **near real-time** (not real-time — buffers flush after 60 seconds to 15 minutes, though some destinations support near-zero-latency options); transformation via Lambda; format conversion JSON → Parquet/ORC (built-in, no Lambda needed); compression via GZIP/ZIP/Snappy; automatic retries, with failures landing in an S3 backup bucket; **no replay** (nothing is retained after delivery); **no consumer concept** (only a destination).

**Buffer configuration** (S3 example) ⭐⭐: buffer size **1-128 MiB** (default ~5 MiB); buffer interval **60-900 seconds** (default ~300 sec); whichever threshold is hit first triggers a flush. When throughput exceeds processing capacity, the buffer grows dynamically to avoid data loss.

**Data Transformation** (optional): a Lambda processes each record before delivery, and must return `result: Ok/Dropped/ProcessingFailed`. Lambda buffer limit: roughly 3 MB / 5 minutes.

**Source options**: Direct PUT (producers call the Firehose API directly); a Kinesis Data Stream as the source (KDS handles real-time processing, Firehose archives to S3 — a classic combination); CloudWatch Logs/Events, IoT, WAF logs.

**Destinations**: Amazon S3 (most common), Redshift (via S3), OpenSearch, HTTP endpoints, Splunk, third parties like Datadog, Snowflake, Apache Iceberg Tables.

### KDS vs KDF ⭐⭐⭐

|  | **Kinesis Data Streams (KDS)** | **Kinesis Data Firehose (KDF)** |
|---|---|---|
| Real-time-ness | **Real-time** (~200ms classic / ~70ms enhanced) | **Near real-time** (minutes-scale buffer, seconds-scale in some cases) |
| Data retention | 24h – 365 days | **Not retained** (deleted after delivery) |
| Replay | ✅ Supported | ❌ Not supported |
| Consumers | Multiple, each with a cursor | **Destination only** |
| Capacity management | Provisioned (shard) or On-Demand | **Fully automatic** |
| Producer manages partition key | ✅ | ❌ |
| Transformation | Written by the consumer | **Lambda transform** (just configure it) |
| Pricing model | Per shard hour or GB | Per GB ingested |
| Use case | **Real-time processing / multiple consumers** | **Streaming ETL + delivery** |

**Classic choice**: "real-time processing + multiple consumers + replay" → KDS; "stream data into S3/Redshift, want it simple" → KDF; "need both" → KDS → KDF (KDS as the main stream, KDF for delivery).

### Kinesis Data Analytics / Managed Apache Flink (Brief)

**= real-time analysis on a stream, using SQL or Flink.** Core capabilities: tumbling windows, sliding windows, real-time join/aggregate/filter, anomaly detection. Input sources: Kinesis Data Streams, Amazon MSK. Output: Kinesis Data Streams, Firehose, Lambda. This isn't heavily tested on the DVA — knowing it's "SQL/Flink analysis on a stream" is enough.

### SQS vs Kinesis ⭐⭐⭐

| Dimension | **SQS** | **Kinesis Data Streams** |
|---|---|---|
| Model | Queue (deleted on consume) | Stream (data retained, replayable) |
| Consumers | Multiple consumers share the work (each message handled by one consumer) | Multiple consumers each replay independently (every one can read everything) |
| Replay | ❌ | ✅ Any point within the retention window |
| Message size | 256 KB | 1 MB |
| Fits | **Decouple + buffer + process-once** | **Multiple consumers + streaming + replay** |

**Scenario mapping**: a task queue with workers deleting on completion → SQS; multiple services all need the same data → Kinesis (or SNS + multiple SQS); need to rewind and reprocess the past → Kinesis; real-time analytics + ML → Kinesis.

### Kinesis Data Streams + Lambda Traps ⭐⭐

**Default behavior** (different from SQS!): the whole batch retries on failure, until it succeeds or hits the max retry / record expiration. ⚠️ **The poison-pill problem**: a single bad record can retry the entire batch endlessly, blocking the whole shard!

**Fixes**: `MaximumRetryAttempts` (sets a retry cap; defaults to -1/unlimited, and must be changed in production); `MaximumRecordAgeInSeconds` (skips a record once it's too old to still be worth processing); `OnFailure Destination` (sends failed-record metadata to SQS/SNS); `BisectBatchOnFunctionError` (bisects a failing batch to isolate the bad record); `ParallelizationFactor` (1-10, allows multiple parallel Lambda instances per shard, though records with the same partition key still process serially to preserve ordering).
