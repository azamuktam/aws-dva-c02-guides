## Gap-Fill & Extra One-Liners

Small DVA-C02 facts that are easy to leave out. None of these need deep study — recognise the trigger word and the answer usually follows.

**AWS Lambda Powertools**

A suite of utilities (Python / TypeScript / Java) that implement the patterns you would otherwise hand-roll:

- **Logger** — structured JSON logs.
- **Tracer** — wraps X-Ray so you get subsegments without manual instrumentation.
- **Metrics** — emits CloudWatch **EMF** (Embedded Metric Format) so you can log a metric without the PutMetricData API.
- **Idempotency** — deduplicates invocations using **DynamoDB** as the store.
- **Batch** — handles partial-batch failures for SQS/Kinesis/DynamoDB streams.
- **Parameters** — caches and fetches SSM Parameter Store / Secrets Manager values.
- **Feature flags** — reads AWS AppConfig flags.

**AWS SDK — waiters, paginators and retries**

- **Waiter** — polls an API until a resource reaches a state (for example, wait until an EC2 instance is *running* or an S3 bucket *exists*) instead of writing your own loop.
- **Paginator** — iterates results that AWS returns in pages (for example `ListObjectsV2`), so you do not silently miss items. In boto3: `client.get_paginator(...)`.
- **Retries** — the SDK automatically retries transient errors with exponential backoff (about 3 attempts by default; tune with `max_attempts` / `retry_mode`).
- **Credential provider chain** — resolved roughly in this order: environment variables → shared config/credentials file (`~/.aws/credentials`, selected by profile) → ECS task role / EC2 instance profile. Roles always beat static keys.

**AWS IAM Identity Center (formerly AWS SSO)**

Central **workforce** sign-in across many accounts in AWS Organizations, with **permission sets** mapped to groups. This is for *your employees*, not your application's end users — customer identities are **Amazon Cognito**.

- On-premises or hybrid (non-AWS) workloads that need AWS credentials → **IAM Roles Anywhere**.

**Amazon Aurora — Global Database and Backtrack**

- **Aurora Global Database** — one primary Region and up to five secondary Regions; storage is replicated with **sub-second** latency (RPO around 1 second). Secondaries serve **read-only** traffic, and a Region can be promoted for disaster recovery in under a minute.
- **Aurora Backtrack** — rewinds the cluster **in place** to an earlier point without restoring from a backup, up to **72 hours**. MySQL-compatible Aurora only.
- **Aurora Serverless v2** — scales capacity automatically with demand; good for intermittent or unpredictable workloads.

**ECS Service Discovery (AWS Cloud Map)**

Registers ECS tasks in a service registry so other services find them by **DNS name or API** instead of a hard-coded IP address. This is the answer when services must discover each other **without** sitting behind a load balancer.

**KMS encryption context**

A set of **non-secret key–value pairs** passed to a KMS encrypt/decrypt call. It is **not encrypted** and it is recorded in **CloudTrail**, which makes it useful for auditing — but the **exact same context must be supplied to decrypt**. Think of it as authenticated associated data.

**Amazon S3 Storage Lens**

**Account- and organisation-wide** storage metrics and dashboards (usage, activity, cost-efficiency, data-protection). It answers "how much am I storing, where, and at what cost" across the whole organisation, not one bucket.

**AWS Copilot**

A CLI that builds and deploys **containerised services to ECS / Fargate** from a manifest, creating the load balancer, pipeline and infrastructure for you. The quickest path to run a container service on ECS with minimal YAML (App Runner is the answer when you want no infrastructure to manage at all).

**AWS Well-Architected Framework**

Six pillars: **Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability**. Review a workload with the **Well-Architected Tool**, and use **lenses** (for example the Serverless Lens) to adapt it to a domain. On the exam it is usually the *framing* of a question rather than the answer — if a scenario names a pillar, pick the practice that serves it (runbooks and CI/CD for operational excellence, and so on).

**AWS SAM policy templates**

Pre-written IAM policies referenced by name inside a SAM template, such as `S3ReadPolicy`, `DynamoDBCrudPolicy` and `SQSPollerPolicy`. Use them to give a function least-privilege access to one resource with the least effort.

# Question patterns

> A developer must make a Lambda function idempotent with the least custom code. What should they use?
> → **The Powertools Idempotency utility (backed by DynamoDB)**

> A function must emit structured JSON logs without hand-building the format. Which Powertools utility?
> → **Logger**

> A team wants custom CloudWatch metrics from Lambda without calling PutMetricData directly. Which utility?
> → **Powertools Metrics (Emits Embedded Metric Format, EMF)**

> Code must poll an EC2 instance until it is running instead of writing its own retry loop. What SDK feature?
> → **A waiter**

> A script pages through every object in a large S3 bucket and is missing items. What SDK feature fixes this?
> → **A paginator (get_paginator)**

> What is the SDK's default behaviour when a request fails with a transient error?
> → **Automatic retries with exponential backoff (about 3 attempts by default)**

> In what order does the AWS SDK resolve credentials?
> → **Environment variables → shared config/credentials file → ECS/EC2 role**

> Employees need a single sign-on across many AWS accounts. Which service?
> → **IAM Identity Center (AWS SSO)**

> An on-premises workload must obtain AWS credentials without long-term keys. Which feature?
> → **IAM Roles Anywhere**

> Where should customer sign-in for an application live — Identity Center or Cognito?
> → **Amazon Cognito (Identity Center is for employees, not customers)**

> A database must replicate across Regions with sub-second latency and serve read-only traffic in secondary Regions. Which feature?
> → **Aurora Global Database**

> A MySQL-compatible Aurora cluster must be rewound to 40 minutes ago without restoring a backup. Which feature?
> → **Aurora Backtrack (up to 72 hours)**

> An Aurora database has unpredictable, intermittent traffic and should scale capacity automatically. Which option?
> → **Aurora Serverless v2**

> Two ECS services must find each other by DNS without a load balancer. Which feature?
> → **ECS Service Discovery (AWS Cloud Map)**

> A security team must attach auditable, non-secret data to every KMS operation. Which feature?
> → **KMS encryption context**

> A company needs storage usage and cost metrics across the whole organisation, not per bucket. Which feature?
> → **Amazon S3 Storage Lens**

> A team wants the fastest way to deploy a container service to ECS/Fargate from a CLI. Which tool?
> → **AWS Copilot**

> Name the six pillars of the Well-Architected Framework.
> → **Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability**

> A workload should be reviewed against serverless best practices. What does the Well-Architected Tool use for this?
> → **A lens (for example the Serverless Lens)**

> A SAM template must grant a function least-privilege access to a single DynamoDB table with minimal effort. What to use?
> → **A SAM policy template (for example DynamoDBCrudPolicy)**

> What store does the Powertools Idempotency utility use to detect duplicates?
> → **Amazon DynamoDB**

> Which is NOT the right service for external customer identities?
> → **IAM Identity Center**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Make Lambda idempotent | **Powertools Idempotency (DynamoDB)** |
| Structured logs | **Powertools Logger** |
| Custom metrics without code | **Powertools Metrics (EMF)** |
| Partial-batch failures | **Powertools Batch** |
| Poll until a state | **SDK waiter** |
| Iterate paged results | **SDK paginator** |
| SDK retries | **exponential backoff (~3 attempts)** |
| Credential order | **env → shared files → role** |
| Workforce SSO | **IAM Identity Center** |
| On-prem → AWS creds | **IAM Roles Anywhere** |
| Customer identities | **Amazon Cognito** |
| Cross-Region Aurora | **Global Database (sub-second RPO ~1 s)** |
| Rewind Aurora MySQL | **Backtrack (≤ 72 h)** |
| Auto-capacity Aurora | **Serverless v2** |
| ECS service registry | **Cloud Map / Service Discovery** |
| Auditable KMS metadata | **encryption context** |
| Org-wide S3 metrics | **Storage Lens** |
| Containers to ECS via CLI | **AWS Copilot** |
| No-infra container service | **App Runner** |
| Well-Architected pillars | **Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability** |
| Review a workload | **Well-Architected Tool (+ lenses)** |
| SAM least privilege | **policy templates** |
| Powertools idempotency store | **DynamoDB** |
