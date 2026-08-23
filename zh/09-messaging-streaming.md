# Module 9：消息与流式（SQS / SNS / Kinesis）

> **覆盖日期**：5/8、5/9
>
> English version: [`en/09-messaging-streaming.md`](../en/09-messaging-streaming.md)

---

## 一、SQS 基础 ⭐⭐⭐

**= AWS 完全托管的消息队列服务**（point-to-point 模式）。Producer 发消息到 Queue，Consumer 主动 poll 消息处理。

**关键特性**：✅ 完全托管；✅ 几乎无限扩展；✅ 持久存储（跨多个 AZ 复制）；✅ 解耦生产者和消费者（异步）；✅ 不限保留期内消息数。

### SQS 两种 Queue 类型 ⭐⭐⭐

|  | **Standard Queue** | **FIFO Queue** |
|---|---|---|
| 顺序 | 尽力（best-effort）——**可能乱序** | **严格 FIFO**（按 message group） |
| 重复 | **至少一次投递**——可能重复 | **恰好一次投递**（5 分钟去重窗口） |
| 吞吐 | 几乎无限 | 无批时较低，批量发送时更高（具体数字以官方文档为准） |
| 名字后缀 | 任意 | **必须以 `.fifo` 结尾** |
| Message Group ID | 不需要 | **必须**（同 group 内严格 FIFO，不同 group 可并行） |
| Deduplication ID | 不需要 | **必须**（或启用 ContentBasedDeduplication） |

**选择**：✅ Standard——大多数场景，要高吞吐，业务能容忍重复/乱序（用幂等处理）；✅ FIFO——必须严格顺序（银行交易、订单状态机），不能重复。

### 通用关键限制 ⭐⭐⭐

| 项 | 值 |
|---|---|
| **Message size 上限** | **256 KB**（整个 message） |
| **更大消息** | 用 S3 + Extended Client Library（消息体放 S3，SQS 存指针） |
| **Message retention** | 60 秒 - **14 天**，**默认 4 天** |
| **Visibility Timeout** | 0 秒 - **12 小时**，**默认 30 秒** |
| **Delay queue / 单消息 delay** | 0 秒 - **15 分钟** |
| **Long polling 最大等待** | **20 秒** |
| **Batch size** | 最多 **10 条** per send/receive/delete batch |
| **Message attributes** | 最多 **10 个** per message |

**Encryption**：In-transit 始终 HTTPS；At-rest 可选 SSE——**SSE-SQS**（AWS 管 key，免费，默认开启）或 **SSE-KMS**（customer managed key）。

### Visibility Timeout ⭐⭐⭐

**= 消息被接收后，对其他 consumer 不可见的时间窗口**

```
T=0   Consumer A poll → 拿到 message X, 进入"in-flight"状态,倒计时开始(默认 30s)
T=0-30s    Consumer B poll → 看不到 X(被锁定)
T=30s+ (若 Consumer A 没在 30s 内 delete)
       X 重新可见 → Consumer B 能拿到
```

**三种结局**：① Consumer 在 timeout 内 delete 消息 → 永久删除；② Consumer 没在 timeout 内 delete（crash 或处理慢）→ 重新可见 → 可能被重复处理 → ReceiveCount 累加达到 maxReceiveCount → 进 DLQ；③ Consumer 主动 `ChangeMessageVisibility` 延长 timeout。

**设置策略**：太短（< 处理时间）→ 被其他 consumer 重复处理；太长（几小时）→ crash 后消息要等很久才重新可见，延迟高；理想是略长于平均处理时间 + 关键任务用 heartbeat 定期延长。

**判断**："想消息处理失败后立即可见" → 调用 `ChangeMessageVisibility(0)`。

### Polling ⭐⭐⭐

**Short Polling**（默认，但不推荐）：`WaitTimeSeconds=0`，立刻返回，即使队列有消息也可能返回空（只查了部分后端服务器）。缺点：大量空响应、API 调用次数多、收费贵。

**Long Polling**（推荐）⭐：`WaitTimeSeconds=1-20`，等到有消息或超时才返回，查询所有 SQS 后端服务器（no false empty）。优点：大幅降低 API 调用、降低延迟、几乎没空响应。⚠️ AWS 强烈推荐用 long polling（20 秒）。

**陷阱**：Lambda 触发 SQS（Event Source Mapping）时，Lambda 服务自动用 long polling，不用手动配；应用层手动 poll 时要显式开启。

### Dead Letter Queue (DLQ) ⭐⭐

**= 用来"接住"处理失败多次的消息的特殊队列**

Redrive Policy：

```json
{
  "deadLetterTargetArn": "arn:aws:sqs:...:my-dlq",
  "maxReceiveCount": 5
}
```

消息被接收 5 次都没被成功处理（没 delete）→ 自动转移到 DLQ（原 message ID 保留）。

**关键约束**：✅ DLQ 必须和源 queue 同类型（Standard 配 Standard，FIFO 配 FIFO）；✅ DLQ 必须在同账号 + 同 region；✅ 一个 DLQ 可以服务多个源 queue；⚠️ maxReceiveCount 别设为 1（没机会重试）。

**用途**：排查失败原因；隔离 poison message；手动重试——用 **Redrive to source** 把 DLQ 消息批量送回源 queue 重新处理。

### Delay Queue & Per-Message Delay

**Delay Queue**（队列级）：消息进入队列后延迟 N 秒才可见，范围 0-15 分钟，适合"延迟通知"场景。vs Visibility Timeout：Delay queue 是消息进入队列时就被隐藏；Visibility timeout 是消息被 consumer 拿到后才被隐藏。

**Per-Message Delay**：`SendMessage` 时设 `DelaySeconds`（0-15 分钟），单条消息独立延迟。⚠️ FIFO queue 不支持 per-message delay，只能在队列级设。

### Lambda + SQS ⭐⭐

Lambda 服务主动 poll SQS（Event Source Mapping，用你的 execution role），拿到一批 messages → 调 Lambda function → 成功自动 delete，失败等 visibility timeout 后重新可见 → 达到 maxReceiveCount 进 DLQ。

**关键配置**：`BatchSize`（单次给 Lambda 多少消息）、`MaximumBatchingWindow`（0-300 秒）、`ReportBatchItemFailures`（支持部分失败）。

**陷阱 1**：Visibility Timeout 必须 ≥ Lambda 超时 × 6（AWS 推荐），否则 Lambda 还没处理完，SQS 已经把消息释放给其他 consumer → 重复处理。

**陷阱 2**：Standard 默认整批失败 = 整批重试——如果一批 10 条只有 1 条失败，默认整批 10 条都重试。**解决**：用 `ReportBatchItemFailures`——Lambda response 里指定哪几条失败，只那几条重试：

```json
{
  "batchItemFailures": [
    {"itemIdentifier": "msg-id-3"},
    {"itemIdentifier": "msg-id-7"}
  ]
}
```

**陷阱 3**：FIFO + Lambda 并发——同一个 message group 内消息串行处理（只一个 Lambda 实例），不同 group 并行；想要高并发 FIFO → 用多个 message group。

### SQS Auto Scaling 集成

关键 CloudWatch metric：`ApproximateNumberOfMessagesVisible`（队列里待处理的消息数，最常用）、`ApproximateNumberOfMessagesNotVisible`（in-flight 数）、`ApproximateAgeOfOldestMessage`（监控延迟）。

**更好的"backlog per instance"指标**：用 Lambda 周期性算 `messages_visible / current_instance_count`，推到 CloudWatch 作为 custom metric 做 target tracking auto scaling。

---

## 二、SNS 基础 ⭐⭐⭐

**= AWS 完全托管的 pub/sub 消息服务**。核心价值：**Fanout**——一条消息发给多个订阅者（并行）。

### SNS Topic 类型

|  | **Standard Topic** | **FIFO Topic** |
|---|---|---|
| 顺序 | 尽力 | **严格 FIFO** |
| 投递 | 至少一次 | **恰好一次**（5 分钟去重窗口） |
| 吞吐 | 几乎无限 | 较高但有明确上限（以官方文档为准） |
| 订阅者数 | 单 topic 支持极大规模 | 单 topic 上限少得多 |
| 支持的订阅类型 | SQS、Lambda、HTTP/S、Email、SMS、Mobile push、Firehose | **SQS、HTTP/S、Lambda**（Lambda 支持是较新添加） |
| 名字后缀 | 任意 | **必须以 `.fifo` 结尾** |
| 消息归档/重放 | ❌ | ✅（较新功能：archive policy + replay） |

**关键限制**：Message size 上限 **256 KB**；更大消息用 SNS Extended Client Library（消息体到 S3）。

### SNS Subscriptions ⭐⭐

**支持的订阅类型**（Standard Topic）：SQS（最常见）、Lambda、HTTP/HTTPS（webhook）、Email/Email-JSON、SMS、Mobile Push、Kinesis Data Firehose。

**Confirmation 流程**（HTTP/Email/SMS）：订阅创建后 SNS 发确认消息到订阅 endpoint → endpoint 必须点击确认链接（Email）或调用 ConfirmSubscription（HTTP）→ 确认后才开始接收消息。Lambda、SQS 同账户订阅自动确认（IAM 验证身份）。

**Delivery Retry Policy**：Lambda/SQS/Firehose/SMS/Mobile 由 AWS 内部 retry，你不能配；HTTP/S endpoint 可自定义 retry 策略（次数、延迟、backoff function）。

**DLQ for SNS**：SNS 投递失败时把消息送到 SQS DLQ；配置在 subscription 级（不是 topic 级）；每个订阅可以有自己的 DLQ；DLQ 必须是 SQS queue。

### Message Filtering ⭐⭐⭐

**= 让订阅者只接收符合条件的消息**，在订阅上配 Filter Policy，SNS 在投递前过滤。

```json
// Subscription 1 的 filter policy
{ "event_type": ["order_placed"] }

// Subscription 2 的 filter policy
{ "event_type": ["order_cancelled"], "amount": [{"numeric": [">", 100]}] }
```

**Filter Policy Scope** ⭐：**`MessageAttributes`**（默认，匹配 message attributes）或 **`MessageBody`**（匹配消息 body JSON 内容——适合无法改 publisher 发 attributes 的老应用）。

过滤匹配类型：精确匹配、数字比较、前缀匹配、后缀匹配、`exists`、`anything-but`。

### SNS + SQS Fanout 模式 ⭐⭐⭐

**= 经典架构：SNS 作为 fanout 入口，多个 SQS 队列订阅同一个 topic**

```
                  ┌── SQS Queue A ─→ Service A (订单处理)
                  │
Publisher → SNS Topic ── SQS Queue B ─→ Service B (库存更新)
                  │
                  └── SQS Queue C ─→ Service C (邮件通知)
```

**为什么用这个模式（不直接 SNS → Lambda/Service）**：

|  | 直接 SNS → Service | SNS → SQS → Service |
|---|---|---|
| Service 暂时挂了 | 消息可能丢 | 消息**安全留在 SQS** |
| 消息持久化 | SNS 不持久 | **SQS 持久，14 天** |
| Replay 能力 | 没有 | 可 redrive |
| 监控 | SNS metrics | SQS 队列长度 metrics → auto scaling |

**判断**：任何"event 分发到多个下游 service" + "下游可能慢/挂" + "要可靠投递" → **SNS + SQS Fanout**。

### SQS vs SNS vs EventBridge vs Kinesis ⭐⭐⭐

|  | **SQS** | **SNS** | **EventBridge** | **Kinesis Data Streams** |
|---|---|---|---|---|
| 模式 | Point-to-point queue | Pub/Sub topic | Event bus + 路由 | Streaming |
| 持久化 | ✅ 14 天 | ❌（不存，直接投递） | 短期 | 默认 1 天，最长 365 天 |
| 顺序 | Standard 不保证，FIFO 保证 | Standard 不保证，FIFO 保证 | 保证 | 按 partition key 保证 |
| Consumer 数 | 1 个逻辑 consumer（多 instance 共同消费） | 多个订阅（广播） | 多 rule（广播） | 多 consumer 各自 cursor |
| Replay | 不能（消费即删） | ❌（FIFO 可 archive） | ❌ | ✅（回到 N 天前） |

**速记**：解耦 + 缓冲 + 处理一次 → **SQS**；广播给多个消费者 → **SNS**；基于规则路由/AWS 服务事件 → **EventBridge**；流处理/多 consumer 各自 replay → **Kinesis**。

---

## 三、Kinesis — 流式数据处理

### Kinesis 全家桶概览

| 服务 | 用途 |
|---|---|
| **Kinesis Data Streams (KDS)** | 实时数据流（底层，需要自己消费） |
| **Kinesis Data Firehose (KDF)** | 流式 ETL + 落地到 S3/Redshift/OpenSearch（完全托管） |
| **Kinesis Data Analytics / Managed Apache Flink** | 用 SQL 或 Flink 实时分析流数据 |
| **Kinesis Video Streams** | 视频流 |

> ⚠️ AWS 已把 Kinesis Data Firehose 改名为 "Amazon Data Firehose"，Kinesis Data Analytics 改名为 "Managed Service for Apache Flink"，但考试题库和很多文档仍广泛使用旧名字。以官方最新命名为准。

### Kinesis Data Streams ⭐⭐⭐

**= 实时数据流服务**，producer 写入，consumer 实时读取处理。

**Shard**（容量单位）⭐⭐⭐：

|  | 写 | 读 |
|---|---|---|
| 数据量 | **1 MB/s** | **2 MB/s** |
| 记录数 | **1,000 records/s** | / |
| GetRecords API | / | **5 TPS** per shard |

**Record 大小限制**：单个 record 最大 **1 MB**。

**Partition Key 工作机制**：`PutRecord(stream, data, partitionKey)` → hash(partitionKey) 决定落到哪个 shard → shard 内按 sequence number 严格有序。同一个 partition key 的所有 record 落到同一个 shard；跨 shard 不保证全局顺序；hot partition key → 该 shard 撞上限，其他 record throttle。

**partition key 设计**：✅ 高基数（user_id、device_id、UUID）；❌ 低基数（country、status）；❌ 时间（会导致今天所有写到一个 shard）。

**Stream 容量模式** ⭐⭐：

**1. Provisioned mode**：自己指定 shard 数量，按 shard 小时收费，适合流量可预测，用 SplitShard/MergeShards/UpdateShardCount 手动 scale。

**2. On-Demand mode**：自动 scale shards，按 GB 读写收费，适合流量不可预测/不想管理。切换限制：24 小时内最多切换 2 次。

**Retention** ⭐⭐：默认保留期 **24 小时**；最长保留期 **365 天**（较新提升，之前是 7 天）。KDS 的关键价值：可以 replay（在保留期内从任意时间点重新读取）；多 consumer 独立 cursor；数据不能 delete 或修改直到保留期过期。

**Producers（写入）**：① AWS SDK/CLI——`PutRecord`（单条）、`PutRecords`（批量）；② **Kinesis Producer Library (KPL)**（推荐高吞吐场景，Java/C++，自动 batching/aggregation/retries）；③ **Kinesis Agent**（日志文件场景，独立 daemon）。

**Consumers — 两种 Fan-out 模型** ⭐⭐⭐：

**1. Classic Fan-Out**（共享 fan-out，默认+便宜）：多个 consumer 共享一个 shard 的 2 MB/s 读吞吐，用 `GetRecords` API（pull 模型），延迟约 200ms，**免费**。

**2. Enhanced Fan-Out**（增强 fan-out，收费+高性能）：每个 consumer 独享 2 MB/s 读吞吐 per shard，用 `SubscribeToShard` API（push 模型，HTTP/2），延迟约 70ms（明显更快），每个 stream 最多 20 个 enhanced fan-out consumers，按 consumer-shard hour + GB retrieved 收费。

**选择**：1-2 个 consumer + 不介意延迟 → Classic Fan-Out；3+ consumer 或需要低延迟 → Enhanced Fan-Out。

**Kinesis Client Library (KCL)**：帮你写 consumer 的库。自动处理跨 shard 负载均衡、checkpoint（进度保存到 **DynamoDB 表**）、故障恢复、resharding 适应。⚠️ KCL 会自动创建 DynamoDB 表存 checkpoint，需要 DDB 权限，有额外费用。

**Resharding** ⭐：**Split**（一个 shard 拆成 2 个，提高容量）；**Merge**（2 个 shard 合成 1 个，降低容量）。陷阱：⚠️ Resharding 后旧 shard 不立即消失，状态变 CLOSED，但数据仍在直到保留期过期；consumer 必须先消费完旧 shard 再消费新 shard（KCL 自动处理）。

### Kinesis Data Firehose ⭐⭐⭐

**= 流式 ETL + 落地服务**（完全托管，无服务器）。

**核心特性**：完全托管，自动 scale；**近实时**（非实时，buffer 60 秒到 15 分钟才 flush，部分场景支持接近零延迟的选项）；数据转换用 Lambda；格式转换 JSON → Parquet/ORC（内置，不需 Lambda）；压缩 GZIP/ZIP/Snappy；失败重试，失败的进 S3 backup bucket；**没有 replay**（数据投递后不留拷贝）；**没有 consumer 概念**（只有 destination）。

**Buffer 配置**（S3 为例）⭐⭐：Buffer Size **1-128 MiB**（默认约 5 MiB）；Buffer Interval **60-900 秒**（默认约 300 秒）；两个条件先满足谁就 flush。Buffer 处理跟不上时会动态增大，避免数据丢失。

**Data Transformation**（可选）：投递前用 Lambda 处理每条 record。Lambda 必须返回 `result: Ok/Dropped/ProcessingFailed`。Lambda buffer 限制：约 3 MB / 5 分钟。

**Source 选项**：Direct PUT（Producer 直接调 Firehose API）；Kinesis Data Stream 作为 source（KDS 做实时处理 + Firehose 落地到 S3 做归档，经典组合）；CloudWatch Logs/Events、IoT、WAF logs。

**Destination**：Amazon S3（最常用）、Redshift（经 S3 中转）、OpenSearch、HTTP endpoints、Splunk、Datadog 等第三方、Snowflake、Apache Iceberg Tables。

### KDS vs KDF 对比 ⭐⭐⭐

|  | **Kinesis Data Streams (KDS)** | **Kinesis Data Firehose (KDF)** |
|---|---|---|
| 实时性 | **实时**（约 200ms classic / 约 70ms enhanced） | **近实时**（分钟级 buffer，部分场景可到秒级） |
| 数据保留 | 24h - 365 天 | **不保留**（投递后即删） |
| Replay | ✅ 支持 | ❌ 不支持 |
| Consumer | 多 consumer 各自 cursor | **只有 destination** |
| 容量管理 | 分 Provisioned（shard）和 On-Demand | **完全自动** |
| Producer 管理 partition key | ✅ | ❌ |
| 转换 | Consumer 自己写 | **Lambda 转换**（配置即可） |
| 费用模型 | per shard hour 或 GB | per GB ingested |
| 用途 | **实时处理/多 consumer** | **流式 ETL + 落地** |

**经典选型**："实时处理 + 多 consumer + replay" → KDS；"把数据流式落地到 S3/Redshift，要省事" → KDF；"两个都要" → KDS → KDF（KDS 做主流，KDF 落地）。

### Kinesis Data Analytics / Managed Apache Flink（简略）

**= 用 SQL 或 Flink 在流上做实时分析**。核心能力：Tumbling window（固定时间窗口）、Sliding window（滑动窗口）、实时 join/aggregate/filter、异常检测。输入源：Kinesis Data Streams、Amazon MSK。输出：Kinesis Data Streams、Firehose、Lambda。DVA 考试对这块考察不多，知道它"是 SQL/Flink 在流上做分析"即可。

### SQS vs Kinesis ⭐⭐⭐

| 维度 | **SQS** | **Kinesis Data Streams** |
|---|---|---|
| 模型 | Queue（消费即删） | Stream（数据保留，可 replay） |
| 消费者 | 多个 consumer 共同消费（一条消息一个 consumer 处理） | 多个 consumer 各自 replay（每个都能读全部） |
| Replay | ❌ | ✅ 保留期内任意时间点 |
| 单消息大小 | 256 KB | 1 MB |
| 适合场景 | **解耦 + 缓冲 + 处理一次** | **多消费者 + 流式处理 + replay** |

**场景对照**：任务队列，worker 处理后删除 → SQS；多个 service 都要消费相同数据 → Kinesis（或 SNS + 多 SQS）；需要回到过去重新处理 → Kinesis；实时分析 + ML → Kinesis。

### Kinesis Data Streams + Lambda 陷阱 ⭐⭐

**默认行为**（和 SQS 不同！）：整批失败重试，直到成功或达到 max retry/record 过期。⚠️ Poison pill 问题：一条坏 record 会无限重试整批，堵塞整个 shard！

**解决方案**：`MaximumRetryAttempts`（设最大重试次数，默认 -1 无限，生产必须改）；`MaximumRecordAgeInSeconds`（record 超过 N 秒还没处理成功就跳过）；`OnFailure Destination`（失败的 record metadata 发到 SQS/SNS）；`BisectBatchOnFunctionError`（批失败时二分定位坏 record）；`ParallelizationFactor`（1-10，同一 shard 可并行起多个 Lambda 实例，同 partition key 仍串行以保序）。
