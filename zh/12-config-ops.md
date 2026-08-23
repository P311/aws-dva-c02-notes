# Module 12：配置与运维（Secrets Manager / Parameter Store / Systems Manager）

> **覆盖日期**：5/14
>
> English version: [`en/12-config-ops.md`](../en/12-config-ops.md)

---

## 一、AWS Secrets Manager ⭐⭐⭐

**= AWS 完全托管的密钥/凭证管理服务**，主打**自动轮换**。

**核心价值**：✅ 集中管理数据库密码、API key、token、应用凭证；✅ **自动轮换密码**（最大卖点，与 RDS/Aurora/Redshift/DocumentDB 深度集成）；✅ KMS at-rest 加密；✅ 跨 region 复制（DR 场景）；✅ 跨账号共享（resource policy）；✅ CloudTrail 审计、CloudWatch 监控；✅ IAM 细粒度访问控制。

**关键限制**：单个 secret 最大 **64 KB**；计费按 secret 数量 + API 调用次数（具体价格以官方定价页为准）；加密用 KMS（默认 `aws/secretsmanager`，可换 customer key）；支持自动轮换（Lambda 或 managed）；支持多 region 复制。

### Secret 版本和标签 ⭐⭐

每个 secret 可以有多个版本，每个版本有 staging labels：

| Label | 含义 |
|---|---|
| **AWSCURRENT** | 当前生效的版本（应用读到的就是这个） |
| **AWSPENDING** | 轮换中，新生成但未生效的版本 |
| **AWSPREVIOUS** | 上一个 current 版本（轮换完成后保留，用于回滚） |

**关键洞察**：轮换不是"删旧建新"，而是在 staging label 之间移动指针——新版本写入，label = AWSPENDING → 测试通过后 label 切到 AWSCURRENT（旧的 AWSCURRENT → AWSPREVIOUS）→ 应用 `GetSecretValue` 默认读 AWSCURRENT。

### Automatic Rotation ⭐⭐⭐

**两种形式**：

**1. Managed Rotation**（托管轮换）：不用写 Lambda，AWS 直接管理。仅适用于这些"managed secrets"：RDS/Aurora（MySQL/PostgreSQL/Oracle/SQL Server/MariaDB）、Amazon Redshift、Amazon DocumentDB。

**2. Rotation by Lambda function**（Lambda 轮换）：适用于其他类型 secrets（自定义 API key、第三方服务凭证）。AWS 提供预置模板，也可自定义。

**4 步轮换流程** ⭐⭐⭐：Secrets Manager 调用 Lambda 时传 `Step` 参数：

```
1. createSecret    → 生成新凭证(如新密码),写入 secret 的 AWSPENDING 版本
2. setSecret       → 把新凭证更新到目标服务(如 RDS 改密码)
3. testSecret      → 用新凭证连接服务,验证是否生效
4. finishSecret    → 把 AWSPENDING 标签移到 AWSCURRENT,旧 AWSCURRENT → AWSPREVIOUS
```

⚠️ 如果任何一步失败，Secrets Manager 会在轮换窗口内重试整个流程。

### 两种 Rotation Strategy ⭐⭐

**1. Single User Rotation**（单用户轮换）：一个 DB user 用于应用，轮换时修改它的密码。流程：创建新密码 → 改 DB 中该 user 的密码 → 验证 → 完成。⚠️ 轮换瞬间旧密码失效、新密码生效，应用如果缓存了旧密码会短暂报错。适合简单场景，可容忍瞬间中断。

**2. Alternating Users Rotation**（交替用户轮换）：两个 DB user 交替使用。流程：用 admin/superuser secret 克隆当前用户 → 创建另一个用户（同权限，不同密码）→ 切换。**零停机**：旧 user 仍有效一段时间，应用慢慢切到新 user。需要额外提供 superuser secret（因为要 CREATE USER）。⚠️ RDS Proxy 不支持 alternating users。

**选择**：简单+容忍瞬间中断 → Single User；零停机/严格 SLA → Alternating Users。

### Cross-Account / Cross-Region

**Resource Policy**（跨账号）：

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::OTHER-ACCOUNT:role/AppRole" },
  "Action": ["secretsmanager:GetSecretValue"],
  "Resource": "*"
}
```

KMS key policy 也要允许对方账号使用 KMS key。

**Multi-Region Replication**：控制台/API 配置 replica region；复制是异步的（秒级）；应用在每个 region 直接读本地 replica（低延迟）；主 secret 改了副本自动同步；副本可以单独轮换（独立的 KMS key）。

### 缓存与最佳实践

**问题**：每次 Lambda invocation 都调 `GetSecretValue` → API 调用费爆炸 + 性能差。

**解决**：**AWS Parameters and Secrets Lambda Extension** ⭐——作为 Lambda Layer 附加；本地缓存 secret（默认 TTL 300 秒）；Lambda 代码通过 `localhost:2773` 读 secret；大幅减少 API 调用。也可以用各语言的 AWS SDK 客户端缓存库。

---

## 二、SSM Parameter Store ⭐⭐⭐

**= AWS Systems Manager 的配置管理服务**。核心价值：存储配置数据/密钥；层次化路径便于组织；版本跟踪；集成 KMS 加密（SecureString）；IAM 细粒度访问控制；Standard tier 完全免费。

### Parameter 类型 ⭐

| 类型 | 含义 |
|---|---|
| **String** | 普通字符串（明文） |
| **StringList** | 逗号分隔的字符串列表 |
| **SecureString** | 用 KMS 加密的敏感数据 |

### Standard vs Advanced Tier ⭐⭐⭐

| 维度 | **Standard** | **Advanced** |
|---|---|---|
| 每 region parameter 数上限 | **10,000** | **100,000** |
| 单 parameter 大小 | **4 KB** | **8 KB** |
| 价格 | **免费** | 按 parameter 数量收费 |
| Parameter Policies（TTL、过期通知） | ❌ | ✅ |
| 加密方式（SecureString） | KMS Encrypt（直接） | Envelope encryption with AWS Encryption SDK |
| 是否可降级 | / | ❌ **Advanced 不能降回 Standard**（必须删了重建） |

**Intelligent Tier**：AWS 自动决定用 Standard 或 Advanced（超过 4KB 自动升级）。

### 命名规则 / Hierarchy ⭐

**最佳实践**：用 `/` 分层命名：

```
/myapp/dev/db/url
/myapp/dev/db/password
/myapp/prod/db/url
/myapp/prod/db/password
/myapp/prod/api/stripe-key
```

好处：✅ IAM policy 可以基于路径授权（`"Resource": "arn:aws:ssm:us-east-1:111:parameter/myapp/dev/*"`）；✅ API `GetParametersByPath` 可批量取一个 prefix 下所有 parameters。

**重要细节**：Parameter name 顶层不要以 `aws` 或 `ssm` 开头（保留前缀）。

### Parameter Policies（Advanced tier 专属）⭐

**= 给 parameter 加策略**（类似 lifecycle）：

| Policy | 用途 |
|---|---|
| **Expiration** | 到期自动删除 |
| **ExpirationNotification** | 到期前 N 天 EventBridge 发通知 |
| **NoChangeNotification** | 超过 N 天没变更，发通知（提醒轮换） |

### Versions

每次 `PutParameter --overwrite` 创建一个新版本（自动递增）；默认读最新版本；可读特定版本（`Name: my-param:3`）；默认保留一定数量的历史版本（超过自动删除最老的）。

### 引用 Secrets Manager ⭐

Parameter Store 可以直接引用 Secrets Manager 的 secret：`/aws/reference/secretsmanager/my-secret-name`。读这个 parameter 会透明地拿到 secret 的当前值。用途：统一在 Parameter Store API 取所有配置，即使部分实际存在 Secrets Manager（享受轮换）。

### 引用 Public Parameters

AWS 在 Parameter Store 公开了一些常用值，如最新的 Amazon Linux AMI ID：`/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2`。EC2 Auto Scaling、CloudFormation 用得很多。

---

## 三、Secrets Manager vs Parameter Store ⭐⭐⭐

| 维度 | **Secrets Manager** | **Parameter Store** |
|---|---|---|
| 价格 | 按 secret 数量收费 | **Standard 免费**；Advanced 按数量收费 |
| 大小 | 64 KB | Standard 4 KB / Advanced 8 KB |
| 自动轮换 | ✅ **原生支持**（Lambda 或 managed） | ❌ 只能自己写（用 EventBridge + Lambda） |
| 跨 region 复制 | ✅ 原生 | ❌ |
| 加密 | 默认 KMS | SecureString 类型用 KMS |
| Resource Policy | ✅（跨账号） | ❌（只 IAM） |
| 整合 RDS/Redshift | ✅ 深度集成 | ❌ |
| 典型用途 | 数据库密码、API key（需要轮换） | 配置值、URL、feature flag、不需轮换的 secret |

### 选择决策 ⭐⭐⭐

| 场景 | 选 |
|---|---|
| RDS/Aurora/Redshift 密码 | **Secrets Manager**（原生 managed rotation） |
| 第三方 API key 需要定期轮换 | **Secrets Manager**（Lambda 轮换） |
| 应用配置（database URL、feature flag） | **Parameter Store** Standard |
| 静态 API key（不轮换） | **Parameter Store** SecureString（省钱） |
| 大量 secrets 都需要轮换 | **Secrets Manager**（虽贵但功能完善） |
| 大量配置值，大多不变 | **Parameter Store**（省钱） |
| 跨 region DR 数据库密码 | **Secrets Manager**（原生复制） |
| 需要 N 天后自动失效的临时凭证 | **Parameter Store Advanced**（Parameter Policies） |
| 需要 8 KB+ 的大配置 | **Parameter Store Advanced** 或 **Secrets Manager** |

**经典 trick**：在 Parameter Store 引用 Secrets Manager（`/aws/reference/secretsmanager/xxx`）→ 应用统一 API 取值，享受 Secrets Manager 轮换 + Parameter Store 价格。

---

## 四、AWS Systems Manager 工具集 ⭐⭐

**= AWS 的"运维瑞士军刀"**，管理 EC2/on-premises/多云节点。

**核心机制**：节点装 SSM Agent（预装在大多数 AMI），Agent 连接 SSM service → AWS 可以管理这台机器。

**配置 SSM Agent 需要**：✅ IAM Instance Profile，attach `AmazonSSMManagedInstanceCore` 托管 policy；✅ SSM Agent 已安装并运行；✅ 出站 HTTPS 访问到 SSM endpoint（VPC Endpoint 或公网）。

**或更简单**：启用 **Default Host Management Configuration**（较新功能），所有 EC2 自动作为 managed node，不需要手动配 IAM instance profile。

### 主要工具

**1. Session Manager**（⭐⭐⭐ 高频）：浏览器/CLI shell 连接 EC2，无 SSH/无 bastion/无开放端口。核心价值：不需要 SSH key（全 IAM 管理）；不需要 22 端口开放；不需要 bastion host；完整审计（所有 session 写入 CloudTrail + CloudWatch Logs + S3）；跨 AWS 账号；支持 Windows/Linux/macOS。

**判断**："想 SSH 到 private subnet EC2 但不想开 22 端口/bastion" → Session Manager；"合规要求 shell 操作全审计" → Session Manager（把日志发到 S3/CloudWatch）。

**2. Run Command**（⭐⭐ 高频）：在多台 managed nodes 上一次性执行 predefined command/script。用 SSM Documents（JSON/YAML）定义要做的事；AWS 预置很多 documents（`AWS-RunShellScript`、`AWS-UpdateSSMAgent`）；可以批量在标签匹配的所有 instances 上执行。vs Session Manager：Run Command = 预定义命令、批量、无交互；Session Manager = 交互式 shell、单机。

**3. Patch Manager**（⭐ 中频）：自动化 OS/应用 patch。核心概念：Patch Baseline（定义哪些 patch 自动批准/拒绝/批准延迟）、Patch Group（用 tag 分组 instances，不同组用不同 baseline）、Maintenance Window（计划特定时间应用 patches）、Compliance（扫描报告哪些 instance 缺哪些 patch）。

**4. State Manager**（⭐ 中频）：持续保持 instance 处于"期望状态"，类似 Ansible/Puppet。定义"我要 instance 满足这些条件"，按 schedule 检查+修正 drift。典型用途：确保所有 EC2 装了 CloudWatch Agent；确保 firewall 配置不变。

**5. Maintenance Windows**（⭐ 中频）：定时维护任务的窗口期，配置 cron schedule，关联 Run Command/State Manager/Lambda/Step Functions 作为 task。

**6. Inventory**（⭐ 低频）：自动收集 instance 元数据（软件、文件、网络、注册表等）。

**7. Distributor**（⭐ 低频）：软件包分发管理，把内部软件打包成 package，用 Run Command/State Manager 部署，支持版本管理、回滚。

**8. Automation (Runbooks)**（⭐⭐ 中频）：多步骤的运维自动化流程，用 SSM Documents 定义，AWS 提供大量预置 runbooks（创建 AMI、重启 instance、应急响应），可以跨账号/跨 region 执行。

**9. OpsCenter**（⭐ 低频）：集中查看/处理 OpsItems，来源包括 CloudWatch 告警、Config 不合规、Security Hub findings，关联 Automation runbook 一键处理。

**10. Change Manager**（⭐ 低频）：运维变更的审批工作流，模板化 change request，多级审批，关联 runbook 自动执行变更。

**11. Fleet Manager**（⭐ 中频）：统一 UI 管理 instance fleet，浏览器界面查看 file system/registry/processes/log，不需要 SSH/RDP，一键启动 Session Manager/Run Command。

### SSM Documents ⭐

**= Systems Manager 工具的"剧本"**（JSON 或 YAML）

| 类型 | 用途 | 由谁用 |
|---|---|---|
| **Command** | 执行 shell 命令/脚本 | Run Command、State Manager |
| **Automation** | 多步运维流程（Runbook） | Automation |
| **Session** | 定义 Session Manager 行为 | Session Manager |

**Document Sharing**：Private（默认，只本账号能用）、Shared（跨账号共享）、Public（全网公开，AWS 维护的 `AWS-*` documents）。

---

## 五、综合应用场景

**完整场景：Lambda + RDS + Secrets Manager + Parameter Store**——Lambda 启动 → Lambda Extension 读 Secrets Manager（cached）拿到 DB password → 读 SSM Parameter Store（cached）拿到 DB endpoint 等配置 → 连 RDS 处理业务。定期自动轮换：Secrets Manager 调 Lambda → 改 RDS 密码 → 切换 secret 版本 → Lambda 下次读到新密码。

**IaC 中的引用**：

CloudFormation 引用 SSM Parameter：

```yaml
Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2:1}}'
      KeyName: '{{resolve:ssm:my-keypair-name}}'
```

CloudFormation 引用 Secrets Manager：

```yaml
DBPassword: '{{resolve:secretsmanager:my-db-secret:SecretString:password}}'
```

⚠️ CloudFormation 的 `resolve` 不会在 template 里存明文——它在 deploy 时拉值。
