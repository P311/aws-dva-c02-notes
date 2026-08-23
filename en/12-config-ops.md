# Module 12: Config & Ops (Secrets Manager / Parameter Store / SSM)

> **Covered**: May 14
>
> 中文版本：[`zh/12-config-ops.md`](../zh/12-config-ops.md)

---

## 1. AWS Secrets Manager ⭐⭐⭐

**= AWS's fully managed secrets/credential management service**, built around **automatic rotation**.

**Core value**: ✅ centralizes database passwords, API keys, tokens, and application credentials; ✅ **automatic password rotation** (its biggest selling point, deeply integrated with RDS/Aurora/Redshift/DocumentDB); ✅ KMS at-rest encryption; ✅ cross-region replication (for DR); ✅ cross-account sharing (via resource policy); ✅ CloudTrail auditing, CloudWatch monitoring; ✅ fine-grained IAM access control.

**Key limits**: a single secret caps at **64 KB**; billing is based on the number of secrets plus API calls (check the current pricing page for exact figures); encrypted via KMS (defaults to `aws/secretsmanager`, a customer key can be substituted); supports automatic rotation (Lambda or managed); supports multi-region replication.

### Secret Versions and Labels ⭐⭐

Every secret can have multiple versions, each carrying staging labels:

| Label | Meaning |
|---|---|
| **AWSCURRENT** | The version currently in effect (what the application reads) |
| **AWSPENDING** | Newly generated during rotation, not yet in effect |
| **AWSPREVIOUS** | The previous AWSCURRENT version (kept after rotation, for rollback) |

**Key insight**: rotation isn't "delete the old, create the new" — it's *moving a pointer* between staging labels. A new version is written with the AWSPENDING label; once it passes testing, the label moves to AWSCURRENT (and the old AWSCURRENT becomes AWSPREVIOUS); `GetSecretValue` reads AWSCURRENT by default.

### Automatic Rotation ⭐⭐⭐

**Two forms**:

**1. Managed Rotation**: no Lambda to write — AWS manages it directly. Applies only to these "managed secrets": RDS/Aurora (MySQL/PostgreSQL/Oracle/SQL Server/MariaDB), Amazon Redshift, Amazon DocumentDB.

**2. Rotation by Lambda function**: for other secret types (custom API keys, third-party service credentials). AWS provides pre-built templates, and you can also write your own.

**The four-step rotation flow** ⭐⭐⭐: Secrets Manager passes a `Step` parameter when invoking the Lambda:

```
1. createSecret    → generates new credentials (e.g. a new password), writes them to the AWSPENDING version
2. setSecret       → updates the target service with the new credentials (e.g. changes the RDS password)
3. testSecret      → connects to the service with the new credentials to confirm they work
4. finishSecret    → moves the AWSPENDING label to AWSCURRENT; the old AWSCURRENT becomes AWSPREVIOUS
```

⚠️ If any step fails, Secrets Manager retries the entire flow within the rotation window.

### The Two Rotation Strategies ⭐⭐

**1. Single User Rotation**: one DB user is used by the application, and rotation changes its password. Flow: generate a new password → change that user's password in the DB → verify → done. ⚠️ At the moment of rotation, the old password stops working and the new one takes over — if the application has cached the old password, it will briefly error out. Fits simple scenarios that can tolerate a brief interruption.

**2. Alternating Users Rotation**: two DB users are used alternately. Flow: using the admin/superuser secret, clone the current user → create a second user (same permissions, different password) → switch. **Zero downtime**: the old user stays valid for a while as the application gradually switches to the new one. Requires an additional superuser secret (since it needs `CREATE USER`). ⚠️ RDS Proxy doesn't support alternating users.

**Choosing**: simple + tolerant of a brief blip → Single User; zero downtime / strict SLA → Alternating Users.

### Cross-Account / Cross-Region

**Resource Policy** (cross-account):

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::OTHER-ACCOUNT:role/AppRole" },
  "Action": ["secretsmanager:GetSecretValue"],
  "Resource": "*"
}
```

The KMS key policy must also allow the other account to use the key.

**Multi-Region Replication**: configure a replica region via the console/API; replication is asynchronous (seconds); applications in each region read the local replica directly for low latency; updates to the primary secret sync automatically; a replica can rotate independently (with its own KMS key).

### Caching and Best Practices

**The problem**: calling `GetSecretValue` on every Lambda invocation causes API costs to explode and hurts performance.

**The fix**: **the AWS Parameters and Secrets Lambda Extension** ⭐ — attached as a Lambda layer, it caches secrets locally (default TTL 300 seconds), and Lambda code reads secrets via `localhost:2773` — dramatically cutting API calls. SDK-level client-side caching libraries are also available for various languages.

---

## 2. SSM Parameter Store ⭐⭐⭐

**= the configuration management service within AWS Systems Manager.** Core value: stores configuration data/secrets; hierarchical paths for organization; version tracking; integrated KMS encryption (SecureString); fine-grained IAM access control; the Standard tier is completely free.

### Parameter Types ⭐

| Type | Meaning |
|---|---|
| **String** | Plain text |
| **StringList** | A comma-separated list of strings |
| **SecureString** | Sensitive data encrypted with KMS |

### Standard vs. Advanced Tier ⭐⭐⭐

| Dimension | **Standard** | **Advanced** |
|---|---|---|
| Max parameters per region | **10,000** | **100,000** |
| Max size per parameter | **4 KB** | **8 KB** |
| Price | **Free** | Billed per parameter |
| Parameter Policies (TTL, expiration notices) | ❌ | ✅ |
| SecureString encryption | Direct KMS encrypt | Envelope encryption with the AWS Encryption SDK |
| Can downgrade | / | ❌ **Advanced cannot revert to Standard** (must be deleted and recreated) |

**Intelligent Tier**: AWS automatically decides between Standard and Advanced (upgrading automatically past 4 KB).

### Naming Conventions / Hierarchy ⭐

**Best practice**: use `/` for hierarchical naming:

```
/myapp/dev/db/url
/myapp/dev/db/password
/myapp/prod/db/url
/myapp/prod/db/password
/myapp/prod/api/stripe-key
```

Benefits: ✅ IAM policies can grant access by path prefix (`"Resource": "arn:aws:ssm:us-east-1:111:parameter/myapp/dev/*"`); ✅ `GetParametersByPath` can batch-fetch every parameter under a prefix.

**Important detail**: a top-level parameter name must not start with `aws` or `ssm` (reserved prefixes).

### Parameter Policies (Advanced Tier Only) ⭐

**= attaching policies to a parameter** (similar to a lifecycle):

| Policy | Purpose |
|---|---|
| **Expiration** | Automatically deletes the parameter on expiry |
| **ExpirationNotification** | Sends an EventBridge notification N days before expiry |
| **NoChangeNotification** | Notifies if the parameter hasn't changed in N days (a rotation reminder) |

### Versions

Every `PutParameter --overwrite` creates a new version (auto-incrementing); the latest version is read by default; a specific version can be read (`Name: my-param:3`); a limited number of historical versions is retained (the oldest are pruned once the cap is reached).

### Referencing Secrets Manager ⭐

Parameter Store can directly reference a Secrets Manager secret: `/aws/reference/secretsmanager/my-secret-name`. Reading this parameter transparently returns the secret's current value. Use case: fetching all configuration through a single Parameter Store API, even when some values actually live in Secrets Manager (getting its rotation for free).

### Referencing Public Parameters

AWS publishes commonly needed values in Parameter Store, such as the latest Amazon Linux AMI ID: `/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2`. Heavily used by EC2 Auto Scaling and CloudFormation.

---

## 3. Secrets Manager vs. Parameter Store ⭐⭐⭐

| Dimension | **Secrets Manager** | **Parameter Store** |
|---|---|---|
| Price | Billed per secret | **Standard is free**; Advanced billed per parameter |
| Size | 64 KB | Standard 4 KB / Advanced 8 KB |
| Automatic rotation | ✅ **Native support** (Lambda or managed) | ❌ Must be built yourself (EventBridge + Lambda) |
| Cross-region replication | ✅ Native | ❌ |
| Encryption | KMS by default | KMS for SecureString type |
| Resource Policy | ✅ (cross-account) | ❌ (IAM only) |
| RDS/Redshift integration | ✅ Deeply integrated | ❌ |
| Typical use | Database passwords, API keys (needing rotation) | Config values, URLs, feature flags, secrets that don't need rotation |

### Choosing Between Them ⭐⭐⭐

| Scenario | Choose |
|---|---|
| RDS/Aurora/Redshift passwords | **Secrets Manager** (native managed rotation) |
| Third-party API keys needing periodic rotation | **Secrets Manager** (Lambda rotation) |
| Application config (database URL, feature flags) | **Parameter Store** Standard |
| Static API keys (no rotation) | **Parameter Store** SecureString (cheaper) |
| Many secrets, all needing rotation | **Secrets Manager** (pricier but full-featured) |
| Many config values, mostly static | **Parameter Store** (cheaper) |
| Cross-region DR database passwords | **Secrets Manager** (native replication) |
| Temporary credentials that must auto-expire after N days | **Parameter Store Advanced** (Parameter Policies) |
| Large config values over 8 KB | **Parameter Store Advanced** or **Secrets Manager** |

**A classic trick**: reference Secrets Manager from Parameter Store (`/aws/reference/secretsmanager/xxx`) — the application fetches everything through one API, getting Secrets Manager's rotation at Parameter Store's price.

---

## 4. The AWS Systems Manager Toolset ⭐⭐

**= AWS's operations "Swiss Army knife"** for managing EC2, on-premises, and multi-cloud nodes.

**Core mechanism**: a node runs the SSM Agent (pre-installed on most AMIs), which connects to the SSM service, letting AWS manage the machine.

**Requirements for the SSM Agent**: ✅ an IAM instance profile with the `AmazonSSMManagedInstanceCore` managed policy attached; ✅ the SSM Agent installed and running; ✅ outbound HTTPS access to the SSM endpoint (via a VPC Endpoint or the public internet).

**Or, more simply**: enable **Default Host Management Configuration** (a newer feature) so every EC2 instance is automatically a managed node — no manual IAM instance profile configuration required.

### Main Tools

**1. Session Manager** (⭐⭐⭐ frequently tested): browser/CLI shell access to EC2, with no SSH, no bastion host, no open ports. Core value: no SSH key needed (entirely IAM-managed); port 22 doesn't need to be open; no bastion host needed; full auditing (every session is logged to CloudTrail + CloudWatch Logs + S3); works cross-account; supports Windows/Linux/macOS.

**Judgment calls**: "want to SSH into a private-subnet EC2 without opening port 22 or using a bastion" → Session Manager; "compliance requires full shell-session auditing" → Session Manager (streaming logs to S3/CloudWatch).

**2. Run Command** (⭐⭐ frequently tested): runs a predefined command/script across multiple managed nodes at once. Uses SSM Documents (JSON/YAML) to define the action; AWS provides many built-in documents (`AWS-RunShellScript`, `AWS-UpdateSSMAgent`); can execute across all instances matching a tag. Vs. Session Manager: Run Command is predefined, batch, non-interactive; Session Manager is interactive, single-instance.

**3. Patch Manager** (⭐ moderately tested): automated OS/application patching. Core concepts: Patch Baseline (defines which patches are auto-approved/rejected, and approval delay), Patch Group (groups instances by tag, e.g. "prod"/"dev," each using its own baseline), Maintenance Window (schedules when patches get applied), Compliance (scans and reports which instances are missing which patches).

**4. State Manager** (⭐ moderately tested): keeps instances in a continuously enforced "desired state," similar to Ansible/Puppet. Defines the conditions an instance should satisfy, checking and correcting drift on a schedule. Typical uses: ensuring every EC2 instance has the CloudWatch Agent installed; ensuring firewall configuration doesn't drift.

**5. Maintenance Windows** (⭐ moderately tested): a scheduled window for maintenance tasks — configure a cron schedule, and attach Run Command/State Manager/Lambda/Step Functions as the task.

**6. Inventory** (⭐ lightly tested): automatically collects instance metadata (software, files, network, registry, etc.).

**7. Distributor** (⭐ lightly tested): software package distribution management — packages internal software, deploys via Run Command/State Manager, supports versioning and rollback.

**8. Automation (Runbooks)** (⭐⭐ moderately tested): multi-step operational automation, defined via SSM Documents. AWS provides many pre-built runbooks (creating AMIs, restarting instances, incident response), and they can run cross-account/cross-region.

**9. OpsCenter** (⭐ lightly tested): a central place to view/handle OpsItems, sourced from CloudWatch alarms, Config non-compliance, and Security Hub findings — can trigger an Automation runbook to resolve them.

**10. Change Manager** (⭐ lightly tested): an approval workflow for operational changes — templated change requests, multi-level approval, and runbook execution once approved.

**11. Fleet Manager** (⭐ moderately tested): a unified UI for managing a fleet of instances — a browser interface to view the file system, registry, processes, and logs without SSH/RDP, with one-click launch into Session Manager/Run Command.

### SSM Documents ⭐

**= the "scripts" behind Systems Manager's tools** (JSON or YAML)

| Type | Purpose | Used by |
|---|---|---|
| **Command** | Executes shell commands/scripts | Run Command, State Manager |
| **Automation** | Multi-step ops flows (runbooks) | Automation |
| **Session** | Defines Session Manager behavior | Session Manager |

**Document sharing**: Private (default, account-only), Shared (cross-account), Public (published for anyone — the AWS-maintained `AWS-*` documents).

---

## 5. Bringing It All Together

**A complete scenario: Lambda + RDS + Secrets Manager + Parameter Store** — Lambda starts up → the Lambda Extension reads (cached) Secrets Manager for the DB password → reads (cached) SSM Parameter Store for the DB endpoint and other config → connects to RDS and processes business logic. On a regular schedule, automatic rotation happens: Secrets Manager invokes a Lambda → updates the RDS password → switches the secret's version → the next Lambda invocation reads the new password.

**Referencing these from IaC**:

CloudFormation referencing an SSM parameter:

```yaml
Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2:1}}'
      KeyName: '{{resolve:ssm:my-keypair-name}}'
```

CloudFormation referencing Secrets Manager:

```yaml
DBPassword: '{{resolve:secretsmanager:my-db-secret:SecretString:password}}'
```

⚠️ CloudFormation's `resolve` never stores the plaintext value in the template — it fetches it at deploy time.
