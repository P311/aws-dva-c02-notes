# Module 3：EC2 + 存储（EBS / Instance Store / EFS / FSx）

> **覆盖日期**：4/23、4/24、4/25
>
> English version: [`en/03-ec2-storage.md`](../en/03-ec2-storage.md)

---

## 一、EC2 基础

### Instance Types 命名规则

```
m       5        .xlarge
│       │         │
│       │         └─ Size（规模）
│       │
│       └─────────── Generation（代数）
│
└───────────────── Family（家族，用途分类）
```

**带额外字母**：`m5n.xlarge`、`c6g.large`、`r5dn.2xlarge`

- `a` = AMD CPU
- `g` = ARM (AWS Graviton)
- `n` = 高网络带宽
- `d` = 本地 NVMe SSD ⭐
- `i` = Intel（少见明写）

### 主要 Family

| 首字母 | 类型 | 典型用途 | 代表 |
|---|---|---|---|
| **T** | Burstable（突发型） | 低负载日常，允许短时突发 | t2, t3, t4g |
| **M** | General Purpose（通用） | 均衡 CPU/内存，Web server | m5, m6i, m7g |
| **C** | Compute Optimized | CPU 密集：批处理、HPC | c5, c6i, c7g |
| **R** | Memory Optimized | 大内存：数据库、缓存 | r5, r6i |
| **X** | High Memory | 超大内存：SAP HANA | x1, x2 |
| **I** | Storage (IOPS) Optimized | 高 IOPS 本地 SSD：NoSQL | i3, i4i |
| **D** | Dense Storage (HDD) | 大容量便宜存储 | d2, d3 |
| **G / P** | GPU | ML 训练/推理、图形渲染 | g4, p4 |
| **Inf** | ML 推理专用 | Inferentia 芯片 | inf1, inf2 |
| **Trn** | ML 训练专用 | Trainium 芯片 | trn1 |

**记忆口诀**：T = Tiny/Thrifty，M = Medium/Main，C = Compute，R = RAM，X = eXtreme memory，I = IOPS，D = Dense storage，G/P = Graphics/Parallel（GPU）

**Size 递进**：`nano → micro → small → medium → large → xlarge → 2xlarge → 4xlarge → 8xlarge → 16xlarge → 24xlarge`（每升一级，vCPU 和内存大约翻倍）

### T 系列 Burstable 机制

**工作原理**：

- 每个 T 实例有一个 CPU Credit 账户
- 基线（baseline）低于 100% CPU 时，攒 credit
- 需要高于基线的 CPU 时，消耗 credit
- Credit 用完就只能跑在基线水平

**两种模式**：

- **Standard 模式**：credit 用完就降速到基线
- **Unlimited 模式**（T3/T3a/T4g 默认!）：credit 用完继续全速，超出部分按小时付费

**T2 vs T3/T4g 默认模式不一样**：

- T2 默认是 Standard
- T3、T3a、T4g 默认是 **Unlimited** ⭐

**常见考点**：

- "T 实例 CPU 突然慢下来" → credit 耗尽（Standard 模式）
- "T3 账单意外多出 CPU 突发费用" → 因为 T3 默认 Unlimited

### AMI (Amazon Machine Image)

**AMI 来源**：

- AWS 提供（Amazon Linux、Ubuntu、Windows Server）
- AWS Marketplace（第三方，可能含 license 费）
- Community AMI（⚠️ 安全风险，不推荐生产用）
- 自己创建

**自己创建 AMI 流程**：

1. 启动一台 EC2，装软件、改配置、部署应用
2. 右键 → Create Image（或 `aws ec2 create-image`）
3. AWS 后台：停止 instance（可 `--no-reboot` 但可能快照不一致）→ 给所有 EBS 卷做 snapshot → 注册 AMI

**AMI 跨区域**：

- AMI 是 **Region 级资源**，不能跨 region 直接用 → 要 **Copy AMI** 到目标 region
- AMI 跨 region 复制时 AWS 会同时复制底层 snapshot
- 跨 region 复制加密 AMI：目标 region 必须有 KMS key

**跨账户共享 AMI 的两层权限**：

1. AMI 的 `launchPermission` — 加上目标账户 ID
2. 底层 EBS snapshot 的 `createVolumePermission` — 也加上目标账户 ID（易漏！）
3. （加密 AMI）KMS Key Policy — 允许目标账户使用这个 key

### Golden AMI vs User Data

|  | Golden AMI | User Data |
|---|---|---|
| 变更频率 | 慢（重做 AMI 几十分钟） | 快（改 Launch Template 即可） |
| 启动速度 | 快（软件预装） | 慢（要执行脚本） |
| 灵活性 | 低 | 高 |
| 网络依赖 | 启动时不依赖外网 | 通常依赖（要拉包） |
| 适合放什么 | OS、基础软件、安全加固 | 环境变量、动态配置、注册到服务发现 |

**最佳实践组合**：

- Golden AMI 包含：OS、基础工具（CloudWatch Agent、SSM Agent）、应用 runtime、应用代码、安全加固
- User Data 启动时做：从 Parameter Store 拉取配置、写入 properties 文件、启动应用、注册到 Service Discovery

**EC2 Image Builder** = AWS 托管的"自动构建 Golden AMI"服务，适合企业定期自动构建带最新补丁的标准镜像。

### User Data（启动脚本）

**特点**：

- 第一次启动时自动执行（只在第一次！）
- 以 **root** 身份执行
- 最大 **16 KB**（超过要放 S3，在 User Data 里下载）
- Linux：shell 脚本；Windows：PowerShell 或 cmd

**Linux 示例**：

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
```

**关键点**：

- User Data 默认**只运行一次**（在 `/var/lib/cloud/instance/sem/` 有标记文件）
- 强制每次启动都运行：配置 cloud-init `cloud_final_modules` 里 `scripts-user: always`
- **修改已有 instance 的 User Data**：必须先 **stop** instance（不是 reboot，不是 terminate）→ 改完 → start
- ⚠️ 但默认情况下 stop/start 后的新 User Data 不会再跑（除非配置了 always）

**User Data 执行日志位置**：

| OS | 日志位置 |
|---|---|
| Amazon Linux 2 / 2023 | `/var/log/cloud-init-output.log`（最常用） |
| Amazon Linux | `/var/log/cloud-init.log` |
| Ubuntu | 同上 |
| Windows | `C:\ProgramData\Amazon\EC2-Windows\Launch\Log\UserdataExecution.log` |

> ⚠️ User Data 日志**不在 CloudTrail**！CloudTrail 只记录 AWS API 调用。

**User Data + IMDS 经典模式**：

```bash
#!/bin/bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
REGION=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/region)

# 用 instance profile 凭证调 AWS（不用配 access key）
aws s3 cp s3://my-config-bucket/app.conf /etc/myapp/app.conf --region $REGION
systemctl start myapp
```

### Instance Identity Document

**路径**：`/latest/dynamic/instance-identity/document`

**内容**：JSON，包含 instance 身份信息（accountId、instanceId、region、AMI、pendingTime 等），**由 AWS 私钥签名**。

**用途**：第三方系统可以验证"这个请求真的来自某台 AWS EC2"。类比：身份证 + 政府公章。

### SSH Key Pair 和连接方式

**Key Pair**：

- AWS 保存公钥，你保存私钥（.pem 文件）
- 启动 EC2 时关联 → AWS 把公钥放到 EC2 的 `~/.ssh/authorized_keys`
- **私钥丢了 AWS 帮不了你** — 只能停止 instance、detach root volume、挂到另一台 EC2、重写 authorized_keys、再挂回去

**连接 EC2 的方式对比**：

|  | EC2 Instance Connect | Session Manager | 传统 SSH |
|---|---|---|---|
| 协议 | SSH | SSM | SSH |
| 入站端口 | 22 端口必须开（仅对 AWS IP 段） | **不需要任何入站端口** ⭐ | 22 必须开 |
| 私钥管理 | 不需要（AWS 推 60 秒临时 key） | 不需要 | 需要管 .pem 文件 |
| 认证方式 | IAM | IAM | SSH key |
| 审计 | CloudTrail | CloudTrail + 全程 session 录制 ⭐ | OS 日志 |
| 网络要求 | EC2 在 public subnet 或有公网 IP | EC2 出站能到 SSM 服务 | EC2 可达 |
| 推荐度 | 中 | **生产推荐** ⭐ | 已不推荐 |

**Session Manager 工作机制**：

1. EC2 装 SSM Agent（Amazon Linux 2/2023 默认装好）
2. EC2 有合适的 IAM role（含 `AmazonSSMManagedInstanceCore` policy）
3. EC2 主动连接 SSM 服务（出站 443）
4. 你在 console 点 "Connect" → 走 SSM → 进入 EC2 shell
5. 完全不开任何入站端口！

**EC2 Instance Connect 工作机制**：

1. 在 console 点 "Connect" → 浏览器 SSH
2. AWS 把临时 SSH 公钥推送到 EC2 的 `authorized_keys`，有效期 60 秒
3. AWS 用浏览器 WebSocket 模拟 SSH 客户端连接 EC2
4. 60 秒后临时公钥失效

**选择速记**：

- "完全不开入站端口" → **Session Manager**
- "不想管 SSH 密钥但仍用 SSH" → **EC2 Instance Connect**
- "全程 session 录制审计" → **Session Manager**

---

## 二、EC2 Advanced — Pricing + Placement Groups

### EC2 购买选项总览 ⭐⭐⭐

| 选项 | 价格 | 承诺 | 中断风险 | 典型场景 |
|---|---|---|---|---|
| **On-Demand** | 100%（基准） | 无 | ❌ 不会 | 短期、不确定的工作负载 |
| **Reserved Instance (RI)** | 省 ~30-72% | 1 或 3 年 | ❌ 不会 | 稳定的长期负载 |
| **Savings Plans** | 类似 RI（30-72%） | 1 或 3 年承诺消费金额 | ❌ 不会 | 灵活的长期负载 |
| **Spot Instance** | 省 ~70-90% | 无 | ⚠️ AWS 可随时回收 | 容错、批处理、CI |
| **Dedicated Host** | 最贵 | 可选 RI | ❌ 不会 | License 绑定物理机、合规 |
| **Dedicated Instance** | 较贵 | 可选 | ❌ 不会 | 物理机隔离 |
| **Capacity Reservations** | 按 On-Demand 价付 | 无承诺 | ❌ 不会 | 保证容量（不是省钱） |

### Reserved Instances (RI)

**类型**：

- **按支付方式**：All Upfront（折扣最大）/ Partial Upfront / No Upfront（折扣最小）
- **按灵活性**：
  - **Standard RI** — 折扣大（最高 72%），不能改 instance family
  - **Convertible RI** — 折扣小一点（~54%），可以换 family
- **Reserved Instance Marketplace** — 不再需要的 RI 可以卖给其他客户（仅 Standard RI 可卖）

**要点**：RI 是**对账折扣**，不是"预留某台机器"。Capacity Reservation 才是真"预留容量"。

### Savings Plans

|  | Compute Savings Plans | EC2 Instance Savings Plans |
|---|---|---|
| 灵活性 | 极高 | 中 |
| 适用范围 | EC2 + Fargate + Lambda | 仅 EC2 + 特定 region 的特定 family |
| 折扣 | 最高 ~66% | 最高 ~72% |
| 推荐 | 多数场景首选 | 已经知道用哪个 family |

**Savings Plans vs RI**：Savings Plans 更灵活（可换 region、family）；RI 折扣略大（同 instance 锁死）；AWS 推荐 Savings Plans。

**常见判断**：

- "想要折扣但担心未来换 instance 类型" → Convertible RI 或 Compute Savings Plans
- "Lambda 也要折扣" → Compute Savings Plans（RI 不覆盖 Lambda）

### Spot Instances ⭐⭐⭐

**核心机制**：买 AWS 闲置算力，最高省 90%；AWS 可以 **2 分钟通知**后回收。

**Spot 中断的两分钟通知**：

```bash
# 在 EC2 上检查是否被回收
curl http://169.254.169.254/latest/meta-data/spot/instance-action
```

返回 `{"action": "stop", "time": "..."}` → 即将被回收。

**Spot 中断时三种行为**：

- **Stop**（默认）— EBS 数据保留，可后来恢复
- **Hibernate** — 内存状态写到 EBS，恢复时连内存都还原
- **Terminate** — 直接销毁

**Spot 适合**：✅ 批处理（大数据 ETL、视频渲染）、✅ CI/CD（GitHub Actions self-hosted runner）、✅ 机器学习训练（中断了重启，结合 checkpoint）、✅ Web 服务器中的"弹性那部分"（base 用 on-demand，扩容用 spot）

**Spot 不适合**：❌ 长跑批处理（中断要重头来）、❌ 数据库、❌ 状态服务

**EC2 Fleet** 取代了 Spot Fleet — 同时买多种 spot + on-demand，AWS 自动选最便宜的组合。

### Dedicated Host vs Dedicated Instance

|  | Dedicated Host | Dedicated Instance |
|---|---|---|
| 隔离粒度 | 整台物理机器给你 | 不和别人共享 host，但你看不到 host |
| 物理机可见 | ✅ 看到 sockets、cores、host ID | ❌ AWS 透明 |
| BYOL | ✅ 可以用自带的 Windows / Oracle / SQL Server license | 部分支持 |
| 计费 | 按物理 host 收费 | 按 instance 收费 |
| 主要场景 | License 绑物理机、合规审计 | 仅需要"不和别人共享"的隔离 |
| 控制摆放 | ✅ 可以指定具体 host | ❌ 不能 |

**License 场景**：Per-socket / per-core 的 license（老 Oracle、SQL Server）→ 必须 Dedicated Host（看到 socket 数）；Per-VM 的 license → Dedicated Instance 也行；政府合规要求物理隔离 → Dedicated Host。

### Capacity Reservations

**和折扣无关，只是"占坑"**。

**特点**：按 on-demand 全价付费（不省钱！）；即使你不启动 instance 也要付钱（占坑费）；**可以和 Savings Plans / RI 折扣叠加**（reserve 提供容量保证，SP/RI 提供折扣）；随时取消。

**判断**："需要保证某天能启动 X 台 instance" → Capacity Reservations；"要省钱" → 不是 Capacity Reservations，是 RI / Savings Plans。

### Placement Groups ⭐⭐⭐

| 类型 | 摆放方式 | 网络性能 | 容错 | 场景 |
|---|---|---|---|---|
| **Cluster** | 同一 AZ 同一机架 | 极高（10 Gbps+ 低延迟） | 极差（机架挂全挂） | HPC、机器学习、大数据低延迟 |
| **Spread** | 不同物理机（同 AZ 或跨 AZ） | 普通 | 极好 | 关键应用、不能容忍单点故障 |
| **Partition** | 多个 partition，每个 partition 一组机器 | 普通 | 好（partition 间隔离） | Hadoop、Cassandra、HDFS |

**Spread Placement Group 限制**：每个 AZ 最多 **7 个 instance**。

**详解**：

- **Cluster** — 同一机架，网络延迟极低（亚毫秒），但机架/AZ 挂全挂。用途：HPC、大规模 MPI 任务。
- **Spread** — 任何机架/物理机/AZ 挂只影响一个 instance。用途：关键的小规模应用（数据库 master）。
- **Partition** — 每个 partition 是独立硬件（不同机架），可装很多 instance。用途：大型分布式系统（HDFS、Cassandra、Kafka），把数据分片放到不同 partition，单 partition 挂只丢一部分。

**场景速记**：

| 场景 | 选什么 |
|---|---|
| 节点间需要超低延迟通信（HPC、ML 训练） | Cluster |
| 关键应用、绝对不能多个同时挂、规模小 | Spread |
| 大型分布式数据库（Cassandra、HDFS） | Partition |

---

## 三、存储 — EBS, Instance Store, EFS, FSx

### EBS (Elastic Block Store)

**核心特性**：**Block storage**（像一块裸硬盘，可以装文件系统）；**网络挂载** — 通过网络访问，不是物理插在 EC2 上；**AZ 级资源** — 只能挂载到同一个 AZ 的 EC2；**持久化** — EC2 终止了 EBS 还在（除非配了"随 EC2 删除"）；可以 detach 然后 attach 到另一台 EC2（同 AZ 内）。

### EBS Volume Types ⭐⭐⭐

| 类型 | 介质 | 用途 | 性能 | 关键特性 |
|---|---|---|---|---|
| **gp3** | SSD | 通用，新默认 | 3,000-16,000 IOPS（独立配置） | IOPS 和容量解耦 ⭐ |
| **gp2** | SSD | 老的通用 | IOPS 随容量涨（3 IOPS/GB） | 已被 gp3 取代 |
| **io1 / io2** | SSD | 高性能、关键 DB | 最高 64,000 IOPS（io1）/ 256,000（io2 Block Express） | **多挂载（仅 io1/2）** ⭐ |
| **st1** | HDD | 大数据吞吐 | 高吞吐、低 IOPS | 不能做 boot volume |
| **sc1** | HDD | 冷数据、便宜 | 最低性能 | 不能做 boot volume |

**gp2 vs gp3**：gp2 的 IOPS 和容量挂钩（要更多 IOPS 必须扩容量）；gp3 **独立设置 IOPS 和容量**，便宜 ~20%，默认推荐；gp3 默认给 3,000 IOPS 和 125 MB/s。

**io1 / io2 的 Multi-Attach**：同一个 io1/io2 卷可以同时挂载到**最多 16 台 EC2**，必须**同一个 AZ**；⚠️ 应用必须支持并发写（普通 ext4 不行，需要 cluster-aware 文件系统如 GFS2、OCFS2）。

**HDD 限制**：st1 和 sc1 **不能做 root volume**（启动盘必须是 SSD）。

### EBS Snapshot

**特点**：**增量备份**（第一次全量，后续只存变化的 block）；存到 S3（你看不到 bucket）；可以**跨 region 复制**（用于 DR）；可以从 snapshot 创建新 volume（甚至换 type、换 size、换 AZ）；可以设为 public 或共享给特定账户。

**EBS Snapshot Archive**：存到归档层，**便宜 75%**，但**恢复需要 24-72 小时**，适合长期保留的合规备份。

**Recycle Bin**：防止误删 snapshot，可设置 retention period（最多 1 年）。

### EBS 加密

创建时启用加密，用 KMS key。加密的 volume：**snapshot 自动加密、从 snapshot 创建的 volume 自动加密**。一旦加密，衍生（snapshot、volume、AMI）都自动加密。

**已存在的未加密卷怎么加密**（常见操作流程）：

1. 给未加密 volume 做 snapshot（snapshot 也未加密）
2. **Copy snapshot，复制时勾选"启用加密 + 选 KMS key"** ⭐ 关键步骤
3. 从加密的 snapshot 创建新 volume（自动加密）
4. 替换原 volume（stop EC2 → detach 老 volume → attach 新加密 volume → start EC2）

**关键**：现有未加密 volume 本身**不能"原地加密"**，必须通过 CopySnapshot 时启用加密这条路径。

**EBS 跨 AZ/Region 迁移流程**：给原 volume 做 snapshot → （跨 region）CopySnapshot 到目标 region → 在目标 AZ 从 snapshot 创建新 volume → attach 到目标 AZ 的 EC2 → 验证后删原 volume。

### Instance Store

**= 物理上插在 EC2 物理机上的本地 SSD/NVMe**。

|  | EBS | Instance Store |
|---|---|---|
| 物理位置 | 网络存储 | 物理插在 host 机器上 |
| 速度 | 快 | **超快**（低延迟、高 IOPS） |
| 持久化 | ✅ 持久 | ❌ EC2 stop/terminate 数据全丢 |
| Snapshot | ✅ 支持 | ❌ 不支持 |
| 大小 | 灵活 | 固定（看 instance type） |
| 价格 | 单独计费 | **包含在 instance 价格里** |

**操作时数据行为**：

| 操作 | EBS 数据 | Instance Store 数据 |
|---|---|---|
| Reboot | ✅ 保留 | ✅ **保留**！ |
| Stop | ✅ 保留 | ❌ 丢失 |
| Terminate | 看 "delete on termination" | ❌ 丢失 |
| Hibernate | ✅ 保留 | ❌ 丢失 |
| Host 物理故障 | ✅ 保留（网络存储） | ❌ 丢失 |

**关键**：Reboot 不会丢 instance store 数据（host 没变，只是 OS 重启）；Stop = AWS 把 instance 从 host 搬走，下次 start 可能在另一个 host。

**带 d 字母的 instance type 含 instance store**：`m5d.xlarge`、`c5d.xlarge`、`r5d.xlarge`（中间 d）；`i3.xlarge`、`i4i.xlarge`（整个 i 系列）；`d2.xlarge`、`d3.xlarge`（整个 d 系列）。

**用途**：✅ 临时文件、缓存、buffer / scratch space；✅ 视频转码的临时文件；✅ 有自己复制机制的分布式系统的本地存储（Cassandra）；❌ 任何需要持久化的数据。

### EFS (Elastic File System)

**= AWS 托管的 NFS 文件系统（仅 Linux）**

**核心特性**：**多 EC2 同时挂载**（不像 EBS 默认单挂载）；**跨 AZ**（同一个 file system 在所有 AZ 都有 mount target）；跨 region（可以 replication）；**自动扩容**（按用量计费，不需要预先分配）；POSIX 兼容（标准 Linux 文件系统语义）；**仅 Linux**（Windows 用 FSx）；协议：**NFS v4**。

**EFS Performance Modes**：**General Purpose**（默认，绝大多数场景）、**Max I/O**（高度并行，10000+ EC2 同时访问，延迟略高）。

**EFS Throughput Modes**：**Bursting**（默认，按存储量给吞吐）、**Provisioned**（独立配置吞吐）、**Elastic**（自动伸缩，推荐，AWS 新方案）。

**EFS Storage Classes**：

| Class | 价格 | 访问 | 用途 |
|---|---|---|---|
| EFS Standard | 标准 | 多 AZ | 频繁访问 |
| EFS Standard-IA | 便宜 92% | 多 AZ，首次访问慢 | 不常访问的活数据 |
| EFS One Zone | 便宜 47% | **单 AZ**（不容灾） | 可重生成的数据 |
| EFS One Zone-IA | 最便宜 | 单 AZ + IA | 单 AZ 的不常访问 |

**Lifecycle Management**：自动把 N 天没访问的文件移到 IA（7、14、30、60、90 天可选）。

**用途**：✅ WordPress 等需要多服务器共享文件的应用；✅ 容器（ECS/EKS）的共享存储；✅ 内容管理系统、CI/CD 共享；✅ 大数据 / ML 训练数据；❌ Windows 应用 → FSx for Windows；❌ 块存储语义（数据库 raw device）→ EBS。

### FSx 家族

| FSx 类型 | 文件系统 | 用途 |
|---|---|---|
| **FSx for Windows File Server** | Windows SMB / NTFS | Windows 应用共享存储 |
| **FSx for Lustre** | Lustre（HPC 文件系统） | HPC、ML 训练，超高性能 |
| **FSx for NetApp ONTAP** | NetApp 企业文件系统 | 从企业 NetApp 迁移到 AWS |
| **FSx for OpenZFS** | OpenZFS | 从 ZFS / Linux NFS 迁移 |

**FSx for Lustre 特殊点**：可以**链接到 S3 bucket**——S3 里的对象自动映射成 Lustre 文件系统里的文件；任务完成后，结果可以同步回 S3；用于 HPC：S3 是"数据湖"，Lustre 是"高速计算缓存"。

**速记**：Windows 共享文件 → **FSx for Windows File Server**；HPC / 机器学习训练高速读 → **FSx for Lustre**（可与 S3 集成）；企业 NetApp 客户上云 → FSx for ONTAP。

### 存储类型对比总表

| 服务 | 类型 | 共享性 | 持久 | Linux/Win | 用途速记 |
|---|---|---|---|---|---|
| **EBS** | 块 | 单挂载（io1/2 可多） | ✅ | 都行 | EC2 的"硬盘" |
| **Instance Store** | 块 | 单 EC2 | ❌ | 都行 | 临时高速 |
| **EFS** | 文件 (NFS) | 多 EC2 共享 | ✅ | 仅 Linux | Linux 共享文件 |
| **FSx Windows** | 文件 (SMB) | 多 EC2 共享 | ✅ | Windows | Windows 共享文件 |
| **FSx Lustre** | 文件 | 多 EC2 共享 | ✅ | Linux | HPC 高速 |
| **S3** | 对象 | 全球 | ✅ | API 访问 | 对象存储 |

**场景判断**：

| 需求 | 选什么 |
|---|---|
| 单 EC2 高速临时空间 | Instance Store |
| 多 EC2 共享高速 + S3 集成（HPC/ML） | FSx for Lustre |
| 多 EC2 共享 Linux 文件 | EFS |
| 多 EC2 共享 Windows 文件 | FSx for Windows |
| EC2 持久化块存储 | EBS |
| 数据库的存储 | **EBS**（块设备语义、低延迟、高 IOPS） |
| 视频转码的本地高速 buffer | **Instance Store**（临时数据 + 超高速 + 包含在价格里） |
