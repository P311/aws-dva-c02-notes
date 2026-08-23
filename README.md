# AWS DVA-C02 Notes / AWS 开发者认证笔记

Module-organized study notes for the AWS Certified Developer – Associate (DVA-C02) exam, in both English and Chinese.

按 AWS 服务模块组织的 DVA-C02 考试笔记，中英双语。

---

## Contents / 目录

| # | Module | 模块 | EN | 中文 |
|---|--------|------|----|----|
| 1 | Fundamentals (DNS / JSON / YAML) | 基础（DNS + JSON/YAML） | [en](en/01-fundamentals.md) | [zh](zh/01-fundamentals.md) |
| 2 | IAM + STS + KMS | IAM + STS + KMS | [en](en/02-iam-sts-kms.md) | [zh](zh/02-iam-sts-kms.md) |
| 3 | EC2 + Storage (EBS / Instance Store / EFS / FSx) | EC2 + 存储 | [en](en/03-ec2-storage.md) | [zh](zh/03-ec2-storage.md) |
| 4 | S3 (Storage, Security, Performance) | S3（存储、安全、性能） | [en](en/04-s3.md) | [zh](zh/04-s3.md) |
| 5 | VPC (Networking) | VPC（网络） | [en](en/05-vpc.md) | [zh](zh/05-vpc.md) |
| 6 | Databases (RDS / Aurora / ElastiCache / DynamoDB) | 数据库 | [en](en/06-databases.md) | [zh](zh/06-databases.md) |
| 7 | Lambda | Lambda（基础 + 性能 + 部署 + 监控） | [en](en/07-lambda.md) | [zh](zh/07-lambda.md) |
| 8 | API Gateway (REST / HTTP / WebSocket + Authz) | API Gateway | [en](en/08-api-gateway.md) | [zh](zh/08-api-gateway.md) |
| 9 | Messaging & Streaming (SQS / SNS / Kinesis) | 消息与流式 | [en](en/09-messaging-streaming.md) | [zh](zh/09-messaging-streaming.md) |
| 10 | Orchestration & Events (Step Functions / EventBridge) | 编排与事件 | [en](en/10-orchestration-events.md) | [zh](zh/10-orchestration-events.md) |
| 11 | Cognito (AuthN / AuthZ) | Cognito（认证与授权） | [en](en/11-cognito.md) | [zh](zh/11-cognito.md) |
| 12 | Config & Ops (Secrets Manager / Parameter Store / SSM) | 配置与运维 | [en](en/12-config-ops.md) | [zh](zh/12-config-ops.md) |
| 13 | Containers (ECS / Fargate / ECR) | 容器服务 | [en](en/13-containers.md) | [zh](zh/13-containers.md) |
| 14 | Infrastructure as Code (CloudFormation / SAM / CDK) | 基础设施即代码 | [en](en/14-iac.md) | [zh](zh/14-iac.md) |
| 15 | CI/CD & Build Tools (Code* suite) | CI/CD 与构建工具 | [en](en/15-cicd.md) | [zh](zh/15-cicd.md) |
| 16 | Deployment & Load Balancing (Beanstalk + ELB/ALB/NLB) | 部署平台 + 负载均衡 | [en](en/16-deployment-lb.md) | [zh](zh/16-deployment-lb.md) |
| 17 | CDN & Edge (CloudFront) | CDN 与 Edge | [en](en/17-cloudfront.md) | [zh](zh/17-cloudfront.md) |
| 18 | Monitoring & Observability (X-Ray / CloudWatch / CloudTrail) | 监控与可观测性 | [en](en/18-monitoring.md) | [zh](zh/18-monitoring.md) |
| 19 | Misc & Emerging Services (AppConfig / Amplify / ACM / AppSync) | 杂项与新兴服务 | [en](en/19-misc.md) | [zh](zh/19-misc.md) |

## Methodology / 方法说明

See [METHOD.md](METHOD.md) for how AI was used to plan and study for this exam — not just how these notes were translated. / 关于如何用 AI 规划并完成整个备考过程（而不只是这份笔记怎么翻译的），见 [METHOD.md](METHOD.md)。

---

## Exam Domain Weights / 考试域权重

| Domain / 考试域 | Weight / 占比 | Modules |
|---|---|---|
| Development with AWS Services | 32% | 6, 7, 8, 9, 10 |
| Security | 26% | 2, 11, 12 |
| Deployment | 24% | 14, 15, 16 |
| Troubleshooting / Optimization | 18% | 7, 8, 12, 18 |

---

## Notes / 说明

- These are personal study notes, written while preparing for and passing the exam. They are not affiliated with or endorsed by AWS.
- 这是备考期间的个人笔记，与 AWS 官方无关。
- Written in 2026. Service limits, defaults, and pricing change — **check anything load-bearing against the current AWS docs.**
- 写于 2026 年。服务限额、默认值与定价会变，**关键数字请以 AWS 官方最新文档为准**。
- The English version is a technical adaptation rather than a literal translation.
- 英文版为技术改写，非逐字直译。

## License

This work (the notes themselves) is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — you may share and adapt it, including commercially, as long as you give appropriate credit.

本笔记内容采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.zh) 协议——可自由分享、修改，包括商用，但需署名。

AWS service names, trademarks, and official documentation remain the property of Amazon Web Services, Inc. This repo is an independent study resource and is not affiliated with or endorsed by AWS.

AWS 服务名称、商标及官方文档版权归 Amazon Web Services, Inc. 所有。本仓库为独立备考资料，与 AWS 官方无关，未获其认可或背书。
