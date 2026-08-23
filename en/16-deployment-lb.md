# Module 16: Deployment & Load Balancing (Elastic Beanstalk + ELB/ALB/NLB)

> **Covered**: May 21
>
> 中文版本：[`zh/16-deployment-lb.md`](../zh/16-deployment-lb.md)

---

# 1. Elastic Beanstalk ⭐⭐

## What Elastic Beanstalk Is ⭐⭐⭐

**= AWS's PaaS (Platform as a Service).**

**Core value**: ✅ "just upload your code, I'll handle the rest" (EC2 + ASG + ALB + health monitoring + rolling deployments); ✅ completely free (you only pay for the underlying EC2/ALB/RDS resources); ✅ full access to underlying resources (you can SSH into the EC2 instances); ✅ fits traditional web applications/APIs.

**Three core components**: **Application** (a logical container — the project name); **Environment** (the actual AWS resources running — each environment has its own ASG/ALB); **Application Version** (packaged code, zip/war).

## Supported Platforms

Java (SE, Tomcat), .NET (Windows Server, Linux), Node.js, PHP, Python, Ruby, Go (all Linux), and **Docker** (single- or multi-container, Linux).

⚠️ Custom Platform has been deprecated — use Docker instead.

## Environment Tiers — Web vs. Worker ⭐⭐⭐

**Web Server Environment** (Web tier): handles HTTP requests. `Internet → ALB → ASG (EC2 instances running your app)`. Fits web applications, REST APIs, websites.

**Worker Environment** (Worker tier): handles background/async work. `SQS Queue → Beanstalk Worker (EC2 polls SQS) → processes → deletes the message`.

Mechanism: Beanstalk creates the SQS queue automatically; each EC2 runs a daemon that polls SQS; the daemon POSTs the message body to a local HTTP endpoint (`localhost:80` by default); the application handles the POST → processes it → returns 200 → the daemon deletes the SQS message.

**Key point**: the Worker tier has no ALB — it fits time-consuming tasks like sending email, image processing, or PDF generation.

**cron.yaml** (Worker scheduled tasks) ⭐: the Worker tier supports scheduled triggers (like cron):

```yaml
version: 1
cron:
  - name: "daily-cleanup"
    url: "/cron/cleanup"
    schedule: "0 0 * * *"
  - name: "hourly-stats"
    url: "/cron/stats"
    schedule: "0 * * * *"
```

The daemon POSTs to the specified URL on schedule automatically — no EventBridge required.

⭐ Judgment call: "Beanstalk + scheduled background task" → Worker tier + cron.yaml.

## Deployment Policies ⭐⭐⭐

**= EB's five strategies for deploying a new version.**

| Policy | Behavior | Downtime | Cost | Failure impact |
|---|---|---|---|---|
| **All at once** | Updates every instance at once | **Yes (brief)** | Lowest | Everything rolls back |
| **Rolling** | Updates existing instances in batches | None (reduced capacity) | Lowest | Updated batch rolls back |
| **Rolling with additional batch** | Adds a new batch first, then rolls through updates | None | Moderate | Updated batch rolls back |
| **Immutable** | Launches a brand-new ASG, cuts over after validation, removes the old one | None | High (temporarily double) | Just delete the new ASG — safe |
| **Traffic Splitting** | Immutable + percentage-based traffic shifting (canary) | None | High | Automatically shifts back to old — safest |

**Choosing**: dev/test → All at once; production, capacity dip acceptable → Rolling; production, must maintain capacity → Rolling with additional batch; production, maximum stability → Immutable; production, canary testing → Traffic Splitting (requires an ALB).

## Blue/Green Deployment ⭐

**EB has no native "Blue/Green" deployment policy**, but it can be achieved via **swap URLs**: create a second environment (green) → deploy the new version → thoroughly test green → click "Swap Environment URLs" in the EB console (CNAME swap) → if something goes wrong, swap again for an instant rollback.

vs. Traffic Splitting: Blue/Green means two complete environments with a manually managed cutover; Traffic Splitting is a progressive rollout within a single environment.

## .ebextensions ⭐⭐

**= configuration files that customize an EB environment.** Location: `.ebextensions/*.config` at the root of the application source (YAML or JSON).

```yaml
option_settings:
  aws:autoscaling:asg:
    MinSize: 2
    MaxSize: 8

packages:
  yum:
    git: []

commands:
  01_install_deps:
    command: "yum install -y mypackage"

container_commands:
  01_migrate:
    command: "python manage.py migrate"
    leader_only: true
```

**commands vs. container_commands** ⭐:

|  | commands | container_commands |
|---|---|---|
| When they run | **Before** the application code is deployed | After the code is deployed but **before it starts** |
| Access to app code | ❌ | ✅ |
| Used for | Installing OS packages, system configuration | DB migrations, build steps |
| leader_only | ❌ | ✅ |

## RDS in Beanstalk ⭐⭐

⚠️ **The problem**: terminating the EB environment also deletes the RDS instance if it was created inside EB!

**The recommended approach** (frequently tested): keep RDS independent of EB, connected via environment variables — create RDS separately → set EB environment variables (DB_HOST/DB_USER/DB_PASSWORD) → update the security group to allow the EB instances to reach RDS.

**Decoupling an RDS instance already inside EB**: set `DeletionPolicy: Retain` on the RDS → create an RDS snapshot → create an independent RDS instance from that snapshot → update the EB environment variables to point at the new instance → terminate the old EB environment.

## Other EB Details ⭐

**HTTPS**: add an HTTPS listener to the EB ALB, using an ACM certificate, and force redirect HTTP → HTTPS.

**Health Check Reporting**: Basic (CloudWatch metrics, 5-minute granularity, free); Enhanced (per-instance and overall environment health, 1-minute granularity, billed, 7 health states).

**Important EB CLI commands**: `eb init` (initialize), `eb create` (create an environment), `eb deploy` (deploy), `eb status`, `eb health`, `eb logs`, `eb ssh`, `eb swap` (swap environment URLs), `eb terminate`.

**Limits**: roughly 75 applications per region; roughly 1000 application versions per app; max deployment package size **512 MB**.

---

# 2. Elastic Load Balancer ⭐⭐

## Comparing the ELB Types ⭐⭐⭐

| Type | Layer | Fits |
|---|---|---|
| **CLB** (Classic, legacy) | L4+L7 | Legacy, on a deprecation path |
| **ALB** (Application LB) | **L7 (HTTP)** | Web applications, APIs, microservices |
| **NLB** (Network LB) | **L4 (TCP/UDP/TLS)** | Extreme performance, static IPs, long-lived connections |
| **GWLB** (Gateway LB) | **L3 (IP)** | Third-party network appliances (firewalls, IDS) |

⭐ The DVA mainly tests ALB and NLB — GWLB is worth knowing exists, nothing more.

## ALB ⭐⭐⭐

**= an L7 HTTP/HTTPS load balancer.**

**Multi-target routing rules**: **path-based** (`/api/*` → backend A); **host-based** (`api.example.com` → service A); **header-based** (`X-Version: v2` → a new backend); **query string**; **source IP** (CIDR matching).

**Target types**: `instance` (an EC2 instance ID); `IP` (any IP, including on-prem — **ECS Fargate awsvpc mode must use IP**); `lambda` (the ALB invokes Lambda directly).

**SNI (Server Name Indication)** ⭐: a single ALB can bind multiple SSL certificates (for different domains) — the client presents an SNI header during the TLS handshake, and the ALB selects the matching certificate.

**Listener rule actions**: `forward` (route to a target group); `redirect` (URL rewriting, e.g. HTTP → HTTPS); `fixed-response` (return fixed HTML/JSON, e.g. a maintenance page); `authenticate-cognito` (Cognito login integration); `authenticate-oidc` (any OIDC provider).

**Sticky sessions**: routes a given client's requests consistently to the same target — via an LB-generated cookie (the ALB creates an `AWSALB` cookie) or an app-based cookie (the application generates it, and the ALB uses that cookie for stickiness).

**Cross-Zone Load Balancing**: enabled by default on ALB, free; disabled by default on NLB, and cross-AZ traffic is billed if enabled.

**WebSocket / gRPC**: ALB natively supports WebSocket and HTTP/2/gRPC (NLB just does TCP, protocol-agnostic).

**ALB + Cognito authentication**: configuring the `authenticate-cognito` action on a listener rule handles user login directly, without needing API Gateway.

**Dynamic port mapping** (ECS bridge mode): multiple copies of the same container run on one EC2 host, each on a random port, and the ALB target group tracks instance + dynamic port.

## NLB ⭐⭐

**= an L4 TCP/UDP/TLS load balancer.**

**Traits**: ✅ extreme performance (millions of RPS, tens-of-microseconds latency); ✅ static IPs (one Elastic IP per AZ); ✅ preserves the client's source IP (unmodified, so the backend sees the real client IP); ✅ cross-AZ balancing disabled by default (must be manually enabled, and is billed); ❌ no HTTP-level features (path routing, header routing).

**Target types**: instance/IP/ALB (an NLB can sit in front of an ALB).

## ELB Connection Draining / Deregistration Delay

CLB calls this Connection Draining; ALB/NLB call it **Deregistration Delay** (defaults to **300 seconds**, configurable 0-3600). When a target is deregistered, it keeps serving existing connections while new connections stop being routed to it.

The corresponding mechanism for a newly added target is **Slow Start** — gradually ramping up traffic to a new target instead of sending it full load immediately.
