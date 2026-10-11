# AWS Certified Developer – Associate (DVA-C02) — Topic Guides

**Exam-focused study guides for the AWS Certified Developer – Associate (DVA-C02) exam**, grouped by the role each service plays in an application rather than alphabetically — because that is how the exam presents them: a scenario describes an application, and you have to know which piece is missing or broken.

This repository holds a complete study reference moved out of Notion into a plain topic-guide layout. The wording is preserved from the original document.

## How it is organised

Each service or topic is written as:

- **Why it exists** — the problem that makes the service necessary. If you know this, you can usually derive the rest.
- **How it works** — the mechanism. Most exam questions are answerable from mechanism alone.
- **The decisions** — why you would pick this over the alternative, with the *reason*, not just a mapping.
- **How it fails** — failure modes and their causal chains.
- **Numbers** — limits, where they matter.
- **Recall check** — the compressed keyword→answer pairs.

## Exam domains (from Part 8)

| Exam domain | Weight |
|---|---:|
| Development with AWS Services | 32% |
| Security | 26% |
| Deployment | 24% |
| Troubleshooting and Optimization | 18% |

## Start here

📖 **[Topic Guides Index](topic-guides/00-README.md)** — all guides, grouped by part.

## Contents

- **Part 0** — How to think about this exam
- **Part 1 — Compute:** Lambda · EC2 & Auto Scaling · ECS/Fargate/ECR · Elastic Beanstalk
- **Part 2 — Data:** DynamoDB · DynamoDB Streams · DAX · RDS & Aurora · ElastiCache · S3 · EBS vs EFS
- **Part 3 — APIs, events, streams:** API Gateway · ALB · CloudFront · Route 53 · SQS · SNS · EventBridge · Step Functions · Kinesis · AppSync
- **Part 4 — Security:** IAM · STS · Cognito · KMS · Secrets Manager / Parameter Store / AppConfig · S3 encryption · WAF · VPC, security groups & NACLs
- **Part 5 — Deployment:** CloudFormation · SAM · CDK · CLI · CodeCommit · CodePipeline · CodeBuild · CodeDeploy · CodeArtifact
- **Part 6 — Observability:** Retries/backoff/idempotency · CloudWatch · X-Ray · CloudTrail
- **Part 7 — The rest:** lower-frequency and distractor services
- **Part 8** — The Reflex List, plus **How to study from this**

## Disclaimer

These are personal study notes, not official AWS exam content. AWS service features, quotas, and behaviours can change — verify anything critical against current AWS documentation.
