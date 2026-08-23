# Module 6：数据库（RDS / Aurora / ElastiCache / DynamoDB）

> **覆盖日期**：4/30、5/1、5/2、5/3
>
> English version: [`en/06-databases.md`](../en/06-databases.md)

---

## 一、RDS + Aurora + Read Replicas vs Multi-AZ

### RDS 是什么

**RDS = Relational Database Service**。AWS 托管的关系型数据库服务，你不用管 OS、补丁、备份、failover。

**支持的引擎**：MySQL、PostgreSQL、MariaDB、Oracle、Microsoft SQL Server、Amazon Aurora（MySQL/PostgreSQL 兼容，AWS 自研）。

**RDS 不支持 SSH 进 OS**（完全托管），想要 OS 级访问 → **RDS Custom**（只支持 Oracle 和 SQL Server）。

**核心组件**：DB Instance（数据库服务器，底层是 EC2 但你看不到、不能改）；DB Subnet Group（跨多 AZ 部署所需的 subnet 集合）；Parameter Group（引擎参数，类似 my.cnf）；Option Group（引擎特定功能，如 Oracle 的 TDE）；Endpoint（DNS 名字，客户端连接用）。

**RDS 存储**：默认 gp2 / gp3 / io1（SSD）；可以启用 Storage Auto Scaling（磁盘满了自动扩容，配上限）；不能直接 detach（和 EBS 不同）。

### Multi-AZ（高可用）⭐⭐⭐

**= 主备同步复制，跨 AZ**

```
AZ-1                       AZ-2
┌──────────────┐          ┌──────────────┐
│  Primary DB  │ ──同步─→  │  Standby DB  │
│  (读+写)     │          │  (不接流量)  │
└──────────────┘          └──────────────┘
       ↑
   Endpoint(DNS)
       ↑
    应用程序
```

**关键特性**：✅ **同步复制**（synchronous）；✅ Standby **不接受任何流量**（不能用于读写）——纯热备；✅ 主库挂了 → AWS 自动 failover（DNS 切到 standby）→ 通常 60-120 秒；✅ 跨 AZ（默认 2 个 AZ；部分引擎支持 3 个 AZ Multi-AZ Cluster）；✅ Failover 触发场景：AZ 故障、主库故障、维护、强制 reboot with failover。

**用途**：**HA（高可用）**，**不是性能优化**。

**记忆**：Multi-AZ = 灾备，Standby 不分担读流量。

### Read Replicas（读扩展）⭐⭐⭐

**= 主库异步复制到副本，副本可读**

```
              Primary DB(读+写)
                   │
        ┌──────────┼──────────┐
        │ 异步     │          │
        ▼          ▼          ▼
    Replica 1  Replica 2  Replica 5
    (只读)     (只读)     (只读)
```

**关键特性**：✅ **异步复制**（asynchronous）——副本可能稍微落后（replica lag）；✅ 副本**只读**（应用要分流读到 read replica endpoint）；✅ **最多 15 个 Read Replicas**（MySQL/PostgreSQL/MariaDB/SQL Server/Db2），**Oracle 是 5 个**（Aurora 也是 15 个）；✅ 可以在同 AZ、跨 AZ、**跨 region**；✅ 副本可以"晋升"成独立数据库（promote）；✅ 跨 region 副本可以用于 DR。

**用途**：**读性能扩展**，**报表查询**（不影响主库），**跨 region DR**。

**注意**：跨 region Read Replica 走公网（除非配 VPC peering）→ 收跨 region 流量费。

### Multi-AZ vs Read Replica ⭐⭐⭐

|  | Multi-AZ | Read Replica |
|---|---|---|
| 复制方式 | **同步** | **异步** |
| 副本是否可读 | ❌ 不行（纯热备） | ✅ 只读 |
| 主要目的 | **高可用 / DR** | **读性能扩展** |
| 自动 failover | ✅ 有 | ❌ 没有（要手动 promote） |
| 跨 region | ❌ 不行（同 region 跨 AZ） | ✅ 可以 |
| 数量 | 1 个 standby | **最多 15 个**（Oracle 5 个，Aurora 15 个） |
| 数据一致性 | 强一致 | 最终一致（有 lag） |

**经典场景题**："数据库需要高可用" → **Multi-AZ**；"数据库读慢，主库 CPU 高" → **Read Replica**（分摊读流量）；"需要在不影响生产的情况下跑大查询/报表" → **Read Replica**；"DR（灾备到另一个 region）" → **Cross-region Read Replica**（或 Aurora Global Database）。

**陷阱题**：Read Replica 不能用作 HA（主库挂了不会自动切），Multi-AZ 不能用作读扩展。

### RDS 加密 ⭐

**关键规则**：✅ 创建时启用 KMS 加密（选择 KMS key）；❌ **已存在的未加密 RDS 不能直接启用加密**！想加密已有数据库：**snapshot → copy snapshot 时启用加密 → 从加密 snapshot 还原**（类似 EBS 流程）。

**其他加密细节**：加密包括数据、备份、snapshot、Read Replica、log；Read Replica 必须和主库使用同样的加密设置；**Transparent Data Encryption (TDE)**：Oracle 和 SQL Server 引擎层加密（应用层）——在 Option Group 配置。

### RDS 备份

|  | Automated Backup | Manual Snapshot |
|---|---|---|
| 触发 | 自动（每天） | 手动 |
| 保留期 | 0-35 天（0 = 关闭） | 永久（直到你删） |
| 删除 RDS 后 | 删除 RDS 时一起删（可选保留） | 永久保留 |
| 用途 | 短期恢复（point-in-time） | 长期归档、跨账户共享 |

**Point-in-Time Recovery (PITR)**：从备份恢复到任意时间点（精度到秒），通过 transaction log 重放，只能在 backup retention 期内。

**RDS 没有"原地恢复"**：任何 restore 都是创建新的 RDS instance（新 endpoint！）。

### RDS Proxy ⭐

**问题**：Lambda 调 RDS 时，每个 invocation 建一个新连接 → 高并发时把 RDS 连接数打满。

**解决**：RDS Proxy = 连接池 + failover 加速。

**特性**：✅ **连接池**（应用看到很多连接，实际共享少数 RDS 连接）；✅ **failover 速度**：Multi-AZ failover 时，RDS Proxy 帮你保持连接，**failover 时间从 60s 降到 < 30s**；✅ **IAM 认证**（不用密码连数据库）；✅ 与 **Secrets Manager** 集成；✅ 在 VPC 内，不暴露到公网；✅ 适合：**Lambda 高并发**、Auto Scaling 应用。

**判断**："Lambda 频繁连 RDS，连接数打爆" → **RDS Proxy**；"想要数据库 IAM 认证 + 隐藏密码" → RDS Proxy + Secrets Manager。

### Aurora ⭐⭐⭐

**= AWS 自研的云原生关系型数据库**（MySQL / PostgreSQL 兼容）。为什么特殊：它的存储层是 AWS 重写的，不是传统 MySQL/PG 的 InnoDB/heap。

**Aurora 存储架构**：

```
        Aurora Cluster
              │
   ┌──────────┴──────────┐
   │                     │
Compute Layer         Storage Layer (Cluster Volume)
- Writer instance     - 6 copies across 3 AZs
- Reader instances    - 4/6 for write quorum
                      - 3/6 for read quorum
                      - Self-healing
                      - Auto-grow up to 128 TB
```

**关键特性**（全是高频考点 ⭐）：✅ **6 个数据副本，跨 3 个 AZ**（每 AZ 2 个）；✅ **写入需要 4/6 副本确认**（quorum write）；✅ **读取需要 3/6 副本响应**（quorum read）；✅ 任意 1 个 AZ 挂（2 副本丢失）→ 不影响写；✅ 任意 2 个副本丢失 → 不影响读；✅ **自愈**：坏的 block 自动从其他副本恢复；✅ 存储自动伸缩，**最大 128 TB**。

> Aurora 声称比 RDS MySQL/PostgreSQL 快数倍，具体倍数以 AWS 官方最新数据为准。

### Aurora Replicas

最多 **15 个 read replica**（RDS 是 5 个）；**复制延迟通常 < 100ms**（共享存储，不是真的"复制"，是直接读同一存储）；**自动 failover**：主挂了，某个 replica 在 < 30 秒内升主。

### Aurora Cluster Endpoints ⭐

| Endpoint | 用途 |
|---|---|
| **Writer Endpoint** | 写入（总是指向当前 master） |
| **Reader Endpoint** | 读取（自动 load balance 到所有 read replica） |
| **Custom Endpoint** | 自定义 instance 子集（如把分析 query 路由到特定 replica） |
| **Instance Endpoint** | 直接连特定 instance（很少用，因为 instance 可能切换） |

**应用最佳实践**：写 → Writer Endpoint；读 → Reader Endpoint（自动负载均衡）；不要硬编码 instance endpoint（failover 后可能不是主了）。

### Aurora Serverless ⭐

**= 自动启停 + 自动伸缩的 Aurora**

**v1 vs v2**：**Aurora Serverless v1**——capacity 单位是 ACU（Aurora Capacity Unit），从 0 缩到很高，有冷启动；**Aurora Serverless v2**——**扩缩速度快得多**，精细到 0.5 ACU 增量，推荐生产。

**适合**：✅ 间歇性/不可预测工作负载、✅ 开发测试环境、✅ 新应用（不知道流量大小）；❌ 持续高负载（provisioned 更便宜）。

### Aurora Global Database ⭐

**= 跨 region Aurora**

**特性**：1 个 primary region + 最多 5 个 secondary region；**跨 region 复制延迟通常 < 1 秒**；Secondary region 只读；**DR**：从 secondary promote 到 primary，RTO/RPO 都很短（以官方文档具体数字为准）；走 AWS 内部专线，不走公网。

**对比 Cross-region Read Replica**：Global Database 更快、更专业，但只支持 Aurora。

### Aurora Backtrack（只 Aurora MySQL）

**= 把数据库"倒回"到过去某个时间点**（无需 restore）。不删除当前数据，直接"时间机器"回到过去；可以反复 backtrack（前进/后退）；有时间窗口上限（以文档为准）；**比 PITR 快得多**（秒级）。用途：误删数据、误操作、应用 bug 写错数据。

### Aurora 其他特性（速览）

**Aurora Multi-Master**：多个 writer（罕见，处理复杂）；**Aurora Clone**：从现有 cluster 创建新 cluster，**copy-on-write**（几乎瞬间完成，不复制数据）——适合给开发环境一份生产数据副本；**Database Activity Streams**：CloudTrail 记录不到的数据库内部操作审计。

---

## 二、ElastiCache（Redis vs Memcached）+ 缓存模式

### ElastiCache 是什么

**= AWS 托管的内存缓存服务**。两个引擎：**Redis**、**Memcached**。核心价值：把热点数据放内存，减少 DB 负载，降低延迟。

### Redis vs Memcached ⭐⭐⭐

|  | Redis | Memcached |
|---|---|---|
| 数据结构 | 丰富（String, List, Set, Sorted Set, Hash, Stream, Bitmap, HyperLogLog, Geo） | 只有 key-value（String） |
| 持久化 | ✅ 支持（AOF + RDB snapshot） | ❌ 纯内存，重启丢数据 |
| 复制 | ✅ Master + Read Replica | ❌ 不支持 |
| Multi-AZ | ✅ 支持（自动 failover） | ❌ 不支持 |
| 备份 | ✅ 支持 snapshot | ❌ 不支持 |
| 多线程 | ❌ 单线程（每个 shard） | ✅ 多线程 |
| 分片 (sharding) | ✅ Cluster Mode | ✅ 自动 sharding |
| Pub/Sub | ✅ | ❌ |
| 事务 | ✅ MULTI/EXEC | ❌ |
| Lua 脚本 | ✅ | ❌ |
| 加密（at rest / in transit） | ✅ | ❌ |
| HIPAA / 合规 | ✅ | ❌ |

### 选型决策

**用 Redis 的场景**：✅ 需要持久化（重启不丢数据）、✅ 需要 HA / Multi-AZ、✅ 需要复杂数据结构（排行榜、消息队列、地理位置）、✅ 需要 pub/sub、✅ 合规要求（加密、HIPAA）、✅ 缓存 + Session Store + Leaderboard 等高级用例。

**用 Memcached 的场景**：✅ 纯简单 key-value 缓存、✅ 需要多线程吞吐、✅ 数据丢了无所谓（重新缓存）、✅ 横向 scale（自动 sharding）。

**默认选 Redis**（覆盖大多数场景）。Memcached 只在"我就要纯多线程缓存，数据可丢"时考虑。

### Redis 部署模式

**1. Cluster Mode Disabled**（单 shard）：1 个 master + 最多 5 个 read replica；Multi-AZ 自动 failover；数据全在一个 shard（容量受单节点内存限制）。

**2. Cluster Mode Enabled**（多 shard）：数据分片到多个 shard；每个 shard 有 1 个 master + replica；横向扩展容量；应用要支持 cluster client。

### 缓存模式（Caching Patterns）⭐⭐

**模式 1：Lazy Loading（Cache-Aside）— 最常用**

```
应用读数据:
  1. 查 Cache
  2. 命中? → 返回
  3. 未命中? → 查 DB → 写入 Cache → 返回
```

```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if user is None:
        user = db.query("SELECT * FROM users WHERE id = ?", user_id)
        cache.set(f"user:{user_id}", user)
    return user
```

优点：✅ 只缓存被请求的数据（节省内存）、✅ Cache 挂了应用还能用（降级到 DB）。缺点：❌ Cache miss 时延迟高（三次往返）、❌ **数据可能 stale**（DB 更新了，Cache 还是旧的）。

**模式 2：Write-Through**

```
应用写数据:
  1. 写 DB
  2. 同步写 Cache
```

优点：✅ Cache 永远和 DB 一致、✅ Cache 命中率高。缺点：❌ 写延迟高（两次往返）、❌ 大部分缓存的数据可能从未被读（浪费内存）、❌ Cache 挂了，写也失败（除非异步）。

**模式 3：TTL（过期时间）**

**= 给缓存数据设过期时间，过期自动失效**。通常和 Lazy Loading 配合：Lazy Loading 解决"缓存什么"，TTL 解决"缓存多久"。

**TTL 选择**：数据变化频繁 → 短 TTL（几秒到几分钟）；数据相对稳定 → 长 TTL（几小时到几天）；完全不变 → 无 TTL，主动失效（Write-Through 或显式 invalidate）。

### 缓存常见问题

**Cache Stampede（缓存击穿）**：热点 key 过期瞬间，大量请求同时穿透到 DB。解决：加锁（只一个请求重建缓存）、提前刷新。

**Cache Avalanche（缓存雪崩）**：大量 key 同时过期。解决：TTL 加随机抖动，分散过期时间。

**Cache Penetration（缓存穿透）**：查询不存在的 key，每次都打到 DB。解决：**缓存"不存在"的结果**（空值缓存，短 TTL）。

### ElastiCache 用途场景

| 场景 | 引擎选择 |
|---|---|
| Session Store（用户登录态） | Redis（持久化） |
| Leaderboard（游戏排行榜） | Redis（Sorted Set） |
| Real-time Analytics（实时统计） | Redis（HyperLogLog） |
| Geo / 附近的人 | Redis（Geo） |
| Pub/Sub 消息分发 | Redis |
| 简单 key-value 缓存，纯性能 | Memcached |
| 数据库查询缓存 | 任一（Redis 优先） |

### ElastiCache 安全

**VPC 内部署**，默认不暴露公网；**Security Group** 控制访问。Redis 支持：**In-transit encryption**（TLS）、**At-rest encryption**（KMS）、**Redis AUTH**（密码）、**IAM 认证**（Redis 7 起）。Memcached：**只支持 SASL 认证**，不支持加密。

### 考试速记

"Session 存储 + 跨多个 web server" → ElastiCache Redis；"排行榜" → Redis Sorted Set；"需要简单纯缓存 + 多线程" → Memcached；"数据库读慢，加缓存怎么实现" → Lazy Loading + TTL；"缓存和 DB 一致性优先" → Write-Through；"缓存挂了应用不能挂" → Lazy Loading；"需要 Multi-AZ HA 的缓存" → Redis（Memcached 不支持）。

---

## 三、DynamoDB Part 1 — Tables, Partition/Sort Key, Capacity Modes

### DynamoDB 是什么

**= AWS 全托管的 NoSQL 数据库**（键值 + 文档）。核心特性：✅ **完全托管**（AWS 管硬件、复制、scaling、patching）；✅ **Serverless**（没有服务器要管，按用量付费）；✅ **毫秒级延迟**（可选 DAX 做微秒级）；✅ **跨多个 AZ 自动复制**（高可用，持久性 11 个 9 默认）；✅ **几乎无限扩展**（单表 PB 级别都没问题）；✅ **支持 ACID 事务**（2018 年起）；✅ **集成 IAM、CloudWatch、CloudTrail**。

**对比 RDS**：RDS = SQL 关系型，适合复杂查询、JOIN、事务；DynamoDB = NoSQL key-value，适合海量、高并发、简单查询、固定查询模式。

### DynamoDB 数据模型

```
Table(表)
  └── Item(项,类似一行)
        └── Attributes(属性,类似列)
```

| RDB | DynamoDB |
|---|---|
| Table | Table |
| Row | Item |
| Column | Attribute |
| Primary Key | Primary Key（必须有） |
| Schema 严格 | **Schemaless**（每个 item 的属性可以不一样） |

**关键限制** ⭐⭐⭐：**单个 Item 最大 400 KB**（整个 item 的所有属性总和）；表名 3-255 字符；属性名 1-255 字节。

### Primary Key ⭐⭐⭐

**两种 Primary Key 类型**：

**1. Simple Primary Key（只有 Partition Key）**——也叫 **Hash Key**，每个 item 的 partition key 必须**唯一**。

```
Users 表
┌─────────────┬──────────────┐
│ user_id (PK)│ name         │
├─────────────┼──────────────┤
│ "u1"        │ Alice        │
│ "u2"        │ Bob          │
│ "u3"        │ Carol        │
└─────────────┴──────────────┘
```

**2. Composite Primary Key（Partition Key + Sort Key）**——也叫 **Hash and Range Key**；(partition key, sort key) 的组合必须唯一；**同一个 partition key 下可以有多个 item**（用 sort key 区分）。

```
Orders 表(Partition Key = user_id, Sort Key = order_date)
┌─────────────┬─────────────────┬──────────┐
│ user_id     │ order_date (SK) │ amount   │
├─────────────┼─────────────────┼──────────┤
│ "u1"        │ "2026-01-01"    │ 50.00    │
│ "u1"        │ "2026-01-15"    │ 80.00    │
│ "u1"        │ "2026-02-03"    │ 120.00   │ ← 同一个 user 多个订单
│ "u2"        │ "2026-01-10"    │ 30.00    │
└─────────────┴─────────────────┴──────────┘
```

**Sort Key 的好处**：同一 partition 内 item 按 sort key 排序；支持范围查询（`>=`、`BETWEEN`、`begins_with`）；适合"一对多"的关系（一个用户多个订单）。

### Partition Key 工作机制 ⭐⭐

```
Item: { user_id: "u1", name: "Alice" }
       │
       ▼
DynamoDB 对 partition key 做 hash
       │
       ▼
Hash 决定 item 存到哪个物理 partition
       │
       ▼
┌────────────┬────────────┬────────────┐
│ Partition  │ Partition  │ Partition  │
│     A      │     B      │     C      │
└────────────┴────────────┴────────────┘
```

**关键洞察**：DynamoDB 内部把数据**自动分到很多 partition**（每个 partition 约 10 GB）；partition key 决定 item 落到哪个 partition；**同一个 partition key 的所有 item 在同一个 partition**——这就是为什么 partition key 设计很重要。

### Hot Partition ⭐⭐⭐

**问题**：某个 partition key 的访问量远高于其他 → 该 partition 撞 RCU/WCU 上限，而其他 partition 闲置 → throttle。

**例子**：❌ 用 `country` 做 partition key，90% 用户在 USA → "USA" 这个 partition 极热；❌ 用 `today_date` 做 partition key → 所有今天的写都到一个 partition。

**好的 partition key 特性**：✅ **高基数**（High Cardinality）——不同值很多；✅ **均匀分布**（Uniform Distribution）——访问量均匀；✅ **请求量分散**。

**最佳实践**：✅ `user_id`（用户多 + 访问均匀）、✅ `device_id`、`session_id`、`order_id`（UUID）；❌ `country`、`status`、`category`（基数低 + 分布不均）；❌ `created_date`（所有今天的写撞一起）。

**Adaptive Capacity**（AWS 自动救援）：AWS 自动给"热 partition"分配更多 capacity，某种程度缓解 hot partition 问题，但**不是万能**，partition key 设计仍然关键。

### Capacity Modes ⭐⭐⭐

|  | Provisioned | On-Demand |
|---|---|---|
| 计费方式 | 预分配 RCU/WCU，按容量收费 | 按实际请求收费（per-request） |
| 适合场景 | 流量可预测、稳定 | 流量不可预测、突发、新应用 |
| 成本（满载） | 便宜 | 贵不少（一般在几倍量级） |
| 成本（闲置） | 还在付 | 几乎不付 |
| Throttle 风险 | 超过容量会 throttle | 几乎不会（AWS 自动 scale） |
| 切换 | 可以切换（每 24 小时一次） | 同左 |

**选择决策**：✅ **Provisioned**——流量稳定（报表、订单系统）、已知 baseline 可预测、成本敏感、配 Auto Scaling 弹性；✅ **On-Demand**——新应用（不知道流量）、流量极不稳定（突发性）、不想容量规划、开发/测试环境。

### RCU 和 WCU 计算 ⭐⭐⭐

**RCU = Read Capacity Unit**。1 RCU 能做什么：**1 个 strongly consistent read**，**4 KB**，每秒；或 **2 个 eventually consistent reads**，**4 KB**，每秒；或 **0.5 个 transactional read**（transactional 是 2 倍 RCU）。

计算公式：

```
读 N KB 的 item:
  - Strongly consistent: ceil(N/4) RCU
  - Eventually consistent: ceil(N/4) / 2 RCU
  - Transactional: ceil(N/4) × 2 RCU
```

例子：读 1 KB item（eventually consistent）→ 0.5 RCU（向上取整 1 RCU）；读 6 KB item（strongly consistent）→ 2 RCU（ceil(6/4) = 2）；读 8 KB item（eventually consistent）→ 1 RCU（2/2 = 1）；读 10 KB item（transactional）→ 6 RCU（ceil(10/4) × 2 = 3 × 2 = 6）。

**WCU = Write Capacity Unit**。1 WCU 能做什么：**1 个 standard write**，**1 KB**，每秒；或 **0.5 个 transactional write**（transactional 是 2 倍 WCU）。

计算公式：

```
写 N KB 的 item:
  - Standard: ceil(N) WCU
  - Transactional: ceil(N) × 2 WCU
```

例子：写 0.5 KB item → 1 WCU（向上取整）；写 3 KB item → 3 WCU；写 5 KB item（transactional）→ 10 WCU。

**口诀**：WCU = 1 KB / sec；RCU（strong）= 4 KB / sec；RCU（eventual）= 8 KB / sec（2 × 4 KB）；Transactional = 2 倍。

### Consistency Models ⭐⭐

**1. Eventually Consistent Read**（默认，最终一致）：✅ 性能好（便宜一半）；❌ 可能读到 stale 数据（最近写入可能没反映），通常 1 秒内一致。

**2. Strongly Consistent Read**（强一致）：✅ 一定读到最新写入；❌ 性能差（2 倍 RCU）；❌ 不能跨 region（只本 region）；❌ 在网络分区时可能失败。

**3. Transactional Read/Write**（ACID 事务）：✅ 多个操作原子性（全成功或全失败）；❌ 2 倍 RCU/WCU 成本。

**选择**：默认用 Eventually（便宜 + 快）；关键查询（账户余额、库存）用 Strong；涉及多个 item 原子操作 → Transactional。

### Throttling ⭐⭐

**Throttle 错误**：`ProvisionedThroughputExceededException`——超过 RCU/WCU 容量，HTTP 400，要客户端重试。

**原因**：① 超过表的 provisioned capacity（总量不够）；② **Hot partition**（单个 partition 撞 partition 上限，即使总量没满）；③ Item size 太大（读一个 100 KB item 要 25 RCU (strong)，很容易撞）。

**解决**：临时——**SDK 自动 retry with exponential backoff**（默认行为）；长期——增加 capacity（provisioned）、切到 On-Demand、改善 partition key 设计（避免 hot partition）、用 DAX 缓存（读密集场景）、拆大 item。

### Burst Capacity

**= 5 分钟未使用的 capacity 累积起来，可以应对短期突发**。类似 EC2 T 系列的 CPU credit；AWS 不保证一定可用（尽力而为）；让你能"超用一会儿"不被立刻 throttle。

### Auto Scaling（Provisioned 模式下）

**= 根据使用率自动调整 RCU/WCU**：设目标使用率（如 70%），AWS 监控自动上下调整；设最小/最大值（防止极端情况）。

**注意**：Auto Scaling **不是瞬间反应**，突发流量仍可能被 throttle。极端突发用 On-Demand。

### Reserved Capacity

**类似 EC2 RI**：承诺 1 年或 3 年的 RCU/WCU，拿折扣。适合：provisioned 模式 + 流量非常稳定。

### DynamoDB API 操作（基础）

| 操作 | 用途 | 消耗 |
|---|---|---|
| `PutItem` | 写入/覆盖 item | WCU |
| `GetItem` | 按 primary key 读 1 个 item | RCU |
| `UpdateItem` | 修改 item 部分属性 | WCU |
| `DeleteItem` | 删除 1 个 item | WCU |
| `Query` | 按 partition key 查多个 item（可选 sort key 范围） | RCU（扫描的所有 item） |
| `Scan` | **扫描整个表**（慢 + 贵！） | RCU（整个表） |
| `BatchGetItem` | 一次读多个 item（最多 100） | RCU |
| `BatchWriteItem` | 一次写多个 item（最多 25） | WCU |
| `TransactGetItems` / `TransactWriteItems` | 事务 | 2 倍 RCU/WCU |

**Query vs Scan**：**Query**——精确 + 高效（用 partition key）；**Scan**——扫全表（避免使用！成本高、慢）。

查询时可以添加 expression：**condition expression**（决定哪些 item 该被修改，用于数据操作）；**projection expression**（指定返回哪些属性）；**filter expression**（对 Query 结果做进一步过滤，在 partition key 和 sort key 之后生效，**不影响 Query 消耗的 RCU**）。

### DynamoDB 连接关系

**IAM**：用 IAM policy 控制表/item 级访问。**VPC Endpoint**：DynamoDB 用 **Gateway Endpoint**（免费，和 S3 一样）。**加密**：KMS（默认 AWS owned，可改 customer managed）。**跨账户访问**：Resource policy（较新支持）。

### 考试速记

"选 Partition Key" → 高基数、均匀分布、避免 hot partition；"RCU 计算" → 4 KB strong / 8 KB eventual；"WCU 计算" → 1 KB；"Transactional 成本" → 2 倍 RCU/WCU；"流量不可预测" → On-Demand；"流量稳定 + 省钱" → Provisioned + Auto Scaling；"突然 throttle 但容量没满" → Hot partition；"事务原子" → Transactional read/write（2 倍成本）；"强一致读" → Strongly consistent（2 倍 RCU）；"默认读一致性" → Eventually consistent。

---

## 四、DynamoDB 深入

### Part 1：Secondary Indexes ⭐⭐⭐

**为什么需要**：DynamoDB 的 Query 必须基于 Primary Key（partition key + 可选 sort key）。如果想按其他属性查询怎么办？

```
Orders 表:
- Partition Key: user_id
- Sort Key: order_date

可以查:"user u1 在 2026-01 的订单" ✅
不能查:"所有 status='pending' 的订单" ❌(只能 Scan,慢且贵)
```

**解决**：Secondary Index——给表"额外的索引"，让你能按其他属性查询。

### 两种 Secondary Index

|  | LSI (Local Secondary Index) | GSI (Global Secondary Index) |
|---|---|---|
| Partition Key | **必须和主表相同** | **可以不同** |
| Sort Key | 必须有，**不同于主表** | 可选 |
| 创建时机 | **只能在创建表时创建** | **任何时候**都能创建/删除 |
| 数量限制 | 每表最多 **5 个** | 每表最多 **20 个** |
| Capacity | **共享主表的 RCU/WCU** | **独立的 RCU/WCU** |
| 一致性 | 支持 strongly consistent read | **只支持 eventually consistent** |
| 大小限制 | 每个 partition key value 总数据 ≤ **10 GB** | 无限制 |

**LSI**：partition key 必须和主表一样，只换 sort key；"Local" = 只在同一个 partition 内提供"另一种排序方式"；共享主表 capacity。

```
主表 Orders:
- PK: user_id, SK: order_date
- 查询模式:"某个用户的订单按下单时间排序"

LSI:
- PK: user_id (相同), SK: order_amount
- 查询模式:"某个用户的订单按金额排序"
```

**GSI**：partition key 可以**完全不同**（因此叫 "Global"——跨整个表的全局索引）；"把表用另一个 key 重新组织一遍"；独立的 capacity（可以单独配置 RCU/WCU）；异步从主表复制（写入主表 → 后台同步到 GSI）→ **eventually consistent**。

```
主表 Orders:
- PK: user_id, SK: order_date

GSI(按 status 查):
- PK: status, SK: order_date
- 查询模式:"所有 status='pending' 的订单按时间排序"
```

### LSI vs GSI 选择决策 ⭐

| 场景 | 选 LSI | 选 GSI |
|---|---|---|
| Partition key 不变，只换排序 | ✅ | ❌（浪费） |
| Partition key 完全不同 | ❌（不行） | ✅ |
| 需要 strongly consistent read | ✅ | ❌（只支持 eventual） |
| 表已经创建好了 | ❌（不能加） | ✅ |
| 想独立控制 capacity | ❌ | ✅ |

**实战经验**：**多数场景用 GSI**（更灵活）。LSI 主要用在"创建表时就知道要按某个属性排序"的特殊场景。

### Projection（投影）⭐

| Projection | 包含 | 存储成本 | 查询能力 |
|---|---|---|---|
| **KEYS_ONLY** | 只有 partition key + sort key + 主表 PK | 最小 | 只能拿到 key，要原始数据要回主表查 |
| **INCLUDE** | KEYS_ONLY + 你指定的额外属性 | 中等 | 可以拿到指定属性 |
| **ALL** | 主表所有属性 | 最大（等于复制一份） | 完全自包含 |

**选择逻辑**：**KEYS_ONLY**——Index 只用来过滤，需要数据时再去主表（节省存储 + WCU）；**INCLUDE**——常用查询的少数几个属性；**ALL**——查询频繁、数据量小、不在乎成本。

### GSI 的写入放大效应 ⭐

**关键概念**：每次主表写入，都会触发**所有 GSI 的同步写入**。

**WCU 计算**：主表写 1 KB item → 1 WCU；表上有 3 个 GSI（都包含被改的属性）→ 每个 GSI 也消耗 WCU；**总消耗 = 主表 WCU + 所有 GSI 的 WCU**。

**陷阱**：如果 **GSI 的 WCU 配置不足**，会导致**主表的写入也被 throttle**（主表必须等 GSI 写成功）——这是 GSI 区别于 LSI 的重要点。

**最佳实践**：GSI 的 WCU 一般要 ≥ 主表的 WCU；减少 GSI 数量（每个 GSI 都是写入开销）；用 KEYS_ONLY 减少同步数据量。

### Part 2：DynamoDB Streams ⭐⭐⭐

**= DynamoDB 的"变更日志"，记录表里的所有 item 修改**，类似关系型数据库的 binlog/CDC（Change Data Capture）。

```
应用写 DynamoDB → DynamoDB Streams 自动记录变更 →
Lambda / Kinesis 消费 → 处理(发通知、同步到其他系统)
```

**关键特性**：✅ **24 小时保留期**（不可改）；✅ **每条变更记录按顺序**（同一个 partition 内）；✅ near real-time（变更后秒级出现在 stream）；✅ 可被 **Lambda Trigger** 自动消费；✅ 可被 **Kinesis Client Library (KCL)** 消费；❌ Streams **本身免费**，但 Lambda 调用、Kinesis 处理另收费。

**4 种 View Type**：

| View Type | Stream 中包含 |
|---|---|
| **KEYS_ONLY** | 只有改变 item 的 key |
| **NEW_IMAGE** | 改变后的 item 完整内容 |
| **OLD_IMAGE** | 改变前的 item 完整内容 |
| **NEW_AND_OLD_IMAGES** | 改变前后都有（最完整） |

**Streams 典型用途**：① 触发 Lambda 处理变更（最常见，如用户注册 → 发送欢迎邮件）；② 跨表数据同步/数据复制；③ CloudWatch 指标/实时分析；④ 跨 region 复制（自定义，Global Tables 是托管版）；⑤ Event Sourcing 架构。

**Streams vs Kinesis Data Streams for DynamoDB**：

|  | DynamoDB Streams | Kinesis Data Streams for DynamoDB |
|---|---|---|
| 保留期 | 24 小时 | **最长可达 365 天** |
| 消费者 | Lambda、KCL | Lambda、KCL、Firehose、自定义 |
| 顺序 | 按 partition | 按 partition |
| 集成 | DynamoDB 原生 | 通过 Kinesis 生态 |

**判断**：需要长保留期 / 接入 Kinesis 生态 → **Kinesis Data Streams for DynamoDB**。

### Part 3：DAX (DynamoDB Accelerator) ⭐⭐

**= DynamoDB 的内存缓存层（完全托管）**。DynamoDB 延迟：毫秒级；DAX 延迟：微秒级——数量级的性能提升。

```
应用 → DAX Cluster → DynamoDB
       ↑ 缓存命中直接返回
       ↑ 缓存未命中查 DDB,然后缓存
```

**关键特性**：✅ **完全兼容 DynamoDB API**（代码几乎不用改，只换 endpoint）；✅ **写穿透 (write-through)**：写入时先写 DDB，再更新 DAX；✅ 在 VPC 内（不暴露公网）；✅ 多节点 cluster，支持 replica。

**DAX 两种缓存**：**Item Cache**（缓存 GetItem / BatchGetItem 的结果，默认 TTL 5 分钟）；**Query Cache**（缓存 Query / Scan 的结果，默认 TTL 5 分钟）。

### DAX vs ElastiCache ⭐

|  | DAX | ElastiCache |
|---|---|---|
| 专门为 | **DynamoDB** | 任何应用（通用缓存） |
| 接入 | 改 endpoint 即可 | 应用代码要改（Lazy Loading 等模式） |
| 缓存策略 | 自动管理 | 应用自己写 |
| 兼容性 | DynamoDB API | 自己的 API（Redis/Memcached） |
| 用途 | DynamoDB 加速 | 通用 / Session / Pub-Sub 等 |

**选择**：只是想给 DynamoDB 加缓存 → **DAX**（零代码改动）；通用缓存 / 多种数据源 / Session / 排行榜 → **ElastiCache**。

**DAX 不适合的场景**：❌ 写入密集型应用（DAX 是 read cache，写入还是要打 DDB）；❌ 强一致性读（DAX 缓存可能 stale，strongly consistent read 会跳过 DAX 直查 DDB）；❌ 数据频繁变化（缓存命中率低，无收益）；❌ 不同应用查相同数据（每个应用建自己的缓存浪费）。

**判断**："DDB 读慢，想毫秒级 → 微秒级" → DAX；"DDB 读 + 复杂数据结构（排行榜）" → ElastiCache Redis（不是 DAX）；"写入热点 throttle" → DAX **不能解决**（改 partition key / On-Demand）。

### Part 4：TTL (Time To Live) ⭐

**= DynamoDB 自动删除过期 item 的机制**

**工作流程**：① 给 item 加一个属性（数字类型），值为 Unix 时间戳（过期时间）；② 启用表的 TTL，指定哪个属性是 TTL 属性；③ DynamoDB 后台扫描，当前时间 > TTL 值的 item 自动删除。

**关键特性**：✅ **完全免费**（删除不消耗 WCU）；✅ 后台异步删除，**通常在 48 小时内**（不保证精确时间）；✅ 删除事件会写到 Streams（可以用 Lambda 后续处理）；✅ 已过期但还没删的 item **不会出现在 Query/Scan 结果**（对应用透明）。

**精度**：TTL 是 best-effort，不保证准时；不要依赖 TTL 做精确时间控制；想精确用 Lambda + EventBridge schedule。

**典型用途**：① Session 数据（登录 token 到期失效）；② 临时数据（验证码、一次性密码）；③ IoT 时序数据（只保留 N 天）；④ 缓存数据；⑤ 合规要求（用户数据保留 N 天后必须删除，GDPR）。

**优势 vs 手动清理**：手动——写 Lambda + EventBridge 定时扫描删除，消耗 WCU，贵；TTL——免费 + 自动。

### Part 5：Transactions ⭐⭐

**= DynamoDB 支持 ACID 事务**（2018 年起）

| API | 用途 |
|---|---|
| `TransactWriteItems` | **多个写操作原子性**（全成功或全失败） |
| `TransactGetItems` | **多个读操作一致性快照** |

**Transactions 限制**：✅ 单次 transaction 支持较多 action（AWS 已从最初的 25 提升到更高上限，以官方文档为准）；✅ **最多 4 MB** 总数据；✅ 可以跨多个表，但**必须在同 region**；❌ 不能跨账户。

**TransactWriteItems 支持的操作**：`Put`（写入/覆盖）、`Update`（修改）、`Delete`（删除）、`ConditionCheck`（条件检查，不修改，只用作前提）。

例子（转账场景）：

```
Transaction:
  - Update(account_A, balance -= 100)
  - Update(account_B, balance += 100)
  - ConditionCheck(account_A, balance >= 100)
要么三个都成功,要么三个都不发生。
```

**事务的成本**：Transactional 操作消耗 2 倍 capacity——内部要做 2-phase commit（第 1 阶段 prepare 检查+锁，第 2 阶段 commit 执行）。

**事务 vs Conditional Writes**：很多场景不需要完整事务，用 Conditional Writes 就够：

```
PutItem({
  TableName: "Orders",
  Item: {...},
  ConditionExpression: "attribute_not_exists(order_id)"
})
```

Conditional Writes：只针对单个 item；"如果条件成立才写"；成本和普通 write 一样（1 WCU）；适合"乐观锁"场景（防止覆盖、防止重复创建）。

**选择**：单 item 条件性更新 → Conditional Writes（便宜）；多 item 原子操作 → Transactions（贵但安全）。

### Part 6：DynamoDB 其他重要特性（速览）

**Global Tables** ⭐：跨 region 的多主复制。多个 region 都能写入（multi-master）；异步复制（通常秒级延迟）；**必须启用 Streams**（底层用 Streams 复制）；冲突解决：**Last Writer Wins**（最后写入获胜）。用途：全球应用（各 region 用户访问最近的副本）、DR、跨 region 数据冗余。

**Backup & Restore**：

|  | On-Demand Backup | Continuous Backups (PITR) |
|---|---|---|
| 触发 | 手动 | 自动持续 |
| 保留 | 永久（直到删除） | 35 天 |
| 恢复粒度 | 备份时刻 | **任意秒**（point-in-time recovery） |
| 用途 | 长期归档、合规 | 误删恢复、PITR |

**Capacity Reservations**：类似 EC2 RI，DynamoDB 支持 Reserved Capacity——承诺 1 或 3 年的 RCU/WCU，享有折扣；仅 Provisioned 模式。

**IAM 控制粒度**：DynamoDB IAM 可以做到非常细的粒度控制：

```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem", "dynamodb:Query"],
  "Resource": "arn:aws:dynamodb:us-east-1:111:table/Users",
  "Condition": {
    "ForAllValues:StringEquals": {
      "dynamodb:LeadingKeys": ["${aws:userid}"]
    }
  }
}
```

这让用户**只能访问自己的 partition**（常用于多租户应用，每个用户只看自己的数据）。

关键 Condition Keys：`dynamodb:LeadingKeys`（限制 partition key）、`dynamodb:Attributes`（限制可访问的属性）、`dynamodb:Select`（限制能不能 SELECT *）。

### Part 7：DynamoDB 完整考点速记

| 场景 | 答案 |
|---|---|
| 想按非主键属性查询 | GSI（灵活）或 LSI（只换 sort key） |
| 表已创建好，想加索引 | **GSI**（LSI 只能创建表时加） |
| 需要 strongly consistent 二级索引 | **LSI**（GSI 不支持） |
| 监听数据变更触发 Lambda | DynamoDB Streams |
| 跨 region 多主复制 | Global Tables |
| 给 DDB 加缓存，毫秒 → 微秒 | **DAX**（零代码改动） |
| 给 DDB 加缓存 + 复杂数据结构 | ElastiCache Redis |
| 写入密集 + throttle | 改 partition key / On-Demand（DAX 无效） |
| 自动删除过期数据 | TTL |
| 多 item 原子操作 | Transactions（2 倍成本） |
| 单 item 条件更新 | Conditional Writes |
| 误删恢复 | Point-in-Time Recovery |
| 多租户每个用户只看自己数据 | IAM `dynamodb:LeadingKeys` |
