# Module 1: Fundamentals (DNS + JSON/YAML)

> **Covered**: Apr 19
> **Topics**: DNS / JSON / YAML / CloudFormation intrinsic functions
>
> 中文版本：[`zh/01-fundamentals.md`](../zh/01-fundamentals.md)

---

## DNS Core Concepts

**Resolution chain**: Root (.) → TLD (.com) → Authoritative Nameserver → Resolver

### Record Types (commonly tested on DVA)

- **A** → IPv4; **AAAA** → IPv6
- **CNAME** → Points one domain name at another. **Cannot be used at the zone apex** (the root domain, e.g. example.com) — only on subdomains (e.g. www.example.com)
- **Alias** → Route 53-specific. Works at the zone apex, is free of charge when pointing at AWS resources (ELB, CloudFront, S3 website, API Gateway, etc.), and Route 53 automatically tracks IP changes on the target
- **NS** → Delegated nameservers; **SOA** → Start of authority for the zone
- **MX** → Mail exchange; **TXT** → Verification (SPF, domain ownership)

**Exam pattern**: For any requirement about pointing the *zone apex* at an ELB / CloudFront / S3 website / API Gateway, the answer is **Alias**, not CNAME (CNAME itself isn't allowed at the zone apex).

---

## Route 53 Routing Policies

- **Simple** — Returns one value, or several at random
- **Weighted** — Splits traffic by weight (standard for canary/gradual rollouts)
- **Latency-based** — Routes by measured latency from the user to each region
- **Failover** — Active/passive switchover, driven by health checks
- **Geolocation** — Routes by the user's geographic location (country/continent). A default record is mandatory as a catch-all
- **Geoproximity** — Routes by geographic distance between resource and user. Requires Traffic Flow; the coverage radius is tuned with a bias value (-99 to +99)
- **Multi-value** — Returns several healthy records and lets the client pick one

### Exam Keywords

- "shift traffic" or "expand coverage area" → **Geoproximity + bias**
- Legal compliance or language localization → **Geolocation**

**TTL**: A high TTL means fewer DNS queries (cheaper, lower latency) but slower propagation after a record change; a low TTL is the reverse. TTL on alias records is managed by AWS.

---

## JSON vs YAML

- In AWS, JSON shows up mainly in IAM policies
- YAML is more common in CloudFormation / SAM / buildspec / appspec, because it supports comments and is more concise

### YAML Gotchas

- Indentation must use spaces — tabs are not allowed
- `key: value` requires a space after the colon
- CloudFormation short form: `!Ref`, `!GetAtt`, `!Sub`
- **Two `!` tags cannot be nested directly** — use the long form `Fn::GetAtt` instead

### Nesting Rules for CloudFormation Intrinsic Functions

```yaml
# ❌ Wrong - YAML does not allow two nested ! tags
!Sub "arn:aws:s3:::${!Ref MyBucket}/*"
!GetAtt !Ref MyResource.Arn

# ✅ Correct - use the long form
Fn::Sub:
  - "arn:aws:s3:::${BucketName}/*"
  - BucketName: !Ref MyBucket

# ✅ Correct - !Sub references the logical name directly, no nested function
!Sub "arn:aws:s3:::${MyBucket}/*"
```

### Rule of Thumb

- Single function on its own → short form (`!Ref`, `!GetAtt`)
- Function nested inside another function → the outer one must use the long form `Fn::Xxx:`

### Common Intrinsic Functions (tested on DVA)

- `Ref` — References a parameter or resource
- `Fn::GetAtt` — Retrieves an attribute of a resource (e.g. `!GetAtt MyBucket.Arn`)
- `Fn::Sub` — String interpolation (the most frequently used)
- `Fn::Join` — String concatenation (the older style)
- `Fn::ImportValue` — References an Output exported by another stack
- `Fn::If` / `Fn::Equals` — Conditional logic
- `Fn::FindInMap` — Looks up a Mappings table (commonly used for per-region AMI IDs)
