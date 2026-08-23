# Module 14：基础设施即代码（CloudFormation / SAM / CDK）

> **覆盖日期**：5/16、5/17、5/18
>
> English version: [`en/14-iac.md`](../en/14-iac.md)

---

# 一、CloudFormation Part 1 — IaC 核心

## CloudFormation 是什么 ⭐⭐⭐

**= AWS 官方的 Infrastructure as Code (IaC) 服务**

**核心价值**：✅ 用模板（JSON/YAML）声明式定义 AWS 资源；✅ AWS 自动管理资源创建、更新、删除的顺序和依赖；✅ 失败自动 rollback；✅ 版本控制+审计；✅ 免费（只为创建的资源付费）；✅ 覆盖几乎所有 AWS 服务；✅ 跨 region、跨账户复制基础设施（用 StackSets）。

**核心概念**：**Template**（YAML/JSON 文件）；**Stack**（一次创建的资源集合）；**Change Set**（"如果执行这次更新会发生什么"的预览）；**Drift**（实际资源和模板定义不一致）；**StackSet**（一个 template 跨多个 account/region 部署）；**Nested Stack**（一个 stack 嵌套在另一个 stack 中）。

**vs 其他 IaC**：Terraform 是 HashiCorp 出品，支持多云，语言是 HCL；CDK 是 AWS 官方，底层是 CFN，用编程语言（TypeScript/Python/Java/C#/Go）。DVA 主要考 CloudFormation，SAM/CDK 也有考察，Terraform 不考。

## Template 9 大区段 ⭐⭐⭐

```yaml
AWSTemplateFormatVersion: '2010-09-09'   # 1. 模板版本(固定值)
Description: 'My web app stack'          # 2. 描述(可选)
Metadata: {}                              # 3. 元数据(可选)
Parameters: {}                            # 4. 输入参数(可选)
Mappings: {}                              # 5. 静态映射表(可选)
Conditions: {}                            # 6. 条件(可选)
Transform: []                             # 7. Transform(SAM/Include)(可选)
Resources: {}                             # 8. 资源 ⭐必需
Outputs: {}                               # 9. 输出值(可选)
```

⭐ 只有 `Resources` 是必需的，其他都可选。但 `Description` 强烈推荐。

## Resources ⭐⭐⭐

**= 你要创建的 AWS 资源，核心区段**

```yaml
Resources:
  LogicalNameHere:           # 逻辑 ID(模板内唯一)
    Type: AWS::S3::Bucket    # 资源类型
    Properties:              # 资源属性
      BucketName: my-bucket
      VersioningConfiguration:
        Status: Enabled
```

**Type 命名规则**：`AWS::Service::ResourceType`。例：`AWS::S3::Bucket`、`AWS::EC2::Instance`、`AWS::Lambda::Function`、`AWS::DynamoDB::Table`、`AWS::IAM::Role`。

**资源间依赖**：**Implicit**（用 `!Ref` 或 `!GetAtt` 引用，CloudFormation 自动推断）；**Explicit**（用 `DependsOn` 属性强制声明）。⭐ 大多数情况不用 DependsOn（`!Ref` 自动产生依赖），显式 DependsOn 用在"没有引用但确实需要顺序"的情况。

## Parameters ⭐⭐

**= 让模板"参数化"，同一模板部署不同环境**

```yaml
Parameters:
  EnvType:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
  DBPassword:
    Type: String
    NoEcho: true                     # ⭐ 不显示在控制台/日志里
    MinLength: 8
  VpcId:
    Type: AWS::EC2::VPC::Id           # ⭐ 控制台自动列出可选 VPC
```

**关键类型**：`String`、`Number`、`CommaDelimitedList`、`AWS::EC2::VPC::Id`、`AWS::EC2::Subnet::Id`、`AWS::EC2::KeyPair::KeyName`、`AWS::SSM::Parameter::Value<String>`（从 SSM Parameter Store 拉取值，常考）。

**NoEcho**：⚠️ 不是加密，只是不显示。真正敏感的值应该用 Secrets Manager/Parameter Store，模板里用动态引用拉取：`'{{resolve:secretsmanager:my-secret:SecretString:password}}'`。

**SSM Parameter 引用**：

```yaml
Parameters:
  LatestAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'
```

## Intrinsic Functions ⭐⭐⭐

**Ref**（最常用）：返回 Logical ID 对应的"主标识符"（看资源类型决定）——Parameter 返回参数值；S3 Bucket 返回 bucket name；EC2 Instance 返回 instance ID；SNS Topic 返回 **topic ARN**；IAM Role 返回 role name。⚠️ 不同资源 `!Ref` 返回不同！需要 ARN 时优先用 `!GetAtt`。

**Fn::GetAtt**（获取资源属性）：`!GetAtt MyBucket.Arn`、`!GetAtt MyInstance.PrivateIp`、`!GetAtt MyDB.Endpoint.Address`。

**Fn::Sub**（字符串插值，最现代）：`!Sub 'my-app-${EnvType}-${AWS::AccountId}'`——推荐生产首选，可读性远好于 Join。

**Fn::Join**（老式拼接）：`!Join ['', ['arn:aws:s3:::', !Ref MyBucket, '/*']]`。

**Fn::FindInMap**（查映射表）：配合 `Mappings` 区段，常用于"按 region 查 AMI"。

**Fn::ImportValue**（跨 Stack 引用）⭐⭐：引用另一个 stack 的 Output（源 stack 需要用 `Export`）。**关键限制**：被 import 的 export 不能删除（必须先删所有 import 它的 stack）；不能跨 region；不能动态名字（name 必须是字面量）。

**Fn::If / Conditions**（条件）⭐：

```yaml
Conditions:
  IsProd: !Equals [!Ref EnvType, prod]
Resources:
  MyDB:
    Properties:
      DBInstanceClass: !If [IsProd, db.r5.large, db.t3.micro]
  ProdOnlyAlarm:
    Condition: IsProd    # ⭐ 只在 prod 创建这个资源
```

**AWS::NoValue**：等于"删除这个属性"，配合 `!If` 使用。

**其他辅助函数**：`Fn::Select`/`Fn::Split`/`Fn::GetAZs`（拿 AZ 列表、拆字符串）；`Fn::Cidr`（切 CIDR 块）；`Fn::Base64`（常用于 EC2 user data）；`Fn::Equals`/`Fn::Not`/`Fn::And`/`Fn::Or`（条件函数）。

## Pseudo Parameters ⭐

**= AWS 预定义、不需要声明就能用的参数**：`AWS::AccountId`、`AWS::Region`、`AWS::StackName`、`AWS::StackId`、`AWS::Partition`（aws/aws-cn/aws-us-gov）、`AWS::URLSuffix`、`AWS::NoValue`、`AWS::NotificationARNs`。

## Mappings

**= 静态查找表**（不能动态）。典型场景：按 region 查 AMI ID。**Mappings vs Parameters**：Mappings 静态、内置在模板里，适合 region/env 映射；Parameters 用户运行时输入，适合环境差异化配置。⭐ 现代做法：用 SSM Parameter 替代 Mappings（更动态、AWS 维护）。

## Outputs

**= stack 创建后导出的值，可被 import/在控制台显示**

```yaml
Outputs:
  VpcId:
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VpcId'   # ⭐ 加 stack 名前缀避免冲突
```

## Resource Attributes ⭐⭐⭐

**= 给资源加的"元行为属性"**，不属于资源本身配置。5 个核心 attributes：**DeletionPolicy**、**UpdateReplacePolicy**、**DependsOn**、**CreationPolicy**、**UpdatePolicy**。

### DeletionPolicy ⭐⭐⭐

**stack 删除时，资源的命运**：`Delete`（默认，删除资源）、`Retain`（保留资源）、`Snapshot`（先做 snapshot 再删除，适合 RDS/EBS/Redshift/ElastiCache）。

**支持 Snapshot 的资源**：EC2 Volume、ElastiCache CacheCluster/ReplicationGroup、Neptune DBCluster、RDS DBInstance/DBCluster、Redshift Cluster。**S3/DynamoDB 不支持 Snapshot**，生产 critical 数据用 Retain。

⚠️ S3 删除 Retain 注意：CFN 不会清空 bucket，如果 bucket 有 object，删 stack 会失败（除非 bucket 为空或设 Retain）。

### UpdateReplacePolicy ⭐⭐

stack **更新**时如果某属性改动触发"替换"（创建新资源+删除旧资源），旧资源怎么办：`Delete`（默认）、`Retain`、`Snapshot`。

**典型触发替换**的场景：改 `BucketName`（S3 必须新 bucket）、改 `DBInstanceIdentifier`（RDS）、改 EC2 的 `ImageId`/`InstanceType`。

⚠️ DeletionPolicy 和 UpdateReplacePolicy 是两个独立场景：DeletionPolicy 在删 stack 时生效；UpdateReplacePolicy 在 update 替换时生效。生产最佳实践：两个都设 Retain 或 Snapshot。

### CreationPolicy（EC2/ASG 必考）⭐

**= 等资源主动 signal 才算"创建完成"**。典型用法：EC2 跑完 user data 才算"准备好"。

```yaml
WebServer:
  Type: AWS::EC2::Instance
  CreationPolicy:
    ResourceSignal:
      Count: 1
      Timeout: PT15M           # ISO 8601 duration
```

**用 cfn-signal**：AWS 提供的 helper script，EC2 跑完关键步骤后调用告诉 CFN "我 ready 了"。

### UpdatePolicy（Auto Scaling/Lambda 必考）⭐

**= 控制更新时 CFN 如何处理 ASG/Lambda**。最常考的是 `AutoScalingRollingUpdate`（ASG 实例分批替换）：`MaxBatchSize`、`MinInstancesInService`、`PauseTime`、`WaitOnResourceSignals`。适合 EC2 fleet 滚动更新而非全部替换。

## Stack 生命周期 ⭐⭐⭐

**Create → Update → Delete** 三个核心操作，每个有 IN_PROGRESS/COMPLETE/ROLLBACK 状态。

**Rollback**：默认行为——任何资源创建/更新失败 → 整 stack 自动回滚到之前状态。可控选项：`OnFailure: DELETE`（默认，新 stack）——失败时删除已创建的资源；`OnFailure: ROLLBACK`（默认，更新）；`OnFailure: DO_NOTHING`——保留失败状态便于排查（开发推荐）。

**Rollback Configuration**：配 CloudWatch alarm，更新中如果 alarm 触发就自动回滚，适合"部署后看监控，有问题立刻回滚"。

## Change Sets ⭐⭐

**= "如果执行这次更新，会发生什么"的预览**。两种模式：`CREATE`（创建新 stack 的预览）、`UPDATE`（更新现有 stack 的预览）。

典型流程：修改 template → 创建 change set → 查看 change set（Add/Modify/Remove，以及是否触发资源 replacement）→ execute（正确）或删除重试（有问题）。

⭐ Change Set 显示哪些资源会被"替换"——防止意外删除生产数据。不 execute change set，资源不会变。

## Stack Policy ⭐

**= 防止某些资源被意外更新或删除的 JSON 策略**。⚠️ Stack Policy ≠ IAM Policy！仅控制"在 stack update 时，能不能改某资源"。

Update actions：`Update:Modify`、`Update:Replace`、`Update:Delete`、`Update:*`。典型用途：防止生产 RDS 被误改/替换、防止关键 SG 被改。

## Drift Detection ⭐

**= 检查"实际资源"和"模板定义"是否不一致**（被人手动改了）。控制台 Stack → "Detect drift" → CFN 对比每个资源属性 → 输出哪些资源 drift。**Drift 状态**：IN_SYNC、DRIFTED、NOT_CHECKED、UNKNOWN。⚠️ 不是所有资源类型都支持。

## Helper Scripts（EC2 用）⭐

AWS 在 Amazon Linux 上预装的 Python 脚本：`cfn-init`（读 template 的 Metadata 配置初始化 EC2）、`cfn-signal`（EC2 告诉 CFN "我准备好了"）、`cfn-hup`（后台 daemon，监听 stack 更新触发 cfn-init 重新跑）、`cfn-get-metadata`。

## Wait Conditions（简略）

**= 等待外部"信号"才继续 stack 创建**（老式，现代用 CreationPolicy）。用 presigned URL 给外部系统，外部 curl 该 URL 报告 success/failure。优先用 CreationPolicy。

## Termination Protection ⭐

**= 防止误删 stack**。启用后 delete-stack 调用直接报错，必须先关闭 protection 才能删。**生产 stack 必开**。

## Nested Stacks ⭐⭐

**= 一个 stack 中嵌套其他 stack（template 复用）**

```yaml
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/network.yaml
  AppStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/app.yaml
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
```

**优势**：模板复用；简化大模板（规避资源数/大小上限）；解决 export 名称冲突（嵌套用 Outputs 直接传值，不用 ImportValue）。

**Nested Stack vs Cross-Stack Reference**：Cross-Stack（ImportValue）松耦合，stack 独立部署，export 不能删；Nested 紧耦合，子 stack 被父 stack 管理，适合"私有的可复用模块"。

## CloudFormation 限制

单 template 资源数上限 **500**（超过用 Nested Stacks 拆分）；单 template 大小从 S3 加载约 1 MB，直接传约 51,200 字节；Parameters/Outputs/Mappings 数各约 200；Stack 数/account/region 默认约 2000（可调）。

---

# 二、CloudFormation Part 2

## StackSets ⭐⭐

**= 一个 template 跨多个 account/region 部署**

**关键术语**：**StackSet**（跨账户的模板配置，在 admin account）；**Stack Instance**（在 target account/region 的实际 stack）。

**两种权限模式** ⭐⭐⭐：

**Self-managed permissions**：适合单独 AWS 账户（不在 Organization），必须手动在每个 target account 创建 `AWSCloudFormationStackSetExecutionRole`，并在 admin account 创建 `AWSCloudFormationStackSetAdministrationRole`。

**Service-managed permissions**：适合 AWS Organizations，AWS 自动管理 role。特性：自动创建/管理跨账户 role；支持"自动 deploy 到 Organization 中所有 account/新加入的 account"；部署粒度可到整个 Organization 或特定 OU；⚠️ 必须先在 Organizations 启用 trusted access for CloudFormation。

**部署选项**：Concurrent accounts（并发部署到多少 account）；Failure tolerance（容忍多少 account 失败）；Region order；Auto-deployment（Service-managed only，新 account 加入 OU 自动 deploy）。

**经典用例**：给所有账户部署相同的 IAM roles；跨账户部署 AWS Config/GuardDuty/CloudTrail；跨账户跨 region 部署 VPC baseline；跨账户部署 alarm/dashboard 模板。**不适合**应用代码（应用通常账户隔离）。

## Custom Resources ⭐⭐⭐

**= 用 Lambda 扩展 CloudFormation 的能力，做"CFN 原生不支持"的事**

用途：创建第三方资源（GitHub repo、Datadog dashboard）；在 stack 创建时填充初始数据；调用 CFN 没暴露的 API 操作；从外部系统拉取动态值。

**两种类型**：Lambda-backed（`AWS::CloudFormation::CustomResource` 或 `Custom::MyType`，⭐ 几乎所有都用这个）、SNS-backed（给第三方 webhook 服务）。

```yaml
Resources:
  MyCustomResource:
    Type: Custom::MyResource
    Properties:
      ServiceToken: !GetAtt MyLambdaFunction.Arn
      InputParam1: foo
```

**Lambda 的合约** ⭐⭐：必须处理 3 种 RequestType——`Create`（创建资源，返回 PhysicalResourceId）、`Update`（更新资源）、`Delete`（删除资源）。Lambda 必须：收到 CFN 发的 event → 做实际工作 → **向 event 里的 ResponseURL（presigned URL）PUT SUCCESS/FAILED 响应**。⚠️ 必须响应，否则 CFN 等到超时（1 小时）才认为失败。响应必含字段：`Status`、`PhysicalResourceId`、`StackId`/`RequestId`/`LogicalResourceId`、`Data`（可选）。

## CloudFormation Registry & Public Extensions（简略）

**= 第三方资源类型**（类似 Terraform Providers）。AWS 维护的、第三方提供的、自己写的私有 extensions。DVA 不重点考。

## Macros（模板宏）⭐

**= 在 CloudFormation 处理 template 前，先用 Lambda 转换它**。典型用例：把简化语法展开成完整 CFN、自动加 tags、生成重复资源。⭐ **SAM 本质上就是一个 macro**：`AWS::Serverless::Function` 不是 CFN 原生类型，而是 SAM transform 在部署前展开成 `AWS::Lambda::Function` + 其他相关资源。

## Modules（简略）

**= 把多个资源打包成"模块"**，可在多个 template 引用，类似 nested stack 但更轻量，部署后只显示为单个资源。

## CloudFormation Hooks

**= 在 stack 创建/更新前，自动验证模板是否符合策略**。典型用途：强制 S3 bucket 必须加密、强制资源必须有特定 tag。类似"部署前的合规检查"。

## Resource Import ⭐⭐

**= 把现有资源"导入"CFN 管理**（原本是手动建的）。用途：手动建了一堆资源现在想用 IaC 管；修复 drift；拆分 stack。

步骤：写描述这些资源的 template → 创建 IMPORT 类型的 change set → 提供每个资源的 physical ID → execute change set → CFN 验证后资源纳入管理。⚠️ 限制：只支持特定资源类型；import 时资源不会被修改；import 后正常 CFN 操作就能用了。

⭐ 不同于 drift 修复：Drift 是已经管理的资源被手动改了；Import 是没管理的资源开始管理。

## Stack 删除时 S3 Bucket 不为空的坑

**问题**：`DeletionPolicy: Delete`（默认）+ bucket 里有 object → 删 stack 失败。**解决方案**：删 stack 前手动清空 bucket；或设 `DeletionPolicy: Retain`；或写 Custom Resource Lambda 在 stack 删除时清空 bucket。

## CloudFormation Best Practices

安全/数据保护：生产资源加 DeletionPolicy + UpdateReplacePolicy（Retain 或 Snapshot）；启用 Termination Protection；敏感参数用 NoEcho 或 secretsmanager resolve；加 Stack Policy 保护关键资源。部署流程：总是用 Change Set 预览；CI/CD 中跑 `cfn-lint`/`cfn-nag` 静态分析；Dev/staging/prod 分开 stack。模板设计：小 stack 而非大 stack；用 Nested Stack 或 Cross-Stack References 解耦；通过 Parameters 参数化环境差异。

---

# 三、SAM (Serverless Application Model)

## SAM 是什么 ⭐⭐⭐

**= CloudFormation 的扩展**，专门简化 serverless 应用部署。核心组件：SAM Template（简化的 CFN 模板，有 `Transform: AWS::Serverless-2016-10-31`）、SAM CLI（本地开发/测试/部署工具）。

**核心价值**：极简语法（几行 YAML 定义 Lambda+API Gateway+DynamoDB）；本地测试（`sam local invoke`、`sam local start-api`）；快速迭代（`sam sync` 几秒钟更新 cloud）；底层是 CFN，所有 CFN 能力都能用；AWS 官方，开源。

## SAM Template 核心结构 ⭐⭐⭐

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31    # ⭐ 必需

Globals:
  Function:
    Runtime: python3.12
    Timeout: 30
    MemorySize: 512

Resources:
  HelloFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./hello/
      Handler: app.lambda_handler
      Events:
        HelloApi:
          Type: Api
          Properties:
            Path: /hello
            Method: GET
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref MyTable

  MyTable:
    Type: AWS::Serverless::SimpleTable
```

**SAM 核心简化类型**：`AWS::Serverless::Function`（展开为 Lambda+IAM Role+EventSources 全套，核心）、`AWS::Serverless::Api`（API Gateway REST API）、`AWS::Serverless::HttpApi`（HTTP API）、`AWS::Serverless::SimpleTable`（DynamoDB Table）、`AWS::Serverless::StateMachine`（Step Functions）、`AWS::Serverless::LayerVersion`（Lambda Layer）、`AWS::Serverless::Application`（Nested stack）。

**关键洞察**：SAM 类型不是 CFN 原生的，deploy 时 SAM transform 把它们展开成 CFN 资源。

## SAM 的事件源 ⭐⭐

`Function` 的 `Events` 字段一句话搞定 trigger：支持 `Api`（REST）、`HttpApi`、`S3`、`SQS`、`DynamoDB`（Streams）、`EventBridgeRule`、`Schedule`（cron）、`Kinesis` 等类型。每个 Event 自动建好 source + 必要的 IAM 权限。

## SAM Policy Templates ⭐⭐

**= 预定义的 IAM policy 简写**，常用权限不用手写：

```yaml
Policies:
  - DynamoDBCrudPolicy:
      TableName: !Ref MyTable
  - S3ReadPolicy:
      BucketName: my-bucket
  - SQSPollerPolicy:
      QueueName: !GetAtt MyQueue.QueueName
  - SNSPublishMessagePolicy:
      TopicName: !GetAtt MyTopic.TopicName
  - VPCAccessPolicy: {}
```

常考的：`DynamoDBCrudPolicy`、`S3ReadPolicy`、`SQSPollerPolicy`、`SNSPublishMessagePolicy`、`LambdaInvokePolicy`、`KinesisStreamReadPolicy`。

## SAM CLI 命令 ⭐⭐⭐

```bash
# 初始化
sam init --runtime python3.12 --name my-app

# 本地开发+测试
sam build
sam local invoke MyFunction
sam local start-api
sam local start-lambda

# 验证
sam validate --lint

# 部署
sam deploy --guided    # 首次
sam deploy              # 之后

# 快速迭代
sam sync --watch

# 日志
sam logs -n MyFunction --tail

# 清理
sam delete
```

**sam build vs sam deploy**：`sam build`（安装依赖、编译打包）；`sam package`（上传到 S3）；`sam deploy`（用 CFN deploy 部署到 AWS）。⭐ `sam deploy` 内部会先 build + package + 调 CFN。

**SAM Accelerate (sam sync)** ⭐⭐：极速本地→云端同步，跳过 CFN 部署流程。工作机制：检测本地代码变化 → 直接调 Lambda UpdateFunctionCode API（绕过 CFN）→ 几秒钟内代码生效。如果是结构变化才退回到 CFN deploy。适合开发阶段快速迭代，不适合生产部署。⚠️ 更新不通过 CFN，会导致 stack drift！

## Deployment Preferences（渐进式发布）⭐⭐

**= SAM 集成 CodeDeploy 实现 canary/linear 部署**

```yaml
HelloFunction:
  Properties:
    AutoPublishAlias: live
    DeploymentPreference:
      Type: Canary10Percent10Minutes
      Alarms:
        - !Ref ErrorAlarm
      Hooks:
        PreTraffic: !Ref PreTrafficLambda
        PostTraffic: !Ref PostTrafficLambda
```

**部署策略类型**：`AllAtOnce`（一次切完）；`Canary10Percent5/10/15/30Minutes`（先 10%，等 N 分钟，再切剩余）；`Linear10PercentEvery1/2/3/10Minutes`（每 N 分钟切 10%）。

⭐ 底层用 CodeDeploy 做 Lambda traffic shifting。Alarm 触发自动回滚到老版本。

## SAM Globals 区段 ⭐

**= 给所有 Function/Api/SimpleTable 提供默认值**，减少重复。Function 资源可以 override Globals 里的值。

## SAM vs CloudFormation ⭐⭐

|  | **CloudFormation** | **SAM** |
|---|---|---|
| 范围 | 所有 AWS 资源 | **专门 serverless**（扩展 CFN） |
| 语法 | 完整、繁琐 | **简化**（关键资源大幅减少代码量） |
| 本地测试 | ❌ 不支持 | ✅ `sam local` 一键模拟 |
| 快速部署 | 标准 CFN 部署 | **sam sync 秒级** |
| Policy 简写 | ❌ 自己写 IAM JSON | ✅ Policy Templates |

**选择**：纯 serverless 应用 → SAM；大型 IaC（VPC+EC2+数据库） → CloudFormation；想本地测试 Lambda → SAM；多语言/想用 OOP → CDK。

## Nested SAM Applications

**= 引用别人写好的 SAM 应用，作为子 stack**，通过 `AWS::Serverless::Application` + `ApplicationId`。**Serverless Application Repository (SAR)**：AWS 的 marketplace，公开 SAM 应用，一键部署到你账户。

---

# 四、AWS CDK — 用编程语言写基础设施

## CDK 是什么 ⭐⭐⭐

**CDK = Cloud Development Kit** — 用真正的编程语言（TypeScript/Python/Java/C#/Go）写基础设施代码，最终编译成 CloudFormation 模板。

**核心价值**：用熟悉的编程语言（for 循环、if、function、class、类型检查、IDE 自动补全）；代码复用（把基础设施抽象成 class/library/package）；单元测试基础设施；强类型；底层是 CloudFormation，所有 CFN 优势继承。

**关键认识**：CDK 不替代 CFN——它是 CFN 之上的抽象层；synth 产生的就是普通 CFN template；stack drift、rollback、change set 等 CFN 特性全部继承；部署目标始终是 CloudFormation Stack。

## CDK v1 vs v2 ⭐

CDK v2 是当前主流。**v2 特点**：一个统一包 `aws-cdk-lib`（服务在子模块），而不是每个服务一个包（v1 的方式）；解决了版本兼容性痛点；v1 已停止支持。

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
```

## 核心三层构造 ⭐⭐⭐

```
App(应用) = 整个 CDK 项目的容器
  └── Stack(栈) = 一个 CloudFormation Stack
        └── Construct(构造) = AWS 资源(可嵌套)
```

**App**：整个 CDK 项目的根容器，一个 App 可以包含多个 Stack。

**Stack**：一个 CloudFormation Stack——是 CDK 中最小的部署单元。

**Construct**：一个或多个 AWS 资源的封装，最小的"积木"。每个 Construct 接收 3 个参数 ⭐：`scope`（父级）、`id`（在该 scope 内唯一的逻辑 ID）、`props`（配置属性，可选）。

## L1 / L2 / L3 Constructs ⭐⭐⭐

CDK Construct 分 3 个抽象级别，这是高频考点：

**L1 — Low-Level Constructs**：直接对应 CFN 资源，无任何抽象。类名以 `Cfn` 前缀（如 `CfnBucket`）；属性与 CFN template 完全一一对应；自动从 CFN spec 生成；你要自己写所有细节（IAM role、permissions、依赖关系）。适合用最新 AWS 资源（L2 还没出）或需要完整 CFN 控制。

**L2 — Curated Constructs**（默认推荐）：CDK 团队精心设计的抽象，有合理默认值。类名简洁（`Bucket`、`Function`、`Vpc`）；大量合理默认值（加密、权限、最佳实践内置）；辅助方法（如 `bucket.grantRead(lambda)` 自动加 IAM 权限）。**这是日常开发用得最多的级别**。

**L3 — Patterns**（高级模式）：多个资源组合成完整模式。命名包含 `Patterns`；一个 Construct 创建多个相关资源；内置最佳实践架构。例：`ApplicationLoadBalancedFargateService` 一行创建 ECS Fargate Service+Task Definition+ALB+Target Group+Security Group+所有 IAM permissions。适合常见架构模式，但灵活性较低。

**实战建议**：默认 L2，需要新功能时降 L1，重复模式抽 L3。

## CDK 项目工作流 ⭐⭐

```bash
# 创建项目
cdk init app --language=typescript

# Bootstrap(每个 account/region 一次)⭐
cdk bootstrap aws://123456789012/us-east-1

# 看 CFN 模板(不部署)
cdk synth

# 对比已部署与本地变化
cdk diff

# 部署
cdk deploy
cdk deploy --hotswap   # ⭐ 快速更新

# 监听变化自动 deploy
cdk watch

# 销毁
cdk destroy
```

**`cdk synth`**：把代码"合成"为 CloudFormation template，输出到 `cdk.out/`，不实际部署。

**`cdk diff`**：显示"本地代码 vs 已部署 stack"的差异，类似 CFN 的 Change Set 但更直观。

**`cdk deploy --hotswap`** ⭐：快速更新，绕过 CFN 直接调资源 API，支持 Lambda code、State Machine 定义、ECS task definition。**只用于开发**（类似 sam sync，会导致 drift），生产用普通 `cdk deploy`。

## CDK Bootstrap ⭐⭐⭐

**= 在 account/region 准备 CDK 所需的"基础设施"**

CDK 部署时会上传 assets（lambda 代码、Docker 镜像）到 S3/ECR、用临时 IAM role 执行 CFN 部署、维护跨 stack 引用——这些都需要先准备好。

`cdk bootstrap` 部署一个名为 **`CDKToolkit`** 的 CloudFormation Stack，包含：S3 Bucket（存 assets）、ECR Repository（存 Docker images）、5 个 IAM Roles（cfn-exec、deploy、file-publishing、image-publishing、lookup）。

**关键考点**：⭐ 每个 account/region 组合只需 bootstrap 一次；⭐ CDK 第一次部署前必须 bootstrap（不然报错）；⭐ Bootstrap stack 名字固定为 `CDKToolkit`；⭐ 删除 CDK 应用后 bootstrap stack 不会自动删。

## Synth & Assets ⭐⭐

**Assets** = CDK 自动管理的"非 CFN"文件（代码、Docker 镜像等）。类型：File assets（Lambda code zip、Layer zip）、Docker image assets。

```typescript
const fn = new lambda.Function(this, 'MyFunction', {
  code: lambda.Code.fromAsset('./lambda-code'),  // ⭐ 自动打包+上传
});
```

自动流程：`cdk deploy` → 把 assets 打包 → 上传到 bootstrap 的 S3/ECR → CFN template 中引用 URI → 部署 stack。你不用写 S3 bucket 创建逻辑、Docker push 命令、ARN 引用——CDK 全自动。

## Context ⭐

**= "在 synth 时需要查询的运行时值"**（如 AMI ID、VPC ID）。工作机制：`cdk synth` 查 AWS API → 把结果存到 `cdk.context.json`（缓存）→ 后续 synth 用缓存（快、deterministic）。

`ec2.Vpc.fromLookup()` 会触发 context 查询。结果存到 `cdk.context.json`，提交到 git，保证团队/CI 一致。清缓存：`cdk context --clear`。

## Aspects（横切关注点）⭐

**= 给整棵 construct 树应用同一个变换/验证**。典型用例：给所有资源加 tag、强制所有 bucket 加密、验证所有资源符合命名规范。应用范围：加到 App→应用到所有 stack；加到 Stack→应用到该 stack 所有资源；加到 Construct→仅该子树。

## Permissions / Grants ⭐⭐

**= L2 Construct 的辅助方法，自动配 IAM**

```typescript
bucket.grantRead(fn);
bucket.grantWrite(fn);
table.grantReadWriteData(fn);
queue.grantSendMessages(fn);
topic.grantPublish(fn);
```

底层做了什么：给 fn 的执行角色加合适的 IAM permissions；如有需要给目标资源加 resource policy；自动处理 KMS key 权限（如果资源加密）。

⭐ vs CFN/SAM：CFN 要手写 IAM policy JSON；SAM 有 Policy Templates 但只是简化预设；CDK 的 grant 基于资源类型自动生成最小权限，最现代。

## 跨 Stack 引用 ⭐

CDK 让跨 stack 引用变得几乎透明——直接引用另一个 stack 暴露出来的属性（如 `stack1.bucket.bucketName`），CDK 自动在 stack1 加 Output+Export，在 stack2 加 `Fn::ImportValue`，你完全感觉不到底层机制。⭐ 限制继承自 CFN：不能跨 region、被引用的 export 不能删。

## Testing

CDK 支持单元测试基础设施代码，用 `Template.fromStack(stack)` + `template.resourceCountIs(...)` / `template.hasResourceProperties(...)`。类型：Snapshot tests、Fine-grained tests、Validation tests。

## CDK Pipelines ⭐

**= 用 CDK 定义的自我演进的 CI/CD pipeline**。核心特性——**Self-mutating**：pipeline 会更新它自己，改了 pipeline 定义 push 后下次 build 自动更新 pipeline，不需要手动重建。优势：多 stage 部署（dev→staging→prod）；跨账户部署内置；自动 approval 步骤。

## CDK 主要 CLI 命令汇总 ⭐⭐⭐

`cdk init`（初始化项目）、`cdk bootstrap`（准备 account/region）、`cdk synth`（生成 CFN template）、`cdk diff`（对比本地 vs 已部署）、`cdk deploy`（部署）、`cdk deploy --hotswap`（快速更新）、`cdk watch`（监听自动 deploy）、`cdk destroy`（删除）、`cdk ls`（列出所有 stack）、`cdk context`（管理 context 缓存）。

## CDK vs CFN vs SAM 终极对比 ⭐⭐⭐

|  | **CFN** | **SAM** | **CDK** |
|---|---|---|---|
| 语法 | YAML/JSON | YAML（扩展 CFN） | **TypeScript/Python/Java/C#/Go** |
| 适合范围 | 全 AWS | 仅 serverless | 全 AWS |
| 复用 | Nested Stack | SAM 模板 | **真正的代码复用**（class、function） |
| IAM 写法 | 手写 JSON | Policy Templates | **`grantRead()` 自动** |
| 类型检查 | ❌ | ❌ | **✅ IDE 报错** |
| 本地测试 | ❌ | sam local | snapshot + assertion tests |
| 快速迭代 | ❌ | sam sync | **cdk deploy --hotswap / cdk watch** |
| Bootstrap | 不需要 | 不需要 | **需要**（每 account/region） |

**选择**：简单 IaC、团队不熟编程 → CFN；纯 serverless 应用要本地测试 → SAM；大型/复杂/多团队 IaC → CDK；想要类型检查+IDE 智能提示 → CDK；抽象/复用/跨团队共享模式 → CDK。

⭐ 现代趋势：新项目通常 CDK > SAM > CFN，但这取决于团队和具体场景。

## CDK 其他生态（简略）

**CDK for Terraform (cdktf)**：用 CDK 语法生成 Terraform 配置。**CDK for Kubernetes (cdk8s)**：用 CDK 写 Kubernetes YAML。**Constructs Hub**：AWS 维护的 CDK construct 搜索引擎。DVA 不考这些，知道存在即可。
