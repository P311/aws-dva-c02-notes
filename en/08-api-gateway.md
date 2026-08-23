# Module 8: API Gateway (REST / HTTP / WebSocket + Authz)

> **Covered**: May 6, 7
>
> 中文版本：[`zh/08-api-gateway.md`](../zh/08-api-gateway.md)

---

## 1. API Gateway Part 1 — REST / HTTP / WebSocket, Integrations, Stages, CORS

### What API Gateway Is

**= AWS's fully managed API service**, acting as an application's "front door": creating, publishing, maintaining, monitoring, and securing APIs; supporting RESTful, HTTP, and WebSocket APIs; handling traffic management, authorization, and versioning automatically; integrating deeply with the AWS ecosystem (Lambda, Cognito, IAM, CloudWatch, X-Ray, WAF, etc.).

**Typical uses**: a RESTful API backed by Lambda (serverless architecture); an API layer in front of traditional EC2/ECS applications; WebSocket real-time bidirectional communication (chat, gaming, push); exposing AWS services to a frontend; API monetization (API keys + usage plans).

### The Three API Types Compared ⭐⭐⭐

|  | **REST API** | **HTTP API** | **WebSocket API** |
|---|---|---|---|
| Protocol | HTTP/HTTPS (stateless) | HTTP/HTTPS (stateless) | WebSocket (stateful, bidirectional) |
| Price | Higher | **~70% cheaper** | Billed by message + connection duration |
| Latency | Higher | **~60% lower than REST** | Low, persistent connection |
| Caching | ✅ Stage cache | ❌ | ❌ |
| API Keys + Usage Plans | ✅ | ❌ | ❌ |
| Per-client throttling | ✅ (usage plan) | ❌ (stage-level only) | ❌ |
| Request validation | ✅ | ❌ | / |
| Mapping Templates (VTL) | ✅ | ❌ (simple variable substitution only) | ✅ |
| WAF integration | ✅ | ❌ | ❌ |
| Authorizers | IAM, Cognito, Lambda | IAM, JWT, Lambda | IAM, Lambda |
| Endpoint types | Edge / Regional / Private | Regional only | Regional only |
| Canary deployment | ✅ | ❌ | ❌ |
| Private API (VPC) | ✅ (Private endpoint) | ❌ | ❌ |
| Bidirectional | ❌ One-way | ❌ One-way | ✅ **Bidirectional** (server pushes) |

> Check AWS's current pricing page for exact figures — the numbers above are relative comparisons.

### Choosing Between Them ⭐

**Use REST API when**: you need API keys + per-client throttling (multi-tenant APIs, API monetization); you need caching; you need request validation; you need canary deployments; you need WAF; you need edge-optimized endpoints (global users); you need private endpoints (VPC-internal access); you need complex mapping templates.

**Use HTTP API when**: it's a simple Lambda backend; you want lower cost and lower latency; you're mainly using JWT authorization (Cognito or a third-party OAuth 2.0 provider); you don't need REST's advanced features.

**Use WebSocket API when**: you need real-time bidirectional communication (chat, collaborative editing); the server needs to push proactively (notifications, price updates); you need multi-client broadcast (live streaming, gaming).

**Pricing note**: HTTP API bills in 512 KB increments while REST API doesn't — for large request/response payloads, HTTP API can actually end up more expensive. In exam scenarios, "save money" + "simple RESTful API" usually points to HTTP API.

### Endpoint Types (REST API Only) ⭐⭐

REST API requires choosing an endpoint type at creation (HTTP API only offers Regional):

**1. Edge-Optimized (default)**: requests route to the nearest CloudFront edge location; AWS manages the CloudFront configuration automatically; suits globally distributed clients; still deployed in a single region underneath.

**2. Regional**: clients connect directly to the API Gateway in a specific region, with no CloudFront in between. Fits: clients in the same region; wanting to manage your own CloudFront distribution; multi-region deployment with Route 53 latency-based routing.

**3. Private**: the API is only reachable from inside a VPC, via an interface VPC Endpoint (PrivateLink) — completely hidden from the internet. Fits internal microservices and strict compliance scenarios.

**Judgment calls**: "clients are spread globally" → Edge-optimized; "clients are EC2 instances in the same region" → Regional (avoids an unnecessary CloudFront hop); "API must never be exposed to the internet, VPC access only" → Private + Interface VPC Endpoint.

### Integration Types ⭐⭐⭐

| Type | Backend | Description |
|---|---|---|
| **AWS_PROXY** (Lambda Proxy) | Lambda | **The recommended approach** — the whole request is passed to Lambda as-is; Lambda returns an object with status/headers/body; no mapping template needed |
| **AWS** (Lambda Custom / AWS Service) | Lambda or any AWS service | Requires a mapping template to transform request/response, e.g. calling DynamoDB/Kinesis directly, bypassing Lambda |
| **HTTP_PROXY** | Any HTTP endpoint | Forwards the whole request unmodified |
| **HTTP** | Any HTTP endpoint | Forwards the request, supports mapping templates for transformation |
| **MOCK** | No backend | API Gateway returns a static response directly (placeholder, CORS preflight) |
| **VPC Link** | An NLB/ALB inside a VPC | Private integration, API GW → VPC-internal service |

**Lambda Proxy vs. Lambda Custom**:

Lambda Proxy (AWS_PROXY) — the whole HTTP request is packaged into an event JSON passed to Lambda, which must return a fixed format:

```json
{
  "statusCode": 200,
  "headers": { "Content-Type": "application/json" },
  "body": "{...}"
}
```

✅ No mapping template needed; ✅ Lambda sees the raw request in full; ❌ tightly coupled to API Gateway's event format.

Lambda Custom (AWS) — requires writing a mapping template (VTL) to transform the request/response; ✅ flexible; ❌ complex, VTL templates are hard to maintain.

**AWS Service Integration**: API Gateway calls an AWS service directly, no Lambda required. Fits: API → DynamoDB (simple CRUD), API → Kinesis, API → SQS, API → S3, API → Step Functions. Pros: no Lambda cost, low latency (no cold start). Cons: requires writing VTL, can't handle complex business logic.

**VPC Link**: REST API's VPC Link attaches to an NLB; HTTP API's VPC Link is more flexible and can attach to an ALB, NLB, or Cloud Map. Typical scenario: API Gateway exposes a public API backed by a private ECS service in a VPC.

### Stages and Deployments ⭐⭐⭐

**A Deployment is an immutable snapshot of an API's configuration.** Changes to the API don't take effect until you Deploy; every Deploy creates a new immutable Deployment; a Deployment attaches to a Stage (a named reference, like dev/staging/prod).

**Key traits**: each Stage has its own URL: `https://{api-id}.execute-api.{region}.amazonaws.com/{stage-name}`; a Stage can independently configure caching, throttling, logging, stage variables, and X-Ray; one Deployment can attach to multiple Stages; **changes to the API don't take effect until redeployed** (a common exam point and a common bug).

**Stage Variables** ⭐⭐: key-value pairs bound to a Stage, similar to environment variables. Most common use — pointing different stages at different Lambda aliases:

```
API's Lambda integration configuration:
  Lambda ARN: arn:aws:lambda:...:function:myFn:${stageVariables.lambdaAlias}

Stage "dev"   → stageVariables.lambdaAlias = "dev"  → calls myFn:dev (alias)
Stage "prod"  → stageVariables.lambdaAlias = "prod" → calls myFn:prod (alias)
```

**Trap**: if a stage variable is used to select a Lambda alias, API Gateway needs permission to invoke *each* alias — a resource-based policy (`lambda:InvokeFunction`) must be added per alias, or some stages will 500.

**Canary Deployment** ⭐⭐: splits traffic within a single Stage across two deployments, for gradual rollout. **REST API only — not supported on HTTP API.** Same Stage URL, traffic split by percentage between production and canary; the canary can have independent stage variables and independent caching. Flow: create canary settings (percent = 10) → redeploy → monitor metrics → promote the canary if healthy, or set percent to 0 if not.

⚠️ If a question asks about canary deployment and both REST and HTTP API appear as options, choose **REST API**.

### Mapping Templates (REST API Only) ⭐

**= VTL (Velocity Template Language) templates that transform requests/responses at the API GW layer.** Uses: request/response transformation, hiding backend structure, normalizing error responses.

Key built-in variables: `$input.body`, `$input.json('$.fieldName')`, `$input.params('paramName')`, `$context.requestId`, `$context.identity.sourceIp`, `$stageVariables.xxx`.

HTTP API doesn't support full VTL — only simple `${request.path.xxx}` substitution.

### CORS ⭐⭐⭐

**= Cross-Origin Resource Sharing**, a browser security mechanism restricting cross-origin requests. A cross-origin browser call first sends a **preflight OPTIONS** request; the API must return the correct CORS headers before the browser allows the actual request through.

Key headers: `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Allow-Credentials`, `Access-Control-Max-Age`.

**Configuring CORS on REST API** (complex): add an OPTIONS method to every resource (using MOCK integration); add CORS headers on both the method response and the integration response; every method's 200 response must also carry `Access-Control-Allow-Origin`; requires redeployment. The console has an "Enable CORS" shortcut, but existing method responses still need manual updates.

**Configuring CORS on HTTP API** (simple): configured once at the API level; preflight OPTIONS is handled automatically; no redeployment needed. ⚠️ Once CORS is configured, API Gateway ignores any CORS headers returned by the backend.

**Common mistakes**: ① Lambda Proxy Integration responses must also carry CORS headers (otherwise the actual request gets blocked by the browser even if preflight succeeded); ② a Custom Authorizer requiring auth on the OPTIONS preflight (a browser's preflight never carries an Authorization header, so it 401s immediately — fix: don't attach the authorizer to the OPTIONS method); ③ `Allow-Credentials: true` cannot be combined with `Allow-Origin: *` (forbidden by browser spec).

### Throttling ⭐⭐

**Default quota (account-wide, across all APIs in a region)**: tens of thousands of RPS steady-state plus thousands of burst capacity (token bucket — some older regions default lower). Algorithm: token bucket — each token represents one request, tokens refill at the rate limit up to the burst cap, and requests are rejected with 429 when no tokens remain. The account-level rate can usually be raised on request, but burst capacity typically cannot.

**Throttling layers**: ① per-client/per-method limits (usage plan) ② per-method overall limits (stage config) ③ stage-level limits ④ account-level limits ⑤ AWS regional limits (a hard ceiling). Exceeding any layer throttles the request.

**Usage Plans + API Keys** (REST API only) ⭐⭐⭐: gives different clients different rate limits and quotas. Structure: API Key (per-client) → Usage Plan (throttle + quota settings) → API Stages. Typical scenario: API monetization (Free/Pro/Enterprise tiers).

⚠️ **HTTP API does not support API Keys + Usage Plans** (stage-level throttling only).

**Trap**: an API Key is not authentication! Anyone with the key can use it. Real authorization needs IAM/Cognito/Lambda Authorizer — API Keys exist purely for metering/throttling.

### Integration Timeout

| API type | Default timeout | Notes |
|---|---|---|
| **REST API** | 29 seconds | Can be raised on request (a relatively recent change — check current docs) |
| **HTTP API** | 30 seconds | Hard ceiling |
| **WebSocket API** | 29 seconds (per route) | Hard ceiling |

**Trap**: Lambda can run up to 15 minutes, but a synchronous API Gateway → Lambda call is capped at 29-30 seconds regardless. The correct architecture for long-running tasks: the client POSTs → API GW → Lambda fires asynchronously and returns a jobId immediately → the client polls GET `/status/{jobId}`; or use Step Functions/SQS + a worker.

---

## 2. API Gateway Part 2 — Auth, Caching, Custom Domain, Monitoring

### Authorizers ⭐⭐⭐

API Gateway supports **four authorizer types**:

| Authorizer | REST API | HTTP API | WebSocket API |
|---|---|---|---|
| **IAM** (SigV4 signing) | ✅ | ✅ | ✅ |
| **Cognito User Pools** | ✅ (built-in) | ✅ (via JWT) | ❌ |
| **Lambda Authorizer** (custom) | ✅ (TOKEN + REQUEST) | ✅ (REQUEST only) | ✅ |
| **JWT Authorizer** (OAuth/OIDC) | ❌ | ✅ (native) | ❌ |

**1. IAM Authorizer**: callers must be an AWS IAM principal, signing requests with SigV4. Suits internal applications and same-account microservice calls. An IAM policy controls `execute-api:Invoke` permission. Pros: fully leverages AWS IAM, SigV4 signing is built into the SDK. Cons: clients must hold AWS credentials, and signing from browser JS is awkward.

**2. Cognito User Pools Authorizer** (built into REST API only): users log into Cognito, get a JWT, and send it in the Authorization header. ⚠️ **REST API + Cognito Authorizer requires the ID token** (not the access token) — using the wrong one returns 401. HTTP API's JWT Authorizer can use an access token (supporting OAuth scopes).

**3. Lambda Authorizer** ⭐⭐⭐: a custom Lambda function handling authorization — the most flexible option. Two types (REST API): **TOKEN Authorizer** (the client passes a token in a header, an optional RegEx does preliminary format validation, well suited to JWT/Bearer tokens); **REQUEST Authorizer** (Lambda receives the entire request and can judge based on multiple parameters). ⚠️ **HTTP API supports REQUEST only.**

A Lambda Authorizer must return an IAM policy:

```python
def lambda_handler(event, context):
    token = event['authorizationToken']
    if not is_valid(token):
        raise Exception('Unauthorized')
    return {
        'principalId': 'user123',
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [{
                'Effect': 'Allow',
                'Action': 'execute-api:Invoke',
                'Resource': event['methodArn']
            }]
        },
        'context': {'userId': '123', 'role': 'admin'}
    }
```

Throwing an exception → the client gets 401; returning a Deny policy → 403; returning Allow → the request proceeds.

**Lambda Authorizer Caching** ⭐: enabled by default. Default TTL **300 seconds (5 minutes)**, max TTL **3600 seconds (1 hour)**, TTL=0 disables caching. Cache key: for TOKEN authorizers, the token source header value; for REQUEST authorizers, the combination of all identity sources.

**Trap**: ⚠️ the cache is **per stage**, not per resource/per method! The same token calling `/cats` and then `/dogs` returns the *same cached policy*. This means **the returned policy must cover every possible method in the API** (using a wildcard `arn:.../*/*/*`), or you'll get bugs from stale permissions leaking across endpoints. Alternatively, set TTL to 0 (calls the authorizer every time — expensive but precise). Use the `flush-stage-authorizers-cache` API to clear an entire stage's cache at once.

**4. JWT Authorizer** (HTTP API only) ⭐⭐: API Gateway natively validates JWTs issued by any OIDC/OAuth 2.0 provider. Fits Cognito User Pools, Auth0, Okta, and similar. Mechanism: the client sends a JWT in the Authorization header → API Gateway validates the signature, issuer, audience, and expiration → on success, claims are injected into `$context.authorizer.claims.xxx`; OAuth Scopes can add finer-grained authorization. **Fully managed, zero Lambda invocations, free, extremely fast.**

**Judgment call**: HTTP API + Cognito + simple authorization → JWT Authorizer (not Lambda Authorizer).

### Caching (REST API Only) ⭐⭐

**= caching backend responses at the API Gateway layer.** ⚠️ **HTTP API doesn't support caching at all.**

Key parameters: default TTL 300 seconds, max TTL 3600 seconds; max cached response size **1 MB**; billed hourly regardless of usage, not covered by the free tier; encryption is optional (AES-256 at rest); configured at the Stage level, overridable per method.

Default behavior: enabling caching on a stage automatically caches every GET method; POST/PUT/DELETE aren't cached unless explicitly enabled.

The cache key defaults to the full URL (path + query string); it can be customized per method (choosing which path/query/header/stage-variable values factor into the key).

**Cache invalidation**: the client sends `Cache-Control: max-age=0` to force a refresh. By default anyone can invalidate the cache (a potential abuse vector) — best practice is enabling "Require authorization for cache invalidation," which requires `execute-api:InvalidateCache` IAM permission.

Monitoring: `CacheHitCount`/`CacheMissCount`; a healthy hit rate is above 80%.

**Traps**: ① changes to cache configuration require a redeploy to take effect; ② POST/PUT are never cached automatically; ③ after a backend code change, the cache may still return stale results — lower the TTL or invalidate proactively.

### Custom Domain Names ⭐⭐

**= replacing the default `xxx.execute-api.region.amazonaws.com` with your own domain.** Requirements: a registered domain; an SSL/TLS certificate requested or imported via ACM; base path mapping configuration; a DNS alias/CNAME record pointing at API Gateway.

**Key trap — endpoint type vs. ACM region** ⭐⭐: **Edge-optimized custom domains require the ACM certificate to live in us-east-1** (since it's backed by CloudFront); **Regional custom domains require the certificate in the API's own region**. A common mistake: deploying an edge-optimized API in another region and also creating the ACM certificate there — API Gateway then can't find the certificate and deployment fails.

Rule of thumb: global = us-east-1 (CloudFront is a global resource); Regional = same region as the API.

### Logging + Monitoring ⭐⭐

**CloudWatch Metrics** (automatic, free): `Count`, `4XXError`, `5XXError`, `Latency` (total), `IntegrationLatency` (backend-only time), `CacheHitCount`/`CacheMissCount`.

**Diagnosing**: `Latency` ≈ `IntegrationLatency` → the backend is slow; a large gap between them → API Gateway itself is adding overhead (rare); high `5XXError` → backend errors; high `4XXError` → client-side issues.

**Two kinds of logs**: **Execution Logs** (API Gateway's internal processing detail, written to `API-Gateway-Execution-Logs/{api-id}/{stage}`, log levels OFF/ERROR/INFO); **Access Logs** (a fully custom format, similar to a web server access log, can be written to CloudWatch Logs or Kinesis Data Firehose — good for shipping to S3 or long-term analysis).

**Detailed CloudWatch Metrics** (method-level, an extra cost): by default only stage/API-level metrics are free; enabling this gives each method its own metrics for fine-grained diagnosis, but is billed per method.

**X-Ray integration**: enabling X-Ray Active Tracing on a stage makes API Gateway generate a trace ID for every request automatically, instrumenting the API GW segment; combined with Lambda Active Tracing, you get an end-to-end trace (Client → API GW → Lambda → DynamoDB).

### Request / Response Validation (REST API Only) ⭐

**= API Gateway validating request format before calling the backend.** ⚠️ HTTP API doesn't support this.

What's validated: required parameters/headers; body schema validation (JSON Schema). Benefits: invalid requests are rejected directly by API GW (400) without invoking Lambda, saving money; less validation code needed in the backend.

Configuration: create a Model (a JSON Schema describing the request body) → enable a validator on the method, pointing at the model → choose the validator type (body only / query+headers only / everything).

### Gateway Responses

**= customizing the content of API Gateway's own error responses** (401, 403, 429, 503, 504, etc.). You can modify the status code, response body/headers, and add CORS headers. **Important**: error responses need CORS headers too, or the browser will block even the error itself — the frontend won't even see a "401 unauthorized."

### WebSocket API Deep Dive

**Mechanism**: the client sends an HTTP upgrade request → triggers the `$connect` route → connection established; the client sends a message → routed to the matching route → invokes Lambda/an AWS service; the server can push messages at any time; disconnecting triggers `$disconnect`.

**Three special routes**: `$connect`, `$disconnect`, `$default` (unmatched messages).

**Server-initiated push**:

```python
client = boto3.client('apigatewaymanagementapi',
    endpoint_url='https://abc123.execute-api.us-east-1.amazonaws.com/prod')

client.post_to_connection(
    ConnectionId='conn123',
    Data=json.dumps({"message": "hello"})
)
```

**Key limits**: a single connection lasts a maximum of 2 hours (must reconnect); a single message caps at 128 KB; idle timeout of 10 minutes.

**Use cases**: real-time chat/collaborative editing, gaming (syncing player actions), stock/crypto price feeds, sending commands to IoT devices, notification systems.
