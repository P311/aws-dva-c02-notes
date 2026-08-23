# Module 2: IAM + STS + KMS

> **Covered**: Apr 20, 21, 22
> **Topics**: IAM core · STS + Federation + Cross-Account · Encryption + KMS
>
> 中文版本：[`zh/02-iam-sts-kms.md`](../zh/02-iam-sts-kms.md)

---

## 1. IAM Core

### Users / Groups / Roles

**User** — Represents a person or a service account, with long-term credentials (password, access key). Best practice: never issue an access key for root; use root only for account-level operations.

**Group** — Can only contain users. Groups cannot be nested and cannot contain roles. Permissions reach the user through the group's attached policies.

**Role** — Has no long-term credentials. When assumed, STS issues temporary credentials. Used for:

- EC2 instance profiles
- Lambda execution roles
- Cross-account access
- Federated identity (SAML, OIDC, Cognito)

**A role carries two policies**:

- **Trust policy** (who may assume this role) — attached to the role itself and is a **resource-based policy**
- **Permissions policy** (what the role may do) — either managed or inline

### IAM User vs IAM Role

|  | IAM User | IAM Role |
|---|---|---|
| Represents | A specific person or service account | An identity any trusted party can temporarily assume |
| Credential type | Long-term (password, access key) | None inherent; STS issues temporary credentials on assume |
| Can sign in? | Yes (with a password) | No — can only be assumed |
| Principal | Only the user itself | EC2, Lambda, another account, SAML, OIDC… |
| Typical use | Human admins, CI systems | EC2 instance profile, cross-account, federation |

### Policy Types

- **Identity-based policy** — Attached to a user, group, or role
- **Resource-based policy** — Attached to a resource (S3 bucket policy, SQS, SNS, Lambda, KMS key policy, etc.)
- **Permission boundary** — Sets a ceiling on a user's or role's permissions. It grants nothing; it only constrains
- **SCP (Service Control Policy)** — Lives at the Organizations level and applies to an entire account
- **Session policy** — Passed inline when assuming a role; can only narrow permissions
- **ACL** — The legacy mechanism, still present in S3 and VPC

### Policy Evaluation Logic ⭐

**Decision order**:

1. Deny by default (implicit deny)
2. Evaluate every applicable policy
3. **Any explicit deny wins immediately** — it is a veto
4. An explicit allow → allowed
5. No allow at all → still implicit deny

**Rule of thumb**: **Deny > Allow > Implicit Deny**

**When several policy layers apply at once**:

- **Same-account access**: identity policy and resource policy are **OR**-ed — either one allowing is enough
- **Cross-account access**: both sides must allow (**AND**)
- **With a permission boundary**: effective permissions = identity policy ∩ permission boundary
- **With an SCP**: the SCP grants nothing; it only filters

**Effective permissions** = `Identity Policy ∩ Permission Boundary ∩ SCP`

### Permission Boundary vs SCP

|  | Permission Boundary | SCP |
|---|---|---|
| Applies to | A single IAM user or role | An entire OU or account |
| Configured by | IAM admin | Organizations admin |
| Grants permissions? | ❌ Caps only | ❌ Filters only |
| Affects root user | ❌ No | ✅ Yes |

### Policy Details

**Common condition keys**:

- `aws:SourceIp` — Restrict source IP
- `aws:MultiFactorAuthPresent` — Require MFA
- `aws:RequestedRegion` — Restrict region
- `aws:PrincipalTag` / `aws:ResourceTag` — The basis of ABAC
- `aws:SecureTransport` — Require HTTPS

**The NotAction / NotResource trap**:

- Semantics: matches every action *except* the ones listed
- `NotAction: "iam:*"` combined with `Allow` actually means "allow every action on every AWS service except IAM"

**IAM is a global service** — it does not belong to any region.

### Using IAM as a Certificate Store

There is exactly one reason to use IAM as a certificate manager: you need to serve HTTPS in a Region where ACM is not available. Key points:

- IAM encrypts the private key and holds it in the IAM SSL certificate store
- Server certificates can be deployed in any Region, but the certificate must be obtained from an external CA
- **An ACM certificate cannot be uploaded into IAM**
- **Certificates cannot be managed from the IAM console** — CLI/API only

### Instance Profile + IMDS

```text
IAM Role  ←[attached to]←  Instance Profile  ←[attached to]←  EC2 Instance
   │                                                              │
   ├─ Trust Policy                                                │
   └─ Permissions Policy                                          │
                                                                  ▼
                                                          169.254.169.254
                                                          (IMDS endpoint)
```

**Key points**:

- An IAM Role and an Instance Profile are **two distinct objects**
- The instance profile is a container that holds one IAM role
- Working through the console, AWS creates a same-named instance profile automatically
- In CloudFormation you must declare an `AWS::IAM::InstanceProfile` resource explicitly

**SDK credential provider chain** (tried in order):

1. Credentials passed explicitly in code
2. Environment variables `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`
3. The `~/.aws/credentials` file
4. **IMDS** (this is what EC2 uses)
5. ECS container credentials

### IMDS in Depth

**IMDS = Instance Metadata Service**

**Endpoint**: `http://169.254.169.254` — a link-local address, reachable only from the instance itself

**Two main data categories**:

1. **Metadata** — describes what this EC2 instance *is*
   - `instance-id`, `instance-type`, `placement/region`, `placement/availability-zone`
   - `local-ipv4`, `public-ipv4`
   - `iam/security-credentials/<role-name>` → **the temporary credential JSON** ⭐
2. **User data** — the script handed to the instance at launch
3. **Dynamic data** — including the instance identity document

**Metadata does *not* expose**:

- ❌ Memory size or CPU core count
- ❌ Disk size

### IMDSv1 vs IMDSv2 ⭐

|  | IMDSv1 | IMDSv2 |
|---|---|---|
| Request flow | Plain GET | PUT for a token first, then GET carrying the token |
| SSRF protection | ❌ | ✅ |
| Hop limit | Unlimited | 1 by default |
| Default on new instances | (deprecated) | ✅ Enabled by default |

**Three layers of SSRF defense in IMDSv2**:

1. **The token must be fetched with PUT** — most SSRF vulnerabilities can only issue GET requests
2. **The token must travel in a custom header** — SSRF rarely gives the attacker control over request headers
3. **Hop limit defaults to 1** — blocks container escape paths ([docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html))

> In practice: a hop limit of 1 in a container environment can prevent the container from getting credentials at all (reaching the container counts as an extra hop) — AWS recommends setting it to 2 there.

**Historical incident**: The 2019 Capital One breach exposed roughly 100 million records; the root cause was SSRF combined with IMDSv1.

### Recognizing Temporary Credentials

```json
{
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "...",
  "SessionToken": "...",
  "Expiration": "..."
}
```

Three tells:

- `AccessKeyId` starts with **ASIA** (long-term keys start with **AKIA**)
- A **SessionToken** is present (long-term credentials have none)
- An **Expiration** is present

---

## 2. STS + Federation + Cross-Account

### Core STS APIs

| API | Purpose | Called by |
|---|---|---|
| `AssumeRole` | Use existing AWS credentials to assume another role | An IAM user or role |
| `AssumeRoleWithSAML` | Exchange a SAML assertion for AWS credentials | Enterprise employees |
| `AssumeRoleWithWebIdentity` | Exchange an OIDC token for AWS credentials | Mobile app users |
| `GetSessionToken` | Trade long-term credentials for short-term ones (mainly to add MFA) | The IAM user itself |
| `GetFederationToken` | Issue credentials to users with no IAM identity (legacy API) | Rarely used now |

**Mapping shortcuts**:

- Mentions **SAML** → `AssumeRoleWithSAML`
- Mentions **Google / Facebook / Cognito** → `AssumeRoleWithWebIdentity`
- Mentions **cross-account** → `AssumeRole`
- Mentions **MFA** → `GetSessionToken`, or `AssumeRole` with an MFA condition

### Temporary Credential Lifetimes

| API | Default | Minimum | Maximum |
|---|---|---|---|
| AssumeRole | 1 hour | 15 min | 12 hours |
| AssumeRoleWithSAML | 1 hour | 15 min | 12 hours |
| AssumeRoleWithWebIdentity | 1 hour | 15 min | 12 hours |
| GetSessionToken | 12 hours | 15 min | 36 hours (root is capped at 1 hour) |

Sources: [AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) · [GetSessionToken](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetSessionToken.html)

### Cross-Account Access

**Option 1: Bucket policy (more common)**

- The identity policy on account A's user allows access to the target resource
- The bucket policy in account B allows A's principal

**Option 2: Assume role**

- Account B creates a role whose trust policy allows account A to assume it
- A user in account A calls `sts:AssumeRole` for temporary credentials
- CloudTrail records both the AssumeRole event and the subsequent API calls, giving a cleaner audit trail

### External ID (Defending Against the Confused Deputy)

**Scenario**: A third-party SaaS (Datadog, MongoDB Atlas) asks you to create a role for them to assume.

**Fix**: The SaaS gives you a unique External ID; add it to the trust policy:

```json
"Condition": {
  "StringEquals": { "sts:ExternalId": "abc-123-unique" }
}
```

**Exam pattern**: "third-party SaaS integration", "cross-account trust", or "confused deputy" → **External ID**.

### The Two Federation Protocols

|  | SAML 2.0 | OIDC |
|---|---|---|
| Main scenario | Enterprise employee sign-in | Public internet user sign-in |
| Underlying format | XML assertion | JWT |
| STS API | AssumeRoleWithSAML | AssumeRoleWithWebIdentity |
| Common IdPs | AD FS, Okta, OneLogin | Google, Facebook, Cognito |

### Role Chaining

**Constraint**: any role later in a chain is capped at a **1 hour** session, regardless of its `MaxSessionDuration`. Source: [AWS re:Post](https://repost.aws/knowledge-center/iam-role-chaining-limit)

```text
IAM User → AssumeRole → RoleA → AssumeRole → RoleB
                        (12h)                 (1h max!)
```

---

## 3. Encryption Fundamentals + KMS

### Symmetric vs Asymmetric

|  | Symmetric | Asymmetric |
|---|---|---|
| Keys | One | A pair: public + private |
| Speed | Fast | Slow |
| Algorithms | AES-256, ChaCha20 | RSA, ECC |
| AWS usage | KMS default; S3/EBS/RDS encryption | TLS, digital signatures |

**AES-256 is symmetric.** AWS KMS uses **AES-256-GCM** by default.

### KMS Core

- Keys live inside AWS HSMs (Hardware Security Modules)
- Keys never leave the HSM in plaintext
- All encrypt/decrypt operations happen inside KMS

### KMS Key Types ⭐

| Key type | Created by | Auditable | Cost |
|---|---|---|---|
| **AWS Owned Key** | AWS | ❌ | Free |
| **AWS Managed Key** | AWS, automatically | ✅ Visible in console (`aws/` prefix) | Free |
| **Customer Managed Key** | You | ✅ Full control | Monthly charge + API call charges |

**When a Customer Managed Key is mandatory**:

- You need a custom key policy
- You need control over rotation
- You are sharing the key across accounts
- You are importing your own key material (BYOK)
- You need an asymmetric key

### Key Limits

- **A symmetric key can encrypt at most 4 KB (4096 bytes) directly** — anything larger requires envelope encryption ([docs](https://docs.aws.amazon.com/kms/latest/developerguide/programming-encryption.html))
- **Keys do not cross regions** — Multi-Region Keys are the exception
- **Key deletion has a waiting period** — **minimum 7 days, maximum 30 days** (30 is the default) ([ScheduleKeyDeletion](https://docs.aws.amazon.com/kms/latest/APIReference/API_ScheduleKeyDeletion.html))
- **AWS Managed Key rotation**: once a year, not configurable
- **Customer Managed Key rotation**: automatic rotation can be enabled, defaulting to 365 days (configurable from 90 to 2560 days) ([KMS FAQ](https://aws.amazon.com/kms/faqs/))

### Envelope Encryption ⭐⭐⭐

**Why it exists**: a KMS symmetric key tops out at 4 KB, and pushing large files through KMS would be slow and expensive.

**The idea**:

1. KMS generates an ephemeral data key
2. The data key encrypts the large file locally
3. The data key itself is encrypted by the KMS key and stored alongside
4. To decrypt: use KMS to decrypt the data key, then use the data key to decrypt the file

**Full encryption flow**:

```text
1. Call KMS GenerateDataKey(keyId='myKey')
   → returns {plain_data_key, encrypted_data_key}

2. Locally: AES-256 encrypt the large file with plain_data_key
   → encrypted_video

3. Locally: attach encrypted_data_key to the metadata

4. Upload to S3:
   - encrypted_video as the object body
   - encrypted_data_key as object metadata

5. ❗ Wipe plain_data_key from memory immediately
```

### Comparing the Four KMS APIs

| API | Purpose | Returns | When to use |
|---|---|---|---|
| `Encrypt` | Encrypt data directly | Ciphertext | Only for payloads ≤ 4 KB |
| `GenerateDataKey` | Generate and return both plaintext and encrypted data key | Plaintext + Encrypted Data Key | Large data — the standard envelope encryption path |
| `GenerateDataKeyWithoutPlaintext` | Return only the encrypted data key | Encrypted Data Key only | Pre-generating keys not needed immediately |
| `Decrypt` | Decrypt any KMS-encrypted ciphertext | Plaintext | Decrypting a data key, or small payloads |

### Key Policy ⭐⭐⭐

> ⚠️ A KMS key **must have a key policy**. **An IAM policy alone cannot grant access to a KMS key.**

| Resource | Policy relationship (same account) |
|---|---|
| S3 bucket | identity policy **OR** bucket policy |
| KMS key | key policy required **AND** IAM policy must also allow |
| SQS queue | OR |
| Lambda | OR |
| IAM role trust policy | trust policy required (AND) |

**The lockout trap**: if you rewrite a key policy so that nobody — including yourself — is permitted, the key is unusable and only an AWS support ticket can recover it.

**Best practice**: always keep at least one statement granting `Principal: root` with `kms:*`.

**Cross-account KMS access** requires both sides (AND):

- The key policy in account A allows account B's principal
- The IAM policy in account B allows calling the relevant KMS APIs

### KMS Grants

A **grant** temporarily gives a principal permission to use a key **without modifying the key policy**.

**Grants fit when**:

- An AWS service needs to use the key on your behalf
- You need fine-grained, temporary permissions
- You would rather not touch the key policy
- You need revocability (`RetireGrant` / `RevokeGrant`)

**Mnemonic**: key policy = long-lived and declarative; grant = short-lived and programmatic.

### Key Rotation

|  | Automatic | Manual |
|---|---|---|
| Supported on | Symmetric Customer Managed Keys | Any key |
| Frequency | 1 year by default (90–2560 days) | Whenever you choose |
| Existing data | No re-encryption needed | Re-encryption needed |
| Key ID / ARN | **Unchanged** | Changes (a new key) |
| Application impact | Transparent | References must be updated |

**Automatic rotation limits**:

- Symmetric Customer Managed Keys only
- AWS Managed Keys rotate yearly, outside your control
- Imported key material (BYOK) does not support automatic rotation
- Asymmetric keys do not support automatic rotation

> Update: KMS now supports `RotateKeyOnDemand` for on-demand rotation, including for imported keys, capped at 10 rotations per key for life. "Imported key material doesn't support **automatic** rotation" still holds ([KMS FAQ](https://aws.amazon.com/kms/faqs/)).

### KMS Across AWS Services

| Service | How KMS is used |
|---|---|
| S3 | SSE-KMS object encryption |
| EBS | Encrypts volumes and snapshots |
| RDS | Encrypts the database (must be enabled at creation; cannot be changed later) |
| DynamoDB | Encrypts tables |
| Lambda | Encrypts environment variables |
| Secrets Manager | Encrypts the secret value |
| SSM Parameter Store | SecureString parameters are KMS-encrypted |
| CloudWatch Logs | Log-group-level encryption |
