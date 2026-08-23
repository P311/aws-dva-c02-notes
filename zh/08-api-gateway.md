# Module 8：API Gateway（REST / HTTP / WebSocket + 鉴权）

> **覆盖日期**：5/6、5/7
>
> English version: [`en/08-api-gateway.md`](../en/08-api-gateway.md)

---

## 一、API Gateway Part 1 — REST / HTTP / WebSocket、Integrations、Stages、CORS

### API Gateway 是什么

**= AWS 全托管的 API 服务**，作为应用的"front door"：创建、发布、维护、监控、保护 API；支持 RESTful API、HTTP API、WebSocket API；自动处理流量管理、鉴权、版本管理；与 AWS 生态深度集成（Lambda、Cognito、IAM、CloudWatch、X-Ray、WAF 等）。

**典型用例**：Lambda 后端的 RESTful API（serverless 架构）；在传统 EC2/ECS 应用前面加 API 层；WebSocket 实时双向通信（聊天、游戏、推送）；暴露 AWS 服务给前端；API 货币化（API keys + usage plans）。

### 三种 API 类型对比 ⭐⭐⭐

|  | **REST API** | **HTTP API** | **WebSocket API** |
|---|---|---|---|
| 协议 | HTTP/HTTPS（无状态） | HTTP/HTTPS（无状态） | WebSocket（有状态，双向） |
| 价格 | 较贵 | **便宜约 70%** | 按消息 + 连接时长计费 |
| 延迟 | 较高 | **比 REST 低约 60%** | 低，持久连接 |
| Caching | ✅ 支持 stage cache | ❌ | ❌ |
| API Keys + Usage Plans | ✅ | ❌ | ❌ |
| Per-client throttling | ✅（usage plan） | ❌（只 stage 级） | ❌ |
| Request validation | ✅ | ❌ | / |
| Mapping Templates (VTL) | ✅ | ❌（只简单变量替换） | ✅ |
| WAF 集成 | ✅ | ❌ | ❌ |
| Authorizers | IAM、Cognito、Lambda | IAM、JWT、Lambda | IAM、Lambda |
| Endpoint types | Edge / Regional / Private | Regional only | Regional only |
| Canary deployment | ✅ | ❌ | ❌ |
| 私有 API (VPC) | ✅（Private endpoint） | ❌ | ❌ |
| 协议双向 | ❌ 单向 | ❌ 单向 | ✅ **双向**（server 主动推） |

> 具体价格数字建议以 AWS 官方定价页最新数据为准，此处只给相对比例。

### 选择决策 ⭐

**用 REST API 当**：需要 API keys + per-client throttling（多租户 API、API 货币化）；需要 caching；需要 request validation；需要 canary deployment；需要 WAF；需要 edge-optimized endpoint（全球用户）；需要 private endpoint（VPC 内访问）；需要复杂 mapping templates。

**用 HTTP API 当**：简单的 Lambda 后端；想省钱、想快；主要用 JWT authorizer（Cognito 或第三方 OAuth 2.0）；不需要上面那些 REST 高级功能。

**用 WebSocket API 当**：实时双向通信（聊天、协作编辑）；服务器主动推送（通知、价格更新）；多客户端广播（直播、游戏）。

**价格结构提醒**：HTTP API 按 512 KB 增量计费，REST API 不分——如果请求 + 响应体积较大，HTTP API 反而可能比 REST 贵。DVA 题里"省钱" + "简单 RESTful API"经常指向 HTTP API。

### Endpoint Types（REST API 专属）⭐⭐

REST API 创建时必须选 endpoint type（HTTP API 只有 Regional）：

**1. Edge-Optimized（默认）**：请求路由到最近的 CloudFront edge location；AWS 自动管 CloudFront 配置；适合全球分布的客户端；不能跨 region（API 还是部署在某个 region）。

**2. Regional**：客户端直接连到某个 region 的 API Gateway，没有 CloudFront 在中间。适合：客户端就在同一个 region；想自己控制 CloudFront；多 region 部署 + Route 53 latency-based routing。

**3. Private**：API 只能从 VPC 内部访问，通过 interface VPC Endpoint (PrivateLink)，完全不暴露到 internet。适合内部 microservices、严合规场景。

**判断**："客户端遍布全球" → Edge-optimized；"客户端在同 region 的 EC2" → Regional（避免不必要的 CloudFront 跳数）；"API 不能暴露 internet，只 VPC 内访问" → Private + Interface VPC Endpoint。

### Integration Types ⭐⭐⭐

| Type | 后端 | 描述 |
|---|---|---|
| **AWS_PROXY**（Lambda Proxy） | Lambda | **推荐方式**：整个请求原样传给 Lambda，Lambda 返回带 status/headers/body 的对象，无需 mapping template |
| **AWS**（Lambda Custom / AWS Service） | Lambda 或任何 AWS 服务 | 需要 mapping template 转换请求/响应，例：直接调用 DynamoDB/Kinesis，不经 Lambda |
| **HTTP_PROXY** | 任意 HTTP endpoint | 转发整个请求，不做转换 |
| **HTTP** | 任意 HTTP endpoint | 转发请求，支持 mapping template 做转换 |
| **MOCK** | 没有后端 | API Gateway 直接返回静态响应（开发期占位、CORS preflight） |
| **VPC Link** | VPC 内 NLB/ALB | 私有集成，API GW → VPC 内服务 |

**Lambda Proxy vs Lambda Custom**：

Lambda Proxy（AWS_PROXY）——整个 HTTP 请求打包成 event JSON 传给 Lambda，Lambda 返回固定格式：

```json
{
  "statusCode": 200,
  "headers": { "Content-Type": "application/json" },
  "body": "{...}"
}
```

✅ 无需 mapping template；✅ Lambda 直接看到原始请求所有信息；❌ 强耦合 API Gateway 的 event 格式。

Lambda Custom（AWS）——需要写 mapping template（VTL）转换请求/响应；✅ 灵活；❌ 复杂，VTL 模板难维护。

**AWS Service Integration**：API Gateway 直接调用 AWS 服务，不需要 Lambda。适合：API → DynamoDB（简单 CRUD）、API → Kinesis、API → SQS、API → S3、API → Step Functions。优点：省钱、低延迟（无冷启动）；缺点：必须写 VTL、复杂业务逻辑搞不定。

**VPC Link**：REST API 的 VPC Link 关联到 NLB；HTTP API 的 VPC Link 更灵活，可关联 ALB、NLB 或 Cloud Map。典型场景：API Gateway 公开 API，后端是 VPC 私有 ECS 服务。

### Stages 和 Deployments ⭐⭐⭐

**Deployment = API 配置的不可变快照**。改了 API 必须 Deploy 才生效；每次 Deploy 创建一个不可变的 Deployment；Deployment 关联到一个 Stage（命名引用，如 dev/staging/prod）。

**关键特性**：每个 Stage 有独立的 URL：`https://{api-id}.execute-api.{region}.amazonaws.com/{stage-name}`；Stage 可以独立配置 caching、throttling、logging、stage variables、X-Ray；同一个 Deployment 可以关联到多个 Stage；**修改 API 后必须重新 Deploy 才生效**（常见考点 + 常见 bug）。

**Stage Variables** ⭐⭐：类似环境变量的 key-value 对，绑定在 Stage 上。最常见用法——不同 stage 调不同 Lambda alias：

```
API 配置 Lambda integration:
  Lambda ARN: arn:aws:lambda:...:function:myFn:${stageVariables.lambdaAlias}

Stage "dev"   → stageVariables.lambdaAlias = "dev"  → 调 myFn:dev (alias)
Stage "prod"  → stageVariables.lambdaAlias = "prod" → 调 myFn:prod (alias)
```

**陷阱**：如果用 stage variable 指 Lambda alias，API Gateway 必须有权限调每个 alias，必须给每个 alias 单独加 resource-based policy（`lambda:InvokeFunction`），否则部分 stage 报 500。

**Canary Deployment** ⭐⭐：在同一个 Stage 内分流到两个 deployment，用于灰度。**REST API 专属，HTTP API 不支持**。同一个 Stage URL，流量按百分比分到 production 和 canary；canary 可以有独立 stage variables、独立 cache。流程：创建 canary settings（percent = 10）→ 重新 deploy → 监控 metrics → 满意就 Promote canary，不满意就调 percent 到 0。

⚠️ 如果题目要求 canary deployment，选项里同时有 REST 和 HTTP API，选 **REST API**。

### Mapping Templates（REST API 专属）⭐

**= VTL（Velocity Template Language）模板，把请求/响应在 API GW 层转换**。用途：请求/响应转换、隐藏后端结构、错误响应规范化。

关键内置变量：`$input.body`、`$input.json('$.fieldName')`、`$input.params('paramName')`、`$context.requestId`、`$context.identity.sourceIp`、`$stageVariables.xxx`。

HTTP API 不支持完整 VTL，只支持简单的 `${request.path.xxx}` 变量替换。

### CORS ⭐⭐⭐

**= Cross-Origin Resource Sharing**，浏览器安全机制，限制跨域请求。浏览器跨域调用会先发 **preflight OPTIONS** 请求，API 必须返回正确 CORS headers，浏览器才放行实际请求。

关键 headers：`Access-Control-Allow-Origin`、`Access-Control-Allow-Methods`、`Access-Control-Allow-Headers`、`Access-Control-Allow-Credentials`、`Access-Control-Max-Age`。

**REST API 的 CORS 配置**（复杂）：给每个 resource 添加 OPTIONS method（用 MOCK integration）；在 method response 和 integration response 加 CORS headers；每个 method 的 200 响应也要加 `Access-Control-Allow-Origin`；必须 redeploy。控制台有 "Enable CORS" 快捷按钮，但仍需手动加到现有 method 的响应。

**HTTP API 的 CORS 配置**（简单）：在 API 级别一次配置；自动处理 preflight OPTIONS；不需要重新部署；⚠️ 配了 CORS，API Gateway 会忽略后端返回的 CORS headers。

**常见错误**：① Lambda Proxy Integration 的响应也要带 CORS headers（不然即使 preflight 通过，实际请求也会被浏览器拒绝）；② Custom Authorizer 在 OPTIONS preflight 上要求授权（浏览器的 preflight 不带 Authorization header，会直接 401——解决：OPTIONS method 不挂 authorizer）；③ `Allow-Credentials: true` 时不能用 `Allow-Origin: *`（浏览器规范禁止）。

### Throttling ⭐⭐

**默认 quota（全局，跨同 region 所有 API）**：数万级 RPS steady-state + 数千级 burst（token bucket 容量，部分老 region 默认更低）。算法：Token Bucket——每个 token 代表 1 个请求，桶以 rate 速率加 token，没 token 时返回 429。account-level rate 可以申请提高，但 burst 通常不可改。

**Throttling 层次**：① Per-client/per-method limits（usage plan）② Per-method overall limits（stage 配置）③ Stage-level limits ④ Account-level limits ⑤ AWS regional limits（硬上限）。每一层超限都会 throttle。

**Usage Plans + API Keys**（REST API 专属）⭐⭐⭐：给不同客户端配不同的速率限制和配额。架构：API Key（per-client）→ Usage Plan（throttle + quota 设置）→ API Stages。典型场景——API 货币化（Free/Pro/Enterprise 分层）。

⚠️ **HTTP API 不支持 API Keys + Usage Plans**（只 stage 级 throttle）。

**陷阱**：API Key 不是身份验证！别人拿到 key 就能用。要鉴权用 IAM/Cognito/Lambda Authorizer，API Key 只用于计量/限流。

### Integration Timeout

| API 类型 | 默认超时 | 备注 |
|---|---|---|
| **REST API** | 29 秒 | 可申请增加（较新变化，以官方文档为准） |
| **HTTP API** | 30 秒 | 硬上限 |
| **WebSocket API** | 29 秒（per route） | 硬上限 |

**陷阱**：Lambda 最长跑 15 分钟，但 API Gateway → Lambda 同步调用有 29-30 秒的超时上限。长任务正确架构：客户端 POST → API GW → Lambda 异步触发 + 立即返回 jobId → 客户端轮询 GET `/status/{jobId}`；或用 Step Functions/SQS + worker。

---

## 二、API Gateway Part 2 — Auth、Caching、Custom Domain、Monitoring

### Authorizers ⭐⭐⭐

API Gateway 支持 **4 种 authorizer 类型**：

| Authorizer | REST API | HTTP API | WebSocket API |
|---|---|---|---|
| **IAM**（SigV4 签名） | ✅ | ✅ | ✅ |
| **Cognito User Pools** | ✅（内置） | ✅（通过 JWT） | ❌ |
| **Lambda Authorizer**（Custom） | ✅（TOKEN + REQUEST） | ✅（REQUEST only） | ✅ |
| **JWT Authorizer**（OAuth/OIDC） | ❌ | ✅（原生） | ❌ |

**1. IAM Authorizer**：调用方必须是 AWS IAM principal，用 SigV4 签名请求。适用于内部应用、AWS 账号内微服务调用。IAM policy 控制 `execute-api:Invoke` 权限。优点：完全用 AWS IAM，自动 SigV4 签名（SDK 内置）；缺点：客户端必须有 AWS 凭证，浏览器 JS 难做签名。

**2. Cognito User Pools Authorizer**（REST API 专属内置）：用户登录 Cognito，拿 JWT token，带在 Authorization header 上。⚠️ **REST API + Cognito Authorizer 必须用 ID token**（不是 access token），错用 → 401。HTTP API 的 JWT Authorizer 可以用 access token（支持 OAuth scopes）。

**3. Lambda Authorizer** ⭐⭐⭐：自己写 Lambda 函数做鉴权，最灵活。两种类型（REST API）：**TOKEN Authorizer**（客户端传 token 在某个 header，可选 RegEx 初步校验，简单适合 JWT/Bearer token）；**REQUEST Authorizer**（Lambda 接收整个请求，可综合多参数判断）。⚠️ **HTTP API 只支持 REQUEST 类型**。

Lambda Authorizer 必须返回 IAM Policy：

```python
def lambda_handler(event, context):
    token = event['authorizationToken']
    if not is_valid(token):
        raise Exception('Unauthorized')
    return {
        'principalId': 'user123',
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [{
                'Effect': 'Allow',
                'Action': 'execute-api:Invoke',
                'Resource': event['methodArn']
            }]
        },
        'context': {'userId': '123', 'role': 'admin'}
    }
```

抛异常 → 客户端收 401；返回 Deny policy → 403；返回 Allow → 通过。

**Lambda Authorizer Caching** ⭐：默认开启。默认 TTL **300 秒（5 分钟）**，最大 TTL **3600 秒（1 小时）**，TTL=0 关闭缓存。缓存 key：TOKEN authorizer 用 token source header 值；REQUEST authorizer 用所有 identity sources 的组合。

**陷阱**：⚠️ Cache 是 **per stage** 的，不是 per resource/per method！同一个 token 调 `/cats` 后再调 `/dogs`，直接返回缓存的 policy。因此**返回的 policy 必须覆盖 API 内所有可能的 method**（用通配符 `arn:.../*/*/*`），否则会出现串权限的 bug。或者把 TTL 设为 0（每次都调 authorizer，贵但精确）。用 `flush-stage-authorizers-cache` API 一次清空整个 stage 的缓存。

**4. JWT Authorizer**（HTTP API 专属）⭐⭐：API Gateway 原生验证任何 OIDC/OAuth 2.0 提供商签发的 JWT。适合 Cognito User Pool、Auth0、Okta 等。工作机制：客户端带 JWT 在 Authorization header → API Gateway 验证签名、issuer、audience、过期时间 → 通过则放行，claims 注入 `$context.authorizer.claims.xxx`；可指定 OAuth Scopes 做更精细授权。**完全托管，零 Lambda 调用，免费，极快**。

**判断**：HTTP API + Cognito + 简单鉴权 → JWT Authorizer（不是 Lambda Authorizer）。

### Caching（REST API 专属）⭐⭐

**= 在 API Gateway 层缓存后端响应**。⚠️ **HTTP API 完全不支持 caching**。

关键参数：默认 TTL 300 秒，最大 TTL 3600 秒；最大 cached response 大小 **1 MB**；按小时计费（不管用多少次），不在 free tier；可选加密（AES-256 at-rest）；配置在 Stage 级，可被 method 级覆盖。

默认行为：Stage 启用 caching → 所有 GET method 自动缓存；POST/PUT/DELETE 不缓存（除非显式开）。

Cache Key 默认是整个 URL（path + query string），method 级可自定义（选择哪些 path/query/header/stage variable 作为 cache key 的一部分）。

**Cache Invalidation**：客户端发 `Cache-Control: max-age=0` header 强制刷新。默认任何客户端都能 invalidate（可能被恶意利用），最佳实践是启用 "Require authorization for cache invalidation"，要求 `execute-api:InvalidateCache` IAM 权限。

监控：`CacheHitCount`/`CacheMissCount`，理想命中率 > 80%。

**陷阱**：① 改完 cache 配置必须 redeploy 才生效；② POST/PUT 不会自动缓存；③ 改后端代码后 cache 还返回老结果 → 调小 TTL 或主动 invalidate。

### Custom Domain Names ⭐⭐

**= 用自己的域名替代默认的 `xxx.execute-api.region.amazonaws.com`**。要求：注册的 domain；ACM 申请/导入 SSL/TLS 证书；配置 base path mapping；DNS 建 alias/CNAME 记录。

**关键陷阱（endpoint 类型 vs ACM Region）** ⭐⭐：**Edge-optimized custom domain → ACM 证书必须在 us-east-1**（底层是 CloudFront）；**Regional custom domain → ACM 证书必须在 API 所在 region**。很多团队踩坑：在其他 region 部署 edge-optimized API，ACM 证书也建在那个 region → API Gateway 找不到证书 → 部署失败。

记忆：全球 = us-east-1（CloudFront 全球资源）；Regional = 同 region。

### Logging + Monitoring ⭐⭐

**CloudWatch Metrics**（自动，免费）：`Count`、`4XXError`、`5XXError`、`Latency`（总延迟）、`IntegrationLatency`（仅后端耗时）、`CacheHitCount`/`CacheMissCount`。

**关键诊断**：`Latency` ≈ `IntegrationLatency` → 后端慢；两者差距大 → API Gateway 自身开销（罕见）；`5XXError` 高 → 后端报错；`4XXError` 高 → 客户端问题。

**两种 Log**：**Execution Logs**（API Gateway 内部处理细节，写到 `API-Gateway-Execution-Logs/{api-id}/{stage}`，日志级别 OFF/ERROR/INFO）；**Access Logs**（自定义格式，类似 web server access log，可写到 CloudWatch Logs 或 Kinesis Data Firehose，适合发到 S3/长期分析）。

**Detailed CloudWatch Metrics**（method 级，要额外付费）：默认只有 stage 级/API 级 metrics；启用后每个 method 有独立 metrics，能精细诊断但按 method 数收费。

**X-Ray 集成**：启用 X-Ray Active Tracing on stage → API Gateway 自动给每个请求生成 trace ID，自动 instrument API GW segment；配合 Lambda Active Tracing → 端到端 trace（Client → API GW → Lambda → DynamoDB）。

### Request / Response Validation（REST API 专属）⭐

**= API Gateway 在调后端前，先校验请求格式**。⚠️ HTTP API 不支持。

校验内容：required parameters/headers；body schema validation（JSON Schema）。好处：不合法请求被 API GW 直接拒绝（400），不调 Lambda，省钱；减少后端 validation 代码。

配置：创建 Model（JSON Schema）→ 在 method 启用 validator，选 model → 选 validator 类型（只验 body / 只验 query+header / 全验）。

### Gateway Responses

**= 自定义 API Gateway 返回的错误响应内容**（401、403、429、503、504 等）。可以修改 status code、响应 body/header，添加 CORS header。**重要**：错误响应也要 CORS headers，否则浏览器会把错误本身挡掉，前端连"401 unauthorized"都看不到。

### WebSocket API 深入

**工作机制**：客户端 HTTP upgrade → 触发 `$connect` route → 建立连接；客户端发消息 → 路由到对应 route → 调 Lambda/AWS service；服务器可随时主动推送消息；断开触发 `$disconnect` route。

**三个特殊 Route**：`$connect`、`$disconnect`、`$default`（没匹配的消息）。

**服务器主动推送**：

```python
client = boto3.client('apigatewaymanagementapi',
    endpoint_url='https://abc123.execute-api.us-east-1.amazonaws.com/prod')

client.post_to_connection(
    ConnectionId='conn123',
    Data=json.dumps({"message": "hello"})
)
```

**关键限制**：单个连接最大 2 小时（必须重连）；单条消息最大 128 KB；idle timeout 10 分钟。

**用例**：实时聊天/协作编辑、游戏、股票/加密货币价格推送、IoT 设备命令下发、通知系统。
