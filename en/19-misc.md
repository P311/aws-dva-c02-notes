# Module 19: Misc & Emerging Services (AppConfig / Amplify / ACM / AppSync)

> **Covered**: May 24
>
> 中文版本：[`zh/19-misc.md`](../zh/19-misc.md)
>
> The real DVA-C02 exam includes a handful of questions on these smaller services — this module quickly covers the remaining gaps.

---

# 1. AWS AppConfig ⭐⭐

## What AppConfig Is

**= a feature within AWS Systems Manager.** Core value: dynamic management of application configuration/feature flags, changeable without restarting or redeploying the application.

**Typical uses**: feature flags (gradual rollouts), application configuration (API endpoints, timeout values), A/B testing, operational throttling/circuit-breaker switches, allow-lists/block-lists.

## vs. Parameter Store / Secrets Manager ⭐⭐⭐

|  | **AppConfig** | **Parameter Store** | **Secrets Manager** |
|---|---|---|---|
| Purpose | Config + feature-flag rollout management | Simple key-value config | Secrets (with automatic rotation) |
| Change control | ✅ deployment strategies (gradual, auto-rollback) | ❌ takes effect immediately | A rotation process |
| Validation | ✅ JSON Schema/Lambda | ❌ | ❌ |
| Automatic rollback | ✅ triggered by a CloudWatch alarm | ❌ | ❌ |
| Size limit | 1 MB | 4/8 KB | 64 KB |

**Judgment call**: "progressive config rollout + auto-rollback on alarm" → AppConfig; simple environment variables → Parameter Store; DB passwords with automatic rotation → Secrets Manager.

## Core AppConfig Concepts ⭐⭐

```
Application (the project)
  └─ Environment (dev / staging / prod)
      └─ Configuration Profile (freeform or feature-flags type)
          └─ Deployment (rolled out via a deployment strategy)
```

**Configuration Profile types**: **Freeform** (free-form JSON/YAML/text config); **Feature Flags** (a dedicated feature-flag format with enabled/attributes).

**Where configuration is stored**: freeform can live in the AppConfig hosted store/SSM Parameter Store/SSM Document/S3/CodePipeline; **Feature Flags can only use the hosted configuration store**.

## Validators ⭐

**= automatically validating configuration correctness before deployment.** Two types: **JSON Schema** (structural validation), **Lambda Function** (arbitrary custom validation). ⭐ a failed validation means the deployment is rejected and never reaches production.

## Deployment Strategy ⭐⭐

| Preset | Behavior |
|---|---|
| `AppConfig.AllAtOnce` | Deploys everything at once (for testing) |
| `AppConfig.Linear50PercentEvery30Seconds` | 50% every 30 seconds |
| `AppConfig.Canary10Percent20Minutes` | 10% → wait 20 minutes → 90% |

**Custom parameters**: Growth Type (Linear/Exponential), Growth Factor, Deployment Duration, **Final Bake Time** (a final observation window during which rollback remains possible).

## Automatic Rollback ⭐⭐

**= rolling back to the previous version automatically when a CloudWatch alarm fires**:

```
Deploy v2 → 10% → 50% → 100%
              ↓
      A 5xx alarm fires
              ↓
      Automatic rollback to v1
```

⚠️ once the bake time ends, the deployment is marked "deployed" and is no longer monitored.

## How Applications Fetch Configuration ⭐⭐

**Option 1: the AppConfig Agent** (a Lambda Extension/sidecar, recommended) — the app calls a local HTTP endpoint → the AppConfig Agent → AppConfig. Benefits: local caching, automatic polling for updates, minimal application code.

**Option 2: calling the SDK directly** — must use the `start_configuration_session` + `get_latest_configuration` pattern (the older `get_configuration` is deprecated).

## AppConfig Use Cases

Progressive config rollout with automatic rollback → AppConfig; simple string config → Parameter Store; secure secrets with automatic rotation → Secrets Manager; Lambda needing low-latency config reads → AppConfig + Lambda Extension; validating config format before deployment → AppConfig + Validators.

---

# 2. AWS Amplify ⭐

## What Amplify Is

**= a full-stack development platform for frontend and mobile apps**, comprising two distinct products:

**Amplify Hosting** (formerly Amplify Console): hosting for static sites/SSR applications with built-in CI/CD, similar to Netlify/Vercel. Git integration (GitHub/GitLab/Bitbucket/CodeCommit); a push triggers automatic build + deploy; built-in CDN + custom domain + automatic SSL certificate provisioning; supports SSR (Next.js, Nuxt).

**Amplify CLI / Libraries**: a frontend/mobile SDK supporting React, React Native, Angular, Vue, iOS, Android, and Flutter, simplifying integration with Cognito, AppSync, S3, and Lambda.

⭐ The DVA mainly tests Amplify Hosting.

**Key Amplify Hosting features**: Git integration (push-triggered automatic build/deploy); **Preview environments** (one independent environment per PR/branch); atomic deployments (automatic rollback on failure); custom domains + SSL (auto-provisioned via ACM); password protection (branch-level); pull request previews (automatic preview deployment per PR); monorepo support (multiple apps in one repo).

**An `amplify.yml` example**:

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: build
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

**Judgment calls**: a static site with automatic deploys from GitHub push + a CDN → Amplify Hosting; a preview environment per PR → Pull Request Previews; a Next.js/Nuxt SSR application → Amplify Hosting; a React + Cognito + AppSync full-stack app → Amplify CLI + Libraries.

---

# 3. AWS Certificate Manager (ACM) ⭐

## What ACM Is

**= AWS's managed SSL/TLS certificate service.** Core value: ✅ free (public certs attached to AWS services); ✅ automatic renewal; ✅ integrates with CloudFront/ALB/API Gateway/ELB; ✅ DNS/email validation.

**ACM certificate types**: **Public** (AWS-issued, free, for public sites/APIs); **Private** (AWS Private CA, a monthly fee plus per-certificate cost, for internal networks/enterprise PKI); **Imported** (bring your own certificate, free to import, for existing third-party certificates).

## ACM Validation Methods ⭐

**DNS validation** (recommended): ACM provides a CNAME record to add to DNS (one click if using Route 53); ✅ renews automatically.

**Email validation**: ACM emails the domain owner, who must click a link manually; ❌ renewal must be done manually.

⭐ use DNS validation in production — email validation is prone to lapsing.

## Key ACM Exam Points ⭐⭐⭐

### 1. The ACM Certificate Used by CloudFront Must Be in us-east-1!

CloudFront is a global service, but its ACM certificate must be requested in us-east-1 — certificates from any other region simply won't work with CloudFront. **A classic DVA trick question.**

### 2. Regional Requirements by Service

| Service | Region requirement |
|---|---|
| CloudFront | **us-east-1** |
| ALB/NLB | Same region |
| API Gateway (Regional) | Same region |
| API Gateway (Edge-optimized) | **us-east-1** |
| Elastic Beanstalk | Same region |
| **EC2** | ❌ not directly supported |
| **S3** | ❌ must front it with CloudFront |

⭐ EC2/S3 can't use ACM directly — it has to go through CloudFront/ALB.

### 3. Automatic Renewal

Renewal attempts begin a while before expiration; DNS validation + still attached to an AWS service + the DNS record still exists → renews automatically; email validation → cannot renew automatically.

## ACM Quick Reference

AWS's free SSL certificate → ACM; the region for CloudFront's certificate → **us-east-1** (essential to remember!); the region for a Regional API Gateway certificate → same region; for an Edge-optimized API Gateway certificate → us-east-1; the validation method needed for auto-renewal → DNS validation; where ACM can't be used directly → EC2/S3; internal/enterprise PKI certificates → ACM Private CA; wanting to use an existing third-party certificate on AWS → ACM Imported.

---

# 4. AWS AppSync ⭐

## What AppSync Is

**= AWS's managed GraphQL service.** vs. API Gateway: API Gateway handles REST/HTTP/WebSocket; AppSync handles **GraphQL**. GraphQL's advantages: a single endpoint where the client declares which fields it needs, plus built-in real-time (subscriptions).

## Core AppSync Concepts ⭐⭐

**GraphQL Schema → Resolver → Data Source.** Data sources: DynamoDB, RDS Aurora, OpenSearch, Lambda, an HTTP endpoint, EventBridge, or None.

## Resolver Types ⭐⭐

**Unit vs. Pipeline**: a Unit resolver calls a single data source; a Pipeline resolver runs multiple steps in sequence (auth → the main call → post-processing).

**Resolver language**: VTL (Apache Velocity, the older option) vs. JavaScript (APPSYNC_JS, the newer option and the recommended choice for new projects).

**Direct Lambda Resolver**: attaches directly to a Lambda with no VTL/JS mapping — no mapping template needed, and the Lambda receives the full GraphQL context.

## Subscriptions (Real-Time) ⭐⭐

**= GraphQL real-time subscriptions**, built on **WebSocket**:

```graphql
subscription OnNewMessage {
  newMessage(roomId: "abc") {
    id
    content
    user
  }
}
```

Mechanism: the client connects over WebSocket → a Mutation triggers a push to subscribers → AppSync manages the connections automatically. Classic use cases: chat, real-time dashboards, collaborative editing, push notifications.

## Authorization Modes ⭐⭐

AppSync supports 5 authentication methods, and a single API can combine several: **API Key** (simple, for development/public APIs, valid up to roughly 365 days); **AWS IAM** (service-to-service, SigV4-signed); **Cognito User Pools** (for logged-in web/mobile app users); **OpenID Connect (OIDC)** (a third-party IdP like Auth0 or Okta); **Lambda Authorizer** (custom authorization logic).

## Field-Level Permissions

```graphql
type Post @aws_cognito_user_pools {
  id: ID!
  title: String!
  internalNotes: String @aws_iam        # readable by IAM only
  views: Int @aws_api_key               # also readable via API key
}
```

Schema directives: `@aws_api_key`, `@aws_iam`, `@aws_cognito_user_pools`, `@aws_oidc`, `@aws_lambda`.

## AppSync Caching

**= server-side caching of GraphQL responses.** Full request caching or per-resolver caching; TTL 1-3600 seconds; backed by Redis (fully managed).

## Offline Support + Conflict Resolution

**Amplify DataStore + AppSync**: offline access, automatic sync, and conflict resolution (server-wins/client-wins/Lambda). Fits mobile applications with unreliable networks.

## AppSync Limits

Single GraphQL request timeout roughly **30 seconds**; response size roughly **1 MB**.

## AppSync Quick Reference

AWS's GraphQL service → AppSync; real-time GraphQL updates → Subscription + WebSocket; AppSync data sources → DynamoDB/RDS/OpenSearch/Lambda/HTTP/None/EventBridge; resolver language → VTL (legacy) or JavaScript (new, recommended); calling Lambda without writing a mapping → Direct Lambda Resolver; field-level access control → schema directives; mobile app + offline + conflict resolution → AppSync + Amplify DataStore; the 5 auth types → API Key/IAM/Cognito/OIDC/Lambda.

---

# 5. This Module's Combined Quick Reference

Dynamic config + progressive rollout + failure rollback → AppConfig; AppConfig vs. Parameter Store → AppConfig has deployment strategies + auto-rollback, Parameter Store takes effect immediately; feature-flag format validation → AppConfig Validators; a frontend static site with Git-push auto-deploy → Amplify Hosting; PR preview environments → Amplify Pull Request Previews; AWS's free SSL → ACM (Public); ACM for CloudFront → must be us-east-1 (a classic!); ACM auto-renewal → DNS validation + still attached; using ACM with EC2/S3 → ❌ (must go through CloudFront/ALB); a GraphQL API → AppSync; WebSocket-based real-time push → AppSync Subscriptions; AppSync's newer recommended resolver language → JavaScript (APPSYNC_JS); calling Lambda without a mapping → Direct Lambda Resolver; GraphQL field-level permissions → schema directives.
