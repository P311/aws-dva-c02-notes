# Module 13：容器服务（ECS / Fargate / ECR）

> **覆盖日期**：5/15
>
> English version: [`en/13-containers.md`](../en/13-containers.md)

---

## 一、容器基础与 ECR ⭐⭐⭐

### 为什么用容器

|  | EC2 | Lambda | 容器 (ECS/EKS) |
|---|---|---|---|
| 部署单元 | VM (AMI) | function code | container image |
| 启动时间 | 几分钟 | 毫秒到几秒 | 秒级 |
| 运行时长 | 长期 | 最长 15 分钟 | 长期 |
| 隔离 | VM 隔离 | function 沙箱 | 容器/micro-VM 隔离 |
| 资源利用率 | 单机一应用 | 极致（按调用付费） | **多应用密集打包** |
| 适合 | 复杂遗留应用 | 事件驱动、小任务 | 微服务、长跑应用 |

容器核心价值：应用 + 依赖打包到 image，一次构建到处跑；启动快、资源占用低、密集打包；与 CI/CD pipeline 天然契合。

### ECR (Elastic Container Registry) ⭐⭐⭐

**= AWS 完全托管的容器镜像注册表**

**两种类型**：

|  | **Private Registry** | **Public Registry** |
|---|---|---|
| 用途 | 公司内部镜像（生产） | 开源分发 |
| 访问 | 仅 AWS account/IAM principal | 全球任何人匿名 pull |

⭐ DVA 几乎只考 Private，Public 知道即可。

**ECR 关键特性**：镜像格式支持 Docker、OCI、Helm charts；加密 at-rest 默认 SSE-S3 或 SSE-KMS；**认证 token 12 小时 TTL**（必须定期刷新）；**Tag Immutability**（同 tag 不能覆盖，强制版本化）；Image Scanning（Basic 免费/Enhanced 用 Inspector 持续扫）；Lifecycle Policies（自动清理旧镜像）；Cross-region/Cross-account replication；Pull-through cache（缓存外部 registry 到 ECR）。

**URI 结构**：`<account-id>.dkr.ecr.<region>.amazonaws.com/<repository>:<tag>`

**Push 流程**：

```bash
# 1. 拿 token(12 小时有效)
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

**IAM Policy 必要权限**：Push 需要 `ecr:GetAuthorizationToken`、`ecr:BatchCheckLayerAvailability`、`ecr:InitiateLayerUpload`/`UploadLayerPart`/`CompleteLayerUpload`、`ecr:PutImage`；Pull 需要 `ecr:GetAuthorizationToken`、`ecr:BatchGetImage`、`ecr:GetDownloadUrlForLayer`。

⭐ 托管策略：`AmazonEC2ContainerRegistryReadOnly`（只拉）、`AmazonEC2ContainerRegistryPowerUser`（拉+推）、`AmazonEC2ContainerRegistryFullAccess`（全权）。

**ECR Lifecycle Policy**（常考）：自动清理旧镜像的规则，例如"保留最近 10 个 tag，untagged 镜像 7 天后删除"。**生产 ECR 必须配 lifecycle policy**，否则镜像无限累积，存储费爆炸。

**Image Scanning 两种模式** ⭐：

|  | **Basic Scanning** | **Enhanced Scanning** |
|---|---|---|
| 引擎 | ECR 内置 CVE database | **AWS Inspector v2** |
| 触发 | Push 时扫一次（或手动重扫） | **持续扫描**（新 CVE 公布时自动重扫） |
| 扫描范围 | OS 包 | OS 包 + **应用语言包**（Python、Node、Java） |
| 成本 | 免费 | 按镜像收费 |

**Tag Immutability**（常考）：启用后同一个 tag 不能被覆盖，防止"v1.0 实际指向不同 image"导致部署混乱，强制版本化（用 Git SHA 或 semver）。

**Pull-Through Cache**（简略）：把外部 registry（Docker Hub、quay.io、GitHub Container Registry 等）的镜像缓存到 ECR，减少对外部 rate limit 的依赖，拉镜像更快，统一审计。

---

## 二、ECS 核心概念 ⭐⭐⭐

**ECS = Elastic Container Service**，AWS 自家的容器编排服务。

**核心层次结构**：

```
Cluster(集群) = 计算资源池(Fargate 或 EC2)
  └── Service(服务) = 维持 N 个 Task 持续运行的"控制器" + LB 集成 + Auto Scaling
        └── Task 1, Task 2, Task 3...(运行中的容器组)

Task Definition(任务定义,蓝图) = 描述"一个 Task 应该长什么样"
```

| 概念 | 类比 |
|---|---|
| **Cluster** | EC2 fleet / Kubernetes cluster |
| **Task Definition** | Docker compose 文件 / K8s Pod spec |
| **Task** | 运行中的容器组（一个 Pod 实例） |
| **Service** | K8s Deployment / Replica Set |
| **Container Instance**（EC2 only） | 装了 ECS Agent 的 EC2 |

---

## 三、Launch Types — Fargate vs EC2 ⭐⭐⭐

|  | **Fargate** | **EC2** |
|---|---|---|
| 谁管基础设施 | **AWS**（serverless） | **你**（管 EC2 fleet） |
| 计费 | 按 task 用的 CPU/memory 秒计 | 按 EC2 实例小时 |
| 隔离强度 | **每个 task 独立 micro-VM**（强） | tasks 共享 host kernel（弱） |
| 最大资源 | 较大上限（近年持续提升） | 受 instance type 限制 |
| GPU 支持 | ❌ | ✅（p/g 系列） |
| Spot 价格 | ✅ Fargate Spot（约 70% off） | ✅ EC2 Spot |
| 网络模式 | **仅 awsvpc** | awsvpc/bridge/host/none |
| 典型场景 | 微服务、变量负载、不想运维 | 大规模稳定负载、GPU、特殊配置 |
| 价格（单 task） | 较贵 | 单 task 便宜（可密集打包） |

**选择决策**：选 Fargate——不想管 EC2、突发/变量负载、微服务/API 后端、批处理、想用 Fargate Spot 省钱；选 EC2——大规模稳定负载（可用 RI/SP 拿大折扣）、需要 GPU、需要密集打包（单 task 成本远低于 Fargate）、需要特殊 instance type、需要 privileged container 或 host networking。

**ECS Managed Instances**（较新功能）：AWS 管 EC2 但提供接近 Fargate 般的运维体验 + EC2 的灵活性，混合方案。

### Capacity Providers（现代推荐）

老方式：在 task definition 指定 `requiresCompatibilities: ["FARGATE"]` 或 `["EC2"]`（legacy）。

新方式：**ECS cluster 关联多个 capacity provider，task 启动时按策略选**。支持的 capacity providers：FARGATE、FARGATE_SPOT、EC2 Auto Scaling Group capacity provider、ECS Managed Instances。

Capacity provider strategy 例子："70% Fargate Spot（便宜，可中断）+ 30% Fargate（保底，不中断）"——ECS 自动按比例 + 自动伸缩底层资源，一个 cluster 可混合用。

---

## 四、Task Definition ⭐⭐⭐

**= 描述 task 应该是什么样的 JSON 模板**

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

**Task Definition Revisions**：同一个 `family`，每次更新创建新 revision（`my-web-app:1`、`my-web-app:2`...）；revision 是**不可变**，不能改，只能创建新 revision；Service 引用特定 revision，部署新版本 = 让 service 切到新 revision。

**CPU / Memory**（Fargate 严格限定组合，具体范围以官方文档为准）：常见档位从 0.25 vCPU/0.5-2 GB 到 16 vCPU/32-120 GB。Fargate 支持的最大资源近年持续扩展，考试题库数字可能滞后，实际以官方最新文档为准。

### Network Mode ⭐⭐

| Mode | 工作方式 | Fargate | EC2 |
|---|---|---|---|
| **awsvpc** | **每个 task 一个独立 ENI 和私有 IP** | ✅（强制） | ✅ |
| **bridge** | Docker 默认 bridge，task 共享 host IP + 端口映射 | ❌ | ✅ |
| **host** | 直接用 host network（没隔离） | ❌ | ✅ |
| **none** | 没有外部网络 | ❌ | ✅ |

**awsvpc 重要细节**：每个 task 自己的 ENI（算 ENI 限额）；每个 task 自己的 SG；每个 task 自己的 IP（消耗 subnet 的 IP）；⚠️ 大集群要注意 subnet 大小，否则 IP 不够。

**bridge mode 经典用法——Dynamic Port Mapping with ALB**：容器内固定 80 端口；Host 上动态分配端口；ALB target group 自动跟踪每个 task 的动态端口；一台 EC2 上可跑多个相同容器（密集打包）。

**Task Placement Strategy**（任务放置策略）：`binpack`（尽量占满少数 instance，最小化实例数）；`random`（随机放置）；`spread`（按指定属性均匀分布，如按 instanceId 或 host）。

---

## 五、IAM in ECS ⭐⭐⭐

ECS 有 3 种 IAM Role，经常考它们的区别：

```
EC2 Container Instance(EC2 launch type only)
  ECS Container Instance Role → ECS Agent 用,注册 cluster、发心跳
                ↓ 运行
Task(Fargate or EC2)
  Task Execution Role → ECS Agent 用:拉 ECR 镜像、发 logs 到 CloudWatch、从 Secrets Manager/SSM 拿 secrets
  Task Role → 容器内应用代码用:调 AWS API(S3/DynamoDB/SNS/SQS 等业务权限)
```

### 关键对比 ⭐⭐⭐

|  | **Execution Role** | **Task Role** |
|---|---|---|
| 谁用 | **ECS Agent**（基础设施层） | **你的应用代码**（业务层） |
| 何时用 | Task 启动前（拉镜像、设置 logs） | Task 运行中（应用调 AWS API） |
| 典型权限 | `ecr:GetAuthorizationToken`、`ecr:BatchGetImage`、`logs:CreateLogStream`、`secretsmanager:GetSecretValue` | `s3:GetObject`、`dynamodb:PutItem` 等业务权限 |
| 必需吗 | 基本必需（没有就拉不了镜像） | 可选（但应用要调 AWS 就要） |

**记忆口诀**："Execution = 基础设施前置工作；Task = 业务运行时"。

**具体场景**：从 ECR 拉镜像 → Execution Role；发 logs 到 CloudWatch → Execution Role；从 Secrets Manager 拿密码注入容器环境变量 → Execution Role；应用代码读 S3/写 DynamoDB/调 Lambda/发 SNS → **Task Role**。

⚠️ **常见 bug**：把 S3 权限加到 execution role 而不是 task role → 应用调 S3 时失败。

**Secrets 注入容器**：方式 1（推荐）——Task Definition `secrets` 字段，Execution Role 需要 `secretsmanager:GetSecretValue`，容器启动时 ECS Agent 拉 secret 注入环境变量；方式 2——应用代码自己调 Secrets Manager（用 Task Role），更安全（secret 不在 env var 中）但需要应用代码改造。

---

## 六、ECS Service ⭐⭐

**Service = 维持 N 个 task 持续运行的"管控器"**

**特性**：维持期望数量的 task；Task 死了自动重启；集成 Load Balancer；Auto Scaling；滚动部署/蓝绿部署；Service Discovery。

**Service 调度策略**：`REPLICA`（默认，维持指定数量的 task）；`DAEMON`（在每个 container instance 上跑 1 个 task，适合 logging agent、监控等）。

### Deployment Type ⭐⭐

**1. Rolling Update**（默认）：用 minimumHealthyPercent + maximumPercent 控制滚动，简单、内置。

**2. Blue/Green**（需要 CodeDeploy）：全量新版本部署后切流量，支持 canary/linear/all-at-once 流量切换，失败可一键回滚到 blue，适合关键应用。

**3. External**：三方部署工具，自己控制 task 切换。

⚠️ **Blue/Green 必须用 CodeDeploy**，Rolling Update 是 ECS 自带的。

---

## 七、Load Balancing 集成 ⭐⭐

### ALB（最常用）

|  | Fargate (awsvpc) | EC2 (bridge mode) |
|---|---|---|
| Target type | **IP**（每 task 有 ENI/IP） | **instance** 或 **IP** |
| Port mapping | 固定端口 | **Dynamic port mapping**（常用，密集打包） |

**Dynamic Port Mapping**（EC2 bridge mode）：Task definition 设动态 host port；ECS 自动选可用端口；ALB target group 自动追踪；同一台 EC2 跑多个相同容器（每个不同 host port）。

**NLB 适合场景**：极致性能（L4）；静态 IP；长连接/gRPC。

**Service Discovery**：**AWS Cloud Map**（原 Route 53 Auto Naming，为每个 task 注册 DNS 名，task 死了自动注销）；**ECS Service Connect**（较新，推荐，内置 service mesh，带 metrics、retries、Envoy 代理）。

---

## 八、Auto Scaling ⭐⭐

### Service Auto Scaling

**= 自动调整 service 的 task 数量**，底层用 Application Auto Scaling。

**3 种策略**：**Target Tracking**（维持指标在目标值，如 CPU 50%，默认推荐）；**Step Scaling**（阶梯式扩缩，流量曲线明显时用）；**Scheduled Scaling**（按时间，可预测周期）。

### Cluster Auto Scaling（EC2 launch type）

**= 自动伸缩 EC2 capacity provider 的 Auto Scaling Group**。机制：ECS 监控"未启动的 task"（capacity reservation gap）→ 计算需要多少 EC2 才能放下这些 task → 通知 Auto Scaling Group 调整 desired capacity。

Fargate **不需要 cluster auto scaling**（AWS 管底层）。

---

## 九、Storage / Volumes ⭐

ECS 支持几种 volume：

| Volume 类型 | Fargate | EC2 | 持久化 | 适合 |
|---|---|---|---|---|
| **Bind mount** | ✅（临时，task 生命周期） | ✅ | ❌ task 结束就没 | 临时数据、跨容器共享 |
| **Docker volume** | ❌ | ✅ | ⚠️ 看驱动 | 老用法 |
| **EFS volume** | ✅ | ✅ | **✅ 持久** | **Fargate 持久数据首选** |
| **FSx for Windows / Lustre** | ✅ | ✅ | ✅ | Windows/HPC |

⭐ **关键**：**Fargate 持久存储必须用 EFS**（没 EBS！），因为 Fargate 没法 attach EBS。

**Fargate Ephemeral Storage**：每个 Fargate task 默认约 20 GB 临时盘，可配置扩展（较新功能范围以官方文档为准），task 结束就销毁。

---

## 十、ECS Anywhere（简略）

**= 在 on-premises/多云服务器上跑 ECS task**。机制：在 on-prem 服务器装 SSM Agent + ECS Agent，注册为 "EXTERNAL" launch type，通过 SSM 管控。用途：把 AWS 控制平面带到本地数据中心、边缘计算、混合云迁移。

---

## 十一、EKS vs ECS（对比，DVA 不重点）

|  | **ECS** | **EKS** |
|---|---|---|
| 编排引擎 | AWS 自研 | **Kubernetes（开源）** |
| 学习曲线 | 简单 | 陡 |
| 跨云 | ❌ | ✅（K8s 哪都能跑） |
| 控制平面 | 免费 | 按小时收费 |
| 适合 | AWS 原生 + 想省事 | 已有 K8s 经验/跨云 |

DVA 主要考 ECS，EKS 知道存在即可。
