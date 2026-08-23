# Module 1：基础（DNS + JSON/YAML）

> **覆盖日期**：4/19
> **主题**：DNS / JSON / YAML / CloudFormation 内置函数
>
> English version: [`en/01-fundamentals.md`](../en/01-fundamentals.md)

---

## DNS 核心概念

**解析层级**：Root (.) → TLD (.com) → Authoritative Nameserver → Resolver

### 记录类型（DVA 常考）

- **A** → IPv4；**AAAA** → IPv6
- **CNAME** → 把一个域名指向另一个域名。**不能用在 zone apex**（根域名，如 example.com），只能用在子域名（如 www.example.com）
- **Alias** → Route 53 特有，能用在 zone apex，且指向 AWS 资源（ELB、CloudFront、S3 website、API Gateway 等）时免费，AWS 会自动跟踪目标的 IP 变化
- **NS** → 授权的 nameserver；**SOA** → zone 的起始授权信息
- **MX** → 邮件交换；**TXT** → 验证用（SPF、域名所有权）

**考试套路**：带 zone apex（根域名）指向 ELB / CloudFront / S3 website / API Gateway 这类需求，选 **Alias**，不选 CNAME（CNAME 本身就不允许用在 zone apex 上）。

---

## Route 53 路由策略

- **Simple** — 单一或多个值随机返回
- **Weighted** — 按权重分配流量（灰度发布常用）
- **Latency-based** — 按用户到各 region 的延迟路由
- **Failover** — 主备切换，依赖 health check
- **Geolocation** — 按用户地理位置（国家/大洲）。必须配一个 default 记录兜底
- **Geoproximity** — 按资源和用户的地理距离，需要用 Traffic Flow，可以用 bias（-99 ~ +99）调整覆盖圈
- **Multi-value** — 返回多个健康的记录，客户端选一个

### 考试关键词

- "shift traffic" 或 "expand coverage area" → **Geoproximity + bias**
- 法律合规、语言本地化 → **Geolocation**

**TTL**：高 TTL 减少 DNS 查询次数（省钱、低延迟），但记录变更后传播慢；低 TTL 反之。Alias record 的 TTL 由 AWS 托管。

---

## JSON vs YAML

- JSON 在 AWS 主要出现在 IAM policy 里
- YAML 在 CloudFormation / SAM / buildspec / appspec 里更常见，因为支持注释、更简洁

### YAML 坑点

- 缩进只能用空格，不能用 tab
- `key: value` 冒号后必须有空格
- CloudFormation 短格式：`!Ref`、`!GetAtt`、`!Sub`
- **两个 `!` 标签不能直接嵌套**，必须用长格式 `Fn::GetAtt`

### CloudFormation 内置函数嵌套规则

```yaml
# ❌ 错误 - YAML 不允许两个 ! 嵌套
!Sub "arn:aws:s3:::${!Ref MyBucket}/*"
!GetAtt !Ref MyResource.Arn

# ✅ 正确 - 用长格式
Fn::Sub:
  - "arn:aws:s3:::${BucketName}/*"
  - BucketName: !Ref MyBucket

# ✅ 正确 - !Sub 直接引用逻辑名，不嵌套函数
!Sub "arn:aws:s3:::${MyBucket}/*"
```

### 实用记法

- 单独用一个函数 → 短格式 `!Ref`、`!GetAtt`
- 函数要嵌套函数 → 外层必须用长格式 `Fn::Xxx:`

### 常见内置函数（DVA 考试会考）

- `Ref` — 引用参数或资源
- `Fn::GetAtt` — 获取资源的某个属性（如 `!GetAtt MyBucket.Arn`）
- `Fn::Sub` — 字符串插值（最常用）
- `Fn::Join` — 字符串拼接（老式写法）
- `Fn::ImportValue` — 引用另一个 stack 的 Output
- `Fn::If` / `Fn::Equals` — 条件逻辑
- `Fn::FindInMap` — 查 Mappings 表（常用于按 region 查 AMI ID）
