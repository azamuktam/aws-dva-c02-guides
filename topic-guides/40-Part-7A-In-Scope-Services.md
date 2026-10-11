# PART 7 — THE REST: LOWER-FREQUENCY AND DISTRACTOR SERVICES
Two groups here.
**7A — In scope, but they generate few questions.** Learn what each one is *for* and move on.
**7B — Not on the in-scope services list.** They still appear as **wrong answers**, and knowing why they’re wrong is worth as much as knowing the right one. Do not spend study hours here.
---
## 7A — IN SCOPE, LOWER FREQUENCY
---
### Amazon Athena
**Why it exists.** You have terabytes of logs sitting in S3 and a question to answer. Loading them into a database first is slow and expensive. Athena runs **serverless SQL directly against S3** — no cluster, no loading, no schema migration.
**Uses:** ad-hoc analysis of CloudTrail logs, ALB and CloudFront access logs, VPC flow logs, application logs, data lakes.
**The whole exam angle is cost, because Athena is priced per terabyte scanned.** Everything that reduces bytes read reduces the bill:
- **Columnar formats** (**Parquet**, ORC) — a query touching 2 of 50 columns reads only those 2.
- **Compression** (Snappy, GZIP).
- **Partitioning** (`s3://bucket/year=2026/month=08/`) — lets Athena skip whole prefixes rather than reading and discarding.
- **Select only the columns you need.** `SELECT *` is the expensive answer.
- **Fewer, larger files** beat thousands of tiny ones.
Other points: schema lives in the **AWS Glue Data Catalog**; **workgroups** separate teams and can enforce per-query data limits; results are written to an S3 output location; **federated queries** reach beyond S3 via connectors.
**Athena vs S3 Select** — the dividing line is the number of objects. One object → **S3 Select**. A whole dataset with joins → **Athena**.
**Recall check**
- **SQL on S3 without a database** → Athena
- **Reduce Athena cost** → Parquet + compression + partitioning + fewer columns
- **Analyse logs stored in S3** → Athena
- **Query a single object** → S3 Select
- **Schema for Athena** → Glue Data Catalog
---
### Amazon OpenSearch Service
**Why it exists.** Neither DynamoDB nor a relational database does full-text search well — ranking, fuzzy matching, faceting, and aggregation over free text are a different problem. OpenSearch is the managed search and log-analytics engine.
**Uses:** full-text search in an application, log analytics dashboards, observability.
**The pattern that appears on the exam:**
```plain text
DynamoDB (source of truth)
      ↓ DynamoDB Streams
   Lambda
      ↓
OpenSearch (search index)
```
Search hits OpenSearch, then retrieves the full item from DynamoDB by key. This keeps DynamoDB as the authoritative store while getting real search.
Also fed by **Kinesis Data Firehose** and by **CloudWatch Logs subscription filters**. **OpenSearch Dashboards** provides visualisation.
**Recall check**
- **Full-text search** → OpenSearch
- **Search over DynamoDB data** → Streams → Lambda → OpenSearch
- **Log analytics dashboards** → OpenSearch (or CloudWatch Logs Insights for ad-hoc queries)
---
### AWS Systems Manager
**Why it exists.** A collection of operational tools for managing resources at scale. For this exam, three parts matter:
- **Parameter Store** — configuration and secrets (covered fully in Part 4).
- **AppConfig** — feature flags and safe configuration deployment (covered fully in Part 4).
- **Run Command** — execute commands on running instances without SSH. The counterpart to EC2 user data: **user data runs at launch; Run Command manages instances that are already running.**
Also present: Session Manager (browser shell to an instance, no SSH keys or open ports), Patch Manager, State Manager, Automation runbooks.
**Recall check**
- **Manage already-running EC2 instances** → Systems Manager Run Command
- **Configuration values and secrets** → Parameter Store
- **Feature flags with safe rollout** → AppConfig
- **Shell into an instance without SSH keys or open ports** → Session Manager
---
### Amazon Q Developer
Added as an explicit new skill in the December 2024 exam guide revision. Expect one or two conceptual questions, not depth.
**What it is.** An AI assistant for software development, embedded in IDEs, the command line, the AWS Console, and chat platforms.
**What it does:**
- **Inline code suggestions** and completions as you type
- **Chat** about your code, AWS services, and your own account’s resources
- **Agentic tasks** — implement a feature across multiple files, write tests, generate documentation
- **Code transformation** — upgrade Java versions, port .NET applications to Linux
- **Security scanning** of your source code during development, with suggested fixes
- **Console and CLI assistance** — explain an error, suggest a command, troubleshoot a resource
**Where it runs:** VS Code, JetBrains, Visual Studio, Eclipse, the CLI, the AWS Console, Slack and Teams.
**The distinction that will be tested:** Q Developer scans **your source code as you write it**. **Amazon Inspector** scans **deployed artifacts** — EC2 instances, container images, Lambda functions — for known CVEs. Different stage of the lifecycle, different service.
**Recall check**
- **AI code suggestions in the IDE** → Amazon Q Developer
- **Automatically upgrade a legacy Java application** → Q Developer code transformation
- **Find security issues in source code during development** → Q Developer security scan
- **Find CVEs in deployed EC2 / containers / Lambda** → **Amazon Inspector** (different service)
- **Explain an AWS error in the console** → Q Developer
---
### Amazon Q Business
AI assistant for **business information**.
- Search company data
- Answer questions about documents/business content
- Summarize organizational information
> **Q Business = company/business knowledge**
### AWS Amplify
**What it is.** A full-stack platform for web and mobile applications: **hosting + CI/CD + a generated backend**.
Backend resources it provisions for you: **Cognito** (auth), **AppSync/DynamoDB** (data), **S3** (storage), Lambda (functions).
**Amplify Hosting** — connect a Git branch and get automatic build and deploy, **pull-request preview environments**, custom domains, atomic deployments, and password protection.
**When Amplify vs S3 + CloudFront:** Amplify when you want Git-based CI/CD and a generated backend. S3 + CloudFront when you only need static file serving and want full control of caching and distribution behaviour.
**Recall check**
- **Host a React/Vue front end with Git-based CI/CD** → Amplify Hosting
- **Add authentication to a mobile app quickly** → Amplify (Cognito underneath)
- **PR preview environments** → Amplify Hosting
- **Static files only, full control of caching** → S3 + CloudFront
---

### AWS CloudShell
**What it is.** A browser-based shell inside the AWS Console, pre-loaded with the **AWS CLI**, Python, Node.js, and Git.
**Why it’s useful:** it runs with **your console identity’s permissions**, so there are no access keys to configure and nothing to install. The fastest path from “I’m signed in” to “I can run a CLI command”.
**The limit worth knowing:** **1 GB of persistent storage per Region**, in `$HOME`. Everything outside `$HOME` is wiped after a period of inactivity.
**Recall check**
- **Run CLI commands with no local setup and no access keys** → CloudShell
- **CloudShell files disappeared** → only `$HOME` (1 GB per Region) persists
---

# Question patterns

> Run SQL queries directly on S3 data. Which service?
> → **Amazon Athena**

> Full-text search and log analytics. Which service?
> → **Amazon OpenSearch Service**

> Run commands on many EC2 instances without SSH. Which service?
> → **AWS Systems Manager (Run Command)**

> An AI assistant for code in the IDE. Which service?
> → **Amazon Q Developer**

> Deploy a full-stack web/mobile app quickly. Which service?
> → **AWS Amplify**

> A browser-based shell for the AWS account. Which service?
> → **AWS CloudShell**

# Pocket card

| Keyword | Answer |
| --- | --- |
| SQL on S3 | **Athena** |
| Search/analytics | **OpenSearch** |
| Operations at scale | **Systems Manager** |
| Code assistance | **Q Developer** |
| Full-stack | **Amplify** |
| Web shell | **CloudShell** |
