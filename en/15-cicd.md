# Module 15: CI/CD & Build Tools (Code* Suite)

> **Covered**: May 19, 20
>
> 中文版本：[`zh/15-cicd.md`](../zh/15-cicd.md)

---

## AWS Code* Family Overview ⭐⭐

| Service | Role |
|---|---|
| **CodeCommit** | Git source hosting (like GitHub) |
| **CodeBuild** | Build/test service (like Jenkins build) |
| **CodeDeploy** | Application deployment (EC2/Lambda/ECS) |
| **CodePipeline** | CI/CD orchestration (chains the others together) |
| **CodeArtifact** | Package repository (npm/pip/Maven, etc.) |

⭐ A common combination: CodeCommit → CodeBuild → CodeDeploy (orchestrated via CodePipeline).

---

## 1. CodeCommit ⭐⭐

**= AWS's managed Git repository service.**

**Key traits**: private Git repos (standard Git protocol); IAM integration; at-rest encryption (KMS); in-transit encryption (HTTPS/SSH); cross-region replication; CloudTrail auditing; no meaningful size limit (single file caps at 6 MB).

> ⚠️ **Status note**: AWS announced in 2024 that CodeCommit would stop onboarding new customers, then later reversed that decision and reopened it. It's currently available and continues to be supported (though largely in maintenance mode, with few new features). Check AWS's current announcements before relying on it for a new project.

**Authentication methods** ⭐⭐: **HTTPS + Git Credentials** (an IAM user generates a dedicated Git username/password in the console); **HTTPS + git-remote-codecommit (GRC)** (uses AWS CLI credentials, good for CI); **SSH Keys** (uploading an SSH public key to an IAM user).

⭐ Plain AWS access keys cannot be used to `git clone` directly. Each IAM user is limited to 2 sets of Git credentials.

**Notifications**: **Notifications** (recommended — AWS CodeStar Notifications, EventBridge+SNS, notifying SNS or Chatbot/Slack); **Triggers** (legacy — configured directly with SNS/Lambda, up to 10 triggers per repo).

**Cross-Account Access**: Account A creates an IAM role → Account B calls `aws sts assume-role` for temporary credentials → clones using `git-remote-codecommit` with those credentials.

**Pull request workflow**: an Approval Rule Template — a reusable rule template across repos specifying the required number of approvers.

**Migration**: `git clone --mirror <source>` + `git push --mirror <CodeCommit>` — preserves all history, branches, and tags.

---

## 2. CodeBuild ⭐⭐⭐

**= AWS's managed build/test service** (serverless, billed per second, concurrent builds, supports many source types).

### buildspec.yml ⭐⭐⭐

**= CodeBuild's "script"**, defining the build steps.

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

### The Four Core Phases ⭐⭐⭐

| Phase | Purpose |
|---|---|
| **install** | Installing the runtime, system dependencies |
| **pre_build** | Preparation before the build (logging into ECR, generating config) |
| **build** | The main build (compiling, packaging, building an image) |
| **post_build** | Post-build tasks (pushing an image, notifying SNS) |

⚠️ Phase names can't be changed — `install/pre_build/build/post_build` are fixed. Each phase can add a `finally:` block that runs regardless of success or failure.

**Environment variables**: four sources — `variables` (plaintext), `parameter-store`, `secrets-manager`, `exported-variables` (passed to downstream stages).

**Built-in variables**: `CODEBUILD_BUILD_ID` (the build's unique ID), `CODEBUILD_RESOLVED_SOURCE_VERSION` (the actual commit SHA), `CODEBUILD_BUILD_NUMBER` (an incrementing build number), `AWS_DEFAULT_REGION`, `AWS_ACCOUNT_ID`.

### Compute Types ⭐

Several tiers exist from small to large (`BUILD_GENERAL1_SMALL/MEDIUM/LARGE/2XLARGE`), with vCPU and memory scaling up at each step — check current docs for exact specs.

⭐ **Lambda-backed CodeBuild** (a newer feature): millisecond startup, but doesn't support Docker/privileged mode.

### Cache ⭐⭐

**S3 Cache** (stored in an S3 bucket, good for node_modules, .m2, etc.); **Local Cache** (stored on the build host, faster but only reusable on the same host). Local cache subtypes: `LOCAL_DOCKER_LAYER_CACHE`, `LOCAL_SOURCE_CACHE`, `LOCAL_CUSTOM_CACHE`.

### Build in a VPC ⭐⭐

CodeBuild runs in an AWS-managed network by default and can't reach resources inside your VPC. Configuration: the CodeBuild project needs a VPC + subnets + security group; a NAT Gateway or VPC Endpoint is required; the IAM service role needs EC2 networking permissions (`ec2:CreateNetworkInterface`).

**Build timeout / logs / IAM**: defaults to 1 hour, maxing out at 8 hours; logs go to CloudWatch Logs (real-time) and/or S3 (archival); the CodeBuild service role needs permissions to write logs, upload artifacts, pull from ECR, and read SSM/Secrets.

**Security**: ❌ never hardcode passwords/tokens in buildspec.yml; ✅ reference them via `parameter-store`/`secrets-manager` in the env section. Build-failure notifications: an EventBridge rule (BuildState changed) → SNS/Lambda → Slack.

---

## 3. CodeDeploy ⭐⭐⭐

**= AWS's managed application deployment service** (automates deployment to EC2/Lambda/ECS, zero downtime, automatic rollback on failure).

### Three Compute Platforms ⭐⭐⭐

| Platform | Deploys to | Key trait |
|---|---|---|
| **EC2/On-premises** | EC2/self-managed servers | Requires the **CodeDeploy Agent** |
| **AWS Lambda** | Lambda alias traffic shifting | Built-in traffic shifting |
| **Amazon ECS** | ECS Service blue/green deployment | Works with an ALB |

### EC2/On-Prem ⭐⭐

**The CodeDeploy Agent** = a daemon running on EC2/on-prem that pulls deployment tasks, executes lifecycle hooks, and reports status.

**Prerequisites**: an IAM instance profile (`AmazonEC2RoleforAWSCodeDeploy`), the agent installed, network reachability, and appropriate instance tags.

**Deployment types**: **In-place** (updates an existing instance directly — simple and cheap, ❌ no easy rollback); **Blue/Green** (creates a brand-new instance, cuts over after validation — zero downtime + one-click rollback, ❌ double the cost).

⭐ EC2/On-prem supports both types; **Lambda and ECS only support Blue/Green**.

### appspec.yml ⭐⭐⭐

**= CodeDeploy's "script"**, describing the deployment steps. ⚠️ location: for EC2/On-prem it lives at the source code root; for Lambda/ECS it's passed as a deployment input parameter.

**EC2/On-Prem**:

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

**Lambda appspec.yaml**:

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

**ECS appspec.yaml**:

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

For ECS deployments, only `TaskDefinition`, `ContainerName`, and `ContainerPort` are actually required by CodeDeploy in the resources section.

### Lifecycle Hooks ⭐⭐⭐

**= scripts/Lambdas triggered at different points in a deployment.**

**EC2/On-Prem In-place**: ApplicationStop → DownloadBundle → BeforeInstall → Install → AfterInstall → ApplicationStart → ValidateService. DownloadBundle and Install are executed internally by CodeDeploy — you can't script them.

**Lambda Lifecycle Hooks** (only 2): `BeforeAllowTraffic` (before shifting traffic — good for a smoke test), `AfterAllowTraffic` (after shifting). ⭐ a Lambda hook must be a Lambda function.

**ECS Lifecycle Hooks** (5): BeforeInstall/AfterInstall/AfterAllowTestTraffic/BeforeAllowTraffic/AfterAllowTraffic.

**How hooks report their result**: EC2/On-Prem uses a shell exit code (0 = success); Lambda/ECS uses a Lambda calling the `PutLifecycleEventHookExecutionStatus` API. ⚠️ hooks have a 1-hour timeout.

### Deployment Configurations ⭐⭐⭐

**= controlling how traffic shifts.**

**EC2/On-Prem**: `AllAtOnce`/`HalfAtATime`/`OneAtATime`/Custom.

**Lambda** (worth memorizing): `LambdaAllAtOnce` (100% at once); `LambdaCanary10Percent5/10/15/30Minutes` (10% first, wait N minutes, then the remaining 90%); `LambdaLinear10PercentEvery1/2/3/10Minutes` (10% every N minutes).

ECS is identical, just with a `CodeDeployDefault.ECS*` prefix.

**Automatic Rollback**: triggered by a deployment failure or a CloudWatch alarm firing. A rollback is really a *new deployment* using the previous known-good revision — not an in-place restore.

### CodeDeploy IAM

**EC2 Instance Profile** (used by the EC2 instance, `AmazonEC2RoleforAWSCodeDeploy`); **CodeDeploy Service Role** (used by the CodeDeploy service, `AWSCodeDeployRole`); **Lambda Hook Execution Role** (used by Lambda hooks, needing `codedeploy:PutLifecycleEventHookExecutionStatus`).

---

## 4. CodePipeline ⭐⭐⭐

**= AWS's managed CI/CD orchestration service** (a visual pipeline, automatic triggers, integrating every Code* service plus third parties).

**The hierarchy**: Pipeline → Stage → Action. **Pipeline** (the entire CI/CD workflow); **Stage** (a logical phase: Source/Build/Test/Deploy); **Action** (one specific operation within a stage, serial or parallel); **Artifact** (files passed between stages, packaged into S3); **Transition** (the "gate" between stages, which can be disabled).

**Key limits**: stages per pipeline — 2 to 10; manual approval timeout — **7 days**; the first stage must always be Source.

**Six Action Categories** ⭐⭐: **Source** (CodeCommit, GitHub, Bitbucket, GitLab, S3, ECR); **Build** (CodeBuild, Jenkins); **Test** (CodeBuild, Device Farm); **Deploy** (CodeDeploy, CloudFormation, ECS, Elastic Beanstalk, AppConfig, S3); **Approval** (Manual Approval); **Invoke** (Lambda, Step Functions).

⭐ Stages always run serially; actions within a stage sharing the same `runOrder` run in parallel.

**Artifact flow** ⭐⭐: files pass between stages via S3 (Source ZIP → Build Output → Deploy Input). One artifact bucket per region; encrypted with aws/s3 KMS by default; a cross-region pipeline needs an artifact bucket in every region involved.

**Manual Approval** ⭐⭐: pauses the pipeline pending human sign-off. **No response within 7 days = treated as Rejected, and the pipeline fails.** Approvers need `codepipeline:PutApprovalResult`; SNS can notify them.

**Cross-Account / Cross-Region** ⭐⭐: cross-account requires a customer-managed KMS key + a pipeline service role + a cross-account IAM role; cross-region requires an artifact bucket in every region (CodePipeline replicates artifacts automatically).

**Triggers**: **CodeStar Connections** is the modern recommendation (replacing GitHub OAuth tokens).

---

## 5. CodeArtifact ⭐⭐

**= AWS's managed package repository service.**

**Supported formats**: npm, pip, Maven, Gradle, NuGet, twine, Cargo, Swift, Ruby Gems, Generic.

### The Hierarchy ⭐⭐⭐

**Domain → Repository → Package → Version**

**Domain**: the top-level container — all package assets are physically stored once (saving space), and the whole domain shares a single KMS key. **Repository**: a logical package repo, polyglot.

**Key limits**: roughly 1,000 repositories per domain; auth token TTL of **12 hours**.

**Upstream Repositories** ⭐⭐: one repo can link to another as an upstream:

```
Local Repo → Upstream Repo → External Connection (public:npmjs)
```

Even if a package is unpublished from npmjs, a local copy still exists (a supply-chain security benefit). Supported external connections: `public:npmjs`, `public:pypi`, `public:maven-central`, `public:nuget-org`, and more.

**Authentication**:

```bash
aws codeartifact login --tool npm --domain my-org --repository my-repo
npm install axios
```

The token expires after 12 hours, so CI pipelines need to refresh it before every build.

**Cross-account access**: Account A configures a domain policy + repository policy + KMS key policy that allow Account B.
