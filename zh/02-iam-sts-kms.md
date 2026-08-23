# Module 2：IAM + STS + KMS

> **覆盖日期**：4/20、4/21、4/22
> **主题**：IAM 核心 · STS + Federation + Cross-Account · Encryption + KMS
>
> English version: [`en/02-iam-sts-kms.md`](../en/02-iam-sts-kms.md)

---

## 一、IAM 核心

### Users / Groups / Roles

**User** — 代表一个人或服务，长期凭证（密码、Access Key）。最佳实践：不给 root 用 access key，root 只做账户级别操作。

**Group** — 只能装 user，不能嵌套，不能装 role。权限通过 group 的 policy 附加给 user。

**Role** — 没有长期凭证，被 assume 时通过 STS 返回临时凭证。用于：

- EC2 instance profile
- Lambda execution role
- 跨账户访问
- 联合身份（SAML、OIDC、Cognito）

**Role 的两个 policy**：

- **Trust policy**（谁可以 assume 这个 role）— 定义在 role 本身，**是 resource-based policy**
- **Permissions policy**（这个 role 能做什么）— 可以是 managed 或 inline

### IAM User vs IAM Role 核心区别

|  | IAM User | IAM Role |
|---|---|---|
| 代表谁 | 具体的一个人或服务账号 | 可被任何被信任方临时扮演的身份 |
| 凭证类型 | 长期（密码、access key） | 无固有凭证，assume 时 STS 发临时凭证 |
| 能否「登录」 | 可以（用密码） | 不能，只能被 assume |
| Principal | 只能是这个 user 自己 | EC2、Lambda、其他账户、SAML、OIDC… |
| 典型用法 | 人类管理员、CI 系统 | EC2 instance profile、跨账户、Federation |

### Policy 类型

- **Identity-based policy** — 附加到 user/group/role
- **Resource-based policy** — 附加到资源（S3 bucket policy、SQS、SNS、Lambda、KMS key policy 等）
- **Permission boundary** — 给 user/role 设置的权限上限，不授权，只限制
- **SCP（Service Control Policy）** — Organizations 层面的，影响整个 account
- **Session policy** — assume role 时临时传入，只能缩小权限
- **ACL** — 老式的，S3 和 VPC 里还在用

### Policy 评估逻辑 ⭐

**决策顺序**：

1. 默认 deny（implicit deny）
2. 检查所有适用的 policy
3. **只要有 explicit deny，立刻拒绝**（一票否决）
4. 有 explicit allow → 允许
5. 没有 allow → 还是 implicit deny

**口诀**：**Deny > Allow > Implicit Deny**

**多层 policy 同时存在时的逻辑**：

- **同账户内访问**：identity policy 和 resource policy 是 **OR** 关系，任一允许即可
- **跨账户访问**：两边都要 allow（**AND** 关系）
- **有 permission boundary 时**：有效权限 = identity policy ∩ permission boundary
- **有 SCP 时**：SCP 不授权，只是 filter

**最终有效权限** = `Identity Policy ∩ Permission Boundary ∩ SCP`

### Permission Boundary vs SCP

|  | Permission Boundary | SCP |
|---|---|---|
| 作用对象 | 单个 IAM user/role | 整个 OU 或 account |
| 谁配置 | IAM admin | Organizations admin |
| 是否授权 | ❌ 只限制上限 | ❌ 只过滤 |
| 对 root user 生效 | ❌ 不生效 | ✅ 生效 |

### IAM 策略细节

**Condition 常见键**：

- `aws:SourceIp` — 限制来源 IP
- `aws:MultiFactorAuthPresent` — 要求 MFA
- `aws:RequestedRegion` — 限制 region
- `aws:PrincipalTag` / `aws:ResourceTag` — ABAC 核心
- `aws:SecureTransport` — 要求 HTTPS

**NotAction / NotResource 的坑**：

- 语义：匹配「除了列出的这些以外的所有 action」
- `NotAction: "iam:*"` 配 `Allow` → 实际是「允许除 IAM 外所有 AWS 服务的所有操作」

**IAM 是全球服务**，不属于任何 region。

### IAM 证书存储（IAM certificate store）

只在一种情况下把 IAM 当证书管理器用：某个 Region 不支持 ACM，而你又必须在那里提供 HTTPS。要点：

- IAM 会加密私钥并存在 IAM 的 SSL 证书库里
- 所有 Region 都支持部署 server certificate，但证书必须从外部 CA 获取
- **不能把 ACM 证书上传到 IAM**
- **不能从 IAM 控制台管理证书**（只能用 CLI/API）

### Instance Profile + IMDS 完整机制

```text
IAM Role  ←[附加]←  Instance Profile  ←[附加]←  EC2 Instance
   │                                                  │
   ├─ Trust Policy                                    │
   └─ Permissions Policy                              │
                                                      ▼
                                              169.254.169.254
                                              (IMDS endpoint)
```

**关键点**：

- IAM Role 和 Instance Profile **是两个东西**
- Instance Profile 是「容器」，里面装着一个 IAM Role
- 控制台操作时，AWS 自动创建同名 Instance Profile
- CloudFormation 必须显式定义 `AWS::IAM::InstanceProfile` 资源

**SDK 凭证提供者链**（自动按顺序尝试）：

1. 代码里显式传的凭证
2. 环境变量 `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`
3. `~/.aws/credentials` 文件
4. **IMDS**（EC2 用这个）
5. ECS container credentials

### IMDS 全面解析

**IMDS = Instance Metadata Service**

**地址**：`http://169.254.169.254`（link-local 地址，只在本机有效）

**两大类数据**：

1. **Metadata**（元数据）— 描述「这台 EC2 是什么」
   - `instance-id`、`instance-type`、`placement/region`、`placement/availability-zone`
   - `local-ipv4`、`public-ipv4`
   - `iam/security-credentials/<role-name>` → **临时凭证 JSON** ⭐
2. **User Data**（用户数据）— 启动时传给 EC2 的脚本
3. **Dynamic data**（含 identity document）

**Metadata 不提供**：

- ❌ 内存大小、CPU 核数
- ❌ 磁盘大小

### IMDSv1 vs IMDSv2 ⭐

|  | IMDSv1 | IMDSv2 |
|---|---|---|
| 请求方式 | 直接 GET | 先 PUT 拿 token，再带 token GET |
| SSRF 攻击防护 | ❌ | ✅ |
| Hop limit | 无限制 | 默认 1 |
| 新 instance 默认 | （已淘汰） | ✅ 默认启用 |

**IMDSv2 防御 SSRF 的三重设计**：

1. **必须用 PUT 方法拿 token** — 大多数 SSRF 漏洞只能发起 GET 请求
2. **Token 必须放在自定义 header 里** — SSRF 通常控制不了 request header
3. **Hop limit 默认为 1** — 防止容器逃逸（[文档](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)）

> 实战注意：容器环境里 hop limit = 1 会导致容器拿不到凭证（进容器算多一跳），AWS 建议调成 2。

**历史事故**：2019 年 Capital One 数据泄露，1 亿用户信息，根本原因是 SSRF + IMDSv1。

### 临时凭证识别

```json
{
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "...",
  "SessionToken": "...",
  "Expiration": "..."
}
```

三个标志：

- AccessKeyId 以 **ASIA** 开头（长期凭证是 **AKIA**）
- 有 **SessionToken**（长期凭证没有）
- 有 **Expiration**

---

## 二、STS + Federation + Cross-Account

### STS 核心 API

| API | 用途 | 谁调用 |
|---|---|---|
| `AssumeRole` | 用现有 AWS 凭证去 assume 另一个 role | IAM user 或 role |
| `AssumeRoleWithSAML` | 用 SAML assertion 换 AWS 凭证 | 企业员工 |
| `AssumeRoleWithWebIdentity` | 用 OIDC token 换 AWS 凭证 | 移动 app 用户 |
| `GetSessionToken` | 长期凭证换短期凭证（主要为了加 MFA） | IAM user 自己 |
| `GetFederationToken` | 给「没有 IAM 身份的用户」发凭证（老 API） | 很少用了 |

**记忆法**：

- 看到 **SAML** → `AssumeRoleWithSAML`
- 看到 **Google / Facebook / Cognito** → `AssumeRoleWithWebIdentity`
- 看到 **跨账户** → `AssumeRole`
- 看到 **MFA** → `GetSessionToken` 或带 MFA condition 的 `AssumeRole`

### 临时凭证有效期

| API | 默认 | 最小 | 最大 |
|---|---|---|---|
| AssumeRole | 1 小时 | 15 分钟 | 12 小时 |
| AssumeRoleWithSAML | 1 小时 | 15 分钟 | 12 小时 |
| AssumeRoleWithWebIdentity | 1 小时 | 15 分钟 | 12 小时 |
| GetSessionToken | 12 小时 | 15 分钟 | 36 小时（root 只能 1 小时） |

来源：[AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) · [GetSessionToken](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetSessionToken.html)

### Cross-Account Access

**方案 1：Bucket Policy（更常用）**

- 源账户 A 的 user 的 identity policy 允许访问目标资源
- 目标账户 B 的 bucket policy 允许 A 的 principal 访问

**方案 2：Assume Role**

- 目标账户 B 创建 role，trust policy 允许源账户 A assume
- 源账户 A 的 user 调用 `sts:AssumeRole` 拿临时凭证
- CloudTrail 能看到 AssumeRole 事件 + 后续操作事件，审计链路更结构化

### External ID（防御「困惑的副手」攻击）

**场景**：第三方 SaaS（Datadog、MongoDB Atlas）让你创建 role 给他们 assume。

**解法**：SaaS 给你一个唯一的 External ID，在 trust policy 里加：

```json
"Condition": {
  "StringEquals": { "sts:ExternalId": "abc-123-unique" }
}
```

**考点**：看到「第三方 SaaS 集成」「cross-account trust」「confused deputy」→ **External ID**。

### Identity Federation 两大协议

|  | SAML 2.0 | OIDC |
|---|---|---|
| 主要场景 | 企业员工登录 | 公网用户登录 |
| 底层协议 | XML assertion | JWT |
| STS API | AssumeRoleWithSAML | AssumeRoleWithWebIdentity |
| 常见 IdP | AD FS、Okta、OneLogin | Google、Facebook、Cognito |

### Role Chaining

**限制**：role chain 中后续 role 的最大有效期是 **1 小时**（不管 MaxSessionDuration 设多少）。来源：[AWS re:Post](https://repost.aws/knowledge-center/iam-role-chaining-limit)

```text
IAM User → AssumeRole → RoleA → AssumeRole → RoleB
                        (12h)                 (最多 1h!)
```

---

## 三、Encryption 基础 + KMS

### 对称 vs 非对称加密

|  | 对称 | 非对称 |
|---|---|---|
| 密钥 | 一把 | 一对：公钥 + 私钥 |
| 速度 | 快 | 慢 |
| 算法 | AES-256、ChaCha20 | RSA、ECC |
| AWS 场景 | KMS 默认、S3/EBS/RDS 加密 | TLS、数字签名 |

**AES-256 是对称加密**。AWS KMS 默认使用 **AES-256-GCM**。

### KMS 核心

- 密钥存储在 AWS 的 HSM（Hardware Security Module）里
- 密钥不会以明文形式离开 HSM
- 所有加密/解密操作在 KMS 内部完成

### KMS Key 类型 ⭐

| Key 类型 | 谁创建 | 能否审计 | 成本 |
|---|---|---|---|
| **AWS Owned Key** | AWS | ❌ | 免费 |
| **AWS Managed Key** | AWS 自动创建 | ✅ 控制台可见（`aws/` 前缀） | 免费 |
| **Customer Managed Key** | 你 | ✅ 完全控制 | 按月计费 + API 调用费 |

**什么情况必须用 Customer Managed Key**：

- 需要自定义 key policy
- 需要控制轮换
- 跨账户共享密钥
- 导入自己的 key material（BYOK）
- 使用非对称密钥

### KMS Key 核心限制

- **Symmetric key 最多能加密 4 KB（4096 字节）数据** — 超过就必须用 Envelope Encryption（[文档](https://docs.aws.amazon.com/kms/latest/developerguide/programming-encryption.html)）
- **Key 不能跨 Region** — 有 Multi-Region Keys 例外
- **Key 删除有等待期** — **最少 7 天，最多 30 天**（默认 30 天）（[ScheduleKeyDeletion](https://docs.aws.amazon.com/kms/latest/APIReference/API_ScheduleKeyDeletion.html)）
- **AWS Managed Key 轮换**：1 年一次，你不能改
- **Customer Managed Key 轮换**：可启用自动轮换，默认 365 天（可调 90 – 2560 天）（[KMS FAQ](https://aws.amazon.com/kms/faqs/)）

### Envelope Encryption ⭐⭐⭐

**为什么需要**：KMS symmetric key 最多加密 4 KB，大文件直接调 KMS 太慢太贵。

**核心思想**：

1. KMS 生成一个临时的 Data Key
2. 用 Data Key 在本地加密大文件
3. 把 Data Key 本身用 KMS Key 加密存起来
4. 解密时先用 KMS 解密 Data Key，再用 Data Key 解密文件

**完整加密流程**：

```text
1. 调 KMS GenerateDataKey(keyId='myKey')
   → 返回 {plain_data_key, encrypted_data_key}

2. 本地：用 plain_data_key 做 AES-256 加密大文件
   → encrypted_video

3. 本地：把 encrypted_data_key 附加到 metadata

4. 上传 S3：
   - encrypted_video 作为 object body
   - encrypted_data_key 作为 object metadata

5. ❗ 立刻从内存擦除 plain_data_key
```

### 四个 KMS API 的对比

| API | 用途 | 返回 | 何时用 |
|---|---|---|---|
| `Encrypt` | 直接加密数据 | 密文 | 只有 ≤4 KB 数据时 |
| `GenerateDataKey` | 生成并返回明文 + 密文两份 data key | Plaintext + Encrypted Data Key | 大数据（信封加密标准做法） |
| `GenerateDataKeyWithoutPlaintext` | 只返回加密版 data key | Encrypted Data Key only | 预生成，当时不立即使用 |
| `Decrypt` | 解密任何 KMS 加密过的密文 | 明文 | 解密 data key 或小数据 |

### Key Policy ⭐⭐⭐

> ⚠️ KMS Key **必须有 Key Policy**。**IAM Policy 单独不能授权访问 KMS Key**。

| Resource | Policy 关系（同账户内） |
|---|---|
| S3 bucket | identity policy **OR** bucket policy |
| KMS key | Key policy 必需 **AND** IAM policy 也要允许 |
| SQS queue | OR |
| Lambda | OR |
| IAM Role 的 trust policy | trust policy 必需（AND） |

**「自锁」陷阱**：如果不小心把 Key Policy 改成不允许任何人（包括自己），这个 key 就废了，必须开 AWS support ticket。

**最佳实践**：Key Policy 永远保留至少一个 `Principal: root` + `kms:*` 的 statement。

**跨账户 KMS 访问**：两边都要配（AND 关系）

- Account A 的 KMS Key Policy 允许 Account B 的 principal
- Account B 的 IAM Policy 允许调用相关 KMS API

### KMS Grants

**Grant** = 临时给某个 principal 授予使用 key 的权限，**但不修改 Key Policy**。

**Grant 适合**：

- AWS 服务代你访问 key
- 细粒度临时权限
- 不想改 Key Policy
- 可撤销（`RetireGrant` / `RevokeGrant`）

**记忆**：Key Policy = 长期、声明式；Grant = 短期、程序化。

### Key Rotation

|  | 自动轮换 | 手动轮换 |
|---|---|---|
| 谁支持 | Customer Managed Key (symmetric) | 任何 key |
| 频率 | 默认 1 年（可调 90–2560 天） | 随时 |
| 旧数据 | 不需要重新加密 | 需要重新加密 |
| Key ID / ARN | **不变** | 变（新 key） |
| 应用影响 | 无感 | 要更新引用 |

**自动轮换限制**：

- 只支持 Symmetric Customer Managed Key
- AWS Managed Key 每年自动轮换，你控制不了
- Imported Key Material（BYOK）不支持自动轮换
- Asymmetric Key 不支持自动轮换

> 补充：KMS 现已支持 `RotateKeyOnDemand` 按需轮换，包括导入的 key，每个 key 终身上限 10 次。“导入 key 不支持**自动**轮换”仍然成立（[KMS FAQ](https://aws.amazon.com/kms/faqs/)）。

### KMS 在 AWS 服务中的应用

| 服务 | KMS 用途 |
|---|---|
| S3 | SSE-KMS 加密 object |
| EBS | 加密 volume 和 snapshot |
| RDS | 加密数据库（创建时启用，事后不能改） |
| DynamoDB | 加密表 |
| Lambda | 加密环境变量 |
| Secrets Manager | 加密密钥 value |
| SSM Parameter Store | SecureString 类型用 KMS 加密 |
| CloudWatch Logs | Log group 级加密 |
