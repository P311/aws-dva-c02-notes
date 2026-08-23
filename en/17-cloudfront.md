# Module 17: CDN & Edge (CloudFront)

> **Covered**: May 22
>
> 中文版本：[`zh/17-cloudfront.md`](../zh/17-cloudfront.md)

---

## What CloudFront Is ⭐⭐⭐

**= AWS's global CDN (Content Delivery Network) + edge computing platform.**

**Core value**: ✅ global acceleration (hundreds of edge locations, so users hit the nearest PoP); ✅ caching (both static content and dynamic API responses can be cached); ✅ reduces origin load; ✅ DDoS protection (AWS Shield Standard built in); ✅ HTTPS termination (custom certificates supported); ✅ edge computing (Lambda@Edge/CloudFront Functions); ✅ security (WAF integration, private content, geo restriction).

**How it works**: a user's request → hits a CloudFront edge location (a cache hit returns immediately, a miss continues) → the regional edge cache → the origin (S3/ALB/EC2/Lambda URL) → content is fetched and cached at both the edge and regional level → the user gets the response.

## Origins ⭐⭐

| Origin | Use case | Key authentication |
|---|---|---|
| **S3 Bucket** | Static websites/file downloads | **OAC** |
| **S3 Website Endpoint** | S3 static website hosting | Custom origin (OAC not usable) |
| **ALB** | Dynamic applications/APIs | Custom origin headers + SG |
| **EC2/Custom HTTP** | Any web server | Same as above |
| **API Gateway** | Putting a CDN in front of an API | Custom origin |
| **Lambda Function URL** | Serverless HTTP | **OAC** (a more recent addition) |

**A key distinction: S3 Bucket vs. S3 Website Endpoint** ⭐⭐

|  | S3 Bucket (direct) | S3 Website Endpoint |
|---|---|---|
| URL format | `mybucket.s3.amazonaws.com` | `mybucket.s3-website-region.amazonaws.com` |
| OAC support | ✅ | ❌ (must be public) |
| index.html/error.html support | ❌ | ✅ |
| Redirect rules | ❌ | ✅ |
| Direct HTTPS access | ✅ | ❌ (HTTP only) |

## OAC vs. OAI ⭐⭐⭐

**= mechanisms restricting an S3/Lambda origin to CloudFront-only access.**

**OAI** (the older approach): limited to S3 only; doesn't support SSE-KMS; doesn't support newer regions.

**OAC** (the newer, recommended approach) ⭐⭐⭐: advantages include SSE-KMS support; support for all regions; support for S3, Lambda Function URLs, and MediaStore; short-lived, frequently rotated credentials.

**OAC configuration (S3 bucket policy)**:

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "cloudfront.amazonaws.com" },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::mybucket/*",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT:distribution/DIST_ID"
    }
  }
}
```

**The three OAC SigningBehavior options** (frequently tested): `always` (CloudFront always signs every request — the default and recommended setting); `never` (never signs); `no-override` (signs only when the request lacks an `Authorization` header).

## Signed URLs vs. Signed Cookies ⭐⭐⭐

**= granting on-demand access to private content.**

|  | **Signed URL** | **Signed Cookie** |
|---|---|---|
| Granularity | **One URL per file** | **A group of files** (via a path pattern) |
| Fits | Single-file downloads, temporary download links | An entire site/app (multiple files) |
| URL changes | The URL itself carries signature parameters | The URL is unchanged (the signature lives in a cookie) |
| Client requirements | Any HTTP client | Must support cookies |
| Typical use case | "Download this PDF," "share this one image" | "Logged-in users can view every paid video" |

**Signed URL types**: **Canned policy** (simple — only expiration is configurable); **Custom policy** (flexible — expiration, source IP, and path pattern are all configurable).

**Trusted Key Groups**: a public/private key pair used to sign URLs/cookies. Flow: generate an RSA 2048-bit key pair → upload the public key to CloudFront to create a Key Group → associate the Key Group with a distribution behavior → the application signs URLs/cookies with the private key → the CloudFront edge verifies with the public key.

⚠️ Trusted Signer Accounts (the older approach) has been deprecated.

## Lambda@Edge vs. CloudFront Functions ⭐⭐⭐

**One of the DVA's most frequently tested comparisons.**

### CloudFront Functions (Lightweight)

Runtime: JavaScript only (V8 isolate); lifecycle has just 2 events — **Viewer Request / Viewer Response**; execution time **under 1 ms**; memory **2 MB**; max code size **10 KB**; extremely cheap; **cannot call external APIs**; runs at every edge location worldwide.

Fits: HTTP header manipulation/URL rewriting/simple authentication (JWT validation)/A/B testing/cache-key normalization.

### Lambda@Edge (Powerful)

Runtime: **Node.js/Python** (the full Lambda runtime); lifecycle has 4 events — Viewer Request/Origin Request/Origin Response/Viewer Response; execution time — 5 seconds for viewer triggers, 30 seconds for origin triggers; memory 128 MB–10,240 MB; max code size 1 MB (viewer) / 50 MB (origin); **can call external APIs**; runs at Regional Edge Caches (far fewer locations than the full edge network).

Fits: complex logic (calling external APIs/DynamoDB), full SDK access, on-the-fly image resizing, traffic rewriting.

**The four Lambda@Edge trigger events** ⭐⭐: **Viewer Request** (every request — authentication, URL rewriting, blocking malicious traffic); **Origin Request** (on a cache miss — modifying the origin, routing to a different origin); **Origin Response** (after receiving the origin's response — modifying response headers); **Viewer Response** (every response — adding security headers).

**Decision table** ⭐⭐⭐: simple header operations/URL rewriting → CloudFront Functions; needing extremely low latency → CloudFront Functions; needing npm packages/an SDK → Lambda@Edge; calling external APIs/DynamoDB → Lambda@Edge; changing origin behavior on a cache miss → Lambda@Edge (Origin Request); on-the-fly image resizing → Lambda@Edge; lightweight A/B testing → CloudFront Functions; simple JWT validation → CloudFront Functions; complex OAuth flows → Lambda@Edge.

## Cache Behaviors ⭐⭐

**= configuring different caching strategies for different URL paths.**

```
Default Cache Behavior: *
  TTL: 86400, Origin: S3

Cache Behavior 1: /api/*
  TTL: 0 (no cache), Forward all headers, Origin: ALB

Cache Behavior 2: /static/*
  TTL: 31536000 (1 year), Origin: S3
```

**Path pattern priority**: matched by specificity — `/api/v2/users` > `/api/v2/*` > `/api/*` > `/*`.

**Cache Key**: can include the URL path/query string/headers/cookies. ⚠️ more fields = more cache variants = lower hit rate.

**The three TTL fields**: **Min TTL** (the shortest possible cache duration); **Max TTL** (the longest possible cache duration); **Default TTL** (used when the origin doesn't set Cache-Control).

## Invalidations ⭐

**= proactively expiring specific cached objects.**

```bash
aws cloudfront create-invalidation \
  --distribution-id ABCD1234 \
  --paths "/index.html" "/static/*"
```

✅ supports wildcards like `/static/*`; ✅ a monthly free allowance exists (beyond which paths are billed); ⚠️ it isn't instant — global propagation takes a few minutes.

**Best practice**: use versioned filenames (`app.v123.js`) instead of relying on invalidations.

## Origin Failover ⭐

**= automatically falling back to a secondary origin when the primary fails.**

```
Origin Group:
  Primary: ALB (us-east-1)
  Secondary: S3 (us-west-2, a static backup)

Failover criteria: 500, 502, 503, 504, 404, 403
```

A typical use: the ALB goes down → falls back to an S3-hosted maintenance page.

## Geo Restriction

**= allowing/denying access by country**: a whitelist (only specific countries allowed) or a blacklist (specific countries denied). Determined via an IP-based GeoIP database. Fits legal/compliance requirements. ⚠️ VPNs can bypass this — for genuine security, use IAM/Signed URLs/WAF.

## Field-Level Encryption ⭐

**= encrypting specific fields at the CloudFront layer with a public key** (e.g. a credit card number). The client POSTs plaintext → the CloudFront edge encrypts the specific field with a public key → the origin only ever sees the encrypted blob → a payment service decrypts it with the private key.

vs. HTTPS: HTTPS encrypts the whole transport channel (plaintext once it reaches the origin); field-level encryption encrypts specific fields (still ciphertext by the time it reaches the backend).

## SSL / Custom Domains ⭐⭐

### ⭐⭐⭐ The ACM Certificate Must Live in us-east-1!

CloudFront is a global service, but **the ACM certificate used by CloudFront must be requested in us-east-1** — ACM certificates from any other region simply won't work with CloudFront. **A classic DVA trick question.**

**SSL configuration options**: the default CloudFront certificate (`*.cloudfront.net`); a custom SSL certificate (via ACM, must be us-east-1); SNI (a single distribution supports multiple domains — recommended); a dedicated IP (rarely used, expensive).

**HTTPS behavior**: Viewer Protocol Policy — HTTP and HTTPS / Redirect HTTP to HTTPS (recommended) / HTTPS Only; Origin Protocol Policy — HTTP Only / HTTPS Only / Match Viewer.

⭐ Recommended for production: Viewer = Redirect to HTTPS, Origin = HTTPS Only.

## Custom Error Pages

The S3 origin returns a 403 → CloudFront returns `/error.html` instead. Configurable: which HTTP status codes trigger it, the returned path, the returned status code, and the TTL for how long the error response is cached.

## Monitoring + WAF + Pricing

**Real-Time Logs**: second-level granularity → delivered to Kinesis Data Streams (good for real-time monitoring). **Standard Logs**: batched periodically → delivered to S3 (good for archiving). **WAF integration**: associate an AWS WAF Web ACL to block malicious traffic at the edge.

**Price Class**: All Edge Locations / US+Canada+Europe / a smaller subset — choose based on where your audience actually is.

## CloudFront Limits

Roughly 25 cache behaviors per distribution; roughly 25 origins per distribution; roughly 10 origin groups per distribution; roughly 100 alternate domain names (CNAMEs); max single-file size **30 GB** (check current docs for the latest figure).
