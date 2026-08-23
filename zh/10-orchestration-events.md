# Module 10：编排与事件（Step Functions / EventBridge）

> **覆盖日期**：5/10
>
> English version: [`en/10-orchestration-events.md`](../en/10-orchestration-events.md)

---

## 一、Step Functions 是什么 ⭐⭐⭐

**= AWS 完全托管的工作流编排服务**。核心价值：把多个 AWS 服务（Lambda、ECS、DynamoDB、SNS、SQS 等）的调用编排成一个有序、可视化、可容错的工作流。

没有 Step Functions：Lambda A 调 Lambda B 调 Lambda C → 错误处理散落各处，状态管理混乱。用 Step Functions：把流程定义成 **state machine**，AWS 帮你管状态、错误、重试。

**关键概念**：State Machine（工作流定义，JSON/ASL）；Execution（一次运行）；State（工作流中的一步）；ASL（Amazon States Language，描述 state machine 的 JSON DSL）。

### Standard vs Express Workflow ⭐⭐⭐

**创建 state machine 时必须选，创建后不能改！**

| 维度 | **Standard** | **Express** |
|---|---|---|
| **最大执行时长** | **1 年** | **5 分钟** |
| **执行语义** | **Exactly-once**（除非显式 retry） | **At-least-once**（可能重复） |
| **吞吐** | 较低（每秒数千级） | **明显更高**（每秒数十万级） |
| **执行历史** | 90 天可查 | **不保留**（只在 CloudWatch Logs） |
| **可视化历史** | ✅ 控制台可查每次执行 | ❌（看 CloudWatch Logs） |
| **计费** | 按 state transitions | 按请求数 + 时长 + 内存 |
| **价格（高吞吐）** | 明显更贵 | 便宜 |
| **支持 `.waitForTaskToken`** | ✅（可暂停最长 1 年） | ❌ |
| **支持 `.sync`** | ✅ | ❌ |
| **同步/异步调用** | 只异步 | **同步 + 异步**（两种子类型） |
| **典型场景** | 长流程、订单履约、ETL、人工审批 | 高频事件、IoT 摄取、API 后端、流处理 |

**Express 两种子类型**：Asynchronous Express（`StartExecution`，返回 confirmation）；Synchronous Express（`StartSyncExecution`，等结果返回，最多 5 分钟，适合 API Gateway/Lambda 同步调用场景）。

**选择决策**：选 Standard——长流程（> 5 分钟）、要求 exactly-once、需要详细执行历史、需要人工审批、复杂业务流程；选 Express——短流程（< 5 分钟）、高吞吐、业务幂等、微服务编排、想省钱。

**嵌套模式**（高阶）：Standard workflow 调 Express workflow——父 Standard 处理长流程+人工审批+详细历史，子 Express 处理高频 idempotent 操作，兼顾可靠性+成本。

### States（状态类型）⭐⭐⭐

ASL 定义了 8 种 state：

**1. Task**（执行任务）：调一个 AWS 服务或 Lambda。集成模式：标准 Lambda 调用（异步）；优化的 Lambda invoke；`.waitForTaskToken`（暂停等回调）；`.sync`（同步等待完成）。

**2. Choice**（分支）：类似 if-else，根据输入数据走不同分支。

**3. Wait**（等待）：等指定时间或时间戳。⚠️ Wait 期间不计费 state transitions/时长（Standard 模式），适合"等下游系统返回"。

**4. Parallel**（并行）：多个分支同时执行，全部完成才进入下一步。典型用途：同时调多个 API 并行验证。

**5. Map**（循环/批处理）：对一个数组的每个元素执行同样的工作流。两种模式 ⭐：**Inline Map**（每个 iteration 在父 execution 内，共享执行历史，并发数有限）；**Distributed Map**（较新功能，每个 iteration 是独立 child execution，支持大规模并发，适合处理大量数据，如 S3 大文件每行一个 iter）。

**6. Pass**（传递/数据变换）：不做事，只是传递或转换输入，用于数据塑形、debug、占位。

**7. Succeed**：终止 execution 并标记为成功。

**8. Fail**：终止 execution 并标记为失败，带 error/cause。

### Error Handling ⭐⭐⭐

**Retry（自动重试）**：在每个 Task/Parallel state 上配置：

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

关键参数：`ErrorEquals`（哪些错误重试，`States.ALL` = 所有）、`IntervalSeconds`（首次等待）、`MaxAttempts`（最大重试次数）、`BackoffRate`（指数退避倍数）、`MaxDelaySeconds`（单次等待上限）、`JitterStrategy`（加随机抖动避免雷鸣群效应）。

预定义错误：`States.ALL`、`States.Timeout`、`States.TaskFailed`、`States.Permissions`、`States.HeartbeatTimeout`、`Lambda.ServiceException`。

**Catch（捕获错误，跳转）**：类似 try-catch，重试用尽后跳转到指定 state。

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

`ResultPath`：把错误信息塞到输入哪个字段。**顺序**：Retry 先于 Catch（用尽 retry 后才走 Catch）。

### Service Integrations ⭐⭐

Step Functions 直接集成 AWS 服务，不用经过 Lambda。**三种集成模式**：

| 模式 | 后缀 | 行为 |
|---|---|---|
| **Request Response**（默认） | 无后缀 | 调用 API，立刻返回（不等结果） |
| **Run a Job** | `.sync` | 调用 API，等任务完成才进下一步 |
| **Wait for Callback** | `.waitForTaskToken` | 暂停 execution，等外部调 `SendTaskSuccess` 回调 |

`.sync` 例子（同步等待 ECS RunTask 完成）：适合 ECS/Batch job、Glue job、SageMaker 训练等长任务。

`.waitForTaskToken` 例子（人工审批）：工作流暂停（最长 1 年），等待外部调用 `send-task-success --task-token <token>`。适合人工审批、等待外部 API 异步回调。

⚠️ **Express 不支持 `.sync` 和 `.waitForTaskToken`**！

支持的集成服务超过 200 个，常考的：Lambda、ECS、Batch、Glue、EMR、DynamoDB、S3、SNS、SQS、Kinesis、SageMaker、API Gateway、EventBridge、Step Functions（嵌套调用！）。

### 其他重要特性

**Activity**（老式 worker 集成）：工作流暂停，等待外部 worker 调用 API 取任务（`GetActivityTask` → 处理 → `SendTaskSuccess`）。适合任务需要在自管 EC2/on-prem 处理的场景，现代场景用 `.waitForTaskToken` 更灵活。

**Input / Output Processing**：每个 state 可以配置字段过滤数据——`InputPath`、`Parameters`、`ResultSelector`、`ResultPath`、`OutputPath`。⚠️ DVA 不常考具体语法，知道这些字段控制数据流即可。

**限制**：最大 input/output/数据载荷 **256 KB**（更大用 S3 + reference）；state machine 定义大小 1 MB。

---

## 二、EventBridge 是什么 ⭐⭐⭐

**= AWS 完全托管的 serverless event bus**（发布/订阅 + 路由）。

```
        Events Sources                Rules                Targets
        ┌──────────────┐              ┌─────┐              ┌──────────┐
        │ AWS Service  │              │     │              │ Lambda   │
        │ (S3/EC2/...) │ ──events──→  │ Bus │ ──filter──→  │ SQS      │
        │ Custom App   │              │     │              │ SNS      │
        │ SaaS Partner │              │     │              │ Step Fn  │
        └──────────────┘              └─────┘              │ 30+ more │
                                                           └──────────┘
```

**vs SNS**：SNS 是简单 fanout，高吞吐（几百万 TPS）；EventBridge 支持复杂路由 + 过滤 + transformation + SaaS 集成，但吞吐相对较低。

**前身**：CloudWatch Events（EventBridge 是 CW Events 的进化版，API 兼容）。

### 三种 Event Bus ⭐⭐

**1. Default Event Bus**：每个账户每 region 自动有一个；自动接收所有 AWS 服务事件；不能删，不能创建额外的。用于 S3 上传触发 Lambda、EC2 状态变化、CloudTrail 事件等。

**2. Custom Event Bus**：自己创建，用于自己应用的事件，通过 `PutEvents` API 推送。适合应用间解耦、自定义事件路由。

**3. Partner Event Bus**：从 SaaS 伙伴接收事件（如 Datadog、Auth0、Zendesk、PagerDuty、Salesforce、Shopify、Stripe）。创建时必须和 partner event source 绑定（名字一致）。⚠️ 不能给 partner event bus 加 resource policy（只允许 partner 写入）。

**跨账户事件路由**：通过给目标 bus 加 resource policy 允许源账户，把 Account A 的 default bus 路由到 Account B 的 custom bus。

### EventBridge Rules ⭐⭐⭐

**两种 Rule 触发方式**：

**1. Event Pattern**（事件模式匹配）：

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

**2. Schedule**（定时触发）：Cron expression（`cron(0 12 * * ? *)`）或 Rate expression（`rate(5 minutes)`）。

⚠️ AWS 现在推荐用 **EventBridge Scheduler**（独立服务，更强大）替代 EventBridge 的 scheduled rules，但 scheduled rules 仍可用。

**Pattern 匹配类型**：精确值、前缀（prefix）、后缀（suffix）、排除（anything-but）、数字比较（numeric）、存在性（exists）、IP CIDR、通配符（wildcard，较新功能）。

**Input Transformer**（常考）⭐：在传给 target 前修改 event 数据，用 `InputPathsMap` + `InputTemplate` 让事件适配 target 的输入格式（target 不需要整个 event）。

### EventBridge Targets ⭐⭐

30+ 种 target，常见的：Lambda function（异步触发）、Step Functions（启动工作流）、SQS/SNS（转发）、Kinesis Data Streams/Firehose（流式归档）、ECS task、CodePipeline/CodeBuild（触发 CI/CD）、API Gateway/API Destinations（调外部 HTTP API）、EventBridge event bus（跨账户/region 转发）、Systems Manager Run Command。

**API Destinations** ⭐：让 EventBridge 调用任意外部 HTTP/HTTPS API 作为 target。工作机制：创建 Connection（配置认证：Basic/API Key/OAuth）→ 创建 API Destination（指 HTTP endpoint + connection）→ Rule 把 event 路由到 API Destination。用途：把 AWS 事件推送到 Datadog/Slack/PagerDuty/第三方 webhook。

### EventBridge Pipes ⭐⭐

**= 一对一 point-to-point 集成**（较新功能）

```
Source ──→ [Filter] ──→ [Enrichment] ──→ Target
(SQS/DDB Stream/Kinesis/MSK)       (Lambda/Step Fn/Kinesis/SQS/SNS/EventBridge bus 等)
```

**vs Event Bus**：Event Bus 是 many-to-many（多 source 多 target）；Pipes 是 **one-to-one**（单 source 单 target）。Pipes 的 source 限定为 SQS/DDB Streams/Kinesis/MSK/自管 Kafka/ActiveMQ/RabbitMQ；Pipes 独有 **Enrichment** 能力（Lambda/Step Fn/API Destination/API GW 调用补全数据）。

典型场景：DDB Streams → enrich（查别的表）→ Kinesis；SQS → Lambda enrich → Step Functions。优点：消除胶水代码（原本要写 Lambda 串起来，现在配置就行）。

### EventBridge Schema Registry（简略）

**= 自动发现/管理 event schema**。EventBridge 自动从经过的 events 发现 schema，帮你生成 SDK 代码绑定（Java、Python、TypeScript 等）。两种来源：AWS schemas（所有 AWS 服务事件，预置）、Custom schemas。DVA 不重点考。

### EventBridge vs SNS vs SQS vs Kinesis ⭐⭐⭐

|  | **EventBridge** | **SNS** | **SQS** | **Kinesis** |
|---|---|---|---|---|
| 模式 | Event bus + 路由 | Pub/Sub | Queue | Stream |
| 主要用途 | **基于规则路由/AWS 事件/SaaS 集成** | **大规模 fanout** | **解耦+缓冲** | **流处理+replay** |
| 吞吐 | 较低（可调） | 极高 | 几乎无限 | per shard |
| Targets | 30+ AWS services + HTTP | 多种端点 | 单消费（多 worker 共抢） | 多消费者 |
| 过滤 | **复杂 pattern matching** | message filtering（简单） | ❌ | 消费者侧 |
| Replay | ✅（Event Bus archive） | ❌ | ❌ | ✅ |
| Schema | ✅ Schema Registry | ❌ | ❌ | ❌ |
| SaaS 集成 | ✅（原生） | ❌ | ❌ | ❌ |

**判断**："AWS 服务事件触发 Lambda" → EventBridge default bus；"定时执行 Lambda" → EventBridge schedule rule 或 EventBridge Scheduler；"把 Datadog 告警接到 AWS" → EventBridge Partner Event Bus；"Microservice 之间复杂路由+过滤" → EventBridge custom bus；"1 个 event 推送给海量订阅者" → SNS（EventBridge 吞吐不够）；"DDB Streams → Lambda enrich → Step Functions" → EventBridge Pipes。

### Event Replay（EventBridge 高阶）

**= 把 archive 中的历史事件重新放入 event bus**。流程：event bus 启用 Archive（指定 retention 期）→ 所有匹配的 events 自动归档 → 需要时创建 replay（指定时间范围+filter pattern）→ EventBridge 把 archived events 重新发到 bus（走当前 rules）。用途：bug 修了想重新 process 历史事件；用历史数据演练测新 rule。⚠️ Replay 的 events 也会触发 targets，慎重，不要造成数据重复。
