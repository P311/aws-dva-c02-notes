# Module 11: Cognito (AuthN / AuthZ)

> **Covered**: May 13
>
> 中文版本：[`zh/11-cognito.md`](../zh/11-cognito.md)

---

## What Cognito Is ⭐⭐⭐

**= AWS's fully managed "identity platform"** for web/mobile applications.

**Two core components** (must be kept straight):

| Component | Full name | Purpose | Keyword |
|---|---|---|---|
| **User Pool** (CUP) | Cognito User Pool | **Authentication** | "Who are you" |
| **Identity Pool** (CIP) | Cognito Identity Pool (formerly Federated Identities) | **Authorization** | "What AWS resources can you use" |

**The key distinction**: **User Pool** — users log into the system, which issues **JWT tokens** (ID/Access/Refresh). **Identity Pool** — exchanges a token for **temporary AWS credentials** (via STS), giving the application direct permission to call AWS services.

**They can be used independently or together**: CUP alone — log in, get a JWT, call your own API; CIP alone — you already have an IdP (Google/Facebook) and just want AWS credentials; CUP + CIP — Cognito manages your user directory *and* lets users call S3/DynamoDB directly.

---

## Cognito User Pool (CUP) — Authentication ⭐⭐⭐

**= a fully managed user directory + authentication service.** Provides: user sign-up/sign-in; MFA (SMS or TOTP); email/phone verification; password policies; password recovery/reset; user attribute management; user groups; federated login (Google, Facebook, Amazon, Apple, SAML, OIDC); a Hosted UI; Lambda triggers to extend the auth flow; advanced security (adaptive auth, risk detection).

### What a User Gets After Signing In ⭐⭐⭐

**Three JWT tokens**:

| Token | Purpose | Contains | Default expiry | Configurable range |
|---|---|---|---|---|
| **ID Token** | Proves user identity | Identity claims (name, email, sub, etc.) | **1 hour** | **5 minutes – 1 day** |
| **Access Token** | API authorization (scopes) | OAuth scopes, user identifier | **1 hour** | **5 minutes – 1 day** |
| **Refresh Token** | Exchanges for new ID/Access tokens | Opaque (not parseable) | **30 days** | **1 hour – 10 years** |

**Usage**: when the short-lived tokens (ID/Access) expire, call `InitiateAuth(REFRESH_TOKEN_AUTH)` with the refresh token to get new ones; once the refresh token itself expires, the user must sign in again.

**JWT structure**: three base64-encoded segments separated by `.`: `header.payload.signature`. API Gateway/backends verify the signature using Cognito's public key (exposed via a JWKS endpoint).

**A key exam point**: REST API + Cognito Authorizer uses the **ID token**; HTTP API + JWT Authorizer uses the **Access token + scopes**. Don't mix them up!

### Cognito Groups ⭐

**= grouping users within a User Pool**:

```
User Pool "MyApp":
├── Group "admin"   → mapped to IAM Role "AdminRole"
├── Group "premium" → mapped to IAM Role "PremiumRole"
└── Group "free"    → mapped to IAM Role "FreeRole"
```

A user can belong to multiple groups; group membership is included in the JWT (the `cognito:groups` claim); application code or a Lambda Authorizer can make authorization decisions based on group; combined with an Identity Pool, groups can even auto-select an IAM Role (by priority).

### Hosted UI ⭐

**= Cognito's pre-built sign-in/sign-up pages.**

```
Your app ─→ redirects to ─→ Cognito Hosted UI
            (user enters credentials or picks a social login here)
            │
            ▼ on success
            redirects back to your app + tokens (in the URL)
```

**Traits**: ✅ zero code; ✅ handles sign-in, sign-up, forgot-password, MFA, and social login automatically; ✅ custom CSS/logo; ✅ custom domain support; ✅ standard OAuth 2.0 endpoints.

**A typical OAuth flow**: the user visits the app while unauthenticated → redirected to the Hosted UI login page → the user enters credentials → redirected back to the app with a `code` → the app exchanges the `code` for JWT tokens.

⚠️ **A naming change**: AWS has renamed "Hosted UI" to "Managed Login" (the newer version), while the classic Hosted UI still exists — but exam question banks often still say "Hosted UI." Check current AWS naming.

### Federation ⭐⭐

**= letting users log in with an external identity, without registering directly in Cognito.**

Supported IdPs: social IdPs (Google, Facebook, Amazon, Apple — out of the box); SAML 2.0 (enterprise SSO: AD, Okta, Auth0, Azure AD); OIDC (any OIDC-compatible IdP).

**Key insight**: the external IdP's token is never handed directly to the application! Cognito "translates" the external identity into its own JWT for the app — so the app only ever deals with Cognito tokens, regardless of how many external IdPs are configured.

### OAuth Flows ⭐

Cognito User Pool supports several OAuth 2.0 grants:

| Flow | Use case | Recommended for |
|---|---|---|
| **Authorization Code Grant** | Web apps (backend + frontend) | The production default — tokens aren't exposed |
| **Authorization Code + PKCE** | SPAs/mobile apps | **The standard choice for modern SPAs/mobile apps** |
| **Client Credentials** | Service-to-service (M2M) | Backend service calls, no user involved |
| ~~Implicit Grant~~ | (Legacy, not recommended) | Deprecated under OAuth 2.1 |

**Judgment calls**: a browser SPA → Authorization Code + PKCE; backend API-to-API → Client Credentials; older exam questions may expect Implicit Grant, but newer ones should point to Code + PKCE.

### MFA (Multi-Factor Authentication)

**Two MFA types**: SMS (a text code) and TOTP (Google Authenticator/Authy).

**Three MFA policies**: Off, Optional (user's choice), Required (mandatory for every user).

**Adaptive Authentication**: dynamically decides whether to require MFA based on a risk score — an unusual IP or device can trigger it. Requires enabling Advanced Security Features (a paid add-on).

---

## Cognito Lambda Triggers ⭐⭐⭐

**= Cognito calling a Lambda at key points in the auth flow to extend its behavior.**

### The Full List of Triggers

**Sign-up/confirmation flow**:

| Trigger | When | Typical use |
|---|---|---|
| **Pre Sign-up** | When the sign-up request is submitted | Restrict to a specific email domain (@company.com), auto-confirm certain users |
| **Post Confirmation** | After the account is confirmed | Send a welcome email, initialize a user profile in DynamoDB |
| **Custom Message** | Before sending a verification code/email/SMS | Customize message content (branded emails) |
| **User Migration** | On a user's first login (legacy system migration) | Validate the old password → create a Cognito account (seamless migration) |

**Authentication flow**:

| Trigger | When | Typical use |
|---|---|---|
| **Pre Authentication** | Before password verification | Reject suspicious IPs, custom checks |
| **Post Authentication** | After a successful login | Log the sign-in, risk analysis |
| **Pre Token Generation** | Before issuing the JWT | **Modify JWT claims** (add custom fields, alter scopes) |

**Custom auth challenge flow** (advanced): **Define Auth Challenge** (defines the overall challenge sequence), **Create Auth Challenge** (generates a specific challenge, like a CAPTCHA), **Verify Auth Challenge Response** (validates the user's answer).

**Custom senders**: **Custom Email Sender** (using your own SES or third-party email service), **Custom SMS Sender** (using your own SMS provider).

**Key constraint**: ⚠️ a Lambda trigger **must return within 5 seconds**, or Cognito treats it as a failure — trigger code must be fast and never call a slow downstream service.

**Common judgment calls**: only allow company email sign-ups → Pre Sign-up; initialize a DynamoDB profile after successful confirmation → Post Confirmation; customize the sign-up verification email → Custom Message; seamlessly migrate from a legacy auth system → User Migration; add a custom field to the JWT → Pre Token Generation; add CAPTCHA verification → the Define/Create/Verify Auth Challenge trio.

---

## Cognito Identity Pool (CIP) — Authorization ⭐⭐⭐

**= issuing temporary AWS credentials to authenticated users (or anonymous guests).**

Flow: the user authenticates (via User Pool/Google/Facebook/SAML/custom) → the app gets a token from that IdP → the app calls the Identity Pool's `GetCredentialsForIdentity` with that token → the Identity Pool assumes a role via STS → returns temporary AWS credentials → the app uses those credentials to call AWS services directly.

**Core value**: lets a frontend/mobile app call AWS services securely and directly, without embedding long-term AWS credentials in the client.

**Supported IdPs**: Cognito User Pool, social (Google/Facebook/Amazon/Apple), SAML 2.0, OIDC, developer-authenticated (custom backend auth), anonymous/guest.

### Two User Types

**1. Authenticated**: identity verified through an IdP, mapped to the **Authenticated IAM Role** (broader permissions).

**2. Unauthenticated / Guest**: not signed in, but can still access some AWS resources, mapped to the **Unauthenticated IAM Role** (limited permissions) — used to let guests browse public content.

Every Identity Pool defaults to two IAM roles — one authenticated, one unauthenticated.

### Role Assignment ⭐⭐

**1. Role-Based Access Control (RBAC)** — choosing a role by group/claim: User Pool group "admin" → AdminRole; group "user" → UserRole. Or by token claims (e.g. "if the token has dept=engineering" → EngineeringRole).

**2. Attribute-Based Access Control (ABAC)** — turning token attributes into IAM session tags:

```
JWT claim "tenant_id=abc" → injected as sts:RequestTag on the STS session
IAM policy restricts access using ${aws:PrincipalTag/tenant_id}
```

ABAC's advantage: one IAM role serves every tenant, differentiated purely by tag.

### A Classic Scenario — Mobile App Uploading Directly to S3

The user signs into the mobile app via Cognito User Pool and gets a JWT → the app calls the Identity Pool to exchange the JWT for temporary AWS credentials → the app uses those credentials to call S3 PutObject directly (via the SDK) → no intermediate server needed!

Example IAM policy (for the authenticated role):

```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject"],
  "Resource": "arn:aws:s3:::user-uploads/${cognito-identity.amazonaws.com:sub}/*"
}
```

Using `${cognito-identity.amazonaws.com:sub}` (the Identity Pool's user ID), each user is restricted to writing only within their own folder.

---

## CUP vs. CIP — Full Comparison ⭐⭐⭐

```
Mobile App  ─→  Cognito User Pool (sign-in/sign-up)
     │              │
     │              └─→ returns JWT tokens (ID/Access/Refresh)
     │
     ├─→ calls your own API directly (using the JWT) ────→ API Gateway + Cognito Authorizer
     │
     └─→ Cognito Identity Pool (exchanges the JWT for temporary AWS credentials)
            │
            └─→ STS AssumeRole → returns AccessKey + Secret + Token
                                      │
                                      └─→ App calls AWS directly: S3 / DynamoDB / SES
```

| Dimension | **User Pool** | **Identity Pool** |
|---|---|---|
| Role | **Authentication** | **Authorization** |
| Keyword | "Who are you" | "What AWS operations can you do" |
| Issues | JWT tokens | Temporary AWS credentials (via STS) |
| Stores user data | ✅ A user directory | ❌ (only stores identity ID mappings) |
| Supports passwords | ✅ | ❌ (relies on an IdP) |
| Supports social login | ✅ (federated to a social IdP) | ✅ (exchanges a social token for AWS credentials) |
| MFA | ✅ | ❌ |
| Guest users | ❌ | ✅ Unauthenticated role |
| Pricing | Billed by MAU | Free |

### Choosing Between Them ⭐⭐⭐

| Need | Choose |
|---|---|
| Self-managed users (sign-up/sign-in/MFA) | **User Pool** |
| Users calling your own API | **User Pool** alone is enough |
| Users calling AWS services directly (S3/DynamoDB) | **Identity Pool** is needed |
| A mobile app letting users upload photos to S3 | **User Pool + Identity Pool** |
| Already using Google login, just want AWS access | **Identity Pool** (no User Pool needed) |
| Guests can browse (no login) | **Identity Pool**'s unauthenticated role |
| Enterprise SSO integration + calling AWS | **Identity Pool** connected directly to SAML |

---

## Cognito + API Gateway (Review + Reinforcement) ⭐⭐⭐

### REST API + Cognito User Pool Authorizer

Uses the **ID token** (in the Authorization header); API Gateway verifies the signature automatically (using Cognito's public key, no Lambda involved); on success, claims are injected into `$context.authorizer.claims.xxx`.

### HTTP API + JWT Authorizer

Typically uses the **Access token** (enabling OAuth-scope-based authorization); configure `issuer` as the Cognito User Pool's issuer URL; configure `audience` as the User Pool's app client ID; `authorizationScopes` can restrict access per method.

**A common trap**: ⚠️ REST API + Cognito Authorizer requires the ID token (using an Access token returns 401); ⚠️ once tokens expire, the client must use the refresh token to get new ones; ⚠️ the Cognito Authorizer's cache defaults to 5 minutes — a changed token may still validate against the stale cached result within that window.

---

## Key Cognito Limits ⭐

| Item | Value |
|---|---|
| **MAU free tier** | Tens of thousands (check the current pricing page for the exact figure) |
| **Max users per User Pool** | Tens of millions |
| **Custom attributes** | Up to **50** |
| **Standard attributes** | 25 pre-built (name/email/phone, etc.) |
| **ID/Access Token default lifetime** | **1 hour** |
| **ID/Access Token configurable range** | **5 minutes – 1 day** |
| **Refresh Token default lifetime** | **30 days** |
| **Refresh Token configurable range** | **1 hour – 10 years** |
| **Lambda trigger timeout** | **5 seconds** |

**Pricing (rough structure)**: User Pool — free for the first tier of MAUs (permanently, not a time-limited free trial), then billed per MAU beyond that; Identity Pool — completely free; Hosted UI/SMS — billed by send volume (SMS runs through SNS underneath); Advanced Security — an additional per-MAU charge. Check AWS's current pricing page for exact figures.

---

## Cognito Security Features (Brief)

**Advanced Security Features** (paid): compromised credential detection (checked against known-leaked password databases); adaptive authentication (dynamically requiring MFA based on risk score); account takeover protection (guards against brute force and malicious IPs).

**Encryption**: tokens are signed with RS256 (RSA); User Pool data is encrypted at rest (KMS, AWS-managed); a custom KMS key is only used for Custom Sender Lambda triggers.
