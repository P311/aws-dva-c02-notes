# AWS DVA-C02 — 6-Week Study Plan

Here's a 6-week study plan for the DVA exam. Since you already have AWS fundamentals (you've used Lambda/API Gateway/Cognito/VPC), you can move faster through some of the foundational chapters.

## Overall Strategy

- **Weekdays (evenings):** 1.5–2 hours of lectures + note-taking
- **Weekends:** 4–5 hours (deep study + hands-on demos + weekly review)
- **Suggested exam date:** Tuesday, May 26 or Thursday, May 28, leaving a few days of buffer

## Week 1 (4/19 – 4/25): Fundamentals + IAM + EC2

| Date | Content | Duration |
|---|---|---|
| Sun 4/19 | Course overview + Fundamentals (DNS, JSON/YAML review) | 2-3h |
| Mon 4/20 | IAM core: Users, Groups, Roles, Policy evaluation logic | 1.5h |
| Tue 4/21 | IAM: STS, Identity Federation, Cross-account roles | 1.5h |
| Wed 4/22 | Encryption basics + KMS intro | 1.5h |
| Thu 4/23 | EC2: Instance types, AMIs, User data, Metadata | 1.5h |
| Fri 4/24 | EC2 Advanced: Spot, Dedicated Host, Placement Groups | 1.5h |
| Sat 4/25 | EBS, Instance Store, EFS, FSx + hands-on demo | 4-5h |

## Week 2 (4/26 – 5/2): S3 + VPC + RDS

| Date | Content | Duration |
|---|---|---|
| Sun 4/26 | S3 overview: Storage classes, Lifecycle, Versioning, Replication | 4-5h |
| Mon 4/27 | S3 Security: Bucket policy, ACL, Presigned URL, SSE types | 1.5h |
| Tue 4/28 | S3 Performance + Multipart Upload + Transfer Acceleration | 1.5h |
| Wed 4/29 | VPC review: Subnets, NAT, VPC Endpoints (Gateway vs Interface) | 1.5h |
| Thu 4/30 | RDS + Aurora + Read Replicas vs Multi-AZ | 1.5h |
| Fri 5/1 | ElastiCache (Redis vs Memcached) + caching patterns | 1.5h |
| Sat 5/2 | DynamoDB Part 1: Tables, Partition/Sort Key, Capacity modes | 4-5h |

## Week 3 (5/3 – 5/9): DynamoDB + Serverless Core

> This is the highest-value week for the DVA exam.

| Date | Content | Duration |
|---|---|---|
| Sun 5/3 | DynamoDB deep dive: GSI/LSI, Streams, DAX, TTL, Transactions | 4-5h |
| Mon 5/4 | Lambda: Execution model, Triggers, Layers, Env vars | 1.5h |
| Tue 5/5 | Lambda: Versions/Aliases, Concurrency (reserved vs provisioned), Cold start | 1.5h |
| Wed 5/6 | API Gateway: REST vs HTTP vs WebSocket, Integration types | 1.5h |
| Thu 5/7 | API Gateway: Authorizers, Caching, Stages, Canary deployment | 1.5h |
| Fri 5/8 | SQS (Standard/FIFO, DLQ, Visibility timeout) + SNS (Fan-out) | 1.5h |
| Sat 5/9 | EventBridge + Step Functions + Kinesis (Data Streams vs Firehose) | 4-5h |

## Week 4 (5/10 – 5/16): Security + Developer Tools

| Date | Content | Duration |
|---|---|---|
| Sun 5/10 | Cognito (User Pool vs Identity Pool) + Practice Exam #1 diagnostic | 4-5h |
| Mon 5/11 | KMS deep dive (Key policies, Grants, Envelope encryption) | 1.5h |
| Tue 5/12 | Secrets Manager vs Parameter Store (commonly tested comparison) | 1.5h |
| Wed 5/13 | CodeCommit + CodeBuild (buildspec.yml syntax) | 1.5h |
| Thu 5/14 | CodeDeploy (appspec.yml, deployment strategies: Canary/Linear/All-at-once) | 1.5h |
| Fri 5/15 | CodePipeline + Elastic Beanstalk (deployment policies) | 1.5h |
| Sat 5/16 | SAM + CDK + CloudFormation (intrinsic functions, nested stacks) | 4-5h |

## Week 5 (5/17 – 5/23): Monitoring + Practice-Question Week

| Date | Content | Duration |
|---|---|---|
| Sun 5/17 | CloudWatch (Logs, Metrics, Alarms, Insights) + X-Ray | 4h + Practice Exam #2 |
| Mon 5/18 | CloudTrail + ECS/ECR/Fargate basics | 1.5h |
| Tue 5/19 | Systems Manager (Parameter Store, Session Manager, Run Command) | 1.5h |
| Wed 5/20 | Practice Exam #3 + review wrong answers | 2h |
| Thu 5/21 | Targeted reinforcement of weak areas (based on wrong-answer patterns) | 1.5h |
| Fri 5/22 | Run through cheat sheet + flashcards | 1.5h |
| Sat 5/23 | Practice Exam #4 + full review | 5h |

## Week 6 (5/24 – 5/28): Final Sprint + Exam

| Date | Content |
|---|---|
| Sun 5/24 | Practice Exam #5 + final review of wrong answers |
| Mon 5/25 | Light review of cheat sheets, sleep early |
| Tue 5/26 | 🎯 Exam day (recommended) |
| Backup 5/27–5/30 | If your 5/24 practice exam score isn't stable yet, push the exam to one of these days |

## Recommended Practice Resources (by priority)

1. **Tutorials Dojo (Jon Bonso)** — closest to the real exam questions, a must
2. **Cantrill's built-in Practice Test** — maps well to the course content
3. **AWS Skill Builder Official Practice Question Set** — the free one

**Passing benchmark:** once you're consistently scoring 80%+ on Tutorials Dojo practice exams, you're ready to sit the exam.
