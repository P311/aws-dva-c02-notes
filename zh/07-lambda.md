# Module 7：Lambda（基础 + 性能 + 部署 + 监控）

> **覆盖日期**：5/4、5/5
>
> English version: [`en/07-lambda.md`](../en/07-lambda.md)

---

## 一、Lambda Part 1 — 基础、调用、Triggers、Layers、ENV

### Lambda 是什么

**= AWS 的 serverless 计算服务**。你只关心代码，AWS 管所有底层（server、OS、scaling、patching）。代码不跑时不收钱。

**核心特点**：✅ 完全 serverless；✅ 按用量付费（请求数 + 执行时长 × 内存）；✅ 自动 scale（0 到几千并发）；✅ 支持多种语言：Node.js、Python、Java、Go、Ruby、.NET、Custom Runtime；✅ 集成 200+ AWS 服务作为触发源。

**适合 Lambda**：✅ 短任务（< 15 分钟）、✅ 事件驱动、✅ 流量不稳定/突发、✅ 间歇性任务（定时报表、cron）、✅ 微服务后端。

**不适合 Lambda**：❌ 长时间运行的任务（> 15 分钟）、❌ 持续 24/7 高负载（EC2/Fargate 更便宜）、❌ 重型计算（模型训练、大文件视频转码）、❌ WebSocket 长连接、❌ 需要 GPU 的工作负载。

### Lambda 关键限制 ⭐⭐⭐

| 项目 | 限制 |
|---|---|
| **超时** | 1 秒 - **900 秒（15 分钟）**，默认 3 秒 |
| **内存** | **128 MB - 10,240 MB（10 GB）**，1 MB 增量 |
| **vCPU 比例** | 在 1,769 MB 时约等于 1 vCPU（纯比例分配） |
| **临时存储 /tmp** | **512 MB - 10,240 MB（10 GB）**，默认 512 MB |
| **同步调用 payload** | **6 MB**（请求和响应） |
| **异步调用 payload** | 官方近期已从 256 KB 上调至 **1 MB** ⭐较新变化 |
| **环境变量总大小** | **4 KB** |
| **部署包（直传 zip）** | **50 MB**（zipped） |
| **部署包（via S3）** | **250 MB**（unzipped，含 layers） |
| **部署包（容器镜像）** | **10 GB** |
| **Layers per function** | **5 个** |
| **总 unzipped 大小** | function + 所有 layers ≤ **250 MB** |
| **Concurrent executions（默认）** | **1,000 per region per account**（共享） |

**记忆口诀**："15 / 10 / 6 / 1 / 4" = 15 分钟 / 10GB / 6MB sync / 1MB async / 4KB env。

⚠️ **注意**：异步 payload 限额提升是较新的变化，考试题库可能还停留在旧值 256 KB，如果选项里没有 1 MB，选 256 KB。以上所有数字建议以 AWS 官方文档最新版本核实。

### Execution Model ⭐⭐⭐

**三种调用方式**：

```
1. Synchronous(同步,RequestResponse) — 调用者等待返回,客户端处理重试
2. Asynchronous(异步,Event) — Lambda 排队后立刻返回成功,Lambda 自动重试
3. Event Source Mapping(轮询模式) — Lambda 服务主动 poll 流/队列,批量处理
```

**1. Synchronous（同步）— InvocationType=RequestResponse**

典型源：✅ API Gateway、✅ ALB、✅ CloudFront（Lambda@Edge）、✅ Cognito 触发器、✅ CLI/SDK 直接调用、✅ Lex/Alexa/Kinesis Firehose 转换、✅ Step Functions（默认同步）。

特性：payload 上限 **6 MB**；错误处理**调用者自己重试**（Lambda 不自动重试）；调用者看得到完整响应。

**2. Asynchronous（异步）— InvocationType=Event**

典型源：✅ S3、✅ SNS、✅ EventBridge、✅ CloudWatch Events/Logs、✅ SES、✅ CodeCommit。

特性：payload 上限 **1 MB**（旧值 256 KB）；**Lambda 自动重试**：默认 2 次重试，共 3 次执行（第 1 次失败后等 1 分钟，第 2 次等 2 分钟）；可配置 `MaximumRetryAttempts`（0-2）；可配置 `MaximumEventAgeInSeconds`（60 秒到 6 小时）；重试用尽 → 进入 **DLQ** 或 **Destination on failure**；客户端只看到"已排队"。

**3. Event Source Mapping（轮询）**

典型源（全是流/队列）：✅ SQS（标准 + FIFO）、✅ Kinesis Data Streams、✅ DynamoDB Streams、✅ Amazon MQ、✅ Self-managed Kafka / Amazon MSK。

特性：Lambda 服务做内部 polling，不消耗你的并发做轮询；**批量处理**（`BatchSize` 配置）；batch window（等 N 秒攒一批）；至少一次投递（at-least-once，可能重复）；错误处理：整个 batch 重试/丢弃/DLQ。SQS 特殊：消息处理失败不删 → 等 visibility timeout 过期 → 重新可见。

**速记表**：

|  | Sync | Async | Event Source Mapping |
|---|---|---|---|
| 调用方式 | 等待返回 | 排队后立返回 | Lambda 拉数据 |
| Payload | 6 MB | 1 MB | 视服务而定 |
| 重试 | **客户端管** | **Lambda 自动** | **Lambda 批量重试** |
| 典型源 | API GW / ALB | S3 / SNS / EventBridge | SQS / Kinesis / DDB Streams |

**判断**："S3 上传触发 Lambda，失败了怎么办" → S3 是异步，Lambda 自动重试 2 次，然后 DLQ/Destination；"API Gateway 调用 Lambda 失败" → 同步，客户端重试；"想从 SQS 处理消息" → Event Source Mapping（不是 Trigger）。

### Triggers（触发源）

**两种机制**：**Push 模型**（S3、SNS、API Gateway、ALB、Cognito、CloudFront 等——Trigger 配置在源服务里，自动给 Lambda 加 resource-based policy）；**Pull 模型 = Event Source Mapping**（SQS、Kinesis、DynamoDB Streams、Kafka、MSK——Mapping 配置在 Lambda 里，Lambda 的 execution role 需要相应权限）。

### IAM 集成 ⭐

**Execution Role**：Lambda 运行时假装的角色，决定 Lambda 能调用什么 AWS 服务。

Trust policy（谁能 assume）：

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "lambda.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

常用 managed policy：`AWSLambdaBasicExecutionRole`（写 CloudWatch Logs）、`AWSLambdaVPCAccessExecutionRole`（VPC 内）、`AWSLambdaSQSQueueExecutionRole`、`AWSLambdaDynamoDBExecutionRole`、`AWSLambdaKinesisExecutionRole`、`AWSXRayDaemonWriteAccess`。

最佳实践：❌ 不要给 `:*`；✅ 用 AWS managed policy + 自定义最小化 inline policy；✅ 每个 Lambda 一个独立 role。

**Resource-based Policy**：控制谁能调用这个 Lambda function。

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

用途：跨账户调用 Lambda；AWS 服务调用 Lambda（控制台创建 trigger 时自动加）。**陷阱**：总是加 SourceArn/SourceAccount Condition，防止 confused deputy。

### Environment Variables

**关键限制**：总大小 **4 KB**；key 只能字母/数字/下划线，首字符必须字母。

**默认加密**：所有 ENV var 总是用 KMS 加密 at rest，默认用 AWS managed key（`aws/lambda`，免费），可换 customer managed key。

**注意**：⚠️ ENV var 在 Lambda console 里默认明文显示（只 at-rest 加密）；想要 in-transit 加密 → 启用 "Encryption helpers"；高敏感数据（密码、API key）→ 用 Secrets Manager 或 SSM Parameter Store，不要直接放 ENV。

**保留的环境变量**（Lambda 自动设置，不能自己覆盖）：`AWS_REGION`、`AWS_LAMBDA_FUNCTION_NAME`、`AWS_LAMBDA_FUNCTION_VERSION`、`AWS_LAMBDA_FUNCTION_MEMORY_SIZE`、`AWS_EXECUTION_ENV`、`_HANDLER`、`_X_AMZN_TRACE_ID`、`LAMBDA_TASK_ROOT`、`LAMBDA_RUNTIME_DIR`、`TZ`。

### Lambda Layers ⭐⭐

**= 把"共享代码/依赖"打包成可复用的层**。解决：每个 Lambda 都打包同样依赖 → zip 包大、上传慢；多个 Lambda 共享工具库 → 改一次要更新所有 function。

**限制**：每个 function 最多 **5 个** layers；function + 所有 layers unzipped **≤ 250 MB**；Layer zip 直传上限 50 MB。

**目录结构**：Layer 解压后放到 `/opt`——Node.js: `/opt/nodejs/node_modules`；Python: `/opt/python`；Java: `/opt/java/lib`；Ruby: `/opt/ruby/gems/...`；所有 runtime: `/opt/bin`（可执行文件）。

Python 打 Layer 标准流程：

```bash
mkdir -p python
pip install requests -t python/
zip -r layer.zip python/
aws lambda publish-layer-version --layer-name my-layer --zip-file fileb://layer.zip
```

**版本**：Layer **不可变**，发布后不能改；修改 → 发布新版本；Lambda 引用具体版本号（`my-layer:3`）；删除某个 Layer 版本不影响已引用它的 function。

**跨账户共享**：`aws lambda add-layer-version-permission --layer-name my-layer --version-number 3 --principal 111122223333 --action lambda:GetLayerVersion`。

---

## 二、Versions and Aliases ⭐⭐⭐

### Versions（版本）

**= function 代码 + 配置的不可变快照**。`$LATEST` = 最新的草稿版本（可改）；Published version（v1、v2...）= 不可变（代码、ENV、内存、超时、Layer 配置全部冻结）；发布版本会得到版本特定的 ARN：`arn:aws:lambda:us-east-1:123:function:my-fn:3`；版本号只增，不重用。

### Aliases（别名）

**= 指向某个 version 的命名指针**。常见模式：`prod → version 5`、`staging → version 6`、`dev → $LATEST`。Alias 可变（可以指向不同版本）；有自己的 ARN；API Gateway/Event Source 应该指向 alias，不直接指向 version；部署新版本 → 只更新 alias 指针 → 立即切换。

### Weighted Alias（Canary / Blue-Green）

**= 一个 alias 可以同时指向两个 version，按权重分流**。

限制：最多指向 **2 个 version**；两个 version 必须已发布（不能是 `$LATEST`）；必须有相同的 execution role 和 DLQ 配置。

典型流程：部署新代码 → 发布 version 6 → alias 配 `90% → v5, 10% → v6` → 监控 v6 指标/错误率 → 没问题就逐步调到 100% → 出问题立刻调回 100% v5。

CodeDeploy 预定义 deployment 配置：`Canary10Percent5Minutes`、`Canary10Percent10Minutes`、`Linear10PercentEvery1Minute`、`Linear10PercentEvery2/3/10Minutes`、`AllAtOnce`。

---

## 三、Concurrency（并发）⭐⭐⭐

**默认账户级限制**：1,000 concurrent executions per region per account（所有 function 共享），可申请提高配额。

**计算公式**：所需 concurrency ≈ 请求数/秒 × 平均执行时长（秒）。

**三种类型**：

**1. Unreserved**（默认，共享池）：所有未配置 reserved 的 function 共享总量减去 reserved 的部分；AWS 强制至少留 100 给 unreserved。

**2. Reserved Concurrency**（预留并发）：**同时是上限和下限**——这个 function 永远有 N 个并发可用（其他 function 不能用），同时最多也只能用 N 个（超过会 throttle）；不收额外费用；用途：保护关键 function、限制下游连接数、设为 **0** 完全 disable function。

**3. Provisioned Concurrency**（预置并发）：预先初始化 N 个执行环境，处于"hyper-ready"状态；**避免冷启动**（双位数毫秒响应）；收费（按预置时长计费，不管用没用）；必须配置在 alias 或 version（不能 `$LATEST`）；适合生产环境延迟敏感应用、可预测的流量高峰。

**Reserved vs Provisioned**：

|  | Reserved | Provisioned |
|---|---|---|
| 上限/下限 | 既是上限也是下限 | 只是"预热"数量 |
| 冷启动 | 仍然有（没预热） | **没有**（已预热） |
| 费用 | 免费 | 按预置时长计费 |
| 用途 | 限流/保留容量 | 性能/低延迟 |
| 配置在 | function 级 | alias/version 级 |

**判断**："API Gateway 调 Lambda，首次请求很慢" → **Provisioned Concurrency**；"怕这个关键 Lambda 被流量洪峰挤掉" → **Reserved Concurrency**；"Lambda 把 RDS 连接数打爆" → Reserved Concurrency 限流（也可以加 RDS Proxy）；"想紧急停用这个 function" → Reserved Concurrency 设为 **0**。

**Throttling**：触发条件——超过 account 级并发上限、超过 function 的 reserved/provisioned concurrency。同步调用返回 `429 TooManyRequestsException`；异步调用 Lambda 自动重试；Event Source Mapping 会减慢 polling。

---

## 四、Lambda Destinations ⭐

**= 异步调用完成后（成功或失败）把记录路由到其他服务**。4 种 Destination：SQS queue、SNS topic、Lambda function（链式调用）、EventBridge event bus。配置 `OnSuccess` / `OnFailure`。

**vs DLQ**：

|  | DLQ | Destinations |
|---|---|---|
| 触发时机 | 仅 OnFailure | OnSuccess **和** OnFailure |
| 支持目标 | SQS / SNS | SQS / SNS / Lambda / EventBridge |
| 元数据 | 简单 | 完整（请求/响应/错误细节） |
| 现状 | Legacy | **推荐** |

⚠️ Destinations 只对异步调用和 Event Source Mapping（部分）有效，同步调用无 Destination。

---

## 五、Lambda + VPC ⭐

**默认情况**：Lambda 默认在 AWS 管理的 VPC 里，可以访问公网，**不能**访问你的 VPC 资源（如 private RDS）。

**配置 Lambda 在你的 VPC**：Lambda 服务在你 VPC 创建 **Hyperplane ENI**（共享 ENI 池，冷启动已优化）；ENI 用 VPC 的 IP 地址池。

**陷阱**：⚠️ VPC 里的 Lambda 默认不能访问公网！需要配 NAT Gateway（在 public subnet）；⚠️ execution role 必须有 `AWSLambdaVPCAccessExecutionRole`；⚠️ 不要把 Lambda 放 public subnet（Lambda 没有 public IP，没用）。

**与 IAM 的连接**：访问 S3 等优先用 VPC Endpoint（走 AWS 内网，免 NAT 费用）。

---

## 六、Cold Start（冷启动）

**冷启动阶段**：① Init phase（下载代码/容器镜像、启动 runtime、运行 init 代码）；② Invoke phase（真正调 handler）。

**典型冷启动延迟（参考量级）**：Python ~100-500 ms；Node.js ~100-300 ms；Java/.NET 明显更慢（数秒级，JVM/CLR 启动慢）。

**减少冷启动的方法**：① Provisioned Concurrency（预热）；② SnapStart（Java/Python/.NET 支持）；③ 减小部署包；④ 避免重型依赖；⑤ 延迟初始化；⑥ 保持 warm（非官方推荐做法）；⑦ 选轻量 runtime。

**Init 代码 vs Handler 代码**：

```python
# init 代码 — 容器复用时只执行一次
import boto3
db_client = boto3.client('dynamodb')   # 全局,只创建一次

def lambda_handler(event, context):    # 每次请求都执行
    response = db_client.get_item(...)
    return response
```

最佳实践：✅ 全局初始化 SDK client；✅ 全局缓存配置/静态数据；❌ 不要在 handler 内部 import 或创建 client。

---

## 七、Lambda 监控（基础）

**CloudWatch Metrics**（自动，免费）：`Invocations`、`Errors`、`Duration`、`Throttles`、`ConcurrentExecutions`、`ProvisionedConcurrencyUtilization`。

**CloudWatch Logs**：每个 Lambda 一个 log group `/aws/lambda/<function-name>`；`console.log`/`print` 自动写入；execution role 必须有 `logs:CreateLogStream` + `logs:PutLogEvents` 权限。

**X-Ray**：启用 Active tracing → Lambda 自动发送 trace；execution role 加 `AWSXRayDaemonWriteAccess`；可以看到 Lambda → DDB/S3/外部 API 的调用瀑布图。

---

## 八、Lambda Part 2 — 性能深入、部署、监控进阶

### Cold Start 完整阶段

```
1-4 = "Init Duration"(冷启动专属,只在新容器执行一次)
5   = "Duration"(每次调用都有)
```

**容器复用/Warm Start**：Lambda 跑完一次后容器不立刻销毁，保持一段时间等下次调用复用 → warm start，没有 init 阶段。复用不保证（AWS 随时可能销毁），空闲多久会冷 AWS 不公开。

### SnapStart ⭐⭐

**= 把初始化后的执行环境做 snapshot，后续从 snapshot 恢复，跳过 init 阶段**。基于 Firecracker MicroVM 的 snapshot 能力。

**支持范围**：✅ Java 11+、✅ Python 3.12+、✅ .NET 8+；❌ Node.js/Ruby/Go/Custom runtime；❌ Container image。

**限制**：❌ 不能与 Provisioned Concurrency 同时用（互斥）；❌ 不支持 EFS；❌ 不支持 ephemeral storage > 512 MB；❌ 只能用在 published version 或指向 published version 的 alias（不能 `$LATEST`）。

**收费**：Java 免费；Python/.NET 收两笔费用（caching cost + restoration cost）——建议删除不用的版本以省钱。

**Uniqueness 陷阱**：从同一个 snapshot 恢复多个执行环境 → 状态完全一样 → 如果 init 代码生成了 UUID、随机数等"应该唯一"的值，会被多个并发实例共享！解决：用 runtime hooks（`afterRestore` / `beforeCheckpoint`），在恢复后重新生成唯一值。

**SnapStart vs Provisioned Concurrency**：

|  | SnapStart | Provisioned Concurrency |
|---|---|---|
| 原理 | 跳过 init，从快照恢复 | 预先初始化好 N 个环境 |
| 冷启动 | 仍然是冷启动，但快 | **没有冷启动** |
| 收费 | Java 免费，其他付费 | 付费（预置时长） |
| 支持 runtime | Java/Python/.NET | 全部 runtime |
| 支持容器镜像 | ❌ | ✅ |
| 互斥 | ✅ 互斥 | ✅ 互斥 |

**选择**：Java 应用 + 突发流量 → SnapStart；真正低延迟生产应用 → Provisioned Concurrency；Node.js/Go 或容器镜像 → 只能用 Provisioned Concurrency。

### Container Image Support ⭐

**= 用 Docker 镜像作为 Lambda 部署方式**。为什么用：zip 部署上限 250 MB，大型依赖（ML 模型）容易爆；容器镜像**最大 10 GB**（大幅提升），可用 Dockerfile 定义环境，与现有 CI/CD 无缝集成。

**基础镜像**：AWS 提供各语言的 Lambda 基础镜像（Python、Node.js、Java、.NET、Ruby、Custom runtime）；自定义基础镜像需实现 Lambda Runtime API。

**镜像 vs Zip 对比**：

|  | Zip Package | Container Image |
|---|---|---|
| 大小上限 | 250 MB（unzipped） | **10 GB** |
| 部署源 | S3/直传 | **必须 ECR** |
| Layer 支持 | ✅ | ❌（直接打包到镜像） |
| SnapStart 支持 | ✅（Java/Python/.NET） | ❌ |
| 与现有 CI/CD | 弱 | **强**（Docker 工具链） |

### Function URLs ⭐⭐

**= 给 Lambda 一个直接的 HTTPS endpoint（不用 API Gateway）**。URL 形如 `https://<url-id>.lambda-url.<region>.on.aws/`。

**特性**：✅ HTTPS only；✅ 内置 CORS 支持；✅ 支持 IAM auth 或公开访问；✅ 可附加到 alias/`$LATEST`（不能附加到 published version）；✅ 支持 Streaming response；✅ 同步调用（payload 仍 6 MB 上限）。

**两种 Auth Type**：**NONE**（公开访问，必须在 resource policy 显式 allow `Principal: "*"` + `lambda:InvokeFunctionUrl`）；**AWS_IAM**（SigV4 签名，调用方需要 `lambda:InvokeFunctionUrl` 权限，适合内部 API/跨账户）。

**Function URL vs API Gateway**：

|  | Function URL | API Gateway |
|---|---|---|
| 复杂度 | 极简 | 复杂 |
| 价格 | **免费**（只 Lambda 钱） | 按请求计费 |
| Rate limiting | ❌ | ✅ |
| Custom domain | ❌ | ✅ |
| WebSocket | ❌ | ✅ |
| Streaming response | ✅ | 部分 |

**选择**：内部工具、原型、简单 webhook → Function URL；生产 API、需要复杂鉴权/限流/路由 → API Gateway。

⚠️ **陷阱**：Function URL CORS 配了之后，Lambda 函数的响应 header 里也要返回 CORS headers，否则浏览器仍报错。

### Lambda@Edge ⭐⭐

**= 在 CloudFront 边缘节点执行的 Lambda 函数**。部署位置：必须部署到 **us-east-1**，但执行在全球 CloudFront edge location。

**4 个触发点**：**viewer-request**（每个请求都跑）、**origin-request**（仅 cache miss）、**origin-response**（仅 cache miss）、**viewer-response**（每个请求都跑）。

**限制**（和普通 Lambda 不同）：Viewer 触发器超时 5 秒，函数大小 1 MB；Origin 触发器超时 30 秒，函数大小 50 MB。通用限制：❌ 不支持环境变量、Layers、X-Ray、reserved/provisioned concurrency、ARM64、VPC、DLQ；仅支持 **Node.js 和 Python**；必须用 published version。

**典型用例**：A/B 测试、JWT/自定义鉴权、URL 重写、基于地理位置的内容、响应 header 注入（安全 header）、图片动态处理、多 origin 路由。

### CloudFront Functions ⭐

**= CloudFront 边缘的轻量级 JavaScript 函数**（比 Lambda@Edge 更轻）。

|  | CloudFront Functions | Lambda@Edge |
|---|---|---|
| 语言 | **仅 JavaScript** | Node.js + Python |
| 执行位置 | **所有 edge location**（数百个） | Regional edge cache（数量少得多） |
| 触发点 | **viewer-request / viewer-response** | 4 个（都支持） |
| 最大执行时长 | **亚毫秒级** | 数秒级 |
| 网络访问 | ❌ 不行 | ✅ 可以 |
| 价格 | **极便宜** | 比普通 Lambda 贵 |

**记忆**：CloudFront Functions = 极简 + 极快 + 极便宜（只能 viewer 触发，没有网络访问）；Lambda@Edge = 强大但慢且贵（全 4 个触发点，能调外部 API）。

### Lambda Insights（增强监控）

**= CloudWatch 的 Lambda 增强监控扩展**。额外指标：CPU 使用率、内存利用率细节、网络流量、磁盘 I/O。启用方式：加 AWS 提供的 Lambda Layer（`LambdaInsightsExtension`），execution role 加 `CloudWatchLambdaInsightsExecutionRolePolicy`。按 metric 数量收费。

### X-Ray ⭐⭐⭐

**= AWS 分布式追踪服务**。

**核心概念**：Trace（一个完整请求的端到端记录）→ Segment（一个服务对该请求的处理）→ Subsegment（段内的细分操作）。Service Map：自动从 trace 生成的可视化拓扑图。

**Lambda + X-Ray 集成**：**Active Tracing**（Lambda 自动发送 trace，即使没配置 X-Ray SDK）；**PassThrough**（不主动 trace，只透传 tracing header）。启用：Configuration → Monitoring → Active tracing；execution role 需要 `AWSXRayDaemonWriteAccess`。

启用 Active Tracing 后，X-Ray 看到 2 个 segment：`AWS::Lambda`（Lambda 服务自己的处理）和 `AWS::Lambda::Function`（函数代码本身的执行）。

**X-Ray SDK 细粒度追踪**（Python 例子）：

```python
from aws_xray_sdk.core import xray_recorder, patch_all

patch_all()  # 自动追踪 boto3 / requests / sqlite3 等

@xray_recorder.capture('my_function')
def process_order(order_id):
    pass

def lambda_handler(event, context):
    subsegment = xray_recorder.begin_subsegment('custom-work')
    subsegment.put_annotation('user_id', '12345')
    subsegment.put_metadata('event', event)
    xray_recorder.end_subsegment()
```

**Annotations vs Metadata** ⭐：

|  | Annotations | Metadata |
|---|---|---|
| 数据类型 | string/number/boolean | 任意可序列化 |
| 是否被索引 | ✅ 索引（可用 filter expression 搜） | ❌ 不索引 |
| 用途 | **搜索/过滤 trace** | **存详细上下文** |
| 数量限制 | 每个 trace 最多 50 | 无明确限制（单 segment doc < 64 KB） |

**陷阱（Lambda 特殊）**：⚠️ parent segment 由 Lambda 服务管理，不能直接给它加 annotations/metadata，必须创建 subsegment。

**Sampling**：默认规则——第一秒的第一个请求 100% 采样，之后 5% 采样；可自定义规则（基于 URL、HTTP 方法、service name）。

> **提醒**：AWS X-Ray SDK / Daemon 官方已进入维护模式，推荐迁移到 OpenTelemetry，但 DVA 考试题库仍以 X-Ray SDK 为基础。以官方文档核实当前状态。

### CloudWatch Logs Insights ⭐⭐

**= 用类 SQL 查询语言搜索/分析 logs**

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 25
```

关键命令：`fields`、`filter`、`sort`、`limit`、`stats`（聚合）、`parse`、`bin(N)`（时间分桶）。

Lambda 自动发现的字段：`@timestamp`、`@message`、`@requestId`、`@duration`、`@billedDuration`、`@maxMemoryUsed`、`@memorySize`、`@type`、`@initDuration`。

示例查询（找冷启动率）：

```sql
filter @type = "REPORT"
| stats sum(strcontains(@message, "Init Duration")) / count(*) * 100 as coldStartPct,
        avg(@duration)
        by bin(5m)
```

### DLQ vs Destinations 对比

|  | DLQ | Destinations |
|---|---|---|
| 触发 | 异步调用失败用尽重试 | 异步成功 + 失败都行 |
| 目标 | SQS/SNS | SQS/SNS/Lambda/EventBridge |
| 推荐度 | Legacy | **首选** |

### 故障排查速查表

| 症状 | 可能原因 | 排查方法 |
|---|---|---|
| 首次调用慢，后续快 | 冷启动 | logs 看 `Init Duration`，启用 Provisioned Concurrency / SnapStart |
| 大量 timeout | 函数没在超时前完成 | 加超时时间/优化代码/加内存 |
| 大量 throttle | 超过 concurrency 上限 | 看 `Throttles` 指标，加 reserved 或申请配额 |
| OOM | 内存不够 | logs 看 `Max Memory Used`，加 memory size |
| VPC Lambda 调不到外网 | 没配 NAT Gateway | private subnet + NAT GW |
| Lambda 调 RDS 连接耗尽 | 每个 invocation 一个新连接 | RDS Proxy + Reserved Concurrency |
| DDB throttle 但容量没满 | DDB hot partition | 看 CloudWatch DDB 分区分布 |
| 异步事件丢失 | 重试用尽且没配 DLQ | 配 DLQ 或 Destinations |
| API Gateway 调 Lambda 502 | Lambda 内部错误/超时 | 看 Lambda logs |
