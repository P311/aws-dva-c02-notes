# Module 5: VPC (Networking)

> **Covered**: Apr 29
>
> 中文版本：[`zh/05-vpc.md`](../zh/05-vpc.md)

---

## VPC Basics

**= Virtual Private Cloud** — your own private virtual network inside AWS.

**Key attributes**: a **region-scoped resource** (one VPC lives in one region, spanning multiple AZs); you specify a CIDR block at creation (e.g. `10.0.0.0/16`, holding 65,536 IPs); every AWS account gets one default VPC per region.

**Core components**: Subnets, Internet Gateway (IGW), NAT Gateway, Route Tables, Security Groups (instance-level), NACLs (subnet-level), VPC Endpoints, VPC Flow Logs.

### Subnets

**Traits**: an **AZ-scoped resource** (a subnet lives in exactly one AZ); a VPC can hold many subnets; each subnet has its own CIDR; AWS **reserves 5 IPs** per subnet: `.0` (network address), `.1` (VPC router), `.2` (DNS), `.3` (reserved), `.255` (broadcast address).

### Public Subnet vs Private Subnet ⭐

**A public subnet needs both**: ✅ a route-table default route to an Internet Gateway (`0.0.0.0/0 → IGW`); ✅ instances in the subnet with a public IP (auto-assigned or Elastic IP).

**Private subnet**: no route to an IGW; instances have no public IP; outbound traffic requires a NAT Gateway.

**Classic three-tier layout**:

```
Public Subnet:    ALB (receives external requests)
  ↓
Private Subnet:   EC2 application tier (not directly exposed)
  ↓
Private Subnet:   RDS database tier (most tightly isolated)
```

### Internet Gateway (IGW)

**Function**: gives a VPC internet access (both directions).

**Traits**: VPC-scoped (one IGW per VPC); highly available (AWS-managed); no bandwidth limit (free).

**Three conditions for a subnet to actually reach the internet**: ① the VPC has an IGW; ② the subnet's route table has `0.0.0.0/0 → IGW`; ③ the instance has a public IP (or Elastic IP).

### NAT Gateway ⭐

**The problem**: an instance in a private subnet needs outbound internet access (yum update, calling an external API) without being reachable from outside.

**Mechanism**: NAT translates the private instance's private IP into the NAT Gateway's own Elastic IP.

**Traits**: AWS-managed — highly available, auto-scaling; **an AZ-scoped resource** — one NAT Gateway lives in one AZ; **multi-AZ availability requires deploying a NAT Gateway in each AZ**; billed hourly plus data transfer.

### NAT Gateway vs NAT Instance

|  | NAT Gateway | NAT Instance |
|---|---|---|
| Nature | AWS-managed service | An EC2 instance you run yourself |
| High availability | ✅ Automatic within an AZ | ❌ You configure it yourself |
| Bandwidth | High, auto-scales (up to 100 Gbps) | Limited by instance type |
| Maintenance | None | You patch it |
| Cost | Higher | Cheaper (for low-traffic scenarios) |
| Source/dest check | Handled automatically | **Must be disabled manually** |
| Current recommendation | ✅ Preferred | ❌ Legacy, not recommended |

### VPC Endpoints ⭐⭐⭐

**= let resources inside a VPC reach AWS services over the AWS internal network, without going through the public internet, an IGW, or NAT.**

**Why it matters**:

```
Without a VPC Endpoint:
Private EC2 → NAT Gateway → IGW → public internet → S3
              ↑ expensive data transfer + goes over the public internet

With a VPC Endpoint:
Private EC2 → VPC Endpoint → S3
              ↑ AWS internal network, free (Gateway type)
              ↑ never touches the public internet, more secure
```

### The Two Types of VPC Endpoint ⭐⭐⭐

|  | Gateway Endpoint | Interface Endpoint |
|---|---|---|
| Supported services | **S3 and DynamoDB only** | Almost every other AWS service |
| Underlying mechanism | A route table entry | **An ENI (PrivateLink)** |
| Consumes an IP | ❌ No | ✅ Uses a private IP |
| Cost | **Free** ⭐ | Billed hourly + per GB |
| Scope | VPC-level | Subnet-level (one ENI per AZ) |
| How to enable | Add a route table entry | Create an ENI |

**Gateway Endpoint supports exactly two services**: S3 and DynamoDB.

**Interface Endpoint supports**: KMS, Secrets Manager, SQS, SNS, API Gateway, SSM, CloudWatch, ECR, Lambda, and more.

**Decision table**:

| Accessing this from a private instance | Use |
|---|---|
| S3 | **Gateway Endpoint** (free) |
| DynamoDB | **Gateway Endpoint** (free) |
| KMS / Secrets Manager / SSM | Interface Endpoint |
| SQS / SNS / EventBridge | Interface Endpoint |
| API Gateway / Lambda | Interface Endpoint |
| Any other AWS service | Interface Endpoint |

**Mnemonic**: "S3 + DynamoDB = Gateway," everything else is Interface.

**Important**: services that already live inside a VPC (RDS, ElastiCache, EC2, ELB) **don't need a VPC Endpoint** — endpoints are for reaching AWS services that live *outside* your VPC.

### Security Group vs NACL ⭐⭐

|  | Security Group | NACL |
|---|---|---|
| Scope | Instance-level (ENI) | Subnet-level |
| State | **Stateful** | **Stateless** |
| Rule types | **Allow only** | Allow + Deny |
| Rule evaluation | All rules considered together (any allow lets it through) | **Evaluated in numeric order — first match wins** |
| Can reference another SG | ✅ Yes | ❌ IP addresses only |
| Default inbound | Deny all | Allow all (default NACL) |
| Default outbound | Allow all | Allow all (default NACL) |
| Count | Up to 5 SGs per instance | One NACL per subnet |

### The Practical Impact of Stateful vs Stateless

**Stateful (SG) example**: an SG allowing inbound port 80 means a web server's response to the client **doesn't need an outbound rule allowing port 80** — the return traffic is automatically permitted.

**Stateless (NACL) example**: a NACL allowing inbound port 80 only permits the incoming connection. The response packet leaving the instance **also needs an explicit outbound NACL rule** (typically allowing the ephemeral port range 1024-65535).

**A common trap**: a NACL allows inbound traffic but has no matching outbound rule, so the HTTP connection still fails — because NACLs are stateless, the response can't get out. Fix: allow the 1024-65535 ephemeral port range outbound in the NACL.

### When to Use Which

| Scenario | Use |
|---|---|
| Controlling access to a single instance | Security Group |
| **Need to explicitly deny specific IPs** | **NACL** (SGs have no Deny rules) |
| Coarse-grained protection at the subnet level | NACL |
| Default scenario | Security Group (leave the NACL at its default) |

**A practical pattern — SG referencing SG**:

```
ALB-SG: allows inbound 443 from 0.0.0.0/0
EC2-SG: allows inbound 8080 from ALB-SG   ← reference the SG, not an IP!
```

Benefit: as the ALB scales and its IPs change, referencing the SG means the rule never needs updating. NACLs can't do this.

### VPC Peering

**= a private network connection between two VPCs.**

**Traits**: bidirectional; **not transitive** (A↔B and B↔C don't imply A↔C — a separate A↔C peering is required); works across regions and accounts; **CIDR blocks must not overlap**.

**Rule of thumb**: 2-3 VPCs → VPC Peering; many VPCs (>3) → Transit Gateway.

### Transit Gateway

**= a central hub connecting many VPCs and on-premises networks.**

**Use case**: when the number of VPCs grows, peering becomes unmanageable (n² connections) — Transit Gateway simplifies this into a hub-and-spoke design.

### VPC Flow Logs ⭐

**= records network traffic within a VPC.**

**Destinations**: CloudWatch Logs, S3, Kinesis Data Firehose.

**Capture levels**: VPC-level, Subnet-level, ENI-level (most granular).

**What's recorded**: source/destination IP, port, protocol; packet and byte counts; ACCEPT / REJECT; timestamps.

**Trap — Flow Logs do *not* capture certain traffic**: ❌ traffic to the Amazon DNS server (traffic to a custom DNS server IS captured); ❌ Windows license activation traffic; ❌ **IMDS traffic (169.254.169.254)** ⭐; ❌ DHCP traffic; ❌ traffic to the VPC router's reserved address (.1).

**Important exam point**: since IMDS traffic isn't logged, Flow Logs cannot be used to trace IMDS credential theft — other tools like GuardDuty are required for that.

### Auto-Assigned IP vs Elastic IP

- **Auto-assigned public IP** (default): changes on stop/start (unchanged on reboot)
- **Elastic IP (EIP)**: a fixed public IP you own — stays the same when attached to an instance; can be detached and reattached to a different instance; **an unattached EIP is billed** (to discourage holding one idle); default quota is 5 EIPs per account per region

**Want an IP that never changes** → use an **Elastic IP**.

### Quick Reference for Common Scenarios

| Scenario | Answer |
|---|---|
| How does a private EC2 reach an external API? | NAT Gateway (or a VPC Endpoint first, if it's an AWS service) |
| How does a private EC2 reach S3 without the public internet? | **S3 Gateway Endpoint** |
| How does a private EC2 reach Secrets Manager? | Secrets Manager Interface Endpoint |
| How does a fully isolated VPC use AWS services? | Create a VPC Endpoint for every service needed |
| How do two VPCs communicate privately? | VPC Peering (few VPCs) or Transit Gateway (many) |
| How do you explicitly deny specific IPs? | NACL (SGs have no Deny) |
| Can an EC2 keep the same IP across a reboot? | Elastic IP |
