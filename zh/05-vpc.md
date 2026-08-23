# Module 5：VPC（网络）

> **覆盖日期**：4/29
>
> English version: [`en/05-vpc.md`](../en/05-vpc.md)

---

## VPC 基础

**= Virtual Private Cloud**，你在 AWS 里专属的虚拟网络。

**关键属性**：**Region 级资源**（一个 VPC 在一个 region，跨多个 AZ）；创建时指定 CIDR block（如 `10.0.0.0/16`，可装 65,536 个 IP）；每个 AWS 账户有一个默认 VPC（每 region）。

**核心组件**：Subnets、Internet Gateway (IGW)、NAT Gateway、Route Tables、Security Groups（instance 级）、NACLs（subnet 级）、VPC Endpoints、VPC Flow Logs。

### Subnets（子网）

**特性**：**AZ 级资源**（一个 subnet 只在一个 AZ）；一个 VPC 可以有多个 subnet；每个 subnet 有自己的 CIDR；AWS 在每个 subnet **保留 5 个 IP**：`.0`（网络地址）、`.1`（VPC router）、`.2`（DNS）、`.3`（保留）、`.255`（广播地址）。

### Public Subnet vs Private Subnet ⭐

**Public Subnet 必备两个条件**：✅ 路由表有指向 Internet Gateway 的 default route（`0.0.0.0/0 → IGW`）；✅ subnet 里的 EC2 有公网 IP（自动分配 public IP 或弹性 IP）。

**Private Subnet**：路由表没有指向 IGW；EC2 没有公网 IP；出网需要 NAT Gateway。

**经典三层架构**：

```
Public Subnet:    ALB(接收外部请求)
  ↓
Private Subnet:   EC2 应用层(不直接对外)
  ↓
Private Subnet:   RDS 数据库层(最严格隔离)
```

### Internet Gateway (IGW)

**功能**：让 VPC 能访问互联网（双向）。

**特点**：VPC 级（一个 VPC 一个 IGW）；高可用（AWS 托管）；没有带宽限制（免费）。

**Subnet 真正"通互联网"3 条件**：① VPC 有 IGW；② Subnet 路由表 `0.0.0.0/0 → IGW`；③ EC2 有 public IP（或弹性 IP）。

### NAT Gateway ⭐

**问题**：private subnet 里的 EC2 想访问互联网（yum update、调外部 API），但不想被外部直接访问。

**机制**：NAT 翻译——把 private EC2 的私有 IP 翻译成 NAT Gateway 自己的弹性公网 IP。

**特点**：AWS 托管——高可用、自动伸缩；**AZ 级资源**——一个 NAT GW 在一个 AZ；**多 AZ 高可用要在每个 AZ 部署一个 NAT Gateway**；付费（按小时 + 数据传输）。

### NAT Gateway vs NAT Instance

|  | NAT Gateway | NAT Instance |
|---|---|---|
| 性质 | AWS 托管服务 | 你自己跑的 EC2 |
| 高可用 | ✅ AZ 内自动 | ❌ 需要自己配 |
| 带宽 | 高，自动扩展（可达 100 Gbps） | 受 instance type 限制 |
| 维护 | 无 | 你自己 patch |
| 成本 | 较贵 | 便宜（小流量场景） |
| Source/dest check | 自动处理 | **必须手动关闭** |
| 当前推荐 | ✅ 首选 | ❌ Legacy，不推荐 |

### VPC Endpoints ⭐⭐⭐

**= 让 VPC 内的资源通过 AWS 内网访问 AWS 服务，不需要走公网/IGW/NAT**

**为什么需要**：

```
没有 VPC Endpoint:
Private EC2 → NAT Gateway → IGW → 公网 → S3
              ↑ 流量费贵 + 走公网

用 VPC Endpoint:
Private EC2 → VPC Endpoint → S3
              ↑ AWS 内网,免费(Gateway)
              ↑ 不用走公网,更安全
```

### 两种 VPC Endpoint ⭐⭐⭐

|  | Gateway Endpoint | Interface Endpoint |
|---|---|---|
| 支持服务 | **仅 S3 和 DynamoDB** | 几乎所有其他 AWS 服务 |
| 底层 | Route table entry | **ENI (PrivateLink)** |
| 占 IP 吗 | ❌ 不占 | ✅ 占 private IP |
| 费用 | **免费** ⭐ | 按小时 + GB |
| 作用范围 | VPC 级 | Subnet 级（每 AZ 一个 ENI） |
| 如何启用 | 路由表加条目 | 创建 ENI |

**Gateway Endpoint 支持的服务**：**只有两个**——S3 和 DynamoDB。

**Interface Endpoint 支持**：KMS、Secrets Manager、SQS、SNS、API Gateway、SSM、CloudWatch、ECR、Lambda 等。

**决策表**：

| 想从私有 EC2 访问 | 用什么 |
|---|---|
| S3 | **Gateway Endpoint**（免费） |
| DynamoDB | **Gateway Endpoint**（免费） |
| KMS / Secrets Manager / SSM | Interface Endpoint |
| SQS / SNS / EventBridge | Interface Endpoint |
| API Gateway / Lambda | Interface Endpoint |
| 其他 AWS 服务 | Interface Endpoint |

**记忆口诀**：**"S3 + DynamoDB = Gateway"**，剩下的全是 Interface。

**重要**：VPC 内的服务（RDS、ElastiCache、EC2、ELB）**不需要 VPC Endpoint**——Endpoint 是给 VPC 外的 AWS 服务用的。

### Security Group vs NACL ⭐⭐

|  | Security Group | NACL |
|---|---|---|
| 作用范围 | Instance 级（ENI） | Subnet 级 |
| 状态 | **Stateful**（有状态） | **Stateless**（无状态） |
| 规则类型 | **只 Allow** | Allow + Deny |
| 规则评估 | 全部规则一起看（任一允许即放行） | **按编号顺序，先匹配先生效** |
| 可以引用 SG | ✅ 可以 | ❌ 只能用 IP |
| 默认入站 | 拒绝所有 | 允许所有（默认 NACL） |
| 默认出站 | 允许所有 | 允许所有（默认 NACL） |
| 数量 | 一个 instance 最多 5 个 SG | 一个 subnet 一个 NACL |

### Stateful vs Stateless 的实战影响

**Stateful (SG) 例子**：SG 允许入站 80 端口 → web 服务器响应客户端时**不需要在出站规则里允许 80**。

**Stateless (NACL) 例子**：NACL 允许入站 80 端口 → 仅允许"进来"的连接；响应包出去时，**必须 NACL 出站规则也允许**（一般用 ephemeral ports 1024-65535）。

**陷阱题**：NACL 配了入站允许，但出站没配，HTTP 连接还是失败——因为 stateless，response 出不去。修复：NACL 出站规则要允许 1024-65535 的 ephemeral ports。

### 何时用哪个

| 场景 | 用 |
|---|---|
| 控制单个 EC2 的访问 | Security Group |
| **想 explicitly deny 某些 IP** | **NACL**（SG 没有 Deny 规则） |
| 子网级粗粒度防护 | NACL |
| 默认场景 | Security Group（NACL 用默认即可） |

**SG 引用 SG 的实战用法**：

```
ALB-SG: 允许 0.0.0.0/0 的 443 入站
EC2-SG: 允许 ALB-SG 的 8080 入站   ← 不写 IP,写 SG!
```

好处：ALB 自动伸缩、IP 随时变 → 引用 SG 不需要更新规则。NACL 做不到。

### VPC Peering

**= 两个 VPC 之间建立私网连接**

**特点**：双向连通；**不可传递**（A↔B、B↔C，但 A 和 C 不通，要单独建 A↔C）；跨 region、跨账户都行；**CIDR 不能重叠**。

**记忆**：2-3 个 VPC → VPC Peering；多 VPC（>3）→ Transit Gateway。

### Transit Gateway

**= 多 VPC + on-premise 的中央枢纽**

**用途**：VPC 数量多时，Peering 太复杂（n² 连接），用 Transit Gateway 简化（星型）。

### VPC Flow Logs ⭐

**= 记录 VPC 内的网络流量**

**输出位置**：CloudWatch Logs、S3、Kinesis Data Firehose。

**捕获级别**：VPC 级、Subnet 级、ENI 级（最细）。

**记录内容**：源/目标 IP、端口、协议；包数量、字节数；ACCEPT / REJECT；时间戳。

**陷阱**：Flow Logs **不记录某些流量**：❌ Amazon DNS server 流量（用自定义 DNS 则会记录）；❌ Windows License activation；❌ **IMDS 流量（169.254.169.254）** ⭐；❌ DHCP 流量；❌ VPC router 保留 IP（.1）。

**重要考点**：IMDS 不记录，无法通过 Flow Logs 追溯 IMDS 凭证盗取，必须靠 GuardDuty 等其他工具。

### Auto-assigned IP vs Elastic IP

- **Auto-assigned public IP**（默认）：Stop+Start 后会变（reboot 不变）
- **Elastic IP (EIP)**：你拥有的固定公网 IP——Attach 到 EC2 → IP 不变；Detach 后还可以 attach 到别的 EC2；**EIP 不 attach 到任何 EC2 时收费**（防止占着不用）；每个账户每 region 默认 5 个 EIP 配额

**想 IP 永远不变** → 用 **Elastic IP**

### 常见考题速记

| 题型 | 答案 |
|---|---|
| private EC2 怎么访问外部 API？ | NAT Gateway（如果是 AWS 服务，优先 VPC Endpoint） |
| private EC2 怎么访问 S3 而不走公网？ | **S3 Gateway Endpoint** |
| private EC2 怎么访问 Secrets Manager？ | Secrets Manager Interface Endpoint |
| 完全 isolated 的 VPC 如何用 AWS 服务？ | 所有需要的服务都建 VPC Endpoint |
| 连接两个 VPC 私网通信？ | VPC Peering（小）或 Transit Gateway（大） |
| 怎么 deny 特定 IP？ | NACL（SG 没有 Deny） |
| EC2 重启后能保持 IP 不变？ | Elastic IP |
