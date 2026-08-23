# Module 16：部署平台 + 负载均衡（Elastic Beanstalk + ELB/ALB/NLB）

> **覆盖日期**：5/21
>
> English version: [`en/16-deployment-lb.md`](../en/16-deployment-lb.md)

---

# 一、Elastic Beanstalk ⭐⭐

## Elastic Beanstalk 是什么 ⭐⭐⭐

**= AWS 的 PaaS（Platform as a Service）**

**核心价值**：✅ "上传代码就行，其他我管"（EC2+ASG+ALB+健康监控+滚动部署）；✅ 完全免费（只付底层资源 EC2/ALB/RDS）；✅ 完全访问底层资源（可以 SSH 到 EC2）；✅ 适合传统 web 应用/API。

**三大核心组件**：**Application**（逻辑容器，项目名）；**Environment**（实际跑的 AWS 资源，每个 env 独立 ASG/ALB）；**Application Version**（打包的代码，zip/war）。

## 支持的 Platforms

Java（SE、Tomcat）、.NET（Windows Server、Linux）、Node.js、PHP、Python、Ruby、Go（均 Linux）、**Docker**（单容器/多容器，Linux）。

⚠️ Custom Platform 已经 deprecated，用 Docker 替代。

## Environment Tier — Web vs Worker ⭐⭐⭐

**Web Server Environment**（Web tier）：处理 HTTP 请求。`Internet → ALB → ASG (EC2 instances running your app)`。适合 web 应用、REST API、网站。

**Worker Environment**（Worker tier）：处理后台/异步任务。`SQS Queue → Beanstalk Worker (EC2 拉 SQS) → 处理 → 删除消息`。

工作机制：Beanstalk 自动创建 SQS queue；每个 EC2 上跑一个 daemon 轮询 SQS；daemon POST message body 到本地 HTTP（默认 `localhost:80`）；应用接到 POST → 处理 → 返回 200 → daemon 删除 SQS 消息。

**关键**：Worker tier 没有 ALB；适合邮件发送、image processing、PDF 生成等耗时任务。

**cron.yaml**（Worker 定时任务）⭐：Worker tier 支持定时触发（类似 cron）：

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

daemon 按 schedule 自动 POST 到指定 URL，不需要 EventBridge。

⭐ 判断：看到"Beanstalk+定时后台任务" → Worker tier + cron.yaml。

## Deployment Policies ⭐⭐⭐

**= EB 部署新版本的 5 种策略**

| Policy | 行为 | 停机 | 成本 | 失败影响 |
|---|---|---|---|---|
| **All at once** | 一次性更新所有实例 | **有（短暂）** | 最低 | 全部回滚 |
| **Rolling** | 分批更新现有实例 | 无（容量减少） | 最低 | 已更新批回滚 |
| **Rolling with additional batch** | 先加一批新实例，再分批更新 | 无 | 中 | 已更新批回滚 |
| **Immutable** | 启动全新 ASG，验证后切换，删旧 | 无 | 高（临时双倍） | 直接删新 ASG，安全 |
| **Traffic Splitting** | Immutable + 流量按百分比切（canary） | 无 | 高 | 自动切回旧，最安全 |

**选择决策**：开发/测试 → All at once；生产、容忍容量减少 → Rolling；生产、要保持容量 → Rolling with additional batch；生产、要最稳 → Immutable；生产、Canary 测试 → Traffic Splitting（必须用 ALB）。

## Blue/Green Deployment ⭐

**EB 没有原生"Blue/Green"部署 policy**，但可用 **swap URLs** 实现：创建第二个 environment（green）→ 部署新版本 → 充分测试 green → 在 EB console 点"Swap Environment URLs"（CNAME 互换）→ 失败可再 swap 一次，秒级回滚。

vs Traffic Splitting：Blue/Green 是两套完整 env、人工管理切换；Traffic Splitting 是单 env 内渐进式发布。

## .ebextensions ⭐⭐

**= 自定义 EB 环境的配置文件**。位置：应用源代码根目录 `.ebextensions/*.config`（YAML 或 JSON）。

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

**commands vs container_commands** ⭐：

|  | commands | container_commands |
|---|---|---|
| 何时跑 | 应用代码部署**之前** | 应用代码已部署但还**没启动**前 |
| 能访问应用代码 | ❌ | ✅ |
| 用途 | 装 OS 包、配置系统 | DB migration、build 步骤 |
| leader_only | ❌ | ✅ |

## RDS in Beanstalk ⭐⭐

⚠️ **问题**：EB Environment terminated → RDS 也被删除！

**推荐方案**（必考）：RDS 独立于 EB，通过环境变量连接——独立创建 RDS → EB 环境变量（DB_HOST/DB_USER/DB_PASSWORD）→ 修改 Security Group 允许 EB EC2 访问 RDS。

**已有 EB 内 RDS 怎么"解耦"**：RDS 设置 `DeletionPolicy: Retain` → 创建 RDS snapshot → 用 snapshot 创建独立 RDS → 改 EB 环境变量指向新 RDS → 销毁旧 EB env。

## EB 其他 ⭐

**HTTPS**：在 EB ALB 监听器加 HTTPS listener，用 ACM 证书，强制 HTTPS redirect HTTP → HTTPS。

**Health Check Reporting**：Basic（CloudWatch metrics，5 分钟粒度，免费）；Enhanced（每个 instance+整体环境健康，1 分钟粒度，收费，7 种健康状态）。

**EB CLI 重要命令**：`eb init`（初始化）、`eb create`（创建环境）、`eb deploy`（部署）、`eb status`、`eb health`、`eb logs`、`eb ssh`、`eb swap`（交换环境 URL）、`eb terminate`。

**限制**：Applications per region 约 75；Application versions per app 约 1000；部署文件最大 **512 MB**。

---

# 二、Elastic Load Balancer ⭐⭐

## ELB 类型对比 ⭐⭐⭐

| 类型 | 层级 | 适合 |
|---|---|---|
| **CLB**（Classic，老式） | L4+L7 | Legacy，已在弃用进程中 |
| **ALB**（Application LB） | **L7（HTTP）** | Web 应用、API、microservices |
| **NLB**（Network LB） | **L4（TCP/UDP/TLS）** | 极致性能、静态 IP、长连接 |
| **GWLB**（Gateway LB） | **L3（IP）** | 第三方网络设备（防火墙、IDS） |

⭐ DVA 主要考 ALB 和 NLB，GWLB 知道存在即可。

## ALB ⭐⭐⭐

**= L7 HTTP/HTTPS 负载均衡器**

**Multi-target routing（路由规则）**：**Path-based**（`/api/*` → backend A）；**Host-based**（`api.example.com` → service A）；**Header-based**（`X-Version: v2` → new backend）；**Query string**；**Source IP**（CIDR 匹配）。

**Target Types**：`instance`（EC2 instance ID）；`IP`（任意 IP 包括 on-prem，**ECS Fargate awsvpc mode 必须用 IP**）；`lambda`（ALB 直接调 Lambda）。

**SNI (Server Name Indication)** ⭐：一个 ALB 可绑定多个 SSL 证书（给不同域名），客户端在 TLS 握手时带 SNI header，ALB 选对应证书。

**Listener Rules Action**：`forward`（转发到 target group）；`redirect`（URL 重写，如 HTTP → HTTPS）；`fixed-response`（返回固定 HTML/JSON，维护页面）；`authenticate-cognito`（Cognito 登录集成）；`authenticate-oidc`（任意 OIDC 提供商）。

**Sticky Sessions**：同一个 client 的请求总路由到同一个 target。LB-generated cookie（ALB 生成 `AWSALB` cookie）或 App-based cookie（应用生成，ALB 用这个 cookie 做 sticky）。

**Cross-Zone Load Balancing**：ALB 默认启用，免费；NLB 默认禁用，跨 AZ 流量收费。

**WebSocket / gRPC**：ALB 原生支持 WebSocket 和 HTTP/2/gRPC（NLB 只做 TCP，协议无感知）。

**ALB + Cognito 认证**：ALB listener rule 配 `authenticate-cognito` action，直接做用户登录认证，不需要 API Gateway。

**Dynamic Port Mapping**（ECS bridge mode）：同一个 EC2 上跑多个相同容器，每个容器随机端口，ALB target group 用 instance+动态端口。

## NLB ⭐⭐

**= L4 TCP/UDP/TLS 负载均衡器**

**特点**：✅ 极致性能（数百万 RPS、数十微秒延迟）；✅ 静态 IP（每个 AZ 一个 EIP）；✅ 保留 client 源 IP（不改源 IP，后端看到真实 client IP）；✅ 跨 AZ 均衡默认禁用（需手动开+收费）；❌ 不支持 HTTP level 功能（path routing、header routing）。

**Target Types**：instance/IP/ALB（NLB 后面接 ALB）。

## ELB Connection Draining / Deregistration Delay

CLB 叫 Connection Draining；ALB/NLB 叫 **Deregistration Delay**（默认 **300 秒**，范围 0-3600）。target 从 ELB 撤下时，继续接现有连接，新连接不路由。

新 target 加入时对应的机制是 **Slow Start**——让新 target 的流量逐步预热上升，而不是立刻满负荷接收流量。
