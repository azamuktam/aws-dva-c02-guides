*PART 7 — THE REST: LOWER-FREQUENCY AND DISTRACTOR SERVICES*

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

> A developer has terabytes of logs in S3 and must run ad-hoc SQL without loading them into a database. Which service?
> → **Amazon Athena**

> How is Athena priced, and why does that shape every design decision?
> → **Per terabyte of data scanned — everything that reduces bytes read reduces cost**

> A developer wants to cut Athena query cost on a wide table. Which storage format should they switch to?
> → **Columnar formats — Parquet or ORC (reads only referenced columns)**

> Which compression codecs reduce Athena bytes scanned?
> → **Snappy or GZIP**

> Which S3 layout lets Athena skip whole prefixes instead of reading everything?
> → **Partitioning, e.g. `s3://bucket/year=2026/month=08/`**

> Why is `SELECT *` an expensive answer for Athena?
> → **It reads all columns — select only the columns you need**

> Why do fewer, larger files beat thousands of tiny files for Athena?
> → **Each object incurs overhead; many small files are more expensive/slower**

> A team needs a schema for their Athena tables. Where does it live?
> → **The AWS Glue Data Catalog**

> A team must enforce a per-query data limit and separate teams. Which Athena feature?
> → **Workgroups**

> Where are Athena query results written?
> → **To an S3 output location**

> A developer must query data outside S3 from Athena. What feature enables this?
> → **Federated queries via connectors**

> A developer needs to query a single S3 object rather than a whole dataset. Which service?
> → **S3 Select**

> What is the dividing line between S3 Select and Athena?
> → **Number of objects — one object → S3 Select; a whole dataset with joins → Athena**

> A team wants full-text search with ranking, fuzzy matching, and faceting over free text. Which service?
> → **Amazon OpenSearch Service**

> Why won't DynamoDB or a relational database handle full-text search well?
> → **Ranking, fuzzy matching, faceting, and free-text aggregation are a different problem**

> Which pattern lets an application search DynamoDB data with real full-text search?
> → **DynamoDB Streams → Lambda → OpenSearch, then fetch the item from DynamoDB by key**

> In the DynamoDB + OpenSearch pattern, which store remains the source of truth?
> → **DynamoDB**

> Which services can feed data into OpenSearch besides DynamoDB Streams?
> → **Kinesis Data Firehose and CloudWatch Logs subscription filters**

> Which component provides visualisation for OpenSearch?
> → **OpenSearch Dashboards**

> A team needs ad-hoc log queries rather than a persistent search index. Which is the better fit?
> → **CloudWatch Logs Insights**

> A developer must execute commands on many already-running EC2 instances without SSH. Which service?
> → **AWS Systems Manager Run Command**

> How does Systems Manager Run Command relate to EC2 user data?
> → **User data runs at launch; Run Command manages instances that are already running**

> A developer needs configuration values and secrets for an application. Which capability?
> → **Systems Manager Parameter Store**

> A team wants feature flags with safe configuration rollout. Which capability?
> → **Systems Manager AppConfig**

> A developer needs a browser shell into an instance with no SSH keys and no open ports. Which capability?
> → **Systems Manager Session Manager**

> Which other Systems Manager capabilities exist for patching, state, and automation?
> → **Patch Manager, State Manager, Automation runbooks**

> A developer wants an AI assistant in the IDE for inline code suggestions and completions. Which service?
> → **Amazon Q Developer**

> A team must automatically upgrade a legacy Java application to a newer version. Which service/feature?
> → **Amazon Q Developer code transformation**

> A developer wants security issues found in source code as they write it. Which service?
> → **Amazon Q Developer security scan**

> A team must find known CVEs in deployed EC2 instances, container images, and Lambda functions. Which service?
> → **Amazon Inspector (not Q Developer)**

> What is the key distinction between Q Developer and Amazon Inspector?
> → **Q Developer scans source code during development; Inspector scans deployed artifacts for CVEs**

> A developer wants help understanding an AWS error in the console or a suggested CLI command. Which service?
> → **Amazon Q Developer (console and CLI assistance)**

> Which tasks can Q Developer perform beyond suggestions?
> → **Chat, agentic multi-file tasks, tests/docs, code transformation, security scanning, console/CLI help**

> Where can Q Developer run?
> → **VS Code, JetBrains, Visual Studio, Eclipse, CLI, AWS Console, Slack, Teams**

> A team needs an AI assistant to search company documents and summarize business content. Which service?
> → **Amazon Q Business (company/business knowledge)**

> A developer wants to host a React/Vue front end with Git-based CI/CD and automatic deploys. Which service?
> → **AWS Amplify Hosting**

> What backend resources does Amplify provision for you?
> → **Cognito (auth), AppSync/DynamoDB (data), S3 (storage), Lambda (functions)**

> A team wants automatic preview environments per pull request. Which service?
> → **Amplify Hosting**

> A developer wants to add authentication to a mobile app quickly. Which service, and what powers it?
> → **Amplify, with Cognito underneath**

> When should a team choose S3 + CloudFront over Amplify?
> → **When they only need static file serving and want full control of caching and distribution**

> When should a team choose Amplify over S3 + CloudFront?
> → **When they want Git-based CI/CD and a generated backend**

> A developer needs to run AWS CLI commands with no local setup and no access keys. Which service?
> → **AWS CloudShell**

> What tools are pre-loaded in CloudShell?
> → **AWS CLI, Python, Node.js, and Git**

> With whose permissions does CloudShell run?
> → **Your console identity's permissions**

> What is the persistent storage limit for CloudShell?
> → **1 GB per Region, in `$HOME`**

> A developer's CloudShell files disappeared after a period of inactivity. Why?
> → **Only `$HOME` (1 GB per Region) persists; everything else is wiped**

# Pocket card

| Keyword | Answer |
| --- | --- |
| SQL on S3 without a database | **Athena** |
| Athena pricing | **per TB scanned** |
| Reduce Athena cost | **Parquet/ORC + compression + partitioning + fewer columns** |
| Columnar formats | **Parquet, ORC** |
| Athena compression | **Snappy, GZIP** |
| Athena skip prefixes | **partitioning (`s3://bucket/year=/month=/`)** |
| Expensive Athena query | **`SELECT *`** |
| Athena file sizing | **fewer, larger files** |
| Athena schema | **Glue Data Catalog** |
| Per-query limits / teams | **workgroups** |
| Athena results | **written to an S3 output location** |
| Query outside S3 | **federated queries (connectors)** |
| Query a single object | **S3 Select** |
| Athena vs S3 Select | **dataset with joins → Athena; one object → S3 Select** |
| Full-text search | **OpenSearch** |
| Search over DynamoDB | **Streams → Lambda → OpenSearch** |
| DynamoDB + OpenSearch source of truth | **DynamoDB** |
| Feed OpenSearch | **Firehose, CloudWatch Logs subscription filters** |
| OpenSearch visualisation | **OpenSearch Dashboards** |
| Ad-hoc log queries | **CloudWatch Logs Insights** |
| Manage running EC2 without SSH | **Systems Manager Run Command** |
| User data vs Run Command | **launch time vs already running** |
| Config and secrets | **Parameter Store** |
| Feature flags, safe rollout | **AppConfig** |
| Shell with no SSH keys/ports | **Session Manager** |
| Patching / state / runbooks | **Patch Manager / State Manager / Automation** |
| AI code suggestions in IDE | **Q Developer** |
| Upgrade legacy Java | **Q Developer code transformation** |
| Security issues in source code | **Q Developer security scan** |
| CVEs in deployed resources | **Amazon Inspector** |
| Q Developer vs Inspector | **source during dev vs deployed artifacts** |
| Explain an AWS error in console | **Q Developer** |
| Q Developer locations | **VS Code, JetBrains, VS, Eclipse, CLI, Console, Slack, Teams** |
| Business knowledge AI | **Q Business** |
| Full-stack web/mobile platform | **Amplify** |
| Amplify backend | **Cognito, AppSync/DynamoDB, S3, Lambda** |
| Amplify Hosting CI/CD | **Git branch → build/deploy** |
| Amplify PR previews | **preview deployments** |
| Amplify auth | **Cognito underneath** |
| Static files only, cache control | **S3 + CloudFront** |
| Browser shell in AWS | **CloudShell** |
| CloudShell preloaded | **AWS CLI, Python, Node.js, Git** |
| CloudShell permissions | **your console identity** |
| CloudShell persistence | **1 GB per Region in `$HOME`** |
| CloudShell lost files | **only `$HOME` persists** |
