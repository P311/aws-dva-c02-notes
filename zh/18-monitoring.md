# Module 18：监控与可观测性（X-Ray / CloudWatch / CloudTrail）

> **覆盖日期**：5/23
>
> English version: [`en/18-monitoring.md`](../en/18-monitoring.md)

---

# 一、X-Ray — 分布式追踪 ⭐⭐⭐

## X-Ray 是什么

**= 分布式应用追踪服务**，看清请求在微服务间怎么流动。核心价值：可视化服务拓扑（Service Map）、找性能瓶颈、找错误根因、完整请求链路。

## 核心概念 ⭐⭐⭐

| 概念 | 含义 |
|---|---|
| **Trace** | 一个完整请求穿越所有服务的全程记录 |
| **Segment** | 一个服务处理这个请求的数据 |
| **Subsegment** | 服务内部的细化：DB 调用、外部 HTTP 调用 |
| **Annotations** | 索引的 key-value，可被 filter expression 搜索（最多 50/trace） |
| **Metadata** | 不索引的数据，可任意类型，只能从 trace 详情查看 |

### Annotations vs Metadata ⭐⭐⭐

|  | Annotations | Metadata |
|---|---|---|
| 是否索引 | ✅ 索引 | ❌ 不索引 |
| 可搜索 | ✅（filter expression） | ❌ |
| 值类型 | string/number/boolean | 任意（含 object/array） |
| 限制 | 每 trace 最多 **50** 个 | 大小限制（单 segment 64 KB） |

**判断**：想"按 user_id 搜 trace" → Annotation；想"记录完整 request body 但不需要搜索" → Metadata。

## Service Map ⭐⭐

**= X-Ray 自动生成的"服务依赖图"**：

```
[Client]──→[API Gateway]──→[Lambda]──→[DynamoDB]
                            │
                            └──→[SNS]──→[Lambda 2]
```

每个 node 显示：平均延迟、错误率（4xx/5xx）、请求数。调试的"第一站"。

## Sampling Rules ⭐⭐⭐

X-Ray 不追踪每个请求（成本+性能考虑），按规则采样。

**Default Sampling Rule**：默认的 reservoir（每秒固定数）+ rate（超出部分按比例）——Reservoir size 1 request/second（每秒至少追踪 1 个）；Fixed rate 5%（超出 reservoir 后，5% 概率追踪）。

**Custom Sampling Rules** 例子：`/checkout` → Reservoir 5/s + Rate 100%（关键流程全追）；`/health` → Reservoir 0/s + Rate 0%（健康检查不追）。

**Adaptive Sampling**（较新功能）：异常时自动提高采样率，正常时降低，AWS 自动管理。

## X-Ray Daemon ⭐⭐

**= 本地代理，聚合 trace 数据，批量发到 X-Ray API**。监听 UDP 端口 **2000**。

| 环境 | Daemon |
|---|---|
| **EC2/on-prem** | 手动装 X-Ray daemon |
| **ECS/Fargate** | 在 task definition 加 X-Ray sidecar container |
| **Lambda** | AWS 自动管理（开启 Active Tracing） |
| **Elastic Beanstalk** | 平台内置（配置开启） |

**IAM 权限**：必须有 `xray:PutTraceSegments` + `xray:PutTelemetryRecords`。托管 policy：**`AWSXRayDaemonWriteAccess`** ⭐。

## X-Ray SDK

支持的语言：Node.js、Python、Java、.NET、Ruby、Go。

Lambda 用环境变量配合 X-Ray daemon 通信：`_X_AMZN_TRACE_ID`（追踪 header，含采样决定、trace ID、parent segment ID）；`AWS_XRAY_CONTEXT_MISSING`（没有 tracing header 时的行为，Lambda 默认 `LOG_ERROR`）；`AWS_XRAY_DAEMON_ADDRESS`（daemon 的 IP:PORT）；`AWS_XRAY_TRACING_NAME`；`AWS_XRAY_DEBUG_MODE`。

手动 instrument（Python 例子）：

```python
from aws_xray_sdk.core import xray_recorder

@xray_recorder.capture("process_order")
def process_order(order):
    xray_recorder.put_annotation("order_id", order.id)   # 索引,可搜
    xray_recorder.put_metadata("full_request", order.raw)  # 不索引
    return result
```

## Tracing Header ⭐

```
X-Amzn-Trace-Id: Root=1-5759e988-...; Parent=53995c3f...; Sampled=1
```

**Root**：整个 trace 的 ID（穿越所有服务保持不变）；**Parent**：上游 segment 的 ID；**Sampled**：这个请求是否被采样。

⭐ API Gateway 不传 trace header → Lambda 启动新 trace。想要 end-to-end tracing，API Gateway 和 Lambda 都要开 Active Tracing。

## Filter Expressions

```
service("my-service") AND duration > 5
annotation.user_id = "12345"
fault = true
```

⭐ 只有 `annotation.xxx` 才能搜，metadata 不能。

## AWS Distro for OpenTelemetry（补充）

适用性更广的 tracing 服务。适合用 ADOT 的场景：需要把 trace 发到多个不同的 tracing backend 而不重新 instrument 代码；需要大量社区维护的库 instrumentation；需要 Java/Python/Node.js 的全托管 Lambda layer。适合选 X-Ray SDK 的场景：需要紧密集成的单一供应商方案；需要 X-Ray 集中采样规则的完整集成（Node.js/Python/Ruby/.NET）。

---

# 二、CloudWatch ⭐⭐⭐

## Metrics 基础 ⭐⭐

**标准 vs 高分辨率**：Standard 粒度 60 秒（默认，`StorageResolution=60`）；High Resolution 粒度 1/5/10/30 秒（`StorageResolution=1`）。

**Metric 保留**：1 分钟数据保留 **15 天**；5 分钟数据保留 **63 天**；1 小时数据保留 **15 个月（约 455 天）**。超过后自动删除。

**EC2 默认 monitoring** ⭐：Basic monitoring（免费）5 分钟粒度；Detailed monitoring（收费）1 分钟粒度。⚠️ **EC2 默认不监控 memory 和 disk**！需要装 CloudWatch Agent。

## 自定义 Metrics ⭐⭐

```bash
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "OrdersProcessed" \
  --value 42 \
  --dimensions "Environment=prod" \
  --unit Count \
  --storage-resolution 1
```

三大组件：Namespace/Metric Name/Dimensions（维度，最多约 30 个）。

## CloudWatch Agent ⭐

EC2 默认不收集的，Agent 才能收集：✅ Memory utilization、✅ Disk used %、✅ Swap、✅ Process metrics、✅ 自定义 log files → CloudWatch Logs。

## CloudWatch Logs ⭐⭐⭐

**层次结构**：Log Group → Log Stream → Log Event。

**Retention** ⭐：默认 Never expire（永久，要钱！），生产必须设 retention。

**Subscription Filters** ⭐⭐：实时流出 logs 到其他服务（Kinesis/Lambda/Firehose/OpenSearch）。⚠️ 一个 log group 最多 2 个 subscription filter。

**Metric Filters** ⭐：从 log 中提取数字生成 metric。Pattern 语法：简单文本（`"ERROR"`）；JSON（`{ $.statusCode = 500 }`）。

**CloudWatch Logs Insights** ⭐⭐：强大的日志查询语言（类似 SQL）：

```sql
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100
```

复杂示例（提取字段做统计）：

```sql
fields @timestamp, @message
| parse @message "duration=*ms" as duration
| stats avg(duration), max(duration), pct(duration, 99) by bin(5m)
```

限制：单查询最多扫描约 10 GB；查询超时 60 分钟；最多约 20 个 log group。

## Embedded Metric Format (EMF) ⭐⭐

**= 在 log 里嵌入 metric，CloudWatch 自动提取**。传统方式需要应用先调 `put-metric-data` API（同步，慢）；EMF 只需应用写 log（异步），CloudWatch 自动提取 metric。

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

**EMF 优势**：✅ 异步（写 log 即可，不阻塞应用）；✅ 支持高分辨率（1 秒）；✅ 保留 high-cardinality 上下文（OrderId 在 log 里）；✅ 1 个 EMF log 最多约 100 metrics + 30 dimensions。

## CloudWatch Alarms ⭐⭐⭐

**状态**：**OK**（metric 在阈值内）、**ALARM**（超阈值，触发）、**INSUFFICIENT_DATA**（数据不够无法判断）。

**三个关键设置**：**Period**（评估 metric 的时间粒度，秒为单位——例如 1 分钟一个数据点）；**Evaluation Period**（评估时看最近多少个数据点）；**Datapoints to Alarm**（评估期内有多少个数据点超标才触发 ALARM，超标的数据点不需要连续，但都要落在最近的 Evaluation Period 数据点范围内）。

**Actions**：**SNS**（通知：email/SMS/Lambda）；**EC2 Auto Recovery**（自动重启 instance）；**Auto Scaling**（触发 ASG 伸缩）；**Lambda invoke**（较新功能，直接调 Lambda）。

**Composite Alarms** ⭐：多个 alarm 的逻辑组合，如 `(HighCPU AND HighLatency) OR LowDiskSpace`。减少 alarm 噪音+表达复杂条件。

**Anomaly Detection**：ML 学习 metric 的 baseline，超出 band 触发 alarm。适合没有固定阈值的场景。

## CloudWatch Synthetics ⭐

**= "canary"**——定时模拟用户行为。类型：Heartbeat（简单 HTTP）、API Canary、Broken Link、GUI Workflow（Puppeteer）、Visual Monitoring。结果：metrics+logs+截图+HAR file；触发频率最快约 1 分钟。

⭐ "站点 down 立即知道、不等用户报告" → Synthetics。

## 其他

**Container Insights**：ECS/EKS 容器层监控（自动收集容器、task、pod、cluster 层 metrics）。**Application Insights**：自动监控应用堆栈（Java、.NET、SQL Server）。**ServiceLens**：X-Ray+CloudWatch 整合视图（Service Map+metrics+logs+alarms）。

---

# 三、CloudTrail ⭐⭐

## CloudTrail 是什么

**= 记录"AWS 账户里发生了什么 API 调用"**。用于审计/合规（SOC、PCI）、安全调查、运维排错。

⭐ CloudTrail 在所有账户默认开启（记录最近 90 天 management events 到 Event History，免费）。

## 4 种事件类型 ⭐⭐⭐

**1. Management Events**（默认开启）：**"控制平面"操作**（配置资源）。例子：`iam:CreateRole`、`ec2:RunInstances`、`s3:CreateBucket`。子分类：Write events（修改资源）、Read events（只读查询）。第一份 management events trail 免费。

**2. Data Events**（必须手动开）：**"数据平面"操作**（操作 resource 内容）。例子：`s3:GetObject`、`lambda:Invoke`、`dynamodb:GetItem`。⚠️ 数据事件量极大，默认关闭。

**3. Insights Events** ⭐：AWS 自动检测的"异常 API 模式"。自动学 baseline，异常时（突然大量 delete、IAM 操作激增）生成 Insight event。

**4. Network Activity Events**（较新功能）：VPC endpoint 上的 API 流量分析。

## CloudTrail Event History ⭐

**= 内置的 90 天事件浏览器**。✅ 免费，所有账户默认开启；✅ 保留 90 天；❌ 只 management events；❌ 单账户单 region。

## Trails ⭐⭐

Single-region trail（单一 region）；**Multi-region trail**（推荐，所有 region）；**Organization trail**（整个 AWS Organization）。

**Trail 日志去哪**：S3 bucket（必需，主存储，批量周期性上传）；CloudWatch Logs（可选，实时分析，秒级）；EventBridge（可选，实时 < 1 秒，触发自动化响应）。

## Log File Integrity Validation ⭐

**= 防止 CloudTrail log 被篡改**。每小时生成 digest file（SHA-256 hash），合规场景必开：`aws cloudtrail validate-logs --trail-arn ... --start-time ...`。

## CloudTrail Lake

**= 托管的"事件 data lake"**。SQL 查 trail data；聚合多账户、多 region 事件；不可变存储（write-once）；保留期可选（如 1 年/7 年）。

vs S3+Athena：Lake 全托管；S3+Athena 自己管、更灵活。

## EventBridge 集成 ⭐

```
CloudTrail event(如:有人创建 root user)
  ↓ EventBridge rule
Lambda function(自动撤销 / 通知 security team)
```

---

# 四、三服务定位

| 服务 | 定位 |
|---|---|
| **追踪+分布式调用链+服务拓扑** | X-Ray |
| **数值监控+日志+告警** | CloudWatch |
| **API 调用审计+合规** | CloudTrail |

**常见组合场景**：应用慢找瓶颈服务 → X-Ray Service Map；Lambda 频繁报错自动通知 → CloudWatch Logs filter → SNS/Alarm；某 user 偷偷创建 bucket → CloudTrail event+EventBridge → Lambda；全链路一站式排查 → ServiceLens；端到端追踪 API GW+Lambda → 两个都开 Active Tracing；自定义 metric 高频上报 → EMF（异步）；容器层监控 → Container Insights；模拟用户 24/7 监控 → Synthetics canary。
