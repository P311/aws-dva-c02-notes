# Module 17：CDN 与 Edge（CloudFront）

> **覆盖日期**：5/22
>
> English version: [`en/17-cloudfront.md`](../en/17-cloudfront.md)

---

## CloudFront 是什么 ⭐⭐⭐

**= AWS 全球 CDN（Content Delivery Network）+ Edge Computing 平台**

**核心价值**：✅ 全球加速（数百个 edge locations，用户访问就近 PoP）；✅ 缓存（静态内容、动态 API 响应都能缓存）；✅ 降低 origin 负载；✅ DDoS 保护（AWS Shield Standard 内置）；✅ HTTPS 终止（支持自定义证书）；✅ Edge Computing（Lambda@Edge/CloudFront Functions）；✅ 安全（WAF 集成、private content、geo restriction）。

**工作流程**：User 请求 → CloudFront Edge Location（Cache HIT 直接返回，Cache MISS 继续）→ Regional Edge Cache → Origin（S3/ALB/EC2/Lambda URL）→ 取内容，缓存在 edge+regional cache → User 拿到响应。

## Origins ⭐⭐

| Origin | 用途 | 关键认证 |
|---|---|---|
| **S3 Bucket** | 静态网站/文件下载 | **OAC** |
| **S3 Website Endpoint** | S3 static website hosting | Custom origin（不能用 OAC） |
| **ALB** | 动态应用/API | Custom origin headers+SG |
| **EC2/Custom HTTP** | 任意 web server | 同上 |
| **API Gateway** | API 加 CDN | Custom origin |
| **Lambda Function URL** | Serverless HTTP | **OAC**（较新支持） |

**关键区分：S3 Bucket vs S3 Website Endpoint** ⭐⭐

|  | S3 Bucket（直接） | S3 Website Endpoint |
|---|---|---|
| URL 格式 | `mybucket.s3.amazonaws.com` | `mybucket.s3-website-region.amazonaws.com` |
| 支持 OAC | ✅ | ❌（必须 public） |
| 支持 index.html/error.html | ❌ | ✅ |
| 支持 redirect 规则 | ❌ | ✅ |
| HTTPS 直接访问 | ✅ | ❌（只 HTTP） |

## OAC vs OAI ⭐⭐⭐

**= 让 S3/Lambda origin 只允许 CloudFront 访问的机制**

**OAI**（老版）：局限——只支持 S3；不支持 SSE-KMS；不支持较新的 region。

**OAC**（新版，推荐）⭐⭐⭐：优势——支持 SSE-KMS；支持所有 region；支持 S3、Lambda Function URL、MediaStore；短期凭证+频繁轮换。

**OAC 配置（S3 bucket policy）**：

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "cloudfront.amazonaws.com" },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::mybucket/*",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT:distribution/DIST_ID"
    }
  }
}
```

**OAC SigningBehavior 三种选项**（必考）：`always`（CloudFront 总是签名所有请求，默认推荐）；`never`（不签名）；`no-override`（只在请求没 `Authorization` header 时签名）。

## Signed URLs vs Signed Cookies ⭐⭐⭐

**= 给私有内容做"按需访问授权"**

|  | **Signed URL** | **Signed Cookie** |
|---|---|---|
| 粒度 | **每个 file 一个 URL** | **一组 file**（基于 path 模式） |
| 适合 | 单文件下载、临时下载链接 | 整个网站/app（多文件） |
| URL 改变 | URL 中含签名参数 | URL 不变（签名在 cookie） |
| 客户端要求 | 任何 HTTP client | 必须支持 cookie |
| 用途场景 | "下载这个 PDF"、"分享这一张图片" | "登录后可看所有付费视频" |

**Signed URL 类型**：**Canned policy**（简单，只能指定 expiration）；**Custom policy**（灵活，可指定 expiration、source IP、path pattern）。

**Trusted Key Groups**：公私钥对，用于签 URL/cookie。流程：创建 RSA 2048-bit public/private key pair → 把 public key 上传到 CloudFront 创建 Key Group → distribution behavior 关联 Key Group → 应用用 private key 签 URL/cookie → CloudFront edge 用 public key 验证。

⚠️ Trusted Signer Accounts（老法）已 deprecated。

## Lambda@Edge vs CloudFront Functions ⭐⭐⭐

**DVA 最高频对比考点之一**

### CloudFront Functions（轻量）

运行时 JavaScript（纯 JS，V8 隔离）；生命周期只有 **Viewer Request / Viewer Response**（2 种事件）；执行时间 **< 1 ms**；内存 **2 MB**；最大代码大小 **10 KB**；价格极便宜；**不能调外部 API**；运行位置遍布全球所有 edge location。

适合：HTTP header 操作/URL 重写/简单认证（JWT 验证）/A/B testing/Cache key 标准化。

### Lambda@Edge（强大）

运行时 **Node.js/Python**（完整 Lambda 运行时）；生命周期 **4 种事件**：Viewer Request/Origin Request/Origin Response/Viewer Response；执行时间 Viewer 5 秒，Origin 30 秒；内存 128 MB-10240 MB；最大代码大小 1 MB（viewer）/50 MB（origin）；**能调外部 API**；运行位置在 Regional Edge Cache（数量比全部 edge location 少得多）。

适合：复杂逻辑（调外部 API/DDB）/完整 SDK 操作/Image resizing on the fly/流量重写。

**4 种触发事件**（Lambda@Edge）⭐⭐：**Viewer Request**（每个请求，认证/URL 重写/阻挡恶意请求）；**Origin Request**（cache miss 时，改 origin/路由到不同 origin）；**Origin Response**（从 origin 拿到响应后，改响应 header）；**Viewer Response**（每个响应，加 security headers）。

**决策表** ⭐⭐⭐：简单 header 操作/URL 重写 → CloudFront Functions；极低延迟需求 → CloudFront Functions；需要 npm 包/SDK → Lambda@Edge；调外部 API/DDB → Lambda@Edge；改 origin 行为（cache miss 时）→ Lambda@Edge（Origin Request）；Image resize on the fly → Lambda@Edge；轻量 A/B testing → CloudFront Functions；简单 JWT 验证 → CloudFront Functions；复杂 OAuth 流程 → Lambda@Edge。

## Cache Behaviors ⭐⭐

**= 给不同 URL path 配不同的缓存策略**

```
Default Cache Behavior: *
  TTL: 86400, Origin: S3

Cache Behavior 1: /api/*
  TTL: 0(no cache), Forward all headers, Origin: ALB

Cache Behavior 2: /static/*
  TTL: 31536000(1 year), Origin: S3
```

**Path Patterns 优先级**：按特异性匹配——`/api/v2/users` > `/api/v2/*` > `/api/*` > `/*`。

**Cache Key**：可包含 URL path/query string/Headers/Cookies。⚠️ 越多字段 = 越多缓存版本 = 命中率越低。

**TTL 三大字段**：**Min TTL**（最小缓存时间）；**Max TTL**（最大缓存时间）；**Default TTL**（origin 没设 Cache-Control 时的默认）。

## Invalidations ⭐

**= 主动让某些 cached object 立即失效**

```bash
aws cloudfront create-invalidation \
  --distribution-id ABCD1234 \
  --paths "/index.html" "/static/*"
```

✅ 支持通配符 `/static/*`；✅ 每月有一定免费额度（超出后按 path 收费）；⚠️ 不是立刻完成（全球 propagation 需要几分钟）。

**最佳实践**：用版本化文件名（`app.v123.js`）而不是 invalidate。

## Origin Failover ⭐

**= 主 origin 挂了自动 fallback 到备 origin**

```
Origin Group:
  Primary: ALB (us-east-1)
  Secondary: S3 (us-west-2,静态备份)

Failover criteria: 500, 502, 503, 504, 404, 403
```

典型用法：ALB 挂 → fallback 到 S3 maintenance page。

## Geo Restriction

**= 按国家允许/拒绝访问**：Whitelist（只允许特定国家）或 Blacklist（拒绝特定国家）。判定依据：基于 IP 的 GeoIP 数据库。适合法律合规。⚠️ 用 VPN 可绕过——真正安全用 IAM/Signed URLs/WAF。

## Field-Level Encryption ⭐

**= 在 CloudFront 层用公钥加密特定字段**（如信用卡号）。客户端 POST 明文 → CloudFront Edge 用 public key 加密特定字段 → Origin 只看到加密后的密文 → Payment service 用 private key 解密。

vs HTTPS：HTTPS 加密整个传输通道（到 origin 后是明文）；Field-level encryption 加密特定字段（到 backend 仍是密文）。

## SSL / Custom Domains ⭐⭐

### ⭐⭐⭐ ACM 证书必须在 us-east-1！

CloudFront 是全球服务，但**给 CloudFront 用的 ACM 证书必须申请在 us-east-1**，其他 region 的 ACM 证书 CloudFront 无法用。**DVA 经典 trick 题**。

**SSL 配置选项**：Default CloudFront certificate（用 `*.cloudfront.net` 证书）；Custom SSL Certificate（ACM，必须 us-east-1）；SNI（一个 distribution 支持多 domain，推荐）；Dedicated IP（极少用，费用高）。

**HTTPS Behavior**：Viewer Protocol Policy——HTTP and HTTPS / Redirect HTTP to HTTPS（推荐）/ HTTPS Only；Origin Protocol Policy——HTTP Only / HTTPS Only / Match Viewer。

⭐ 推荐生产：Viewer = Redirect to HTTPS，Origin = HTTPS Only。

## Custom Error Pages

S3 origin 返回 403 → CloudFront 返回 `/error.html`。可配：哪些 HTTP status code、返回的 path、返回 status code、TTL（error 缓存多久）。

## 监控 + WAF + 价格

**Real-Time Logs**：秒级粒度 → Kinesis Data Streams（适合实时监控）。**Standard Logs**：批量周期 → S3（适合归档）。**WAF 集成**：关联 AWS WAF Web ACL，在 edge 层拦截恶意流量。

**Price Class**：All Edge Locations / US+Canada+Europe / 子集——目标用户在哪就选对应 class。

## CloudFront 限制

Cache behaviors per distribution 约 25；Origins per distribution 约 25；Origin groups per distribution 约 10；Alternate domain names (CNAMEs) 约 100；最大单文件 **30 GB**（以官方最新文档为准）。
