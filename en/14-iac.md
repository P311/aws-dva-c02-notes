# Module 14: Infrastructure as Code (CloudFormation / SAM / CDK)

> **Covered**: May 16, 17, 18
>
> 中文版本：[`zh/14-iac.md`](../zh/14-iac.md)

---

# 1. CloudFormation Part 1 — IaC Fundamentals

## What CloudFormation Is ⭐⭐⭐

**= AWS's official Infrastructure as Code (IaC) service.**

**Core value**: ✅ declaratively defines AWS resources with templates (JSON/YAML); ✅ AWS automatically manages the order and dependencies for creating, updating, and deleting resources; ✅ automatic rollback on failure; ✅ version control + auditability; ✅ free (you only pay for the resources it creates); ✅ covers almost every AWS service; ✅ replicates infrastructure across regions/accounts (via StackSets).

**Core concepts**: a **Template** (a YAML/JSON file); a **Stack** (one instantiation of a template's resources); a **Change Set** (a preview of "what would happen if this update ran"); **Drift** (actual resources no longer matching the template definition); a **StackSet** (deploying one template across multiple accounts/regions); a **Nested Stack** (one stack nested inside another).

**vs. other IaC tools**: Terraform is from HashiCorp, multi-cloud, using HCL; CDK is AWS's own tool, built on top of CFN, using real programming languages (TypeScript/Python/Java/C#/Go). The DVA mainly tests CloudFormation, with some SAM/CDK coverage — Terraform isn't tested.

## The Nine Template Sections ⭐⭐⭐

```yaml
AWSTemplateFormatVersion: '2010-09-09'   # 1. Template version (a fixed value)
Description: 'My web app stack'          # 2. Description (optional)
Metadata: {}                              # 3. Metadata (optional)
Parameters: {}                            # 4. Input parameters (optional)
Mappings: {}                              # 5. Static lookup tables (optional)
Conditions: {}                            # 6. Conditions (optional)
Transform: []                             # 7. Transform (SAM/Include) (optional)
Resources: {}                             # 8. Resources ⭐ required
Outputs: {}                               # 9. Output values (optional)
```

⭐ Only `Resources` is required — everything else is optional. `Description` is strongly recommended though.

## Resources ⭐⭐⭐

**= the AWS resources you're creating — the core section.**

```yaml
Resources:
  LogicalNameHere:           # Logical ID (unique within the template)
    Type: AWS::S3::Bucket    # Resource type
    Properties:              # Resource properties
      BucketName: my-bucket
      VersioningConfiguration:
        Status: Enabled
```

**Type naming convention**: `AWS::Service::ResourceType`. Examples: `AWS::S3::Bucket`, `AWS::EC2::Instance`, `AWS::Lambda::Function`, `AWS::DynamoDB::Table`, `AWS::IAM::Role`.

**Dependencies between resources**: **Implicit** (referencing via `!Ref` or `!GetAtt` — CloudFormation infers the dependency automatically); **Explicit** (forced via the `DependsOn` attribute). ⭐ Most of the time `DependsOn` isn't needed (`!Ref` creates an implicit dependency automatically) — explicit `DependsOn` is for cases with "no reference but a real ordering requirement still exists."

## Parameters ⭐⭐

**= parameterizing a template so the same one can deploy different environments.**

```yaml
Parameters:
  EnvType:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
  DBPassword:
    Type: String
    NoEcho: true                     # ⭐ Hidden from the console/logs
    MinLength: 8
  VpcId:
    Type: AWS::EC2::VPC::Id           # ⭐ Console auto-populates a VPC dropdown
```

**Key types**: `String`, `Number`, `CommaDelimitedList`, `AWS::EC2::VPC::Id`, `AWS::EC2::Subnet::Id`, `AWS::EC2::KeyPair::KeyName`, `AWS::SSM::Parameter::Value<String>` (fetches a value from SSM Parameter Store, frequently tested).

**NoEcho**: ⚠️ this is not encryption — it only hides the value from display. Genuinely sensitive values should live in Secrets Manager/Parameter Store, referenced dynamically in the template: `'{{resolve:secretsmanager:my-secret:SecretString:password}}'`.

**Referencing an SSM parameter**:

```yaml
Parameters:
  LatestAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'
```

## Intrinsic Functions ⭐⭐⭐

**Ref** (most common): returns the "primary identifier" for a logical ID, which depends on the resource type — a Parameter returns its value; an S3 Bucket returns its name; an EC2 Instance returns its instance ID; an SNS Topic returns its **ARN**; an IAM Role returns its name. ⚠️ `!Ref` returns something different for every resource type! When an ARN is needed, prefer `!GetAtt`.

**Fn::GetAtt** (fetches a resource attribute): `!GetAtt MyBucket.Arn`, `!GetAtt MyInstance.PrivateIp`, `!GetAtt MyDB.Endpoint.Address`.

**Fn::Sub** (string interpolation, the modern choice): `!Sub 'my-app-${EnvType}-${AWS::AccountId}'` — the recommended default for production, far more readable than Join.

**Fn::Join** (the old-school way to concatenate): `!Join ['', ['arn:aws:s3:::', !Ref MyBucket, '/*']]`.

**Fn::FindInMap** (looks up a mapping table): paired with the `Mappings` section, commonly used for "look up an AMI by region."

**Fn::ImportValue** (cross-stack references) ⭐⭐: references another stack's Output (the source stack must use `Export`). **Key limits**: an exported value that's being imported can't be deleted (every importing stack must be deleted first); can't cross regions; the name can't be dynamic (it must be a literal).

**Fn::If / Conditions** ⭐:

```yaml
Conditions:
  IsProd: !Equals [!Ref EnvType, prod]
Resources:
  MyDB:
    Properties:
      DBInstanceClass: !If [IsProd, db.r5.large, db.t3.micro]
  ProdOnlyAlarm:
    Condition: IsProd    # ⭐ Only created when in prod
```

**AWS::NoValue**: equivalent to "omit this property," used together with `!If`.

**Other helper functions**: `Fn::Select`/`Fn::Split`/`Fn::GetAZs` (fetching an AZ list, splitting a string); `Fn::Cidr` (subdividing a CIDR block); `Fn::Base64` (commonly used for EC2 user data); `Fn::Equals`/`Fn::Not`/`Fn::And`/`Fn::Or` (condition functions).

## Pseudo Parameters ⭐

**= AWS-predefined parameters usable without declaration**: `AWS::AccountId`, `AWS::Region`, `AWS::StackName`, `AWS::StackId`, `AWS::Partition` (aws/aws-cn/aws-us-gov), `AWS::URLSuffix`, `AWS::NoValue`, `AWS::NotificationARNs`.

## Mappings

**= a static lookup table** (cannot be dynamic). A typical use: looking up an AMI ID by region. **Mappings vs. Parameters**: Mappings are static and baked into the template — good for region/environment mapping; Parameters are supplied by the user at deploy time — good for environment-specific configuration. ⭐ Modern practice: use an SSM Parameter instead of Mappings (more dynamic, AWS-maintained).

## Outputs

**= values exported once a stack is created**, importable elsewhere or shown in the console.

```yaml
Outputs:
  VpcId:
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VpcId'   # ⭐ Prefix with the stack name to avoid collisions
```

## Resource Attributes ⭐⭐⭐

**= "meta-behavior" properties attached to a resource** — not part of the resource's own configuration. The five core attributes: **DeletionPolicy**, **UpdateReplacePolicy**, **DependsOn**, **CreationPolicy**, **UpdatePolicy**.

### DeletionPolicy ⭐⭐⭐

**What happens to a resource when the stack is deleted**: `Delete` (default — deletes the resource), `Retain` (keeps the resource), `Snapshot` (snapshots it before deleting — fits RDS/EBS/Redshift/ElastiCache).

**Resources supporting Snapshot**: EC2 Volume, ElastiCache CacheCluster/ReplicationGroup, Neptune DBCluster, RDS DBInstance/DBCluster, Redshift Cluster. **S3/DynamoDB don't support Snapshot** — use Retain for critical production data.

⚠️ A note on S3 with Retain: CFN never empties a bucket — if it still holds objects, deleting the stack will fail (unless the bucket is empty or set to Retain).

### UpdateReplacePolicy ⭐⭐

When an **update** changes a property in a way that forces "replacement" (creating a new resource + deleting the old one), what happens to the old one: `Delete` (default), `Retain`, `Snapshot`.

**Changes that typically trigger replacement**: changing `BucketName` (S3 needs a brand-new bucket), changing `DBInstanceIdentifier` (RDS), changing an EC2's `ImageId`/`InstanceType`.

⚠️ DeletionPolicy and UpdateReplacePolicy are two independent scenarios: DeletionPolicy applies when the *stack* is deleted; UpdateReplacePolicy applies during an *update-triggered replacement*. Production best practice: set both to Retain or Snapshot.

### CreationPolicy (Frequently Tested for EC2/ASG) ⭐

**= waiting for a resource to actively signal before it's considered "created."** Typical use: an EC2 instance isn't really "ready" until its user data script finishes.

```yaml
WebServer:
  Type: AWS::EC2::Instance
  CreationPolicy:
    ResourceSignal:
      Count: 1
      Timeout: PT15M           # ISO 8601 duration
```

**Using cfn-signal**: an AWS-provided helper script the instance calls after completing key setup steps, telling CFN "I'm ready."

### UpdatePolicy (Frequently Tested for Auto Scaling/Lambda) ⭐

**= controlling how CFN handles ASG/Lambda during an update.** The most commonly tested is `AutoScalingRollingUpdate` (replacing ASG instances in batches): `MaxBatchSize`, `MinInstancesInService`, `PauseTime`, `WaitOnResourceSignals`. Fits rolling EC2 fleet updates rather than replacing everything at once.

## Stack Lifecycle ⭐⭐⭐

Three core operations — **Create → Update → Delete** — each moving through IN_PROGRESS/COMPLETE/ROLLBACK states.

**Rollback**: default behavior — any resource creation/update failure automatically rolls the entire stack back to its previous state. Configurable options: `OnFailure: DELETE` (default for new stacks — deletes whatever was already created); `OnFailure: ROLLBACK` (default for updates); `OnFailure: DO_NOTHING` (leaves the failed state in place for debugging — recommended in dev).

**Rollback Configuration**: attach a CloudWatch alarm, and if it fires during an update, the stack rolls back automatically — fits "watch monitoring after deploy, roll back the instant something's wrong."

## Change Sets ⭐⭐

**= a preview of "what would happen if this update ran."** Two modes: `CREATE` (a preview of a new stack), `UPDATE` (a preview of an update to an existing stack).

Typical flow: edit the template → create a change set → review it (Add/Modify/Remove, and whether it would trigger resource replacement) → execute if correct, or delete and retry if there's a problem.

⭐ A change set shows which resources would be "replaced" — protecting against accidental data loss in production. Nothing changes until the change set is actually executed.

## Stack Policy ⭐

**= a JSON policy that prevents specific resources from being accidentally updated or deleted.** ⚠️ Stack Policy ≠ IAM Policy! It only controls "whether a given resource can be changed during a stack update."

Update actions: `Update:Modify`, `Update:Replace`, `Update:Delete`, `Update:*`. Typical use: preventing an accidental change/replacement of a production RDS instance, or a critical security group.

## Drift Detection ⭐

**= checking whether "actual resources" still match the "template definition"** (i.e. whether someone changed something manually). Console: Stack → "Detect drift" → CFN compares every resource's properties → reports which resources have drifted. **Drift states**: IN_SYNC, DRIFTED, NOT_CHECKED, UNKNOWN. ⚠️ not every resource type supports drift detection.

## Helper Scripts (for EC2) ⭐

Python scripts pre-installed on Amazon Linux: `cfn-init` (reads the template's Metadata config to initialize EC2 — installing packages, writing config files, starting services); `cfn-signal` (an instance's way of telling CFN "I'm ready"); `cfn-hup` (a background daemon that listens for stack updates and re-runs cfn-init); `cfn-get-metadata`.

## Wait Conditions (Brief)

**= waiting for an external "signal" before continuing stack creation** (legacy — modern templates use CreationPolicy instead). A presigned URL is handed to an external system, which curls it to report success/failure. Prefer CreationPolicy.

## Termination Protection ⭐

**= preventing a stack from being accidentally deleted.** Once enabled, a `delete-stack` call fails outright until protection is turned off. **Should always be enabled for production stacks.**

## Nested Stacks ⭐⭐

**= nesting one stack inside another (template reuse).**

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

**Advantages**: template reuse; simplifying large templates (avoiding resource-count/size limits); avoiding export-name collisions (a nested stack can pass values via Outputs directly, without needing ImportValue).

**Nested Stack vs. Cross-Stack Reference**: Cross-Stack (ImportValue) is loosely coupled — stacks deploy independently, and an export can't be deleted while still referenced; Nested is tightly coupled — the child stack is managed by the parent, fitting a "private reusable module" pattern.

## CloudFormation Limits

Max resources per template: **500** (use Nested Stacks to split beyond that); max template size loaded from S3 ~1 MB, ~51,200 bytes if submitted directly; Parameters/Outputs/Mappings each capped around 200; stacks per account/region default to roughly 2000 (adjustable).

---

# 2. CloudFormation Part 2

## StackSets ⭐⭐

**= deploying one template across multiple accounts/regions.**

**Key terms**: a **StackSet** (the cross-account template configuration, living in the admin account); a **Stack Instance** (the actual stack in a target account/region).

**Two permission models** ⭐⭐⭐:

**Self-managed permissions**: fits a standalone AWS account (not part of an Organization) — you must manually create `AWSCloudFormationStackSetExecutionRole` in each target account, and `AWSCloudFormationStackSetAdministrationRole` in the admin account.

**Service-managed permissions**: fits AWS Organizations — AWS manages the roles automatically. Traits: creates/manages cross-account roles for you; supports "auto-deploy to every account in the Organization, including newly added ones"; deployment can target the entire Organization or a specific OU; ⚠️ trusted access for CloudFormation must be enabled in Organizations first.

**Deployment options**: concurrent accounts (how many accounts to deploy to in parallel); failure tolerance (how many account failures are acceptable before stopping); region order; auto-deployment (Service-managed only — deploys automatically when a new account joins the OU).

**Classic use cases**: deploying identical IAM roles to every account; deploying AWS Config/GuardDuty/CloudTrail across accounts; deploying a VPC baseline across accounts/regions; deploying alarm/dashboard templates across accounts. **Not a fit** for application code (applications are usually account-isolated).

## Custom Resources ⭐⭐⭐

**= extending CloudFormation with Lambda to do things CFN doesn't natively support.**

Uses: creating third-party resources (a GitHub repo, a Datadog dashboard); seeding initial data at stack creation; calling AWS APIs that CFN doesn't expose; pulling dynamic values from external systems.

**Two types**: Lambda-backed (`AWS::CloudFormation::CustomResource` or `Custom::MyType` — ⭐ nearly everyone uses this), SNS-backed (for third-party webhook services).

```yaml
Resources:
  MyCustomResource:
    Type: Custom::MyResource
    Properties:
      ServiceToken: !GetAtt MyLambdaFunction.Arn
      InputParam1: foo
```

**The Lambda's contract** ⭐⭐: it must handle 3 request types — `Create` (create the resource, return a PhysicalResourceId), `Update` (update the resource), `Delete` (delete the resource). The Lambda must: receive the event from CFN → do the actual work → **PUT a SUCCESS/FAILED response to the event's ResponseURL** (a presigned URL). ⚠️ this response is mandatory — otherwise CFN waits out a full timeout (1 hour) before considering it failed. Required response fields: `Status`, `PhysicalResourceId`, `StackId`/`RequestId`/`LogicalResourceId`, `Data` (optional).

## CloudFormation Registry & Public Extensions (Brief)

**= third-party resource types** (similar to Terraform providers) — AWS-maintained, third-party, or your own private extensions. Not heavily tested on the DVA.

## Macros ⭐

**= transforming a template with Lambda before CloudFormation processes it.** Typical uses: expanding simplified syntax into full CFN, auto-tagging, generating repetitive resources. ⭐ **SAM is essentially a macro**: `AWS::Serverless::Function` isn't a native CFN type — the SAM transform expands it into `AWS::Lambda::Function` plus related resources before deployment.

## Modules (Brief)

**= packaging multiple resources into a "module"** reusable across templates — similar to a nested stack but lighter-weight, appearing as a single resource once deployed.

## CloudFormation Hooks

**= automatically validating a template against a policy before stack creation/update.** Typical uses: enforcing that all S3 buckets are encrypted, enforcing required tags. Essentially a pre-deployment compliance check.

## Resource Import ⭐⭐

**= bringing existing resources under CFN management** (resources that were originally created manually). Uses: managing pre-existing manually created resources under IaC; fixing drift; splitting a stack.

Steps: write a template describing the resources → create an IMPORT-type change set → provide each resource's physical ID → execute the change set → once verified, the resources come under CFN management. ⚠️ limits: only specific resource types are supported; import never modifies the resource itself; after import, normal CFN operations work as usual.

⭐ Different from fixing drift: drift is an already-managed resource that got changed manually; import is bringing an unmanaged resource under management for the first time.

## The S3-Bucket-Not-Empty Trap on Stack Deletion

**The problem**: `DeletionPolicy: Delete` (default) + a bucket that still has objects in it → deleting the stack fails. **Solutions**: manually empty the bucket before deleting the stack; or set `DeletionPolicy: Retain`; or write a Custom Resource Lambda that empties the bucket on stack deletion.

## CloudFormation Best Practices

Security/data protection: add DeletionPolicy + UpdateReplacePolicy (Retain or Snapshot) to production resources; enable Termination Protection; use NoEcho or a Secrets Manager resolve for sensitive parameters; add a Stack Policy to protect critical resources. Deployment flow: always preview with a Change Set; run `cfn-lint`/`cfn-nag` static analysis in CI/CD; separate dev/staging/prod stacks. Template design: prefer small stacks over large ones; decouple with Nested Stacks or Cross-Stack References; parameterize environment differences.

---

# 3. SAM (Serverless Application Model)

## What SAM Is ⭐⭐⭐

**= an extension of CloudFormation specialized for simplifying serverless application deployment.** Core components: the SAM Template (a simplified CFN template with `Transform: AWS::Serverless-2016-10-31`), the SAM CLI (a local development/testing/deployment tool).

**Core value**: minimal syntax (a few lines of YAML define Lambda + API Gateway + DynamoDB); local testing (`sam local invoke`, `sam local start-api`); fast iteration (`sam sync` updates the cloud in seconds); it's built on CFN, so all of CFN's capabilities are still available; official and open-source from AWS.

## The Core SAM Template Structure ⭐⭐⭐

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31    # ⭐ Required

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

**SAM's core simplified types**: `AWS::Serverless::Function` (expands into a Lambda + IAM role + event sources — the core one), `AWS::Serverless::Api` (an API Gateway REST API), `AWS::Serverless::HttpApi` (HTTP API), `AWS::Serverless::SimpleTable` (a DynamoDB table), `AWS::Serverless::StateMachine` (Step Functions), `AWS::Serverless::LayerVersion` (a Lambda layer), `AWS::Serverless::Application` (a nested stack).

**Key insight**: SAM types aren't native to CFN — the SAM transform expands them into CFN resources at deploy time.

## SAM's Event Sources ⭐⭐

A `Function`'s `Events` field defines a trigger in one block: supports `Api` (REST), `HttpApi`, `S3`, `SQS`, `DynamoDB` (Streams), `EventBridgeRule`, `Schedule` (cron), `Kinesis`, and more. Each event automatically wires up the source plus the required IAM permissions.

## SAM Policy Templates ⭐⭐

**= predefined IAM policy shorthand**, so common permissions don't need to be hand-written:

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

Frequently tested ones: `DynamoDBCrudPolicy`, `S3ReadPolicy`, `SQSPollerPolicy`, `SNSPublishMessagePolicy`, `LambdaInvokePolicy`, `KinesisStreamReadPolicy`.

## SAM CLI Commands ⭐⭐⭐

```bash
# Initialize
sam init --runtime python3.12 --name my-app

# Local development + testing
sam build
sam local invoke MyFunction
sam local start-api
sam local start-lambda

# Validate
sam validate --lint

# Deploy
sam deploy --guided    # first time
sam deploy              # subsequently

# Fast iteration
sam sync --watch

# Logs
sam logs -n MyFunction --tail

# Cleanup
sam delete
```

**sam build vs. sam deploy**: `sam build` (installs dependencies, compiles/packages); `sam package` (uploads artifacts to S3); `sam deploy` (deploys to AWS via CFN). ⭐ `sam deploy` internally builds and packages first, then calls CFN.

**SAM Accelerate (sam sync)** ⭐⭐: extremely fast local-to-cloud sync, bypassing the CFN deployment flow. Mechanism: detect local code changes → call the Lambda `UpdateFunctionCode` API directly (bypassing CFN) → code takes effect within seconds. Structural changes (new resources) still fall back to a full CFN deploy. Fits fast iteration during development, not production deployment. ⚠️ these updates never go through CFN, so they cause stack drift!

## Deployment Preferences (Progressive Rollout) ⭐⭐

**= SAM's integration with CodeDeploy for canary/linear deployments.**

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

**Deployment strategy types**: `AllAtOnce` (cuts over 100% at once); `Canary10Percent5/10/15/30Minutes` (10% first, wait N minutes, then the rest); `Linear10PercentEvery1/2/3/10Minutes` (shifts 10% every N minutes).

⭐ Underneath, this uses CodeDeploy's Lambda traffic shifting — an alarm firing triggers an automatic rollback to the previous version.

## The SAM Globals Section ⭐

**= supplying default values shared across every Function/Api/SimpleTable**, cutting down repetition. Individual resources can override values set in Globals.

## SAM vs. CloudFormation ⭐⭐

|  | **CloudFormation** | **SAM** |
|---|---|---|
| Scope | Every AWS resource | **Serverless-specific** (extends CFN) |
| Syntax | Complete but verbose | **Simplified** (much less code for key resources) |
| Local testing | ❌ Not supported | ✅ `sam local` gives one-command simulation |
| Fast deployment | Standard CFN deployment | **sam sync in seconds** |
| Policy shorthand | ❌ Write IAM JSON by hand | ✅ Policy Templates |

**Choosing**: purely serverless applications → SAM; large IaC efforts (VPC + EC2 + databases + more) → CloudFormation; wanting local Lambda testing → SAM; multi-language teams / preferring OOP → CDK.

## Nested SAM Applications

**= referencing someone else's SAM application as a child stack**, via `AWS::Serverless::Application` + `ApplicationId`. **Serverless Application Repository (SAR)**: AWS's marketplace of publicly shared SAM applications, deployable to your account with one click.

---

# 4. AWS CDK — Writing Infrastructure with a Programming Language

## What CDK Is ⭐⭐⭐

**CDK = Cloud Development Kit** — writing infrastructure code in a real programming language (TypeScript/Python/Java/C#/Go), which ultimately compiles into a CloudFormation template.

**Core value**: use a familiar language (for loops, if statements, functions, classes, type checking, IDE autocomplete); genuine code reuse (abstracting infrastructure into classes/libraries/packages); unit-testable infrastructure; strong typing; built on CloudFormation, inheriting all of CFN's advantages.

**Key insight**: CDK doesn't replace CFN — it's an abstraction layer on top of it. What `synth` produces is an ordinary CFN template; stack drift, rollback, change sets, and every other CFN feature still apply; the deployment target is always a CloudFormation Stack.

## CDK v1 vs. v2 ⭐

CDK v2 is the current mainstream. **v2's traits**: a single unified package `aws-cdk-lib` (with services organized into submodules), rather than one package per service (v1's approach); this solved v1's version-compatibility pain points; v1 is no longer supported.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
```

## The Core Three-Layer Structure ⭐⭐⭐

```
App = the container for the entire CDK project
  └── Stack = one CloudFormation Stack
        └── Construct = an AWS resource (can be nested)
```

**App**: the root container for the whole CDK project — one App can contain multiple Stacks.

**Stack**: one CloudFormation Stack — the smallest deployable unit in CDK.

**Construct**: a wrapper around one or more AWS resources — the smallest "building block." Every Construct takes three parameters ⭐: `scope` (its parent), `id` (a logical ID unique within that scope), `props` (configuration, optional).

## L1 / L2 / L3 Constructs ⭐⭐⭐

CDK constructs come in three levels of abstraction — a frequently tested topic:

**L1 — Low-Level Constructs**: map directly to CFN resources, with no abstraction. Class names carry a `Cfn` prefix (e.g. `CfnBucket`); properties correspond exactly to the CFN template; auto-generated from the CFN spec; you write every detail yourself (IAM roles, permissions, dependency wiring). Good for using the newest AWS resources (before an L2 exists) or needing full CFN control.

**L2 — Curated Constructs** (the default recommendation): abstractions carefully designed by the CDK team, with sensible defaults. Class names are simple (`Bucket`, `Function`, `Vpc`); lots of sensible defaults (encryption, permissions, best practices baked in); helper methods (e.g. `bucket.grantRead(lambda)` adds the right IAM permissions automatically). **This is the level used most in everyday development.**

**L3 — Patterns** (advanced): multiple resources bundled into a complete pattern. Named with "Patterns" (e.g. `ecs-patterns`); one construct creates several related resources; ships with a best-practice architecture built in. Example: `ApplicationLoadBalancedFargateService` creates an ECS Fargate Service, Task Definition, ALB, Target Group, Security Group, and all IAM permissions in one line. Good for common architecture patterns, at the cost of reduced flexibility.

**Practical guidance**: default to L2, drop to L1 when you need something newer, and extract repeated patterns into L3.

## The CDK Project Workflow ⭐⭐

```bash
# Create a project
cdk init app --language=typescript

# Bootstrap (once per account/region) ⭐
cdk bootstrap aws://123456789012/us-east-1

# View the CFN template (no deployment)
cdk synth

# Compare deployed state vs. local changes
cdk diff

# Deploy
cdk deploy
cdk deploy --hotswap   # ⭐ fast update

# Watch for changes and auto-deploy
cdk watch

# Tear down
cdk destroy
```

**`cdk synth`**: "synthesizes" the code into a CloudFormation template, output to `cdk.out/` — doesn't actually deploy anything.

**`cdk diff`**: shows the difference between local code and the deployed stack — similar to a CFN change set, but more intuitive.

**`cdk deploy --hotswap`** ⭐: a fast update that bypasses CFN and calls resource APIs directly, supporting Lambda code, State Machine definitions, ECS task definitions. **Development use only** (like sam sync, it causes drift) — production deployments should use plain `cdk deploy`.

## CDK Bootstrap ⭐⭐⭐

**= preparing the "infrastructure" CDK needs in an account/region.**

CDK deployments upload assets (Lambda code, Docker images) to S3/ECR, use a temporary IAM role to execute CFN deployments, and maintain cross-stack references — all of which require this setup ahead of time.

`cdk bootstrap` deploys a CloudFormation stack named **`CDKToolkit`**, containing: an S3 bucket (for assets), an ECR repository (for Docker images), and 5 IAM roles (cfn-exec, deploy, file-publishing, image-publishing, lookup).

**Key points**: ⭐ each account/region combination only needs to be bootstrapped once; ⭐ bootstrapping is mandatory before the first CDK deployment (otherwise it errors out); ⭐ the bootstrap stack's name is always `CDKToolkit`; ⭐ deleting a CDK app does *not* automatically delete the bootstrap stack.

## Synth & Assets ⭐⭐

**Assets** are the "non-CFN" files CDK manages automatically (code, Docker images, etc.). Types: file assets (Lambda code zips, layer zips), Docker image assets.

```typescript
const fn = new lambda.Function(this, 'MyFunction', {
  code: lambda.Code.fromAsset('./lambda-code'),  // ⭐ auto-packaged and uploaded
});
```

Automatic flow: `cdk deploy` → packages the assets → uploads them to the bootstrap S3/ECR → the CFN template references the resulting URI → the stack deploys, pulling from S3/ECR. You never write S3 bucket-creation logic, Docker push commands, or ARN references — CDK handles it all.

## Context ⭐

**= "runtime values that need to be looked up during synth"** (like an AMI ID or VPC ID). Mechanism: `cdk synth` queries the AWS API → caches the result in `cdk.context.json` → subsequent synths use the cache (fast and deterministic).

`ec2.Vpc.fromLookup()` triggers a context lookup. The result is stored in `cdk.context.json`, which should be committed to git to keep the team/CI consistent. Clearing the cache: `cdk context --clear`.

## Aspects (Cross-Cutting Concerns) ⭐

**= applying the same transformation/validation across an entire construct tree.** Typical uses: tagging every resource, enforcing bucket encryption everywhere, validating naming conventions. Scope: applied to the App → affects every stack; applied to a Stack → affects that stack's resources; applied to a Construct → affects only that subtree.

## Permissions / Grants ⭐⭐

**= L2 construct helper methods that configure IAM automatically.**

```typescript
bucket.grantRead(fn);
bucket.grantWrite(fn);
table.grantReadWriteData(fn);
queue.grantSendMessages(fn);
topic.grantPublish(fn);
```

What happens underneath: appropriate IAM permissions are added to fn's execution role; a resource policy is added to the target resource if needed; KMS key permissions are handled automatically (if the resource is encrypted).

⭐ vs. CFN/SAM: CFN requires hand-written IAM JSON; SAM has Policy Templates, which are simplified presets; CDK's grants generate minimal permissions automatically based on resource type — the most modern approach.

## Cross-Stack References ⭐

CDK makes cross-stack references nearly transparent — you reference a property exposed by another stack directly (like `stack1.bucket.bucketName`), and CDK automatically adds an Output+Export to stack1 and an `Fn::ImportValue` to stack2, invisible to you. ⭐ the underlying limits are inherited from CFN: no cross-region references, and a referenced export can't be deleted.

## Testing

CDK supports unit-testing infrastructure code using `Template.fromStack(stack)` plus `template.resourceCountIs(...)` / `template.hasResourceProperties(...)`. Types: snapshot tests, fine-grained tests, validation tests.

## CDK Pipelines ⭐

**= a self-evolving CI/CD pipeline defined with CDK.** Its core trait — **self-mutating**: the pipeline updates itself; when a change to the pipeline's definition is pushed, the next build updates the pipeline automatically, with no manual rebuild needed. Benefits: multi-stage deployment (dev→staging→prod), built-in cross-account deployment, automatic approval steps.

## CDK's Main CLI Commands, Summarized ⭐⭐⭐

`cdk init` (initializes a project), `cdk bootstrap` (prepares an account/region), `cdk synth` (generates the CFN template), `cdk diff` (compares local vs. deployed), `cdk deploy` (deploys), `cdk deploy --hotswap` (fast update), `cdk watch` (watches and auto-deploys), `cdk destroy` (deletes), `cdk ls` (lists all stacks in the app), `cdk context` (manages the context cache).

## CDK vs. CFN vs. SAM — The Ultimate Comparison ⭐⭐⭐

|  | **CFN** | **SAM** | **CDK** |
|---|---|---|---|
| Syntax | YAML/JSON | YAML (a CFN extension) | **TypeScript/Python/Java/C#/Go** |
| Scope | All of AWS | Serverless only | All of AWS |
| Reuse | Nested Stack | SAM templates | **Genuine code reuse** (classes, functions) |
| IAM syntax | Hand-written JSON | Policy Templates | **Automatic via `grantRead()`** |
| Type checking | ❌ | ❌ | **✅ IDE catches errors** |
| Local testing | ❌ | sam local | Snapshot + assertion tests |
| Fast iteration | ❌ | sam sync | **cdk deploy --hotswap / cdk watch** |
| Bootstrap | Not needed | Not needed | **Required** (per account/region) |

**Choosing**: simple IaC with a team unfamiliar with programming → CFN; purely serverless apps needing local testing → SAM; large/complex/multi-team IaC → CDK; wanting type checking + IDE assistance → CDK; abstraction/reuse/cross-team shared patterns → CDK.

⭐ Modern trend: for new projects, CDK is often preferred over SAM, which is often preferred over plain CFN — though this really depends on the team and the specific situation.

## Other CDK Ecosystem Tools (Brief)

**CDK for Terraform (cdktf)**: uses CDK syntax to generate Terraform configuration. **CDK for Kubernetes (cdk8s)**: uses CDK to write Kubernetes YAML. **Constructs Hub**: an AWS-maintained search engine for CDK constructs. None of these are tested on the DVA — worth knowing they exist, nothing more.
