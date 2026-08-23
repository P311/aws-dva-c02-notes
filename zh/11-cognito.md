# Module 11：Cognito（认证与授权）

> **覆盖日期**：5/13
>
> English version: [`en/11-cognito.md`](../en/11-cognito.md)

---

## Cognito 是什么 ⭐⭐⭐

**= AWS 完全托管的"身份平台"**，用于 web/mobile 应用。

**两大核心组件**（必须分清）：

| 组件 | 全称 | 用途 | 关键词 |
|---|---|---|---|
| **User Pool**（CUP） | Cognito User Pool | **认证（Authentication）** | "你是谁" |
| **Identity Pool**（CIP） | Cognito Identity Pool（原名 Federated Identities） | **授权（Authorization）** | "你能用什么 AWS 资源" |

**关键差异**：**User Pool**——用户登录系统 → 颁发 **JWT tokens**（ID/Access/Refresh）。**Identity Pool**——用 token 换 **AWS 临时凭证**（STS），给应用直接调用 AWS 服务的权限。

**两者可独立使用，也可组合使用**：只用 CUP——登录 → 拿 JWT → 调你自己的 API；只用 CIP——已有 IdP（Google/Facebook）→ 拿 AWS 凭证调 AWS 服务；CUP + CIP——用 Cognito 自管理用户库 + 让用户直接调用 S3/DynamoDB。

---

## Cognito User Pool (CUP) — 认证 ⭐⭐⭐

**= 完全托管的"用户目录"+ 认证服务**。提供：用户注册/登录；MFA（SMS 或 TOTP）；邮箱/手机号验证；密码策略；密码找回/重置；用户属性管理；用户组（Groups）；联合登录（Google、Facebook、Amazon、Apple、SAML、OIDC）；Hosted UI；Lambda 触发器扩展认证流程；高级安全（adaptive auth、风险检测）。

### 用户登录后拿到什么 ⭐⭐⭐

**3 种 JWT Token**：

| Token | 用途 | 包含什么 | 默认过期 | 可配置范围 |
|---|---|---|---|---|
| **ID Token** | 证明用户身份 | 用户身份信息（name, email, sub 等 claims） | **1 小时** | **5 分钟 - 1 天** |
| **Access Token** | API 授权（scopes） | OAuth scopes、user 标识 | **1 小时** | **5 分钟 - 1 天** |
| **Refresh Token** | 换取新的 ID/Access token | 不可解析（opaque） | **30 天** | **1 小时 - 10 年** |

**用法**：短期 token（ID/Access）过期 → 用 Refresh token 调 `InitiateAuth(REFRESH_TOKEN_AUTH)` 拿新的；Refresh token 过期 → 用户必须重新登录。

**JWT 结构**：三段 base64 编码，以 `.` 分隔：`header.payload.signature`。API Gateway/后端用 Cognito 的公钥验证签名（Cognito 公开 JWKS endpoint）。

**关键考点**：REST API + Cognito Authorizer 用 **ID token**；HTTP API + JWT Authorizer 用 **Access token + scopes**。别搞反！

### Cognito Groups（组）⭐

**= 在 User Pool 内分组用户**：

```
User Pool "MyApp":
├── Group "admin"   → 关联 IAM Role "AdminRole"
├── Group "premium" → 关联 IAM Role "PremiumRole"
└── Group "free"    → 关联 IAM Role "FreeRole"
```

一个用户可以属于多个 group；Group 信息会进 JWT（`cognito:groups` claim）；应用代码/Lambda Authorizer 可以基于 group 做权限判断；配合 Identity Pool，group 还可以自动选 IAM Role（基于优先级）。

### Hosted UI ⭐

**= Cognito 预置的登录/注册页面**

```
你的应用 ─→ 重定向到 ─→ Cognito Hosted UI
            (用户在这里输入用户名密码 / 选社交登录)
            │
            ▼ 登录成功
            重定向回你的应用 + 带 tokens(在 URL)
```

**特点**：✅ 零代码；✅ 自动处理登录、注册、忘记密码、MFA、社交登录；✅ 自定义 CSS/logo；✅ Custom domain 支持；✅ OAuth 2.0 标准 endpoints。

**典型 OAuth flow**：用户访问应用未登录 → 重定向到 Hosted UI 登录页 → 用户输入凭证 → 重定向回应用带 `code` → 应用用 `code` 换 JWT tokens。

⚠️ **Hosted UI 名称变化**：AWS 已把 "Hosted UI" 改名为 "Managed Login"（新版本），保留老的 Hosted UI（classic），但很多考试题库仍称 "Hosted UI"。以官方最新命名为准。

### 联合登录 / Federation ⭐⭐

**= 让用户用外部身份登录（不需要在 Cognito 注册账号）**

支持的 IdP：Social IdPs（开箱即用：Google、Facebook、Amazon、Apple）；SAML 2.0（企业 SSO：AD、Okta、Auth0、Azure AD）；OIDC（任何 OIDC 兼容的 IdP）。

**关键洞察**：外部 IdP 的 token 不会直接给应用！Cognito 把外部 IdP 的身份"翻译"成自己的 JWT 给应用 → 应用统一处理 Cognito tokens，不用对接多个 IdP。

### OAuth Flows ⭐

Cognito User Pool 支持几种 OAuth 2.0 grant：

| Flow | 用途 | 推荐场景 |
|---|---|---|
| **Authorization Code Grant** | Web 应用（后端+前端） | 生产首选，token 不暴露 |
| **Authorization Code + PKCE** | SPA/Mobile 应用 | **现代 SPA/移动 app 首选** |
| **Client Credentials** | 服务到服务（M2M） | 后端服务调用，无用户 |
| ~~Implicit Grant~~ | （老式，不推荐） | OAuth 2.1 已废弃 |

**判断**：浏览器 SPA → Authorization Code + PKCE；后端 API to API → Client Credentials；老题可能选 Implicit Grant，新题选 Code + PKCE。

### MFA（多因素认证）

**两种 MFA**：SMS（短信验证码）、TOTP（Google Authenticator/Authy）。

**3 种 MFA 策略**：Off（不启用）、Optional（用户可选）、Required（强制所有用户必须用 MFA）。

**Adaptive Authentication**：基于风险评分动态决定是否要 MFA，不寻常的 IP/设备触发 MFA，需要启用 Advanced Security Features（收费）。

---

## Cognito Lambda Triggers ⭐⭐⭐

**= Cognito 调用 Lambda 在认证流程的关键节点扩展功能**

### 完整 Trigger 列表

**注册/确认流程**：

| Trigger | 时机 | 典型用法 |
|---|---|---|
| **Pre Sign-up** | 用户提交注册请求时 | 验证邮箱域（只允许 @company.com）、自动确认特定用户 |
| **Post Confirmation** | 用户确认账号后 | 发欢迎邮件、初始化用户档案到 DynamoDB |
| **Custom Message** | 发送验证码/邮件/SMS 前 | 自定义消息内容（品牌化邮件） |
| **User Migration** | 用户首次登录（老系统迁移） | 验证旧密码 → 创建 Cognito 账号（无感迁移） |

**认证流程**：

| Trigger | 时机 | 典型用法 |
|---|---|---|
| **Pre Authentication** | 提交登录请求，验证密码前 | 拒绝可疑 IP、自定义校验 |
| **Post Authentication** | 登录成功后 | 记录登录日志、风险分析 |
| **Pre Token Generation** | 签发 JWT 前 | **修改 JWT claims**（加自定义字段、改 scope） |

**自定义认证 challenge 流程**（高级）：**Define Auth Challenge**（定义整个 challenge 流程）、**Create Auth Challenge**（生成具体 challenge，如 CAPTCHA）、**Verify Auth Challenge Response**（验证用户的回答）。

**自定义 sender**：**Custom Email Sender**（用自己的 SES 或第三方邮件服务）、**Custom SMS Sender**（用自己的 SMS 服务）。

**关键约束**：⚠️ Lambda Trigger 必须 **5 秒内返回**（否则 Cognito 视为失败），触发器代码必须快，不能调慢服务。

**经典判断**：只让公司邮箱注册 → Pre Sign-up；注册成功后初始化 DynamoDB 用户档案 → Post Confirmation；自定义注册验证邮件内容 → Custom Message；从老 auth 系统无感迁移 → User Migration；给 JWT 加自定义字段 → Pre Token Generation；CAPTCHA 增加额外验证 → Define/Create/Verify Auth Challenge 三件套。

---

## Cognito Identity Pool (CIP) — 授权 ⭐⭐⭐

**= 给认证后的用户（或匿名 guest）颁发 AWS 临时凭证**

工作流程：用户认证（通过 IdP：User Pool/Google/Facebook/SAML/自定义）→ 应用拿到 IdP 的 token → 应用调 Identity Pool 的 `GetCredentialsForIdentity`（传 token）→ Identity Pool 用 STS AssumeRole → 返回临时 AWS 凭证 → 应用直接用临时凭证调 AWS 服务。

**核心价值**：让前端/移动 app 直接安全调 AWS 服务（不用把 AWS 长期凭证暴露给客户端）。

**支持的 IdP**：Cognito User Pool、Social（Google/Facebook/Amazon/Apple）、SAML 2.0、OIDC、Developer-authenticated（自定义后端认证）、Anonymous/Guest（未登录用户）。

### 两种用户类型

**1. Authenticated**（已认证）：通过 IdP 验证身份，关联到 Authenticated IAM Role（权限较大）。

**2. Unauthenticated / Guest**（未认证）：没登录，但可以访问部分 AWS 资源，关联到 Unauthenticated IAM Role（权限受限），用于让游客也能浏览公开内容。

默认每个 Identity Pool 配置两个 IAM Roles（一个 auth，一个 unauth）。

### Role Assignment ⭐⭐

**1. Role-Based Access Control (RBAC)** — 用 group/claim 选 role：User Pool Group "admin" → 用 AdminRole；User Pool Group "user" → 用 UserRole。或基于 token claims（如 "如果 token 里 dept=engineering" → EngineeringRole）。

**2. Attribute-Based Access Control (ABAC)** — 用 token attributes 做 IAM session tags：

```
JWT claim "tenant_id=abc" → 注入到 STS session 的 sts:RequestTag
IAM policy 用 ${aws:PrincipalTag/tenant_id} 限制访问
```

ABAC 优势：一个 IAM role 服务所有 tenants，根据 tag 区分权限。

### 经典场景 — Mobile app 直接上传到 S3

用户在 mobile app 用 Cognito User Pool 登录拿 JWT → App 调 Identity Pool 用 JWT 换 AWS 临时凭证 → App 用临时凭证直接调 S3 PutObject（走 SDK）→ 不需要中间 server！

IAM policy（authenticated role）示例：

```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject"],
  "Resource": "arn:aws:s3:::user-uploads/${cognito-identity.amazonaws.com:sub}/*"
}
```

利用 `${cognito-identity.amazonaws.com:sub}`（Identity Pool 的用户 ID），让每个用户只能写自己目录。

---

## CUP vs CIP 综合对比 ⭐⭐⭐

```
Mobile App  ─→  Cognito User Pool (登录/注册)
     │              │
     │              └─→ 返回 JWT tokens (ID/Access/Refresh)
     │
     ├─→ 直接调你的 API (用 JWT) ────→ API Gateway + Cognito Authorizer
     │
     └─→ Cognito Identity Pool (用 JWT 换 AWS 临时凭证)
            │
            └─→ STS AssumeRole → 返回 AccessKey + Secret + Token
                                      │
                                      └─→ App 直接调 AWS: S3 / DynamoDB / SES
```

| 维度 | **User Pool** | **Identity Pool** |
|---|---|---|
| 作用 | **Authentication**（认证） | **Authorization**（授权） |
| 关键词 | "你是谁" | "你能做什么 AWS 操作" |
| 颁发什么 | JWT tokens | AWS 临时凭证（via STS） |
| 存储用户数据 | ✅ user directory | ❌（只存 identity ID 映射） |
| 支持密码 | ✅ | ❌（必须靠 IdP） |
| 支持社交登录 | ✅（联合到 social IdP） | ✅（用 social token 换 AWS 凭证） |
| MFA | ✅ | ❌ |
| Guest 用户 | ❌ | ✅ Unauthenticated role |
| 价格 | 按 MAU（月活）收费 | 免费 |

### 选择决策 ⭐⭐⭐

| 需求 | 选 |
|---|---|
| 自管理用户（注册/登录/MFA） | **User Pool** |
| 用户调你自己的 API | **User Pool** 就够 |
| 用户直接调 AWS 服务（S3/DynamoDB） | 需要 **Identity Pool** |
| Mobile app 让用户上传图片到 S3 | **User Pool + Identity Pool** |
| 已有 Google 登录，只想给 AWS 访问 | **Identity Pool**（不需要 User Pool） |
| 游客可以浏览（无登录） | **Identity Pool** unauthenticated role |
| 企业 SSO 集成 + 调 AWS | **Identity Pool** 直接接 SAML |

---

## Cognito + API Gateway（回顾 + 强化）⭐⭐⭐

### REST API + Cognito User Pool Authorizer

用 **ID token**（放在 Authorization header）；API GW 自动验证签名（用 Cognito 公钥，不用 Lambda）；通过则 claims 注入 `$context.authorizer.claims.xxx`。

### HTTP API + JWT Authorizer

通常用 **Access token**（可以做 OAuth scopes 授权）；配 `issuer` = Cognito User Pool 的 issuer URL；配 `audience` = User Pool 的 app client ID；支持 `authorizationScopes` 限制 method 访问。

**必考陷阱**：⚠️ REST API + Cognito Authorizer 必须用 ID token（用 Access token → 401）；⚠️ Token 过期后客户端要用 refresh token 拿新的；⚠️ Cognito Authorizer 的 cache 默认 5 分钟，改 token 后 cache 时间内仍验证旧的。

---

## Cognito 关键限制 / 数字 ⭐

| 项 | 值 |
|---|---|
| **MAU 免费层** | 前若干万（具体数字以官方定价页为准） |
| **单个 User Pool 用户上限** | 数千万级 |
| **Custom attributes** | 最多 **50 个** |
| **Standard attributes** | 25 个预置（name/email/phone 等） |
| **ID Token / Access Token 默认有效期** | **1 小时** |
| **ID/Access Token 可配置范围** | **5 分钟 - 1 天** |
| **Refresh Token 默认有效期** | **30 天** |
| **Refresh Token 可配置范围** | **1 小时 - 10 年** |
| **Lambda trigger 超时** | **5 秒** |

**价格（大致结构）**：User Pool 前若干 MAU 免费（永久，不是 free tier），超过按 MAU 收费；Identity Pool 完全免费；Hosted UI/SMS 按发送量收费（SMS 用 SNS 在背后）；Advanced Security 额外按 MAU 收费。具体数字建议以 AWS 官方定价页为准。

---

## Cognito 安全特性（简略）

**Advanced Security Features**（收费）：Compromised credential detection（对比已泄露密码库）；Adaptive authentication（基于风险评分动态加 MFA）；Account takeover protection（防暴力破解、防恶意 IP）。

**加密**：Tokens 用 RS256（RSA）签名；User Pool 数据 at-rest 加密（KMS，AWS managed）；自定义 KMS key 仅用于 Custom Sender Lambda triggers。
