# Module 19：杂项与新兴服务（AppConfig / Amplify / ACM / AppSync）

> **覆盖日期**：5/24
>
> English version: [`en/19-misc.md`](../en/19-misc.md)
>
> DVA-C02 真考会出现少量这些"小服务"题，本模块快速覆盖遗漏点。

---

# 一、AWS AppConfig ⭐⭐

## AppConfig 是什么

**= AWS Systems Manager 的一个 feature**。核心价值：应用配置/feature flags 的动态管理，无需重启/重新部署应用即可改配置。

**典型用途**：Feature flags（灰度发布）、应用配置（API 端点、超时值）、A/B 测试、运维 throttling/circuit breaker 开关、Allow-list/block-list。

## vs Parameter Store / Secrets Manager ⭐⭐⭐

|  | **AppConfig** | **Parameter Store** | **Secrets Manager** |
|---|---|---|---|
| 用途 | 配置+feature flags 动态部署 | 简单 key-value 配置 | 密钥（自动轮换） |
| 变更控制 | ✅ deployment strategy（渐进、自动回滚） | ❌ 立即生效 | 轮换流程 |
| 验证 | ✅ JSON Schema/Lambda | ❌ | ❌ |
| 自动回滚 | ✅ CloudWatch alarm 触发 | ❌ | ❌ |
| 大小限制 | 1 MB | 4/8 KB | 64 KB |

**判断**："渐进发布配置+alarm 触发自动回滚" → AppConfig；简单环境变量 → Parameter Store；DB 密码自动轮换 → Secrets Manager。

## AppConfig 核心概念 ⭐⭐

```
Application(项目)
  └─ Environment(dev / staging / prod)
      └─ Configuration Profile(freeform 或 feature-flags 类型)
          └─ Deployment(用 deployment strategy 部署)
```

**Configuration Profile 类型**：**Freeform**（自由格式 JSON/YAML/文本配置）；**Feature Flags**（专门的 feature flag，带 enabled、attributes）。

**Configuration 存储位置**：Freeform 可存在 AppConfig hosted store/SSM Parameter Store/SSM Document/S3/CodePipeline；**Feature Flags 只能用 hosted configuration store**。

## Validators ⭐

**= 部署前自动验证配置正确性**。两种：**JSON Schema**（验证 JSON 结构）、**Lambda Function**（任意自定义验证）。⭐ 验证失败 → deployment 拒绝，不会发到生产。

## Deployment Strategy ⭐⭐

| 预设 | 行为 |
|---|---|
| `AppConfig.AllAtOnce` | 一次性全部部署（测试用） |
| `AppConfig.Linear50PercentEvery30Seconds` | 每 30 秒 50% |
| `AppConfig.Canary10Percent20Minutes` | 10%/等 20 分钟/90% |

**自定义参数**：Growth Type（Linear/Exponential）、Growth Factor、Deployment Duration、**Final Bake Time**（最后停留观察时间，失败可回滚的窗口）。

## Automatic Rollback ⭐⭐

**= CloudWatch Alarm 触发时自动回滚到上一个版本**：

```
Deploy v2 → 10% → 50% → 100%
              ↓
          5xx alarm 触发
              ↓
          自动 rollback to v1
```

⚠️ bake time 结束后，deployment 标记 deployed，不再监控。

## 应用获取配置 ⭐⭐

**方式 1：AppConfig Agent**（Lambda Extension/Sidecar，推荐）——应用 → localhost HTTP → AppConfig Agent → AppConfig。优势：本地缓存+自动 polling 更新+应用代码极简。

**方式 2：SDK 直接调**——必须用 `start_configuration_session` + `get_latest_configuration` 模式（旧 `get_configuration` 已 deprecated）。

## AppConfig 用例

应用配置渐进式发布+失败自动回滚 → AppConfig；简单 string config → Parameter Store；安全密钥+自动轮换 → Secrets Manager；Lambda 想低延迟读 config → AppConfig+Lambda Extension；部署前验证 config 格式 → AppConfig+Validators。

---

# 二、AWS Amplify ⭐

## Amplify 是什么

**= 前端+移动应用的全栈开发平台**，两个独立产品：

**Amplify Hosting**（以前叫 Amplify Console）：静态网站/SSR 应用托管+内置 CI/CD，类似 Netlify/Vercel。Git 集成（GitHub/GitLab/Bitbucket/CodeCommit）；push 触发自动构建+部署；内置 CDN+自定义域名+SSL 证书自动配；支持 SSR（Next.js、Nuxt）。

**Amplify CLI / Libraries**：前端+移动 SDK，支持 React、React Native、Angular、Vue、iOS、Android、Flutter，简化集成 Cognito、AppSync、S3、Lambda。

⭐ DVA 主要考 Amplify Hosting。

**Amplify Hosting 关键特性**：Git 集成（push 自动构建+部署）；**Preview environments**（每个 PR/branch 一个独立环境）；Atomic deployments（部署原子化，失败立即回滚）；Custom domains+SSL（ACM 自动配置）；Password protection（branch 级密码保护）；Pull request previews（自动给每个 PR 部署预览）；Monorepo support（一个 repo 多个 app）。

**amplify.yml** 示例：

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: build
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

**判断**：静态网站 GitHub push 自动部署+CDN → Amplify Hosting；每个 PR 一个预览环境 → Pull Request Previews；Next.js/Nuxt SSR 应用 → Amplify Hosting；React+Cognito+AppSync 全栈 → Amplify CLI+Libraries。

---

# 三、AWS Certificate Manager (ACM) ⭐

## ACM 是什么

**= AWS 托管的 SSL/TLS 证书服务**。核心价值：✅ 免费（public certs，附在 AWS 服务上）；✅ 自动续期；✅ 集成 CloudFront/ALB/API GW/ELB；✅ DNS/Email 验证。

**ACM 证书类型**：**Public**（AWS issued，免费，公开网站/API）；**Private**（AWS Private CA，有月费+cert 费用，内部网络/企业 PKI）；**Imported**（自带 cert，免费导入，已有第三方证书）。

## ACM 验证方式 ⭐

**DNS 验证**（推荐）：ACM 给一条 CNAME，加到 DNS（Route 53 一键），✅ 自动续期。

**Email 验证**：ACM 给域名所有者发邮件，人工点链接，❌ 必须手动续期。

⭐ 生产用 DNS 验证。Email 验证容易过期失败。

## ACM 关键考点 ⭐⭐⭐

### 1. CloudFront 用的 ACM 证书必须在 us-east-1！

CloudFront 是全球服务，但 ACM 证书必须在 us-east-1！其他 region 的 ACM 证书 CloudFront 无法用。**DVA 经典 trick 题**。

### 2. 各服务区域要求

| 服务 | 区域要求 |
|---|---|
| CloudFront | **us-east-1** |
| ALB/NLB | 同 region |
| API Gateway（Regional） | 同 region |
| API Gateway（Edge-optimized） | **us-east-1** |
| Elastic Beanstalk | 同 region |
| **EC2** | ❌ 不直接支持 |
| **S3** | ❌ 用 CloudFront 前置 |

⭐ EC2/S3 不能直接用 ACM，要透过 CloudFront/ALB。

### 3. 自动续期

到期前一段时间开始尝试 renew；DNS 验证+仍 attach 在 AWS 服务上+DNS 记录还在 → 自动续期；Email 验证 → 不能自动续期。

## ACM 考点速记

AWS 免费 SSL 证书 → ACM；CloudFront 用证书 region → **us-east-1**（必背！）；API GW Regional 证书 region → 同 region；API GW Edge-optimized 证书 region → us-east-1；ACM 自动续期需要的验证 → DNS 验证；ACM 不能直接给 → EC2/S3；内部/企业 PKI 证书 → ACM Private CA；已有第三方证书想用 AWS → ACM Imported。

---

# 四、AWS AppSync ⭐

## AppSync 是什么

**= AWS 托管的 GraphQL 服务**。vs API Gateway：API Gateway 是 REST/HTTP/WebSocket；AppSync 是 **GraphQL**。GraphQL 优势：一个 endpoint+client 声明要什么字段+内置 real-time（subscription）。

## AppSync 核心概念 ⭐⭐

**GraphQL Schema → Resolver（解析器）→ Data Source（数据源）**。Data Sources：DynamoDB、RDS Aurora、OpenSearch、Lambda、HTTP endpoint、EventBridge、None。

## Resolver 类型 ⭐⭐

**Unit vs Pipeline**：Unit 是单一 data source 调用；Pipeline 是多步串行（auth → 主调用 → 后处理）。

**Resolver Language**：VTL（Apache Velocity，老式）与 JavaScript（APPSYNC_JS，新式，新项目首选）。

**Direct Lambda Resolver**：直接关联 Lambda，没有 VTL/JS 映射，不需要写 mapping template，Lambda 拿到完整 GraphQL 上下文。

## Subscriptions（实时）⭐⭐

**= GraphQL 实时订阅**，基于 **WebSocket**：

```graphql
subscription OnNewMessage {
  newMessage(roomId: "abc") {
    id
    content
    user
  }
}
```

工作机制：Client 连 WebSocket → 由 Mutation 触发推送给订阅者 → AppSync 自动管理连接。经典用例：聊天、实时仪表盘、协作编辑、推送通知。

## Authorization Modes ⭐⭐

AppSync 支持 5 种认证方式，一个 API 可配多种：**API Key**（简单，开发/公开 API，最长约 365 天有效）；**AWS IAM**（服务到服务/SigV4 签名）；**Cognito User Pools**（用户登录的 web/mobile app）；**OpenID Connect (OIDC)**（第三方 IdP，Auth0、Okta）；**Lambda Authorizer**（自定义认证逻辑）。

## 字段级权限

```graphql
type Post @aws_cognito_user_pools {
  id: ID!
  title: String!
  internalNotes: String @aws_iam        # 只 IAM 能读
  views: Int @aws_api_key               # API key 也能读
}
```

Schema Directives：`@aws_api_key`、`@aws_iam`、`@aws_cognito_user_pools`、`@aws_oidc`、`@aws_lambda`。

## AppSync Caching

**= 服务端缓存 GraphQL 响应**。Full request caching 或 Per-resolver caching；TTL 1-3600 秒；用 Redis 后端（全托管）。

## 离线 + 冲突解决

**Amplify DataStore + AppSync**：离线访问/自动同步/冲突解决（server-wins/client-wins/Lambda）。适合移动应用（网络不稳定）。

## AppSync 限制

单 GraphQL 请求 timeout 约 **30 秒**；响应大小约 **1 MB**。

## AppSync 考点速记

AWS GraphQL 服务 → AppSync；GraphQL 实时订阅 → Subscription+WebSocket；AppSync 数据源 → DynamoDB/RDS/OpenSearch/Lambda/HTTP/None/EventBridge；Resolver 语言 → VTL（老）或 JavaScript（新推荐）；不写 mapping 直接调 Lambda → Direct Lambda Resolver；字段级权限控制 → Schema directives；移动应用+离线+冲突解决 → AppSync+Amplify DataStore；5 种认证 → API Key/IAM/Cognito/OIDC/Lambda。

---

# 五、本模块综合速记

动态配置+渐进发布+失败回滚 → AppConfig；AppConfig vs Parameter Store → AppConfig 有 deployment strategy+自动回滚，Parameter Store 即时生效；Feature flag 验证格式 → AppConfig Validators；前端静态网站+Git push 自动部署 → Amplify Hosting；PR preview 环境 → Amplify Pull Request Previews；AWS 免费 SSL → ACM（Public）；CloudFront 用 ACM → 必须 us-east-1（经典！）；ACM 自动续期 → DNS 验证+仍 attached；EC2/S3 用 ACM → ❌（必须通过 CloudFront/ALB）；GraphQL API → AppSync；实时推送（WebSocket-based）→ AppSync Subscriptions；AppSync resolver 新推荐语言 → JavaScript（APPSYNC_JS）；不写 mapping 直接 Lambda → Direct Lambda Resolver；GraphQL field 级权限 → Schema directives。
