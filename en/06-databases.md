# Module 6: Databases (RDS / Aurora / ElastiCache / DynamoDB)

> **Covered**: Apr 30, May 1, 2, 3
>
> 中文版本：[`zh/06-databases.md`](../zh/06-databases.md)

---

## 1. RDS + Aurora + Read Replicas vs Multi-AZ

### What RDS Is

**RDS = Relational Database Service.** AWS's managed relational database service — no need to manage the OS, patches, backups, or failover yourself.

**Supported engines**: MySQL, PostgreSQL, MariaDB, Oracle, Microsoft SQL Server, Amazon Aurora (MySQL/PostgreSQL-compatible, built by AWS).

**RDS does not allow SSH into the OS** (it's fully managed) — for OS-level access, use **RDS Custom** (supports Oracle and SQL Server only).

**Core components**: a DB Instance (the database server — an EC2 under the hood, but hidden and unmodifiable); a DB Subnet Group (the set of subnets RDS needs to deploy across multiple AZs); a Parameter Group (engine parameters, similar to my.cnf); an Option Group (engine-specific features, like Oracle's TDE); an Endpoint (the DNS name clients connect to).

**RDS storage**: defaults to gp2 / gp3 / io1 (SSD); Storage Auto Scaling can be enabled (auto-expands as disk fills, with a configurable cap); cannot be detached directly (unlike EBS).

### Multi-AZ (High Availability) ⭐⭐⭐

**= synchronous primary-standby replication across AZs.**

```
AZ-1                       AZ-2
┌──────────────┐          ┌──────────────┐
│  Primary DB  │ ─sync──→  │  Standby DB  │
│  (read+write)│          │  (no traffic)│
└──────────────┘          └──────────────┘
       ↑
   Endpoint (DNS)
       ↑
    Application
```

**Key traits**: ✅ **synchronous replication**; ✅ the standby **accepts no traffic at all** (cannot be used for reads or writes) — it's purely a hot standby; ✅ if the primary fails, AWS **fails over automatically** (DNS switches to the standby), typically in 60-120 seconds; ✅ spans AZs (2 AZs by default; some engines support 3-AZ Multi-AZ Clusters); ✅ failover triggers include AZ failure, primary failure, maintenance, and a forced "reboot with failover."

**Purpose**: **HA (high availability)** — **not** a performance optimization.

**Rule of thumb**: Multi-AZ = disaster recovery; the standby never absorbs read traffic.

### Read Replicas (Read Scaling) ⭐⭐⭐

**= asynchronous replicas of the primary that can serve reads.**

```
              Primary DB (read+write)
                   │
        ┌──────────┼──────────┐
        │ async    │          │
        ▼          ▼          ▼
    Replica 1  Replica 2  Replica 5
    (read-only)(read-only)(read-only)
```

**Key traits**: ✅ **asynchronous replication** — replicas can lag slightly behind; ✅ replicas are **read-only** (the application must route read traffic to the replica endpoint); ✅ **up to 15 read replicas** (MySQL/PostgreSQL/MariaDB/SQL Server/Db2), **5 for Oracle** (Aurora also allows 15); ✅ can be in the same AZ, a different AZ, or **a different region**; ✅ a replica can be "promoted" into a standalone database; ✅ cross-region replicas can serve as a DR target.

**Purpose**: **read-performance scaling**, **reporting queries** (without touching the primary), **cross-region DR**.

**Note**: a cross-region read replica travels over the public internet (unless VPC peering is configured), incurring cross-region data transfer charges.

### Multi-AZ vs Read Replica ⭐⭐⭐

|  | Multi-AZ | Read Replica |
|---|---|---|
| Replication | **Synchronous** | **Asynchronous** |
| Replica readable | ❌ No (pure hot standby) | ✅ Read-only |
| Primary purpose | **High availability / DR** | **Read-performance scaling** |
| Automatic failover | ✅ Yes | ❌ No (manual promote) |
| Cross-region | ❌ No (same-region, cross-AZ only) | ✅ Yes |
| Count | 1 standby | **Up to 15** (5 for Oracle, 15 for Aurora) |
| Data consistency | Strongly consistent | Eventually consistent (has lag) |

**Common judgment calls**: "the database needs high availability" → **Multi-AZ**; "reads are slow, primary's CPU is high" → **Read Replica** (offload reads); "need to run large queries/reports without impacting production" → **Read Replica**; "DR into another region" → **Cross-region Read Replica** (or Aurora Global Database).

**A common trap**: a Read Replica cannot serve as HA (it won't auto-failover if the primary dies), and Multi-AZ cannot serve as read scaling.

### RDS Encryption ⭐

**Key rule**: ✅ enable KMS encryption at creation time (select a KMS key); ❌ **an existing unencrypted RDS instance cannot have encryption turned on directly!** To encrypt an existing database: **snapshot it → copy the snapshot with encryption enabled → restore from the encrypted snapshot** (the same pattern as EBS).

**Other encryption details**: encryption covers data, backups, snapshots, read replicas, and logs; a read replica must use the same encryption settings as the primary; **Transparent Data Encryption (TDE)** — engine-level encryption for Oracle and SQL Server, configured through an Option Group.

### RDS Backups

|  | Automated Backup | Manual Snapshot |
|---|---|---|
| Trigger | Automatic (daily) | Manual |
| Retention | 0-35 days (0 = off) | Forever (until you delete it) |
| After deleting the RDS instance | Deleted along with it (retention optional) | Kept forever |
| Use case | Short-term / point-in-time recovery | Long-term archiving, cross-account sharing |

**Point-in-Time Recovery (PITR)**: restore to any point in time (down to the second) by replaying the transaction log — only possible within the backup retention window.

**RDS has no "in-place restore"**: any restore creates a brand-new RDS instance with a new endpoint.

### RDS Proxy ⭐

**The problem**: every Lambda invocation calling RDS opens a new connection — under high concurrency, this exhausts RDS's connection limit.

**The fix**: RDS Proxy = connection pooling + faster failover.

**Traits**: ✅ **connection pooling** (the application sees many connections, but they share a small pool of actual RDS connections); ✅ **faster failover** — during a Multi-AZ failover, RDS Proxy keeps connections alive, **cutting failover time from ~60s to under 30s**; ✅ **IAM authentication** (no password needed to connect); ✅ integrates with **Secrets Manager**; ✅ stays inside the VPC, never exposed publicly; ✅ good fit for **high-concurrency Lambda** and Auto Scaling applications.

**Judgment call**: "Lambda hammers RDS and connection counts blow up" → **RDS Proxy**; "want IAM authentication for the database and to hide the password" → RDS Proxy + Secrets Manager.

### Aurora ⭐⭐⭐

**= AWS's purpose-built, cloud-native relational database** (MySQL/PostgreSQL-compatible). Why it's different: its storage layer is entirely rewritten by AWS, not the traditional MySQL/PostgreSQL InnoDB/heap engine.

**Aurora storage architecture**:

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

**Key traits** (all frequently tested ⭐): ✅ **6 data copies spread across 3 AZs** (2 per AZ); ✅ **writes require 4-of-6 copies to acknowledge** (write quorum); ✅ **reads require 3-of-6 copies to respond** (read quorum); ✅ losing any single AZ (2 copies) does not affect writes; ✅ losing any 2 copies does not affect reads; ✅ **self-healing** — corrupted blocks are automatically repaired from other copies; ✅ storage auto-scales up to **128 TB**.

> Aurora is marketed as significantly faster than RDS MySQL/PostgreSQL — check AWS's current published figures for the exact multiplier.

### Aurora Replicas

Up to **15 read replicas** (5 for standard RDS); **replication lag is typically under 100ms** (they share the underlying storage — it isn't true replication, they read the same storage directly); **automatic failover** — if the primary fails, a replica is promoted within roughly 30 seconds.

### Aurora Cluster Endpoints ⭐

| Endpoint | Purpose |
|---|---|
| **Writer Endpoint** | Writes (always points to the current primary) |
| **Reader Endpoint** | Reads (automatically load-balanced across all read replicas) |
| **Custom Endpoint** | A defined subset of instances (e.g. routing analytics queries to specific replicas) |
| **Instance Endpoint** | Connects to one specific instance (rarely used, since which instance is primary can change) |

**Best practice**: writes → Writer Endpoint; reads → Reader Endpoint (auto load-balanced); don't hardcode an instance endpoint, since it may no longer be the primary after a failover.

### Aurora Serverless ⭐

**= Aurora that starts, stops, and scales automatically.**

**v1 vs v2**: **Aurora Serverless v1** — capacity measured in ACUs (Aurora Capacity Units), scales down to 0, has a cold start; **Aurora Serverless v2** — **scales much faster**, in fine-grained 0.5 ACU increments, and is the recommended choice for production.

**Good fit**: ✅ intermittent/unpredictable workloads, ✅ dev/test environments, ✅ new applications where traffic is unknown. **Poor fit**: ❌ sustained high load (provisioned is cheaper there).

### Aurora Global Database ⭐

**= cross-region Aurora.**

**Traits**: 1 primary region + up to 5 secondary regions; **cross-region replication lag is typically under 1 second**; secondary regions are read-only; **DR**: promoting a secondary to primary has a short RTO/RPO (check current AWS docs for exact figures); traffic travels over AWS's internal backbone, not the public internet.

**vs. cross-region read replica**: Global Database is faster and more purpose-built, but only available for Aurora.

### Aurora Backtrack (Aurora MySQL only)

**= "rewinding" the database to a past point in time** without a restore. The current data isn't deleted — it's a time-machine style rewind; you can backtrack repeatedly (forward/backward); there's a maximum time window (check the docs for the current figure); **much faster than PITR** (seconds, not minutes). Use case: recovering from accidental deletes, mistakes, or an application bug that wrote bad data.

### Other Aurora Features (Quick Overview)

**Aurora Multi-Master**: multiple writers (rare, adds complexity). **Aurora Clone**: creates a new cluster from an existing one using **copy-on-write** (nearly instantaneous, no data actually copied) — good for giving a dev environment a copy of production data. **Database Activity Streams**: audits internal database operations that CloudTrail doesn't capture.

---

## 2. ElastiCache (Redis vs Memcached) + Caching Patterns

### What ElastiCache Is

**= AWS's managed in-memory caching service.** Two engines: **Redis** and **Memcached**. Core value: keep hot data in memory to reduce database load and lower latency.

### Redis vs Memcached ⭐⭐⭐

|  | Redis | Memcached |
|---|---|---|
| Data structures | Rich (String, List, Set, Sorted Set, Hash, Stream, Bitmap, HyperLogLog, Geo) | Key-value only (String) |
| Persistence | ✅ Supported (AOF + RDB snapshots) | ❌ Pure in-memory, data lost on restart |
| Replication | ✅ Master + Read Replicas | ❌ Not supported |
| Multi-AZ | ✅ Supported (automatic failover) | ❌ Not supported |
| Backups | ✅ Snapshots supported | ❌ Not supported |
| Multi-threaded | ❌ Single-threaded (per shard) | ✅ Multi-threaded |
| Sharding | ✅ Cluster Mode | ✅ Automatic sharding |
| Pub/Sub | ✅ | ❌ |
| Transactions | ✅ MULTI/EXEC | ❌ |
| Lua scripting | ✅ | ❌ |
| Encryption (at rest / in transit) | ✅ | ❌ |
| HIPAA / compliance | ✅ | ❌ |

### Choosing Between Them

**Use Redis when**: ✅ you need persistence (data survives a restart), ✅ you need HA/Multi-AZ, ✅ you need complex data structures (leaderboards, message queues, geolocation), ✅ you need pub/sub, ✅ compliance requires it (encryption, HIPAA), ✅ advanced use cases like a cache + session store + leaderboard combined.

**Use Memcached when**: ✅ you need pure, simple key-value caching, ✅ you need multi-threaded throughput, ✅ losing data is fine (it just gets re-cached), ✅ horizontal scaling via automatic sharding.

**Default to Redis** — it covers most scenarios. Reach for Memcached only when you specifically want pure multi-threaded caching and data loss is acceptable.

### Redis Deployment Modes

**1. Cluster Mode Disabled** (single shard): 1 primary + up to 5 read replicas; automatic Multi-AZ failover; all data lives in one shard (capacity limited by a single node's memory).

**2. Cluster Mode Enabled** (multiple shards): data is sharded across multiple shards; each shard has 1 primary + replicas; capacity scales horizontally; the application must use a cluster-aware client.

### Caching Patterns ⭐⭐

**Pattern 1: Lazy Loading (Cache-Aside) — the most common**

```
Application reads data:
  1. Check the cache
  2. Hit? → return it
  3. Miss? → query the DB → write to cache → return it
```

```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if user is None:
        user = db.query("SELECT * FROM users WHERE id = ?", user_id)
        cache.set(f"user:{user_id}", user)
    return user
```

Pros: ✅ only requested data gets cached (saves memory), ✅ the app still works if the cache goes down (falls back to the DB). Cons: ❌ higher latency on a cache miss (three round trips), ❌ **data can go stale** (the DB updates but the cache still holds the old value).

**Pattern 2: Write-Through**

```
Application writes data:
  1. Write to the DB
  2. Synchronously write to the cache
```

Pros: ✅ the cache always matches the DB, ✅ high cache hit rate. Cons: ❌ higher write latency (two round trips), ❌ much of the cached data may never actually get read (wasted memory), ❌ if the cache is down, writes fail too (unless done asynchronously).

**Pattern 3: TTL (Time-to-Live)**

**= giving cached data an expiration, after which it's automatically invalidated.** Usually paired with Lazy Loading: Lazy Loading answers "what to cache," TTL answers "for how long."

**Choosing a TTL**: fast-changing data → short TTL (seconds to minutes); relatively stable data → long TTL (hours to days); truly static data → no TTL, invalidate explicitly (Write-Through or manual invalidation).

### Common Caching Problems

**Cache Stampede**: a hot key expires and a flood of requests all hit the DB simultaneously. Fix: locking (only one request rebuilds the cache) or refreshing early.

**Cache Avalanche**: a large batch of keys expire at once. Fix: add random jitter to TTLs so expirations spread out.

**Cache Penetration**: repeated queries for a key that doesn't exist keep hitting the DB every time. Fix: **cache the "not found" result too** (a null-value cache with a short TTL).

### ElastiCache Use Cases

| Scenario | Engine choice |
|---|---|
| Session store (login state) | Redis (persistent) |
| Leaderboard (games) | Redis (Sorted Set) |
| Real-time analytics | Redis (HyperLogLog) |
| Geo / "nearby" features | Redis (Geo) |
| Pub/Sub message distribution | Redis |
| Simple key-value caching, pure performance | Memcached |
| Database query caching | Either (Redis preferred) |

### ElastiCache Security

**Deployed inside a VPC**, not exposed publicly by default; **Security Groups** control access. Redis supports: **in-transit encryption** (TLS), **at-rest encryption** (KMS), **Redis AUTH** (password), **IAM authentication** (as of Redis 7). Memcached: **SASL authentication only**, no encryption support.

### Exam Quick Reference

"Session storage shared across multiple web servers" → ElastiCache Redis; "leaderboard" → Redis Sorted Set; "need simple pure caching + multi-threading" → Memcached; "database reads are slow, how to add a cache" → Lazy Loading + TTL; "cache/DB consistency is the priority" → Write-Through; "the app must stay up if the cache fails" → Lazy Loading; "need Multi-AZ HA for the cache" → Redis (Memcached doesn't support it).

---

## 3. DynamoDB Part 1 — Tables, Partition/Sort Key, Capacity Modes

### What DynamoDB Is

**= AWS's fully managed NoSQL database** (key-value + document). Key traits: ✅ **fully managed** (AWS handles hardware, replication, scaling, patching); ✅ **serverless** (nothing to provision, pay by usage); ✅ **single-digit millisecond latency** (microsecond with DAX); ✅ **automatically replicated across multiple AZs** (high availability, 11 nines durability by default); ✅ **near-unlimited scale** (a single table can hold petabytes); ✅ **ACID transaction support** (since 2018); ✅ **integrates with IAM, CloudWatch, CloudTrail**.

**vs. RDS**: RDS is SQL/relational — good for complex queries, joins, transactions. DynamoDB is NoSQL key-value — good for massive scale, high concurrency, simple queries, fixed access patterns.

### DynamoDB Data Model

```
Table
  └── Item (like a row)
        └── Attributes (like columns)
```

| RDBMS | DynamoDB |
|---|---|
| Table | Table |
| Row | Item |
| Column | Attribute |
| Primary key | Primary key (required) |
| Rigid schema | **Schemaless** — each item's attributes can differ |

**Key limits** ⭐⭐⭐: **a single item maxes out at 400 KB** (the sum of all its attributes); table names are 3-255 characters; attribute names are 1-255 bytes.

### Primary Key ⭐⭐⭐

**Two types of primary key**:

**1. Simple Primary Key (partition key only)** — also called a **Hash Key**. Every item's partition key must be **unique**.

```
Users table
┌─────────────┬──────────────┐
│ user_id (PK)│ name         │
├─────────────┼──────────────┤
│ "u1"        │ Alice        │
│ "u2"        │ Bob          │
│ "u3"        │ Carol        │
└─────────────┴──────────────┘
```

**2. Composite Primary Key (partition key + sort key)** — also called a **Hash and Range Key**. The (partition key, sort key) combination must be unique; **multiple items can share the same partition key**, distinguished by sort key.

```
Orders table (Partition Key = user_id, Sort Key = order_date)
┌─────────────┬─────────────────┬──────────┐
│ user_id     │ order_date (SK) │ amount   │
├─────────────┼─────────────────┼──────────┤
│ "u1"        │ "2026-01-01"    │ 50.00    │
│ "u1"        │ "2026-01-15"    │ 80.00    │
│ "u1"        │ "2026-02-03"    │ 120.00   │ ← same user, multiple orders
│ "u2"        │ "2026-01-10"    │ 30.00    │
└─────────────┴─────────────────┴──────────┘
```

**Benefits of a sort key**: items within a partition are ordered by sort key; range queries are supported (`>=`, `BETWEEN`, `begins_with`); a natural fit for one-to-many relationships (one user, many orders).

### How Partition Keys Work ⭐⭐

```
Item: { user_id: "u1", name: "Alice" }
       │
       ▼
DynamoDB hashes the partition key
       │
       ▼
The hash determines which physical partition stores the item
       │
       ▼
┌────────────┬────────────┬────────────┐
│ Partition  │ Partition  │ Partition  │
│     A      │     B      │     C      │
└────────────┴────────────┴────────────┘
```

**Key insight**: DynamoDB automatically splits data across many partitions (roughly 10 GB each); the partition key determines which partition an item lands on; **all items with the same partition key live in the same partition** — which is exactly why partition key design matters.

### Hot Partitions ⭐⭐⭐

**The problem**: one partition key gets disproportionately more traffic than others → that partition hits its RCU/WCU ceiling while other partitions sit idle → throttling.

**Examples**: ❌ using `country` as the partition key when 90% of users are in the USA — the "USA" partition becomes a hotspot; ❌ using `today_date` as the partition key — every write today lands on one partition.

**What makes a good partition key**: ✅ **high cardinality** — many distinct values; ✅ **uniform distribution** — even access across values; ✅ **spread-out request volume**.

**Best practices**: ✅ `user_id` (many users, evenly accessed), ✅ `device_id`, `session_id`, `order_id` (UUIDs); ❌ `country`, `status`, `category` (low cardinality, uneven distribution); ❌ `created_date` (all of today's writes collide).

**Adaptive Capacity** (AWS's automatic mitigation): AWS automatically allocates extra capacity to a hot partition, which helps somewhat — but it's **not a substitute** for good partition key design.

### Capacity Modes ⭐⭐⭐

|  | Provisioned | On-Demand |
|---|---|---|
| Billing | Pre-allocated RCU/WCU, billed by capacity | Billed per actual request |
| Fits | Predictable, steady traffic | Unpredictable, bursty traffic, new applications |
| Cost (fully utilized) | Cheaper | Notably more expensive |
| Cost (idle) | Still billed | Nearly nothing |
| Throttling risk | Throttles above provisioned capacity | Rare (AWS auto-scales) |
| Switching | Can switch once every 24 hours | Same |

**Choosing**: ✅ **Provisioned** — steady traffic (reporting, order systems), a known predictable baseline, cost-sensitive, paired with Auto Scaling for elasticity; ✅ **On-Demand** — a new app with unknown traffic, highly unpredictable/bursty traffic, no appetite for capacity planning, dev/test environments.

### Calculating RCU and WCU ⭐⭐⭐

**RCU = Read Capacity Unit.** One RCU buys: **1 strongly consistent read** of **4 KB** per second; or **2 eventually consistent reads** of 4 KB per second; or **0.5 transactional reads** (transactional costs 2x RCU).

```
Reading an N-KB item:
  - Strongly consistent: ceil(N/4) RCU
  - Eventually consistent: ceil(N/4) / 2 RCU
  - Transactional: ceil(N/4) × 2 RCU
```

Examples: reading a 1 KB item (eventually consistent) → 0.5 RCU, rounded up to 1; reading a 6 KB item (strongly consistent) → 2 RCU (ceil(6/4) = 2); reading an 8 KB item (eventually consistent) → 1 RCU (2/2 = 1); reading a 10 KB item (transactional) → 6 RCU (ceil(10/4) × 2 = 3 × 2 = 6).

**WCU = Write Capacity Unit.** One WCU buys: **1 standard write** of **1 KB** per second; or **0.5 transactional writes** (transactional costs 2x WCU).

```
Writing an N-KB item:
  - Standard: ceil(N) WCU
  - Transactional: ceil(N) × 2 WCU
```

Examples: writing a 0.5 KB item → 1 WCU (rounded up); writing a 3 KB item → 3 WCU; writing a 5 KB item (transactional) → 10 WCU.

**Mnemonic**: WCU = 1 KB/sec; RCU (strong) = 4 KB/sec; RCU (eventual) = 8 KB/sec (2 × 4 KB); transactional = 2x either way.

### Consistency Models ⭐⭐

**1. Eventually Consistent Read** (the default): ✅ better performance (half the cost); ❌ may return stale data (a very recent write might not be reflected yet) — typically consistent within a second.

**2. Strongly Consistent Read**: ✅ guaranteed to reflect the latest write; ❌ worse performance (2x RCU); ❌ cannot cross regions (region-local only); ❌ can fail during a network partition.

**3. Transactional Read/Write** (ACID): ✅ multiple operations are atomic (all succeed or all fail); ❌ 2x RCU/WCU cost.

**Choosing**: default to Eventually Consistent (cheap and fast); use Strong for critical reads (account balances, inventory); use Transactional when multiple items must change atomically.

### Throttling ⭐⭐

**The throttle error**: `ProvisionedThroughputExceededException` — capacity exceeded, HTTP 400, the client must retry.

**Causes**: ① the table's total provisioned capacity is exceeded; ② **a hot partition** hits its own limit even though the table's total capacity isn't maxed out; ③ item size is too large (reading a 100 KB item costs 25 RCU strongly consistent — easy to exceed).

**Fixes**: short-term — **the SDK retries automatically with exponential backoff** (default behavior); long-term — increase provisioned capacity, switch to On-Demand, improve partition key design to avoid hot partitions, add a DAX cache for read-heavy workloads, split large items.

### Burst Capacity

**= unused capacity from the last 5 minutes accumulates and can absorb short bursts.** Similar to EC2 T-series CPU credits; AWS doesn't guarantee it's always available (best-effort); it lets you briefly exceed provisioned capacity without immediate throttling.

### Auto Scaling (in Provisioned Mode)

**= automatically adjusts RCU/WCU based on utilization.** You set a target utilization (e.g. 70%); AWS monitors and scales up/down; min/max bounds guard against extremes.

**Note**: Auto Scaling **doesn't react instantly** — sudden bursts can still be throttled. For extreme bursts, use On-Demand instead.

### Reserved Capacity

**Similar to EC2 RIs**: commit to 1 or 3 years of RCU/WCU for a discount. Fits: provisioned mode with very steady traffic.

### DynamoDB API Operations (Basics)

| Operation | Purpose | Consumes |
|---|---|---|
| `PutItem` | Write/overwrite an item | WCU |
| `GetItem` | Read a single item by primary key | RCU |
| `UpdateItem` | Modify part of an item | WCU |
| `DeleteItem` | Delete a single item | WCU |
| `Query` | Fetch multiple items by partition key (optional sort key range) | RCU (for all items scanned) |
| `Scan` | **Scan the entire table** (slow and expensive!) | RCU (the whole table) |
| `BatchGetItem` | Read multiple items at once (up to 100) | RCU |
| `BatchWriteItem` | Write multiple items at once (up to 25) | WCU |
| `TransactGetItems` / `TransactWriteItems` | Transactions | 2x RCU/WCU |

**Query vs Scan**: **Query** — precise and efficient (uses the partition key); **Scan** — reads the entire table (avoid it! expensive and slow).

Queries can carry expressions: a **condition expression** (determines which items are eligible for a data-modifying operation); a **projection expression** (specifies which attributes to return); a **filter expression** (further filters Query results after the partition/sort key match — it does **not** reduce the RCU the Query consumes).

### How DynamoDB Connects to Other Services

**IAM**: controls table/item-level access via IAM policy. **VPC Endpoint**: DynamoDB uses a **Gateway Endpoint** (free, same as S3). **Encryption**: KMS (AWS-owned by default, can switch to customer-managed). **Cross-account access**: resource policies (a more recent addition — check current docs).

### Exam Quick Reference

"Choosing a partition key" → high cardinality, even distribution, avoid hot partitions; "RCU math" → 4 KB strong / 8 KB eventual; "WCU math" → 1 KB; "transactional cost" → 2x RCU/WCU; "unpredictable traffic" → On-Demand; "steady traffic + cost-conscious" → Provisioned + Auto Scaling; "sudden throttling but capacity isn't maxed" → hot partition; "atomic multi-item operation" → Transactional read/write (2x cost); "strongly consistent read" → 2x RCU; "default read consistency" → eventually consistent.

---

## 4. DynamoDB Deep Dive

### Part 1: Secondary Indexes ⭐⭐⭐

**Why they're needed**: DynamoDB's `Query` must be based on the primary key (partition key + optional sort key). What if you need to query by a different attribute?

```
Orders table:
- Partition Key: user_id
- Sort Key: order_date

Can query: "user u1's orders in Jan 2026" ✅
Cannot query: "all orders with status='pending'" ❌ (only Scan works — slow and costly)
```

**The fix**: a Secondary Index gives the table an "extra index" so it can be queried on other attributes.

### The Two Kinds of Secondary Index

|  | LSI (Local Secondary Index) | GSI (Global Secondary Index) |
|---|---|---|
| Partition key | **Must match the base table's** | **Can be different** |
| Sort key | Required, **different from the base table's** | Optional |
| When it can be created | **Only at table creation time** | **Any time**, and can be dropped too |
| Count limit | Up to **5 per table** | Up to **20 per table** |
| Capacity | **Shares the base table's RCU/WCU** | **Independent RCU/WCU** |
| Consistency | Supports strongly consistent reads | **Eventually consistent only** |
| Size limit | Total data per partition key value ≤ **10 GB** | Unlimited |

**LSI**: the partition key must match the base table's — only the sort key changes. "Local" means it only offers a different sort order within the same partition. It shares the base table's capacity.

```
Base table Orders:
- PK: user_id, SK: order_date
- Access pattern: "a user's orders sorted by order time"

LSI:
- PK: user_id (same), SK: order_amount
- Access pattern: "a user's orders sorted by amount"
```

**GSI**: the partition key can be **entirely different** (hence "Global" — an index spanning the entire table). It's essentially "the table reorganized around a different key." It has independent capacity (configured separately). It replicates asynchronously from the base table → **eventually consistent**.

```
Base table Orders:
- PK: user_id, SK: order_date

GSI (querying by status):
- PK: status, SK: order_date
- Access pattern: "all pending orders, sorted by time"
```

### Choosing LSI vs GSI ⭐

| Scenario | LSI | GSI |
|---|---|---|
| Same partition key, different sort order | ✅ | ❌ (wasteful) |
| Completely different partition key | ❌ (impossible) | ✅ |
| Need strongly consistent reads | ✅ | ❌ (eventual only) |
| The table already exists | ❌ (can't add later) | ✅ |
| Want independent capacity control | ❌ | ✅ |

**In practice**: **most scenarios call for a GSI** (more flexible). LSIs mainly fit the specific case where you know at table-creation time that you'll need to sort by a particular attribute.

### Projections ⭐

| Projection | Contains | Storage cost | Query capability |
|---|---|---|---|
| **KEYS_ONLY** | Just the partition key + sort key + base table PK | Smallest | Only gets keys — a base-table lookup is needed for the rest |
| **INCLUDE** | KEYS_ONLY plus attributes you specify | Moderate | Returns the specified attributes |
| **ALL** | Every attribute from the base table | Largest (effectively a full copy) | Fully self-contained |

**Choosing**: **KEYS_ONLY** — the index is used purely for filtering, and the base table is queried when data is needed (saves storage and WCU); **INCLUDE** — a small set of commonly-needed attributes; **ALL** — frequent queries, small data volume, cost isn't a concern.

### GSI Write Amplification ⭐

**The key concept**: every write to the base table triggers a **synchronous write to every GSI** as well.

**WCU math**: writing a 1 KB item to the base table → 1 WCU; with 3 GSIs (each containing the changed attributes) → each GSI also consumes WCU; **total consumption = base table WCU + every GSI's WCU**.

**The trap**: if a **GSI's WCU is under-provisioned**, it can cause **the base table's writes to be throttled too** (the base write has to wait for the GSI write to succeed) — this is a key way GSIs differ from LSIs.

**Best practices**: a GSI's WCU should generally be ≥ the base table's WCU; minimize the number of GSIs (each one adds write overhead); use KEYS_ONLY to reduce the amount of data synced.

### Part 2: DynamoDB Streams ⭐⭐⭐

**= a "change log" for DynamoDB, recording every item modification in a table** — similar to a relational database's binlog/CDC (Change Data Capture).

```
App writes to DynamoDB → DynamoDB Streams records the change automatically →
Lambda / Kinesis consumes it → processes it (notifications, syncing to other systems)
```

**Key traits**: ✅ **24-hour retention** (fixed, not configurable); ✅ **changes are ordered within a partition**; ✅ near real-time — changes appear in the stream within seconds; ✅ can be consumed automatically by a **Lambda trigger**; ✅ can be consumed by the **Kinesis Client Library (KCL)**; ❌ Streams themselves are **free**, but Lambda invocations or Kinesis processing are billed separately.

**The four view types**:

| View type | What the stream contains |
|---|---|
| **KEYS_ONLY** | Just the key of the changed item |
| **NEW_IMAGE** | The full item after the change |
| **OLD_IMAGE** | The full item before the change |
| **NEW_AND_OLD_IMAGES** | Both before and after (most complete) |

**Common uses for Streams**: ① triggering Lambda on changes (most common — e.g. a user signup writes to DynamoDB, which triggers a welcome email); ② cross-table data sync / replication; ③ CloudWatch metrics / real-time analytics; ④ cross-region replication (custom-built; Global Tables is the managed version); ⑤ event-sourcing architectures.

**DynamoDB Streams vs. Kinesis Data Streams for DynamoDB**:

|  | DynamoDB Streams | Kinesis Data Streams for DynamoDB |
|---|---|---|
| Retention | 24 hours | **Up to 365 days** |
| Consumers | Lambda, KCL | Lambda, KCL, Firehose, custom |
| Ordering | Per partition | Per partition |
| Integration | Native to DynamoDB | Through the Kinesis ecosystem |

**Judgment call**: need longer retention or Kinesis ecosystem integration → **Kinesis Data Streams for DynamoDB**.

### Part 3: DAX (DynamoDB Accelerator) ⭐⭐

**= a fully managed in-memory caching layer for DynamoDB.** DynamoDB latency: single-digit milliseconds; DAX latency: microseconds — an order-of-magnitude improvement.

```
App → DAX Cluster → DynamoDB
      ↑ a cache hit returns directly
      ↑ a cache miss queries DynamoDB, then caches the result
```

**Key traits**: ✅ **fully API-compatible with DynamoDB** (almost no code changes — just swap the endpoint); ✅ **write-through**: writes go to DynamoDB first, then update DAX; ✅ lives inside a VPC, never exposed publicly; ✅ multi-node clusters with replica support.

**DAX has two caches**: an **Item Cache** (caches GetItem/BatchGetItem results, default TTL 5 minutes) and a **Query Cache** (caches Query/Scan results, default TTL 5 minutes).

### DAX vs ElastiCache ⭐

|  | DAX | ElastiCache |
|---|---|---|
| Purpose-built for | **DynamoDB specifically** | Any application (general-purpose cache) |
| Integration | Just swap the endpoint | Application code changes required (Lazy Loading, etc.) |
| Caching strategy | Managed automatically | Implemented by the application |
| API compatibility | DynamoDB API | Its own API (Redis/Memcached) |
| Use case | Accelerating DynamoDB | General purpose / sessions / pub-sub / etc. |

**Choosing**: just want to add caching to DynamoDB → **DAX** (zero code changes); a general-purpose cache / multiple data sources / sessions / leaderboards → **ElastiCache**.

**Where DAX doesn't fit**: ❌ write-heavy applications (DAX is a read cache — writes still hit DynamoDB directly); ❌ strongly-consistent reads (DAX's cache can be stale, and strongly consistent reads bypass DAX to query DynamoDB directly); ❌ rapidly changing data (low hit rate, no benefit); ❌ different applications querying the same data (each building its own cache wastes resources).

**Judgment call**: "DynamoDB reads are slow, want milliseconds → microseconds" → DAX; "DynamoDB reads + complex data structures (leaderboard)" → ElastiCache Redis (not DAX); "write hot-spot throttling" → DAX **doesn't help here** (fix the partition key / use On-Demand instead).

### Part 4: TTL (Time to Live) ⭐

**= DynamoDB's mechanism for automatically deleting expired items.**

**How it works**: ① add a numeric attribute to items whose value is a Unix timestamp (the expiration time); ② enable TTL on the table, specifying which attribute holds it; ③ DynamoDB scans in the background and automatically deletes items where the current time exceeds the TTL value.

**Key traits**: ✅ **completely free** (deletion doesn't consume WCU); ✅ deletion happens asynchronously in the background, **typically within 48 hours** (exact timing isn't guaranteed); ✅ delete events are written to Streams (so Lambda can act on them); ✅ items that have expired but aren't yet deleted **don't appear in Query/Scan results** — this is transparent to the application.

**Precision**: TTL is best-effort, not guaranteed to be prompt — don't rely on it for precise timing. For precision, use Lambda + an EventBridge schedule instead.

**Common uses**: ① session data (login tokens expiring); ② temporary data (verification codes, one-time passwords); ③ IoT time-series data (retaining only the last N days); ④ cache data; ⑤ compliance requirements (e.g. GDPR-mandated deletion after N days).

**vs. manual cleanup**: manual cleanup means writing a Lambda + EventBridge schedule to scan and delete — which consumes WCU and costs money; TTL is free and automatic.

### Part 5: Transactions ⭐⭐

**= DynamoDB's ACID transaction support** (since 2018).

| API | Purpose |
|---|---|
| `TransactWriteItems` | **Atomic multi-write** (all succeed or all fail) |
| `TransactGetItems` | **Consistent snapshot across multiple reads** |

**Transaction limits**: ✅ a generous cap on actions per transaction (AWS has raised this over time from an original 25 — check the current docs for the exact number); ✅ **4 MB max** total data; ✅ can span multiple tables, but **must stay within the same region**; ❌ cannot cross accounts.

**Operations supported by `TransactWriteItems`**: `Put` (write/overwrite), `Update` (modify), `Delete` (remove), `ConditionCheck` (a check that doesn't modify anything — just a precondition).

Example (a money transfer):

```
Transaction:
  - Update(account_A, balance -= 100)
  - Update(account_B, balance += 100)
  - ConditionCheck(account_A, balance >= 100)
All three succeed together, or none of them happen.
```

**The cost of transactions**: transactional operations cost **2x capacity** — internally they require a two-phase commit (phase 1: prepare — check and lock; phase 2: commit — execute).

**Transactions vs. Conditional Writes**: many scenarios don't need a full transaction — a Conditional Write is enough:

```
PutItem({
  TableName: "Orders",
  Item: {...},
  ConditionExpression: "attribute_not_exists(order_id)"
})
```

Conditional Writes: apply to a single item; "write only if this condition holds"; cost the same as a normal write (1 WCU); a good fit for optimistic-locking scenarios (preventing overwrites or duplicate creation).

**Choosing**: conditional update on a single item → Conditional Writes (cheaper); atomic operation across multiple items → Transactions (more expensive but safe).

### Part 6: Other Important DynamoDB Features (Quick Overview)

**Global Tables** ⭐: cross-region multi-master replication. Multiple regions can all accept writes (multi-master); replication is asynchronous (typically seconds of lag); **Streams must be enabled** (replication is built on top of Streams); conflict resolution is **last-writer-wins**. Use cases: global applications (users read from the nearest region), DR, cross-region data redundancy.

**Backup & Restore**:

|  | On-Demand Backup | Continuous Backups (PITR) |
|---|---|---|
| Trigger | Manual | Continuous, automatic |
| Retention | Forever (until deleted) | 35 days |
| Restore granularity | The moment of the backup | **Any second** (point-in-time recovery) |
| Use case | Long-term archiving, compliance | Recovering from accidental deletes, PITR |

**Capacity Reservations**: similar to EC2 RIs — DynamoDB supports Reserved Capacity, committing to 1 or 3 years of RCU/WCU for a discount; Provisioned mode only.

**Fine-grained IAM control**: DynamoDB IAM policies can be very precisely scoped:

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

This restricts a user to **only their own partition** — commonly used in multi-tenant applications where each user should see only their own data.

Key condition keys: `dynamodb:LeadingKeys` (restricts the partition key), `dynamodb:Attributes` (restricts accessible attributes), `dynamodb:Select` (restricts whether SELECT * is allowed).

### Part 7: DynamoDB Full Exam Quick Reference

| Scenario | Answer |
|---|---|
| Need to query by a non-key attribute | GSI (flexible) or LSI (sort-key-only change) |
| The table already exists, want to add an index | **GSI** (LSI can only be added at table creation) |
| Need a strongly consistent secondary index | **LSI** (GSI doesn't support this) |
| Trigger Lambda on data changes | DynamoDB Streams |
| Cross-region multi-master replication | Global Tables |
| Speed up DynamoDB reads from ms to μs | **DAX** (zero code changes) |
| Speed up DynamoDB reads + need complex data structures | ElastiCache Redis |
| Write-heavy + throttling | Fix the partition key / go On-Demand (DAX doesn't help) |
| Auto-delete expired data | TTL |
| Atomic multi-item operation | Transactions (2x cost) |
| Conditional update on a single item | Conditional Writes |
| Recovering from an accidental delete | Point-in-Time Recovery |
| Multi-tenant, each user sees only their own data | IAM `dynamodb:LeadingKeys` |
