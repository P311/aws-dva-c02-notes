# Module 13: Containers (ECS / Fargate / ECR)

> **Covered**: May 15
>
> 中文版本：[`zh/13-containers.md`](../zh/13-containers.md)

---

## 1. Container Fundamentals + ECR ⭐⭐⭐

### Why Containers

|  | EC2 | Lambda | Containers (ECS/EKS) |
|---|---|---|---|
| Deployment unit | VM (AMI) | Function code | Container image |
| Startup time | Minutes | Milliseconds to seconds | Seconds |
| Run duration | Long-lived | Up to 15 minutes | Long-lived |
| Isolation | VM isolation | Function sandbox | Container/micro-VM isolation |
| Resource efficiency | One app per machine | Extreme (billed per invocation) | **Dense multi-app packing** |
| Fits | Complex legacy applications | Event-driven, small tasks | Microservices, long-running apps |

Core value of containers: package the application plus its dependencies into an image — build once, run anywhere; fast startup, low resource overhead, dense packing; a natural fit for CI/CD pipelines.

### ECR (Elastic Container Registry) ⭐⭐⭐

**= AWS's fully managed container image registry.**

**Two types**:

|  | **Private Registry** | **Public Registry** |
|---|---|---|
| Use case | Internal company images (production) | Open-source distribution |
| Access | Only your AWS account/IAM principals | Anyone, anonymously, worldwide |

⭐ The DVA almost exclusively tests Private — Public is worth knowing about but nothing more.

**Key ECR traits**: image formats include Docker, OCI, Helm charts; at-rest encryption defaults to SSE-S3 or SSE-KMS; **the auth token has a 12-hour TTL** (must be refreshed periodically); **Tag Immutability** (a given tag can't be overwritten, enforcing versioning); Image Scanning (Basic — free, or Enhanced — continuous via Inspector); Lifecycle Policies (automatic cleanup of old images); cross-region/cross-account replication; pull-through cache (caching an external registry into ECR).

**URI structure**: `<account-id>.dkr.ecr.<region>.amazonaws.com/<repository>:<tag>`

**Push flow**:

```bash
# 1. Get a token (valid 12 hours)
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com

# 2. Build + tag
docker build -t my-app .
docker tag my-app:latest \
    123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0

# 3. Push
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
```

**Required IAM permissions**: Push needs `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`/`UploadLayerPart`/`CompleteLayerUpload`, `ecr:PutImage`. Pull needs `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`.

⭐ Managed policies: `AmazonEC2ContainerRegistryReadOnly` (pull only), `AmazonEC2ContainerRegistryPowerUser` (pull + push), `AmazonEC2ContainerRegistryFullAccess` (full control).

**ECR Lifecycle Policy** (commonly tested): rules for automatically cleaning up old images — e.g. "keep the 10 most recent tags, delete untagged images after 7 days." **Production ECR repositories must have a lifecycle policy**, or images accumulate indefinitely and storage costs balloon.

**Two Image Scanning modes** ⭐:

|  | **Basic Scanning** | **Enhanced Scanning** |
|---|---|---|
| Engine | ECR's built-in CVE database | **AWS Inspector v2** |
| Trigger | Scans once on push (or manual rescan) | **Continuous scanning** (auto-rescans as new CVEs are published) |
| Scope | OS packages | OS packages + **application-language packages** (Python, Node, Java) |
| Cost | Free | Billed per image |

**Tag Immutability** (commonly tested): once enabled, a given tag can never be overwritten — preventing "v1.0 actually points to a different image" deployment confusion, and enforcing proper versioning (Git SHA or semver).

**Pull-Through Cache** (brief): caches images from an external registry (Docker Hub, quay.io, GitHub Container Registry, etc.) into ECR — reducing exposure to external rate limits, speeding up pulls, and centralizing auditing.

---

## 2. ECS Core Concepts ⭐⭐⭐

**ECS = Elastic Container Service**, AWS's own container orchestration service.

**The core hierarchy**:

```
Cluster = a pool of compute capacity (Fargate or EC2)
  └── Service = a "controller" keeping N tasks running + LB integration + Auto Scaling
        └── Task 1, Task 2, Task 3... (running groups of containers)

Task Definition (a blueprint) = describes what one Task should look like
```

| Concept | Analogy |
|---|---|
| **Cluster** | An EC2 fleet / Kubernetes cluster |
| **Task Definition** | A Docker Compose file / K8s Pod spec |
| **Task** | A running group of containers (one Pod instance) |
| **Service** | A K8s Deployment/ReplicaSet |
| **Container Instance** (EC2 only) | An EC2 instance running the ECS Agent |

---

## 3. Launch Types — Fargate vs. EC2 ⭐⭐⭐

|  | **Fargate** | **EC2** |
|---|---|---|
| Who manages infrastructure | **AWS** (serverless) | **You** (managing the EC2 fleet) |
| Billing | Per-second CPU/memory used by the task | Per EC2 instance-hour |
| Isolation strength | **Each task gets its own micro-VM** (strong) | Tasks share the host kernel (weaker) |
| Max resources | A higher ceiling (has grown over time) | Limited by instance type |
| GPU support | ❌ | ✅ (p/g families) |
| Spot pricing | ✅ Fargate Spot (~70% off) | ✅ EC2 Spot |
| Networking mode | **awsvpc only** | awsvpc/bridge/host/none |
| Typical use | Microservices, variable load, no ops overhead wanted | Large-scale steady load, GPU, special configurations |
| Cost per task | Higher | Cheaper (can pack densely) |

**Choosing**: pick Fargate when you don't want to manage EC2, traffic is bursty/variable, it's a microservice/API backend, batch processing, or you want Fargate Spot savings. Pick EC2 for large-scale steady load (RIs/Savings Plans give a big discount), GPU needs, dense packing (task cost is much lower than Fargate), specific instance types, or privileged containers/host networking.

**ECS Managed Instances** (a newer feature): AWS manages the EC2 layer while offering a near-Fargate operational experience plus EC2's flexibility — a hybrid option.

### Capacity Providers (the Modern Recommendation)

The old way: specifying `requiresCompatibilities: ["FARGATE"]` or `["EC2"]` directly in the task definition (legacy).

The new way: **an ECS cluster is associated with multiple capacity providers, and tasks are placed according to a strategy.** Supported providers: FARGATE, FARGATE_SPOT, an EC2 Auto Scaling Group capacity provider, and ECS Managed Instances.

A capacity provider strategy example: "70% Fargate Spot (cheap, interruptible) + 30% Fargate (a stable baseline)" — ECS automatically maintains that ratio and auto-scales the underlying capacity; a single cluster can mix providers.

---

## 4. Task Definitions ⭐⭐⭐

**= the JSON template describing what a task should look like.**

```json
{
  "family": "my-web-app",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::...:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::...:role/myAppTaskRole",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1",
      "cpu": 256,
      "memory": 512,
      "portMappings": [{ "containerPort": 80, "protocol": "tcp" }],
      "environment": [{ "name": "DB_HOST", "value": "rds-endpoint.xxx" }],
      "secrets": [
        { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:...:secret:db-password" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "essential": true
    }
  ]
}
```

**Task Definition Revisions**: every update to the same `family` creates a new revision (`my-web-app:1`, `my-web-app:2`, ...); revisions are **immutable** — they can't be edited, only superseded; a Service references a specific revision, and deploying a new version means pointing the service at a new revision.

**CPU / Memory** (Fargate enforces specific combinations, exact ranges change over time — check current docs): common tiers range from roughly 0.25 vCPU/0.5-2 GB up to much larger configurations, with the maximum having expanded significantly over the years. Exam question banks may lag — verify against current AWS documentation.

### Network Mode ⭐⭐

| Mode | How it works | Fargate | EC2 |
|---|---|---|---|
| **awsvpc** | **Each task gets its own ENI and private IP** | ✅ (mandatory) | ✅ |
| **bridge** | Docker's default bridge — tasks share the host's IP with port mapping | ❌ | ✅ |
| **host** | Uses the host network directly (no isolation) | ❌ | ✅ |
| **none** | No external networking | ❌ | ✅ |

**Important awsvpc details**: each task gets its own ENI (counts against ENI quotas); its own security group; its own IP (consuming subnet IP space). ⚠️ for large clusters, make sure the subnet is sized appropriately or you'll run out of IPs.

**A classic bridge-mode pattern — dynamic port mapping with an ALB**: the container listens on a fixed internal port (e.g. 80); the host assigns a dynamic port; the ALB target group tracks each task's dynamic port automatically; a single EC2 host can run multiple copies of the same container (dense packing).

**Task Placement Strategies**: `binpack` (packs tasks onto the fewest instances, minimizing count); `random` (random placement); `spread` (evenly distributes based on an attribute like instanceId or host).

---

## 5. IAM in ECS ⭐⭐⭐

ECS involves 3 distinct IAM roles, and the differences between them come up often:

```
EC2 Container Instance (EC2 launch type only)
  ECS Container Instance Role → used by the ECS Agent to register with the cluster and send heartbeats
                ↓ runs
Task (Fargate or EC2)
  Task Execution Role → used by the ECS Agent for: pulling from ECR, sending logs to CloudWatch,
                         fetching secrets from Secrets Manager/SSM
  Task Role → used by the application code inside the container: calling AWS APIs
              (S3/DynamoDB/SNS/SQS and other business-level permissions)
```

### The Key Comparison ⭐⭐⭐

|  | **Execution Role** | **Task Role** |
|---|---|---|
| Used by | **The ECS Agent** (infrastructure layer) | **Your application code** (business layer) |
| When | Before the task starts (pulling images, setting up logs) | While the task is running (the app calls AWS APIs) |
| Typical permissions | `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `logs:CreateLogStream`, `secretsmanager:GetSecretValue` | Business permissions like `s3:GetObject`, `dynamodb:PutItem` |
| Required? | Essentially mandatory (no image pull without it) | Optional (but needed if the app calls AWS) |

**Mnemonic**: "Execution = infrastructure prep work; Task = business-logic runtime."

**Specific scenarios**: pulling an image from ECR → Execution Role; sending logs to CloudWatch → Execution Role; injecting a DB password from Secrets Manager into a container's environment variables → Execution Role; application code reading S3/writing DynamoDB/calling Lambda/publishing to SNS → **Task Role**.

⚠️ **A common bug**: granting S3 permissions to the execution role instead of the task role — causing the application's S3 calls to fail.

**Injecting secrets into a container**: Option 1 (recommended) — the task definition's `secrets` field, requiring `secretsmanager:GetSecretValue` on the Execution Role; the ECS Agent pulls the secret and injects it as an environment variable at container start. Option 2 — the application code calls Secrets Manager itself (using the Task Role) — safer (the secret never lives in an env var), but requires code changes.

---

## 6. ECS Service ⭐⭐

**A Service is the "controller" that keeps N tasks running continuously.**

**Traits**: maintains the desired task count; restarts dead tasks automatically; integrates with a load balancer; Auto Scaling; rolling or blue/green deployments; Service Discovery.

**Service scheduling strategies**: `REPLICA` (default — maintains a specified count); `DAEMON` (runs exactly 1 task per container instance — good for logging agents, monitoring).

### Deployment Types ⭐⭐

**1. Rolling Update** (default): controlled via minimumHealthyPercent + maximumPercent — simple, built into ECS.

**2. Blue/Green** (requires CodeDeploy): deploys the full new version, then cuts over traffic; supports canary/linear/all-at-once traffic shifting; a one-click rollback to blue on failure; fits critical applications.

**3. External**: a third-party deployment tool controls the task cutover yourself.

⚠️ **Blue/Green requires CodeDeploy** — Rolling Update is ECS's own built-in mechanism.

---

## 7. Load Balancer Integration ⭐⭐

### ALB (Most Common)

|  | Fargate (awsvpc) | EC2 (bridge mode) |
|---|---|---|
| Target type | **IP** (each task has its own ENI/IP) | **instance** or **IP** |
| Port mapping | Fixed port | **Dynamic port mapping** (common, for dense packing) |

**Dynamic Port Mapping** (EC2 bridge mode): the task definition sets a dynamic host port; ECS picks an available port automatically; the ALB target group tracks it; a single EC2 host runs multiple copies of the same container, each on a different host port.

**When NLB fits**: extreme performance (L4); static IPs; long-lived connections/gRPC.

**Service Discovery**: **AWS Cloud Map** (formerly Route 53 Auto Naming — registers a DNS name per task, deregistering automatically when it dies); **ECS Service Connect** (newer, recommended — a built-in service mesh with metrics, retries, and an Envoy proxy).

---

## 8. Auto Scaling ⭐⭐

### Service Auto Scaling

**= automatically adjusting a service's task count**, built on Application Auto Scaling.

**Three strategies**: **Target Tracking** (holds a metric at a target, e.g. CPU 50% — the default recommendation); **Step Scaling** (scales in steps based on thresholds, good for a well-understood traffic curve); **Scheduled Scaling** (time-based, for predictable cyclical patterns).

### Cluster Auto Scaling (EC2 Launch Type)

**= scaling the underlying Auto Scaling Group behind an EC2 capacity provider.** Mechanism: ECS monitors the "unplaced task" backlog (a capacity reservation gap) → calculates how much EC2 capacity is needed → tells the Auto Scaling Group to adjust its desired capacity.

Fargate **doesn't need cluster auto scaling** (AWS manages the underlying capacity).

---

## 9. Storage / Volumes ⭐

ECS supports several volume types:

| Volume type | Fargate | EC2 | Persistence | Fits |
|---|---|---|---|---|
| **Bind mount** | ✅ (ephemeral, task lifetime) | ✅ | ❌ Gone when the task ends | Temp data, cross-container sharing |
| **Docker volume** | ❌ | ✅ | ⚠️ Depends on the driver | Legacy pattern |
| **EFS volume** | ✅ | ✅ | **✅ Persistent** | **The go-to for persistent Fargate data** |
| **FSx for Windows / Lustre** | ✅ | ✅ | ✅ | Windows/HPC |

⭐ **Key point**: **persistent storage on Fargate must use EFS** (no EBS!) — Fargate has no way to attach an EBS volume.

**Fargate Ephemeral Storage**: each Fargate task gets roughly 20 GB of ephemeral storage by default, configurable to a larger range (a newer feature — check current docs for exact limits); it's destroyed when the task ends.

---

## 10. ECS Anywhere (Brief)

**= running ECS tasks on-premises or on other clouds.** Mechanism: install the SSM Agent + ECS Agent on the external server, register it under the "EXTERNAL" launch type, and manage it via SSM. Use cases: extending AWS's control plane into an on-prem datacenter, edge computing, hybrid cloud migration.

---

## 11. EKS vs. ECS (Comparison — Not a DVA Focus)

|  | **ECS** | **EKS** |
|---|---|---|
| Orchestration engine | AWS's own | **Kubernetes (open source)** |
| Learning curve | Simple | Steep |
| Cross-cloud portability | ❌ | ✅ (K8s runs anywhere) |
| Control plane | Free | Billed hourly |
| Fits | AWS-native, minimal operational burden | Existing K8s experience / multi-cloud |

The DVA focuses on ECS — EKS is worth knowing exists, nothing more.
