# Module 15：CI/CD 与构建工具（Code* 五件套）

> **覆盖日期**：5/19、5/20
>
> English version: [`en/15-cicd.md`](../en/15-cicd.md)

---

## AWS Code* 系列概览 ⭐⭐

| 服务 | 角色 |
|---|---|
| **CodeCommit** | Git 源代码托管（类似 GitHub） |
| **CodeBuild** | 构建/测试服务（类似 Jenkins build） |
| **CodeDeploy** | 应用部署（EC2/Lambda/ECS） |
| **CodePipeline** | CI/CD 编排（把上面几个串起来） |
| **CodeArtifact** | 包仓库（npm/pip/Maven 等） |

⭐ 高频组合：CodeCommit → CodeBuild → CodeDeploy（via CodePipeline）

---

## 一、CodeCommit ⭐⭐

**= AWS 托管的 Git 仓库服务**

**关键特性**：私有 Git 仓库（标准 Git 协议）；IAM 集成；at-rest 加密（KMS）；in-transit 加密（HTTPS/SSH）；跨 region replication；CloudTrail 审计；无大小限制（单 file 最大 6 MB）。

> ⚠️ **状态提醒**：AWS 曾在 2024 年宣布对新用户停止 CodeCommit，后来又逆转决定重新开放。当前可用，继续支持（但以维护为主，新功能较少）。建议在实际使用前查阅 AWS 官方最新公告确认状态。

**认证方式** ⭐⭐：**HTTPS + Git Credentials**（IAM user 在控制台生成专用 Git username/password）；**HTTPS + git-remote-codecommit (GRC)**（用 AWS CLI 凭证，适合 CI）；**SSH Keys**（上传 SSH public key 到 IAM user）。

⭐ 不能用普通 AWS access key 直接调 git clone。每个 IAM user 最多 2 套 Git credentials。

**通知机制**：Notifications（推荐，AWS CodeStar Notifications，EventBridge+SNS，通知到 SNS 或 Chatbot/Slack）；Triggers（老式，直接配 SNS/Lambda，每个 repo 最多 10 个 triggers）。

**Cross-Account Access**：Account A 创建 IAM Role → Account B 用 `aws sts assume-role` 拿临时凭证 → 用 `git-remote-codecommit` 配合临时凭证 clone。

**Pull Request 工作流**：Approval Rule Template——跨 repo 复用的规则模板，指定必需 approver 数量。

**Migration**：`git clone --mirror <source>` + `git push --mirror <CodeCommit>` → 保留全部历史、branches、tags。

---

## 二、CodeBuild ⭐⭐⭐

**= AWS 托管的"构建/测试"服务**（无服务器，按秒计费，并发 build，支持多种 source）

### buildspec.yml ⭐⭐⭐

**= CodeBuild 的"剧本"**，定义 build 步骤

```yaml
version: 0.2

env:
  variables:
    NODE_ENV: production
  parameter-store:
    DB_HOST: /myapp/prod/db-host
  secrets-manager:
    DB_PASSWORD: prod/db:password
  exported-variables:
    - BUILD_VERSION

phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - npm ci
  pre_build:
    commands:
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URL
  build:
    commands:
      - npm run build
      - docker build -t myapp .
  post_build:
    commands:
      - docker push $ECR_URL/myapp:latest

artifacts:
  files:
    - 'dist/**/*'
  name: my-artifact-$(date +%Y-%m-%d)

cache:
  paths:
    - 'node_modules/**/*'
```

### 4 个核心 Phase ⭐⭐⭐

| Phase | 用途 |
|---|---|
| **install** | 装运行时、安装系统依赖 |
| **pre_build** | 测试前准备（登录 ECR、生成 config） |
| **build** | 主构建（编译、打包、build image） |
| **post_build** | 构建后任务（推送 image、通知 SNS） |

⚠️ Phase 名字不可改——`install/pre_build/build/post_build` 是固定名字。每个 phase 可加 `finally:` block，无论成功失败都会跑。

**Environment Variables**：4 种来源——`variables`（明文）、`parameter-store`、`secrets-manager`、`exported-variables`（传给下游）。

**内置变量**：`CODEBUILD_BUILD_ID`（Build 唯一 ID）、`CODEBUILD_RESOLVED_SOURCE_VERSION`（实际 commit SHA）、`CODEBUILD_BUILD_NUMBER`（增量 build 编号）、`AWS_DEFAULT_REGION`、`AWS_ACCOUNT_ID`。

### Compute Types ⭐

从小到大有多个档位（`BUILD_GENERAL1_SMALL/MEDIUM/LARGE/2XLARGE`），vCPU 和内存逐级增加，具体规格以官方文档为准。

⭐ **Lambda-backed CodeBuild**（较新功能）：毫秒级启动，不支持 Docker/privileged mode。

### Cache ⭐⭐

**S3 Cache**（存 S3 bucket，适合 node_modules、.m2 等）；**Local Cache**（存本地 build host，更快但仅同 host 复用）。Local cache 子类：`LOCAL_DOCKER_LAYER_CACHE`、`LOCAL_SOURCE_CACHE`、`LOCAL_CUSTOM_CACHE`。

### Build in VPC ⭐⭐

CodeBuild 默认在 AWS 管的网络运行，无法访问 VPC 内资源。配置：CodeBuild Project → VPC + Subnets + Security Group；必须配 NAT Gateway 或 VPC Endpoint；IAM service role 需要 EC2 网络权限（`ec2:CreateNetworkInterface`）。

**Build Timeout / Logs / IAM**：默认 1 小时，最长 8 小时；日志写 CloudWatch Logs（实时）/S3（归档）；CodeBuild Service Role 需要写 Logs+上传 artifacts+拉 ECR+读 SSM/Secrets 的权限。

**安全**：❌ 不要把密码/token 写在 buildspec.yml 里；✅ 用 `parameter-store`/`secrets-manager` 在 buildspec env 引用。build 失败通知：EventBridge rule（BuildState changed）→ SNS/Lambda → Slack。

---

## 三、CodeDeploy ⭐⭐⭐

**= AWS 托管的应用部署服务**（自动化部署 EC2/Lambda/ECS，零停机，失败自动回滚）

### 三种 Compute Platform ⭐⭐⭐

| Platform | 部署到哪 | 关键特性 |
|---|---|---|
| **EC2/On-premises** | EC2/自管服务器 | 必须装 **CodeDeploy Agent** |
| **AWS Lambda** | Lambda alias 流量切换 | 内置 traffic shifting |
| **Amazon ECS** | ECS Service 蓝绿部署 | 与 ALB 配合 |

### EC2/On-Prem ⭐⭐

**CodeDeploy Agent** = 运行在 EC2/on-prem 上的 daemon：拉取 deployment 任务/执行 lifecycle hooks/报告状态。

**前提条件**：IAM Instance Profile（`AmazonEC2RoleforAWSCodeDeploy`）+ Agent 已安装 + 网络可达 + 实例有 tag。

**部署类型**：**In-place**（在现有 instance 上更新，简单便宜，❌ 不能轻松回滚）；**Blue/Green**（创建全新 instance，验证后切换，零停机+一键回滚，❌ 成本翻倍）。

⭐ EC2/On-prem 支持两种；**Lambda 和 ECS 只支持 Blue/Green**。

### appspec.yml ⭐⭐⭐

**= CodeDeploy 的"剧本"**，描述部署步骤。⚠️ 路径：EC2/On-prem 在源代码根目录；Lambda/ECS 作为 deployment 输入参数。

**EC2/On-Prem**：

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html
hooks:
  ApplicationStop:
    - location: scripts/stop_server.sh
  BeforeInstall:
    - location: scripts/install_deps.sh
  AfterInstall:
    - location: scripts/configure.sh
  ApplicationStart:
    - location: scripts/start_server.sh
  ValidateService:
    - location: scripts/validate.sh
```

**Lambda appspec.yaml**：

```yaml
version: 0.0
Resources:
  - myLambdaFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: "my-function"
        Alias: "live"
        CurrentVersion: "1"
        TargetVersion: "2"
Hooks:
  - BeforeAllowTraffic: validateBeforeLambda
  - AfterAllowTraffic: validateAfterLambda
```

**ECS appspec.yaml**：

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:...:task-definition/myapp:5"
        LoadBalancerInfo:
          ContainerName: "myapp"
          ContainerPort: 3000
Hooks:
  - BeforeInstall: "arn:aws:lambda:...:function:before-install"
  - AfterInstall: "arn:aws:lambda:...:function:after-install"
  - AfterAllowTestTraffic: "arn:aws:lambda:...:function:test-traffic"
  - BeforeAllowTraffic: "arn:aws:lambda:...:function:before-traffic"
  - AfterAllowTraffic: "arn:aws:lambda:...:function:after-traffic"
```

ECS deployment 的 resources 里，只有 `TaskDefinition`、`ContainerName`、`ContainerPort` 是 CodeDeploy 必需的属性。

### Lifecycle Hooks ⭐⭐⭐

**= 部署不同阶段触发的脚本/Lambda**

**EC2/On-Prem In-place**：ApplicationStop → DownloadBundle → BeforeInstall → Install → AfterInstall → ApplicationStart → ValidateService。DownloadBundle、Install 是 CodeDeploy 内部执行的，不能写脚本。

**Lambda Lifecycle Hooks**（只有 2 个）：`BeforeAllowTraffic`（切流量前，可做 smoke test）、`AfterAllowTraffic`（切流量后）。⭐ Lambda hook 必须是 Lambda function。

**ECS Lifecycle Hooks**（5 个）：BeforeInstall/AfterInstall/AfterAllowTestTraffic/BeforeAllowTraffic/AfterAllowTraffic。

**Hook 返回结果**：EC2/On-Prem 用 shell exit code（0 = 成功）；Lambda/ECS 用 Lambda 调 `PutLifecycleEventHookExecutionStatus` API。⚠️ Hook 有 1 小时超时。

### Deployment Configurations ⭐⭐⭐

**= 控制流量怎么切换**

**EC2/On-Prem**：`AllAtOnce`/`HalfAtATime`/`OneAtATime`/Custom。

**Lambda**（必背）：`LambdaAllAtOnce`（一次切 100%）；`LambdaCanary10Percent5/10/15/30Minutes`（先切 10%，等 N 分钟，再切 90%）；`LambdaLinear10PercentEvery1/2/3/10Minutes`（每 N 分钟切 10%）。

ECS 完全一样，前缀变 `CodeDeployDefault.ECS*`。

**Automatic Rollback**：触发条件——Deployment 失败或 CloudWatch Alarm 触发。Rollback = 创建新部署（用之前 known-good revision），不是原地恢复。

### CodeDeploy IAM

**EC2 Instance Profile**（EC2 instance 用，`AmazonEC2RoleforAWSCodeDeploy`）；**CodeDeploy Service Role**（CodeDeploy 服务用，`AWSCodeDeployRole`）；**Lambda Hook Execution Role**（Lambda hook 用，需要 `codedeploy:PutLifecycleEventHookExecutionStatus`）。

---

## 四、CodePipeline ⭐⭐⭐

**= AWS 托管的 CI/CD 编排服务**（可视化 pipeline，自动触发，集成所有 Code* + 第三方）

**层次结构**：Pipeline → Stage → Action。**Pipeline**（整个 CI/CD workflow）；**Stage**（逻辑阶段：Source/Build/Test/Deploy）；**Action**（Stage 内一个具体操作，可串行或并行）；**Artifact**（Stage 之间传递的文件，打包到 S3）；**Transition**（Stage 间的"传送门"，可禁用）。

**关键限制**：Stages per pipeline——最少 2，最多 10；Manual approval timeout——**7 天**；第一阶段必须是 Source。

**6 种 Action Categories** ⭐⭐：**Source**（CodeCommit、GitHub、Bitbucket、GitLab、S3、ECR）；**Build**（CodeBuild、Jenkins）；**Test**（CodeBuild、Device Farm）；**Deploy**（CodeDeploy、CloudFormation、ECS、Elastic Beanstalk、AppConfig、S3）；**Approval**（Manual Approval）；**Invoke**（Lambda、Step Functions）。

⭐ Stage 之间永远串行；Stage 内 actions 同 `runOrder` → 并行。

**Artifact 流转** ⭐⭐：Stage 之间通过 S3 传递文件（Source ZIP → Build Output → Deploy Input）。每个 region 一个 artifact bucket；默认加密 aws/s3 KMS；跨 region pipeline 必须每个 region 都有 artifact bucket。

**Manual Approval** ⭐⭐：暂停 pipeline 等人审批。**7 天没批 → 视为 Rejected，pipeline 失败**。审批者必须有 `codepipeline:PutApprovalResult` 权限；可配 SNS 通知审批者。

**Cross-Account / Cross-Region** ⭐⭐：Cross-Account 必须用 customer managed KMS key + Pipeline Service Role + 跨账户 IAM Role；Cross-Region 每个 region 都要 artifact bucket（CodePipeline 自动复制 artifacts）。

**Triggers**：现代推荐 **CodeStar Connections**（替代 GitHub OAuth token）。

---

## 五、CodeArtifact ⭐⭐

**= AWS 托管的"包仓库"服务**

**支持格式**：npm、pip、Maven、Gradle、NuGet、twine、Cargo、Swift、Ruby Gems、Generic。

### 层次结构 ⭐⭐⭐

**Domain → Repository → Package → Version**

**Domain**：顶层容器，所有包资产物理存储一次（节省空间），整个 domain 一把 KMS key。**Repository**：逻辑包仓库，polyglot。

**关键限制**：Repositories per domain 上限约 1,000；认证 token TTL **12 小时**。

**Upstream Repositories** ⭐⭐：一个 repo 可链接到另一个 repo 作为 upstream：

```
Local Repo → Upstream Repo → External Connection(public:npmjs)
```

即使 npmjs unpublish 也有本地副本（供应链安全）。External Connection 支持：`public:npmjs`、`public:pypi`、`public:maven-central`、`public:nuget-org` 等。

**认证**：

```bash
aws codeartifact login --tool npm --domain my-org --repository my-repo
npm install axios
```

Token 12 小时过期，CI 流水线每次构建前要刷新。

**Cross-Account 访问**：Account A 设 domain policy + repository policy + KMS key policy 允许 Account B。
