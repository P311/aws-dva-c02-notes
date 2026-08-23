# Module 3: EC2 + Storage (EBS / Instance Store / EFS / FSx)

> **Covered**: Apr 23, 24, 25
>
> 中文版本：[`zh/03-ec2-storage.md`](../zh/03-ec2-storage.md)

---

## 1. EC2 Basics

### Instance Type Naming

```
m       5        .xlarge
│       │         │
│       │         └─ Size
│       │
│       └─────────── Generation
│
└───────────────── Family (purpose category)
```

**With extra letters**: `m5n.xlarge`, `c6g.large`, `r5dn.2xlarge`

- `a` = AMD CPU
- `g` = ARM (AWS Graviton)
- `n` = higher network bandwidth
- `d` = local NVMe SSD ⭐
- `i` = Intel (rarely spelled out)

### Main Families

| Letter | Type | Typical use | Examples |
|---|---|---|---|
| **T** | Burstable | Low, everyday load with short bursts | t2, t3, t4g |
| **M** | General Purpose | Balanced CPU/memory, web servers | m5, m6i, m7g |
| **C** | Compute Optimized | CPU-heavy: batch, HPC | c5, c6i, c7g |
| **R** | Memory Optimized | Large memory: databases, caching | r5, r6i |
| **X** | High Memory | Very large memory: SAP HANA | x1, x2 |
| **I** | Storage (IOPS) Optimized | High-IOPS local SSD: NoSQL | i3, i4i |
| **D** | Dense Storage (HDD) | Large, cheap capacity | d2, d3 |
| **G / P** | GPU | ML training/inference, graphics rendering | g4, p4 |
| **Inf** | ML inference only | Inferentia chip | inf1, inf2 |
| **Trn** | ML training only | Trainium chip | trn1 |

**Mnemonic**: T = Tiny/Thrifty, M = Medium/Main, C = Compute, R = RAM, X = eXtreme memory, I = IOPS, D = Dense storage, G/P = Graphics/Parallel (GPU)

**Size progression**: `nano → micro → small → medium → large → xlarge → 2xlarge → 4xlarge → 8xlarge → 16xlarge → 24xlarge` (roughly doubling vCPU and memory at each step)

### T-Series Burstable Mechanics

**How it works**: Each T instance has a CPU Credit balance. Running below the baseline (below 100% CPU) accrues credits; running above baseline spends them. Once credits run out, performance drops to the baseline.

**Two modes**:

- **Standard mode**: throttles to baseline once credits are exhausted
- **Unlimited mode** (the default on T3/T3a/T4g!): keeps running at full speed after credits run out, and bills the overage by the hour

**T2 vs T3/T4g default differs**: T2 defaults to Standard; T3, T3a, T4g default to **Unlimited** ⭐.

**Common exam angles**:

- "A T instance's CPU suddenly slows down" → credits exhausted (Standard mode)
- "Unexpected CPU-burst charges on a T3 bill" → because T3 defaults to Unlimited

### AMI (Amazon Machine Image)

**Sources**: provided by AWS (Amazon Linux, Ubuntu, Windows Server); AWS Marketplace (third-party, may carry license fees); Community AMIs (⚠️ a security risk — not recommended for production); or self-created.

**Creating your own AMI**:

1. Launch an EC2 instance, install software, configure it, deploy the app
2. Right-click → Create Image (or `aws ec2 create-image`)
3. Behind the scenes: AWS stops the instance (`--no-reboot` is possible but can leave an inconsistent snapshot) → snapshots every EBS volume → registers the AMI

**AMIs across regions**: an AMI is a **region-scoped resource** and cannot be used directly in another region — you must **Copy AMI** to the target region. Copying an AMI across regions also copies the underlying snapshot. Copying an encrypted AMI across regions requires a KMS key to already exist in the target region.

**Sharing an AMI across accounts requires two layers of permission**:

1. The AMI's `launchPermission` — add the target account ID
2. The underlying EBS snapshot's `createVolumePermission` — also add the target account ID (easy to miss!)
3. (For an encrypted AMI) the KMS key policy — allow the target account to use the key

### Golden AMI vs User Data

|  | Golden AMI | User Data |
|---|---|---|
| Change frequency | Slow (rebuilding an AMI takes tens of minutes) | Fast (just edit the launch template) |
| Boot speed | Fast (software pre-installed) | Slower (a script has to run) |
| Flexibility | Low | High |
| Network dependency | None needed at boot | Usually needed (fetching packages) |
| What belongs here | OS, base tooling, security hardening | Env vars, dynamic config, service-discovery registration |

**Best-practice combination**: bake the OS, base tooling (CloudWatch Agent, SSM Agent), app runtime, app code, and security hardening into the Golden AMI; use User Data at boot to pull config from Parameter Store, write property files, start the app, and register with service discovery.

**EC2 Image Builder** is AWS's managed service for automating Golden AMI builds — useful for organizations that need to regularly rebuild a standard image with the latest patches.

### User Data (Launch Script)

**Characteristics**: runs automatically on first boot only; runs as **root**; capped at **16 KB** (larger payloads go in S3 and get downloaded from User Data); shell script on Linux, PowerShell or cmd on Windows.

**Linux example**:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
```

**Key points**:

- User Data **runs only once by default** (a marker file lives at `/var/lib/cloud/instance/sem/`)
- To force it to run on every boot, set `scripts-user: always` in cloud-init's `cloud_final_modules`
- **Changing User Data on an existing instance**: you must **stop** the instance first (not reboot, not terminate) → edit → start
- ⚠️ Even after stop/start, the new User Data still won't re-run unless `always` is configured

**Where User Data execution logs live**:

| OS | Log path |
|---|---|
| Amazon Linux 2 / 2023 | `/var/log/cloud-init-output.log` (most commonly used) |
| Amazon Linux | `/var/log/cloud-init.log` |
| Ubuntu | Same as above |
| Windows | `C:\ProgramData\Amazon\EC2-Windows\Launch\Log\UserdataExecution.log` |

> ⚠️ User Data logs are **not in CloudTrail** — CloudTrail only records AWS API calls.

**A common User Data + IMDS pattern**:

```bash
#!/bin/bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
REGION=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/region)

# Call AWS using the instance profile's credentials (no access keys needed)
aws s3 cp s3://my-config-bucket/app.conf /etc/myapp/app.conf --region $REGION
systemctl start myapp
```

### Instance Identity Document

**Path**: `/latest/dynamic/instance-identity/document`

**Contents**: JSON describing the instance's identity (accountId, instanceId, region, AMI, pendingTime, etc.), **signed with an AWS private key**.

**Use case**: lets a third-party system verify that a request genuinely came from a specific AWS EC2 instance. Think of it as an ID card with an official seal.

### SSH Key Pairs and Connection Methods

**Key pair**: AWS holds the public key, you hold the private key (.pem file). Associating a key pair at launch makes AWS place the public key in the instance's `~/.ssh/authorized_keys`. **If you lose the private key, AWS cannot help you recover it** — the only path is to stop the instance, detach the root volume, attach it to another instance, rewrite `authorized_keys`, and reattach it.

**Comparing connection methods**:

|  | EC2 Instance Connect | Session Manager | Traditional SSH |
|---|---|---|---|
| Protocol | SSH | SSM | SSH |
| Inbound port | Port 22 must be open (to AWS's IP ranges only) | **No inbound port needed at all** ⭐ | Port 22 must be open |
| Private key management | Not needed (AWS pushes a 60-second temporary key) | Not needed | .pem file management required |
| Auth method | IAM | IAM | SSH key |
| Auditing | CloudTrail | CloudTrail + full session recording ⭐ | OS logs |
| Network requirement | Instance in a public subnet or has a public IP | Instance can reach the SSM service outbound | Instance must be reachable |
| Recommended | Moderate | **Recommended for production** ⭐ | No longer recommended |

**How Session Manager works**: the instance runs the SSM Agent (pre-installed on Amazon Linux 2/2023); the instance has an appropriate IAM role (with the `AmazonSSMManagedInstanceCore` policy); the instance initiates an outbound connection to the SSM service (port 443); you click "Connect" in the console and get routed through SSM into a shell — with zero inbound ports open.

**How EC2 Instance Connect works**: click "Connect" in the console for browser-based SSH; AWS pushes a temporary SSH public key into the instance's `authorized_keys`, valid for 60 seconds; AWS's browser client uses WebSocket to simulate an SSH client connecting to the instance; the temporary key expires after 60 seconds.

**Choosing between them**:

- "No inbound ports at all" → **Session Manager**
- "Don't want to manage SSH keys, but still want SSH" → **EC2 Instance Connect**
- "Full session recording for audit" → **Session Manager**

---

## 2. EC2 Advanced — Pricing + Placement Groups

### EC2 Purchasing Options ⭐⭐⭐

| Option | Price | Commitment | Interruption risk | Typical use |
|---|---|---|---|---|
| **On-Demand** | 100% (baseline) | None | ❌ No | Short-term or unpredictable workloads |
| **Reserved Instance (RI)** | ~30-72% savings | 1 or 3 years | ❌ No | Steady, long-term workloads |
| **Savings Plans** | Similar to RI (30-72%) | 1 or 3 year spend commitment | ❌ No | Flexible, long-term workloads |
| **Spot Instance** | ~70-90% savings | None | ⚠️ AWS can reclaim any time | Fault-tolerant, batch, CI |
| **Dedicated Host** | Most expensive | RI optional | ❌ No | License bound to physical hardware, compliance |
| **Dedicated Instance** | Higher cost | Optional | ❌ No | Physical isolation |
| **Capacity Reservations** | On-Demand price | No commitment | ❌ No | Guaranteed capacity (not for saving money) |

### Reserved Instances (RI)

**Types**:

- **By payment**: All Upfront (biggest discount) / Partial Upfront / No Upfront (smallest discount)
- **By flexibility**:
  - **Standard RI** — deeper discount (up to 72%), cannot change instance family
  - **Convertible RI** — slightly smaller discount (~54%), can swap family
- **Reserved Instance Marketplace** — Standard RIs you no longer need can be resold to other customers

**Key point**: an RI is a **billing discount**, not "reserving a specific machine." Capacity Reservations are the actual capacity guarantee.

### Savings Plans

|  | Compute Savings Plans | EC2 Instance Savings Plans |
|---|---|---|
| Flexibility | Very high | Moderate |
| Scope | EC2 + Fargate + Lambda | EC2 only, specific family in a specific region |
| Discount | Up to ~66% | Up to ~72% |
| Recommended when | Most scenarios | You already know which family you'll use |

**Savings Plans vs RI**: Savings Plans are more flexible (region and family can change); RIs have a slightly deeper discount but lock in the instance; AWS generally recommends Savings Plans.

**Common judgment calls**:

- "Want a discount but might change instance type later" → Convertible RI or Compute Savings Plans
- "Need Lambda discounted too" → Compute Savings Plans (RIs don't cover Lambda)

### Spot Instances ⭐⭐⭐

**Core mechanic**: buy AWS's spare capacity for up to 90% savings; AWS can reclaim it with **2 minutes' notice**.

**Checking for the two-minute interruption notice**:

```bash
# From inside the instance, check whether reclamation is coming
curl http://169.254.169.254/latest/meta-data/spot/instance-action
```

A response of `{"action": "stop", "time": "..."}` means reclamation is imminent.

**Three interruption behaviors**:

- **Stop** (default) — EBS data is preserved and can be resumed later
- **Hibernate** — memory state is written to EBS; resuming restores even memory contents
- **Terminate** — the instance is destroyed outright

**Good fit for Spot**: ✅ batch processing (big-data ETL, video rendering), ✅ CI/CD (GitHub Actions self-hosted runners), ✅ ML training (restart on interruption, combined with checkpointing), ✅ the "elastic" portion of a web fleet (baseline on-demand, scale-out on spot).

**Poor fit for Spot**: ❌ long-running batch jobs (interruption means starting over), ❌ databases, ❌ stateful services.

**EC2 Fleet** replaced Spot Fleet — it buys a mix of spot and on-demand capacity simultaneously and lets AWS pick the cheapest combination.

### Dedicated Host vs Dedicated Instance

|  | Dedicated Host | Dedicated Instance |
|---|---|---|
| Isolation granularity | An entire physical machine is yours | Not shared with other tenants, but the host itself is hidden |
| Physical visibility | ✅ Sockets, cores, host ID visible | ❌ AWS abstracts it away |
| BYOL | ✅ Can bring your own Windows / Oracle / SQL Server license | Partial support |
| Billing | Per physical host | Per instance |
| Main use case | Hardware-bound licensing, compliance audits | Just needing "not shared with others" isolation |
| Placement control | ✅ Can target a specific host | ❌ Cannot |

**Licensing scenarios**: per-socket/per-core licenses (legacy Oracle, SQL Server) require Dedicated Host, since socket count is visible; per-VM licenses work fine on Dedicated Instance; government compliance requiring physical isolation calls for Dedicated Host.

### Capacity Reservations

**Unrelated to discounts — this is purely about "holding a spot."**

**Characteristics**: billed at full on-demand price (no savings!); you pay even if you never launch the instance (a holding fee); **can be stacked with Savings Plans / RI discounts** (reservation guarantees capacity, SP/RI provides the discount); cancellable at any time.

**Judgment call**: "must guarantee X instances can launch on a specific day" → Capacity Reservations; "want to save money" → that's RI / Savings Plans, not this.

### Placement Groups ⭐⭐⭐

| Type | Placement | Network performance | Fault tolerance | Use case |
|---|---|---|---|---|
| **Cluster** | Same AZ, same rack | Very high (10 Gbps+, low latency) | Poor (the whole rack can go down together) | HPC, ML training, low-latency big data |
| **Spread** | Different physical machines (same or across AZs) | Normal | Excellent | Critical apps that can't tolerate a single point of failure |
| **Partition** | Several partitions, each a group of machines | Normal | Good (isolated between partitions) | Hadoop, Cassandra, HDFS |

**Spread Placement Group limit**: at most **7 instances per AZ**.

**Detail**:

- **Cluster** — same rack, extremely low latency (sub-millisecond), but the rack/AZ failing takes everything down together. Use: HPC, large-scale MPI jobs.
- **Spread** — a rack/machine/AZ failure only takes down one instance. Use: small, critical applications (a database master).
- **Partition** — each partition sits on independent hardware (a different rack), and can hold many instances. Use: large distributed systems (HDFS, Cassandra, Kafka) that shard data across partitions, so a single partition failure only loses part of the data.

**Quick reference**:

| Scenario | Choose |
|---|---|
| Nodes need ultra-low-latency communication (HPC, ML training) | Cluster |
| Small, critical app that must never lose more than one instance at once | Spread |
| Large distributed database (Cassandra, HDFS) | Partition |

---

## 3. Storage — EBS, Instance Store, EFS, FSx

### EBS (Elastic Block Store)

**Core characteristics**: **block storage** (behaves like a raw disk you can put a filesystem on); **network-attached** — accessed over the network, not physically plugged into the instance; **an AZ-scoped resource** — can only attach to an instance in the same AZ; **persistent** — survives instance termination unless "delete on termination" is set; can be detached and reattached to another instance in the same AZ.

### EBS Volume Types ⭐⭐⭐

| Type | Media | Use case | Performance | Key trait |
|---|---|---|---|---|
| **gp3** | SSD | General purpose, new default | 3,000-16,000 IOPS (configured independently) | IOPS and capacity decoupled ⭐ |
| **gp2** | SSD | Older general purpose | IOPS scales with capacity (3 IOPS/GB) | Superseded by gp3 |
| **io1 / io2** | SSD | High performance, critical databases | Up to 64,000 IOPS (io1) / 256,000 (io2 Block Express) | **Multi-Attach (io1/io2 only)** ⭐ |
| **st1** | HDD | High-throughput big data | High throughput, low IOPS | Cannot be a boot volume |
| **sc1** | HDD | Cold data, lowest cost | Lowest performance | Cannot be a boot volume |

**gp2 vs gp3**: gp2 ties IOPS to capacity (more IOPS requires more capacity); gp3 lets you **set IOPS and capacity independently**, costs ~20% less, and is the recommended default. gp3's baseline is 3,000 IOPS and 125 MB/s.

**Multi-Attach on io1/io2**: a single io1/io2 volume can attach to **up to 16 instances** simultaneously, all in **the same AZ**; ⚠️ the application must handle concurrent writes (plain ext4 won't work — you need a cluster-aware filesystem like GFS2 or OCFS2).

**HDD limitation**: st1 and sc1 **cannot serve as root volumes** (boot volumes must be SSD).

### EBS Snapshots

**Characteristics**: **incremental** (the first is a full copy, later ones store only changed blocks); stored in S3 (though you can't see the bucket); can be **copied across regions** (for DR); a new volume can be created from a snapshot, even with a different type, size, or AZ; can be made public or shared with specific accounts.

**EBS Snapshot Archive**: moves a snapshot into an archive tier at **75% lower cost**, but **restoring takes 24-72 hours** — suited to long-term compliance retention.

**Recycle Bin**: protects against accidental snapshot deletion; retention period configurable up to 1 year.

### EBS Encryption

Enable encryption at creation time, backed by a KMS key. For an encrypted volume: **snapshots are encrypted automatically, and volumes created from an encrypted snapshot are encrypted automatically.** Once encrypted, everything derived from it (snapshots, volumes, AMIs) stays encrypted.

**Encrypting an existing unencrypted volume** (a common workflow):

1. Snapshot the unencrypted volume (the snapshot is also unencrypted)
2. **Copy the snapshot, enabling encryption and selecting a KMS key during the copy** ⭐ the key step
3. Create a new volume from the encrypted snapshot (automatically encrypted)
4. Swap the volume in (stop the instance → detach the old volume → attach the new encrypted volume → start the instance)

**Key point**: an existing unencrypted volume **cannot be encrypted in place** — the only path is through CopySnapshot with encryption enabled.

**Migrating EBS across AZ/Region**: snapshot the source volume → (cross-region) CopySnapshot to the target region → create a new volume from the snapshot in the target AZ → attach it to an instance in the target AZ → verify, then delete the source volume.

### Instance Store

**= local SSD/NVMe physically plugged into the EC2 host machine.**

|  | EBS | Instance Store |
|---|---|---|
| Physical location | Network-attached | Physically attached to the host |
| Speed | Fast | **Extremely fast** (low latency, high IOPS) |
| Persistence | ✅ Persistent | ❌ All data lost on stop/terminate |
| Snapshots | ✅ Supported | ❌ Not supported |
| Size | Flexible | Fixed (depends on instance type) |
| Price | Billed separately | **Included in the instance price** |

**Data behavior across operations**:

| Operation | EBS data | Instance Store data |
|---|---|---|
| Reboot | ✅ Preserved | ✅ **Preserved!** |
| Stop | ✅ Preserved | ❌ Lost |
| Terminate | Depends on "delete on termination" | ❌ Lost |
| Hibernate | ✅ Preserved | ❌ Lost |
| Physical host failure | ✅ Preserved (network storage) | ❌ Lost |

**Key point**: a reboot doesn't lose instance store data (the host hasn't changed — only the OS restarts); a stop means AWS moves the instance off the host, and the next start may land on a different host entirely.

**Instance types with a "d" carry instance store**: `m5d.xlarge`, `c5d.xlarge`, `r5d.xlarge` (embedded d); the entire `i3`/`i4i` family; the entire `d2`/`d3` family.

**Good fit**: ✅ temp files, caches, buffer/scratch space, ✅ transient video-transcoding files, ✅ local storage for distributed systems with their own replication (Cassandra). ❌ Anything requiring persistence.

### EFS (Elastic File System)

**= AWS's managed NFS file system (Linux only)**

**Core characteristics**: **multiple instances can mount it concurrently** (unlike EBS, which defaults to single-attach); **spans AZs** (one filesystem has mount targets in every AZ); can replicate across regions; **auto-scales** (billed by usage, no pre-provisioning); POSIX-compliant (standard Linux filesystem semantics); **Linux only** (Windows uses FSx); protocol: **NFS v4**.

**EFS Performance Modes**: **General Purpose** (default, fits most scenarios), **Max I/O** (highly parallel — 10,000+ concurrent instances — with slightly higher latency).

**EFS Throughput Modes**: **Bursting** (default, throughput scales with stored capacity), **Provisioned** (throughput configured independently), **Elastic** (auto-scales, AWS's newer recommended option).

**EFS Storage Classes**:

| Class | Price | Access | Use |
|---|---|---|---|
| EFS Standard | Standard | Multi-AZ | Frequently accessed |
| EFS Standard-IA | 92% cheaper | Multi-AZ, slower first access | Infrequently accessed, still-live data |
| EFS One Zone | 47% cheaper | **Single AZ** (no DR) | Regeneratable data |
| EFS One Zone-IA | Cheapest | Single AZ + IA | Infrequently accessed, single-AZ |

**Lifecycle Management**: automatically moves files unaccessed for N days into IA (7, 14, 30, 60, or 90 days selectable).

**Good fit**: ✅ apps like WordPress needing shared files across servers, ✅ shared storage for containers (ECS/EKS), ✅ content management systems, shared CI/CD state, ✅ big-data / ML training data. ❌ Windows apps → FSx for Windows instead; ❌ block-storage semantics (a database's raw device) → EBS instead.

### The FSx Family

| FSx type | Filesystem | Use case |
|---|---|---|
| **FSx for Windows File Server** | Windows SMB / NTFS | Shared storage for Windows applications |
| **FSx for Lustre** | Lustre (HPC filesystem) | HPC, ML training, extreme performance |
| **FSx for NetApp ONTAP** | NetApp enterprise filesystem | Migrating an on-prem NetApp estate to AWS |
| **FSx for OpenZFS** | OpenZFS | Migrating from ZFS / Linux NFS |

**FSx for Lustre's distinctive trait**: it can **link to an S3 bucket** — objects in S3 automatically appear as files in the Lustre filesystem, and results can sync back to S3 once a job finishes. In HPC, S3 acts as the "data lake" and Lustre as the "high-speed compute cache."

**Quick reference**: shared Windows files → **FSx for Windows File Server**; HPC / ML training with high-speed reads → **FSx for Lustre** (can integrate with S3); enterprise NetApp customers moving to the cloud → FSx for ONTAP.

### Storage Comparison Table

| Service | Type | Sharing | Persistent | Linux/Windows | Quick summary |
|---|---|---|---|---|---|
| **EBS** | Block | Single-attach (io1/io2 can be multi) | ✅ | Either | EC2's "hard drive" |
| **Instance Store** | Block | Single instance | ❌ | Either | Temporary, high-speed |
| **EFS** | File (NFS) | Shared across instances | ✅ | Linux only | Shared Linux files |
| **FSx Windows** | File (SMB) | Shared across instances | ✅ | Windows | Shared Windows files |
| **FSx Lustre** | File | Shared across instances | ✅ | Linux | HPC high-speed |
| **S3** | Object | Global | ✅ | API access | Object storage |

**Choosing by scenario**:

| Need | Choose |
|---|---|
| Fast temporary space on a single instance | Instance Store |
| Shared high-speed storage across instances + S3 integration (HPC/ML) | FSx for Lustre |
| Shared Linux files across instances | EFS |
| Shared Windows files across instances | FSx for Windows |
| Persistent block storage for an instance | EBS |
| Storage backing a database | **EBS** (block-device semantics, low latency, high IOPS) |
| Local high-speed scratch buffer for video transcoding | **Instance Store** (transient, extremely fast, bundled into the price) |
