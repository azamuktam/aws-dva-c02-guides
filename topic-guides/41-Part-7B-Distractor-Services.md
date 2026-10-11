## 7B — NOT ON THE IN-SCOPE LIST (RECOGNISE AS DISTRACTORS)

---

### AWS Config

**What it is.** Records **resource configuration changes over time** and evaluates resources against **rules** for compliance.

- Shows configuration history and a timeline of changes (correlated with CloudTrail).
- **Config Rules** (AWS-managed or custom Lambda) detect non-compliant resources — “all EBS volumes must be encrypted”, “no security group may allow 0.0.0.0/0 on port 22”.
- Covers EC2, S3, IAM, Lambda, RDS, security groups, and more.
- Supports remediation actions via Systems Manager Automation.

**Why it’s a distractor:** it looks like the answer to any “track changes” question, but the precise question matters — see the CloudTrail and drift-detection comparison in Part 6.

**Recall check**

- **Continuous compliance monitoring / configuration history / audit against rules** → AWS Config
- **Who made the change** → CloudTrail
- **Does it match my CloudFormation template** → Drift Detection

---

### Amazon EMR (Elastic MapReduce)

**What it is.** Managed **big data processing** — a cluster running Apache **Spark, Hadoop, Hive, Presto, HBase**.

**Use:** process very large datasets for analytics or ETL, in batch.

**Why it’s a distractor:** it appears in options whenever data volume is mentioned. The distinguishing feature is that EMR is a **cluster you provision and run frameworks on**, which is the opposite of what a Developer-Associate scenario usually wants.

---

### Amazon Redshift

**What it is.** A fully managed **data warehouse**, columnar and optimised for large analytical SQL queries.

**Use:** business intelligence, reporting, dashboards over structured data loaded from S3, databases, or streams.

**Why it’s a distractor:** it competes with Athena in “query a lot of data with SQL” questions. The difference: Redshift requires **loading data into the warehouse** and is built for repeated, heavy BI workloads; Athena queries data **in place** in S3 and is built for ad-hoc analysis.

**The four-way analytics comparison**

| Service | Best for | Data type |
| --- | --- | --- |
| **EMR** | Processing huge datasets | Batch big data (Spark/Hadoop) |
| **Kinesis Data Streams** | Processing data as it arrives | Real-time streaming |
| **Redshift** | SQL analytics and reporting | Structured analytical data |
| **DynamoDB** | Fast application database | NoSQL key-value / document |

Compressed: **EMR processes. Kinesis streams. Redshift analyses. DynamoDB stores.** And **Athena queries S3 in place** — which is the in-scope member of this family.

---

### The security-services family

Four services that overlap in appearance and differ completely in function. The exam uses them against each other.

**AWS Security Hub** — **aggregates** security findings from other services into one dashboard, and runs standards checks (CIS AWS Foundations Benchmark, AWS Foundational Security Best Practices).

- Integrates with GuardDuty, Inspector, AWS Config, IAM Access Analyzer.
- **Does not fix anything.** It centralises and prioritises.

**Amazon Inspector** — automated **vulnerability management**. Scans **deployed** workloads for known **CVEs**.

- Scans EC2 instances, container images in ECR, and Lambda functions.
- Provides risk scores and remediation guidance.
- Contrast with **Q Developer**, which scans your **source code during development**.

**Amazon GuardDuty** — **threat detection**. Analyses CloudTrail, VPC flow logs, and DNS logs with machine learning to spot malicious or anomalous **activity** — crypto mining, credential exfiltration, communication with known-bad IPs.

**Amazon Macie** — discovers and classifies **sensitive data** (PII, credentials) in S3.

**AWS Trusted Advisor** — **best-practice recommendations** across five categories: Cost Optimisation, Performance, Security, Fault Tolerance, and Service Limits. Examples: unused resources, overly open security groups, IAM usage, low-utilisation instances.

| Service | The question it answers |
| --- | --- |
| **Security Hub** | “Show me all my security findings in one place” |
| **Inspector** | “Do my deployed workloads have known vulnerabilities?” |
| **GuardDuty** | “Is something malicious happening right now?” |
| **Macie** | “Is there sensitive data in my S3 buckets?” |
| **Trusted Advisor** | “Am I following AWS best practices? Can I save money?” |
| **AWS Config** | “Are my resource configurations compliant?” |
| **Q Developer** | “Are there security issues in the code I’m writing?” |

**Recall check**

- **One place for all security findings / compliance dashboard** → Security Hub
- **Vulnerability and CVE scanning of deployed resources** → Inspector
- **Detect attacks and malicious activity** → GuardDuty
- **Find sensitive data in S3** → Macie
- **Cost and best-practice recommendations** → Trusted Advisor
- **Security issues in source code as you write it** → Q Developer
- **Resource configuration compliance** → AWS Config

---

### AWS Service Catalog

**What it is.** Lets an organisation publish a list of **approved AWS products** (CloudFormation templates) that employees can self-serve, without being able to create arbitrary resources.

```plain text
Without Service Catalog:
Developer → creates anything → possible security issues

With Service Catalog:
Admin → defines approved products → developer picks only from that list
```

Approved products might be: a standard application EC2 stack, an approved database configuration, an approved VPC.

**Recall check**

- **Employees can deploy only approved AWS resources** → Service Catalog
- **Self-service with governance** → Service Catalog

---

### AWS Marketplace

**What it is.** An online store for **third-party software** that runs on AWS — security products, database tools, monitoring, ML products, SaaS.

Instead of installing a vendor’s firewall manually, you deploy it from the Marketplace with billing handled through your AWS account.

**Recall check**

- **Buy software from external vendors** → AWS Marketplace
- **Third-party / vendor solution** → AWS Marketplace

---

### AWS Serverless Application Repository (SAR)

**What it is.** A catalogue of **ready-made serverless applications** you can find and deploy, rather than building the same components repeatedly.

```plain text
Developer → search SAR → choose an application → deploy
                                                    ↓
                       AWS creates Lambda, API Gateway, DynamoDB, etc.
```

**Example**: you need an image-resizing pipeline (S3 upload → Lambda resize → store result). A SAR application does it without you writing anything.

Publish to it with `sam publish`.

**Recall check**

- **Ready-made / pre-built serverless application** → SAR
- **Reuse serverless components** → SAR
- **Build your own serverless infrastructure** → SAM / CloudFormation, **not** SAR
- **Commercial third-party software** → AWS Marketplace
- **Control what developers may deploy** → Service Catalog

---

### AWS Certificate Manager (ACM)

**What it is.** Creates, imports, manages, and **automatically renews** SSL/TLS certificates.

**Use:** enable HTTPS on AWS services.

**Supported integrations:** Application Load Balancer, Network Load Balancer, API Gateway, CloudFront, Elastic Beanstalk.

**Two facts that get tested:**

- **ACM-issued public certificates renew automatically.** That is the primary reason to use ACM over uploading your own.
- **CloudFront certificates must be in `us-east-1`**, regardless of where the rest of your infrastructure lives. This is because CloudFront is a global service whose control plane lives in `us-east-1`.

**The alternatives:**

| Service | Difference |
| --- | --- |
| **ACM** | Create, manage, and **auto-renew** public SSL/TLS certificates |
| **IAM certificate store** | Store an **uploaded** server certificate (legacy). **No auto-renewal** |
| **AWS Private CA** | Issue **private** certificates for internal applications |

**Recall check**

- **Enable HTTPS** → ACM
- **Automatically renew a certificate** → ACM
- **Upload an existing SSL/TLS certificate** → ACM import, or IAM (legacy)
- **Which services can store server certificates** → **ACM + IAM**
- **Private / internal certificates** → AWS Private CA
- **CloudFront custom domain certificate** → must be in **us-east-1**

---

### AWS CodeCommit

**Status:** closed to new customers in June 2024, and **no longer on the DVA-C02 in-scope services list.** Kept here because it can still appear in a question *stem* as a source repository, and because one fact about it is a durable trap.

**What it was.** AWS’s managed private Git repository service. Developers used ordinary Git commands (`clone`, `pull`, `push`), with IAM controlling who could access which repository.

**The one fact worth keeping — IAM login credentials are not Git credentials.**

Supported authentication methods:

- **HTTPS Git credentials** — an IAM-generated username and password specifically for Git over HTTPS
- **SSH keys** — an IAM user uploads an SSH public key
- **AWS access keys** — with the AWS CLI or the Git credential helper

**Not supported:**

- **IAM console username and password** — these log you into the AWS Console. They are **not** usable for Git operations.

The trap works because “IAM username and password” sounds correct — it *is* a username and password — but it is a different credential from the Git credentials IAM generates for CodeCommit.

| Scenario | Authentication |
| --- | --- |
| Clone over HTTPS | **Git credentials** |
| Clone over SSH | **SSH key** |
| Application using the AWS SDK | Access keys or an IAM role |
| Console login | IAM password — **not for Git** |

**Two secondary features, for completeness**

- **Triggers and notifications** — a repository could fire an **SNS** notification or invoke a **Lambda** on a push or branch/tag event, and integrate with EventBridge so a commit starts a CodePipeline execution. The modern equivalent for a GitHub source is a webhook through **CodeStar Connections** (now “AWS CodeConnections”), which is what a current pipeline uses.
- **Approval rule templates** — pull requests could require a minimum number of approvals from specified users before merging. Governance rather than developer skill, and now firmly out of scope.

**Recall check**

- **CodeCommit Git access** → ✅ SSH keys, ✅ HTTPS Git credentials, ✅ AWS access keys (CLI/API), ❌ IAM console username/password
- **Assume GitHub or S3** as the source stage in any modern CodePipeline question
- **Connect a pipeline to GitHub** → CodeConnections (formerly CodeStar Connections)

---

### AWS Batch and other compute distractors

Worth one line, because they are the correct answer to “Lambda can’t do this”:

- **AWS Batch** — managed batch computing for long-running jobs. The answer when a job exceeds Lambda’s 15 minutes and is a batch job rather than a service.
- **App Runner** — the simplest possible managed container/web deployment, between Beanstalk and Fargate.
- **Lightsail** — simplified VPS with bundled pricing.

---

### Root user and account-level credentials

**What to know.** The root user represents the entire AWS account. It should be used only for account setup, billing and account management, and the handful of genuinely root-only operations — then locked away behind MFA.

**Root-only operations include:** closing the account, changing the account name or email, changing support plans, enabling IAM access to Billing, and restoring an IAM user’s deleted permissions.

**Not root-only (common traps):**

- **IAM user access keys** — for CLI/API access, created by IAM users or admins
- **IAM user passwords** — for console login, created or reset by IAM users or admins
- **EC2 key pairs** — for SSH, created by anyone with EC2 permissions

**A corrected legacy fact:** older material states that **CloudFront key pairs** (for signed URLs) can only be created by the root user. That was true of the legacy “trusted signer account” mechanism, which AWS has superseded with **key groups** — and key groups require no root access. Treat root-only CloudFront key pairs as a historical footnote, not current practice.

**Recall check**

- **Programmatic access** → IAM access keys
- **Console login** → IAM password
- **SSH into EC2** → EC2 key pair
- **Billing permissions don’t work for an IAM user** → root must enable IAM access to Billing
- **Root user should be used only for** → account setup, billing, and root-only operations
- **“Only root can create a CloudFront key pair”** → legacy; **key groups** are current

## **AWS Device Farm** ❌ → tests mobile/web applications on real devices.

## AWS Amplify Hosting

**Mental model**

**Amplify Hosting = easy hosting for web apps directly from GitHub/GitLab/etc.**

```plain text
GitHub
  ↓
Amplify Hosting
  ↓
Build → Test → Deploy
  ↓
Your website
```

**What it is**

A fully managed service for **hosting and deploying web applications**.

Common apps:

- React
- Next.js
- Vue
- Angular

**Use**

Use it when you want to:

- Connect a Git repository
- Automatically build and deploy your app
- Get a hosted URL
- Automatically redeploy when code changes

**How**

1. Connect GitHub repository to Amplify Hosting.
2. Amplify detects the application.
3. Configure build settings in **`amplify.yml`**.
4. Amplify builds and deploys the application.

**Important points**

| Feature | Meaning |
| --- | --- |
| Git integration | Connect GitHub/GitLab/Bitbucket |
| CI/CD | Automatically build and deploy |
| `amplify.yml` | Build, test, and deployment configuration |
| Preview deployments | Test branches/changes before production |
| Custom domain | Connect your own domain |
| HTTPS | SSL/TLS provided |

**Exam memory**

> **Git repository → Amplify Hosting → build/test/deploy web app**

**`amplify.yml` = tells Amplify what to do during build and testing.**

**`amplify pull` = bring Amplify backend/configuration from AWS to local machine.**

**`aws-exports.js` = is simply a configuration file containing information for an Amplify application's region, user pool, or identity pool. **

**Amplify Studio **- a visual development environment that simplifies the creation of scalable, full-stack web and mobile apps. Use can use Amplify Studio to build your frontend UI with a set of ready-to-use UI components, create an app backend with AWS resources, and then connect the two together.

## AWS Distro for OpenTelemetry (ADOT)

### Mental model

**ADOT = collect + trace your application and send telemetry to different backends.**

What it is

- AWS-supported distribution of **OpenTelemetry**
- Collects **traces, metrics, and logs**
- Works with automatic instrumentation for supported applications
- Uses **ADOT Collector** to process and export telemetry

Use

Use ADOT when you need:

- Distributed tracing
- Automatic instrumentation
- Tracing things like **HTTP requests, AWS SDK calls, and JDBC/SQL**
- Send telemetry to **different observability backends**
- Avoid changing/re-instrumenting application code

ADOT vs X-Ray

| ADOT | X-Ray |
| --- | --- |
| OpenTelemetry-based | AWS X-Ray-specific |
| Multiple backends | Mainly AWS X-Ray |
| Vendor-neutral | AWS-focused |
| Collector available | X-Ray daemon/agent |

Exam memory

**Multiple tracing backends → ADOT / OpenTelemetry**

**AWS X-Ray specifically → X-Ray**

**ADOT Collector → collects, processes, and exports telemetry**

### Amazon GuardDuty

**What it is:** An AWS service that **detects suspicious or malicious activity** in your AWS account.

Think:

> **GuardDuty = security threat detector**

Example:

```plain text
AWS Account
   ↓
GuardDuty
   ↓
Detect suspicious activity
   ↓
Finding / Alert
```

It can detect things like:

- compromised AWS credentials
- unusual API activity
- malicious IP addresses
- cryptocurrency mining activity
- suspicious EC2 behavior

### Example

Someone steals an IAM access key and starts making unusual API calls from another country.

```plain text
Stolen credentials
       ↓
Unusual AWS activity
       ↓
GuardDuty
       ↓
Security finding 🚨
```

**GuardDuty does NOT:**

- block attacks by itself like a firewall
- scan your application source code
- encrypt your data

### Exam memory

> **GuardDuty = detects threats**

**GuardDuty → detection**

**WAF → web application protection**

**Inspector → vulnerability scanning**

**Macie → sensitive data/PII discovery in S3**

### Amazon Pinpoint

**Amazon Pinpoint = customer communication and engagement service.**

It helps you **send messages to users** and track how they interact with those messages.

Example:

```plain text
Your game
   ↓
Amazon Pinpoint
   ↓
Player receives:
📱 Push notification
📧 Email
📩 SMS
```

For example, your game could send:

> “You haven't played for 7 days. Come back and get your daily reward!”

Pinpoint can also track things like:

- who opened/clicked a message
- user engagement
- campaigns
- audience segments

### What it is NOT

**Pinpoint ❌ → real-time data synchronization**

It does not synchronize:

```plain text
Player A's game state
       ↕
Player B's game state
```

That's where **AppSync ✅** fits.

### Exam memory

> **Pinpoint = send messages to users + measure engagement**

**AppSync = synchronize application data in real time**

# Question patterns

> A team wants continuous compliance monitoring and a history of resource configuration changes. Which service?
> → **AWS Config**

> A developer needs to know who changed an IAM role. Is AWS Config the answer?
> → **No — CloudTrail identifies who; Config checks compliance**

> A developer asks whether resources still match the CloudFormation template. Which feature?
> → **CloudFormation Drift Detection**

> Which AWS Config feature detects non-compliant resources against a rule?
> → **Config Rules (AWS-managed or custom Lambda)**

> Which resources does AWS Config cover?
> → **EC2, S3, IAM, Lambda, RDS, security groups, and more**

> How can AWS Config automatically remediate a non-compliant resource?
> → **Remediation actions via Systems Manager Automation**

> A team must process very large datasets with Spark, Hadoop, Hive, Presto, or HBase in batch. Which service?
> → **Amazon EMR (Elastic MapReduce)**

> Why is EMR usually a distractor on DVA-C02?
> → **It requires provisioning a cluster to run big-data frameworks, which developer scenarios rarely want**

> A team needs a managed data warehouse for heavy, repeated BI SQL workloads over structured data. Which service?
> → **Amazon Redshift**

> What is the key difference between Redshift and Athena?
> → **Redshift loads data into the warehouse for repeated BI; Athena queries S3 data in place for ad-hoc analysis**

> In the analytics family, which service processes huge batch datasets?
> → **EMR**

> In the analytics family, which service handles real-time streaming?
> → **Kinesis Data Streams**

> In the analytics family, which service analyses structured analytical data with SQL?
> → **Redshift**

> In the analytics family, which service is the fast application database?
> → **DynamoDB**

> Which member of the analytics family queries S3 in place and is in scope?
> → **Athena**

> A team wants one dashboard aggregating security findings from GuardDuty, Inspector, Config, and IAM Access Analyzer. Which service?
> → **AWS Security Hub**

> Does Security Hub fix security issues?
> → **No — it centralizes and prioritizes findings (and runs standards checks)**

> Which standards checks does Security Hub run?
> → **CIS AWS Foundations Benchmark and AWS Foundational Security Best Practices**

> A team must scan deployed EC2 instances, ECR container images, and Lambda functions for known CVEs. Which service?
> → **Amazon Inspector**

> Which service detects malicious or anomalous activity such as crypto mining or credential exfiltration?
> → **Amazon GuardDuty**

> Which data sources does GuardDuty analyze?
> → **CloudTrail, VPC flow logs, and DNS logs (with machine learning)**

> Does GuardDuty block attacks by itself?
> → **No — it detects and produces findings; it is not a firewall**

> A team must discover and classify sensitive data (PII, credentials) in S3. Which service?
> → **Amazon Macie**

> A team wants best-practice recommendations across cost, performance, security, fault tolerance, and limits. Which service?
> → **AWS Trusted Advisor**

> Which five categories does Trusted Advisor cover?
> → **Cost Optimisation, Performance, Security, Fault Tolerance, Service Limits**

> Which service answers "do my deployed workloads have known vulnerabilities"?
> → **Amazon Inspector**

> Which service answers "is something malicious happening right now"?
> → **Amazon GuardDuty**

> Which service answers "is there sensitive data in my S3 buckets"?
> → **Amazon Macie**

> Which service answers "am I following AWS best practices and can I save money"?
> → **AWS Trusted Advisor**

> A company wants employees to self-serve only approved, pre-defined AWS products without creating arbitrary resources. Which service?
> → **AWS Service Catalog (published CloudFormation templates as approved products)**

> What problem does Service Catalog solve?
> → **Self-service with governance — developers pick only from approved products**

> A team wants to buy third-party software (security, database, monitoring, ML) with billing through AWS. Which service?
> → **AWS Marketplace**

> A developer needs a ready-made serverless application (e.g. image-resizing pipeline) to deploy without writing code. Which service?
> → **AWS Serverless Application Repository (SAR)**

> How is an application published to SAR?
> → **`sam publish`**

> A developer needs to build their own serverless infrastructure. SAR or SAM/CloudFormation?
> → **SAM / CloudFormation — not SAR**

> Which service creates, imports, manages, and automatically renews SSL/TLS certificates?
> → **AWS Certificate Manager (ACM)**

> Why choose ACM-issued public certificates over uploading your own?
> → **They renew automatically**

> Which services integrate with ACM for HTTPS?
> → **ALB, NLB, API Gateway, CloudFront, Elastic Beanstalk**

> A team uses CloudFront with a custom domain. Where must the certificate be created?
> → **In `us-east-1` (CloudFront's control plane is global/us-east-1)**

> Which service stores an uploaded server certificate but does NOT auto-renew (legacy)?
> → **IAM certificate store**

> Which service issues private certificates for internal applications?
> → **AWS Private CA**

> Which services can store server certificates?
> → **ACM and IAM**

> A developer needs to enable HTTPS on an ALB. Which service?
> → **ACM**

> Is CodeCommit on the current DVA-C02 in-scope services list?
> → **No — closed to new customers June 2024 and no longer in scope**

> Which authentication methods are supported for CodeCommit Git access?
> → **SSH keys, HTTPS Git credentials, and AWS access keys (CLI/API)**

> Which credential is NOT valid for CodeCommit Git operations?
> → **The IAM console username and password**

> A developer wants to clone a CodeCommit repository over HTTPS. Which credential?
> → **HTTPS Git credentials (IAM-generated for Git)**

> A developer wants to clone over SSH. Which credential?
> → **An SSH key uploaded to the IAM user**

> How should a modern CodePipeline connect to GitHub?
> → **CodeConnections (formerly CodeStar Connections)**

> What source stage should be assumed in a modern CodePipeline question?
> → **GitHub or S3**

> A job exceeds Lambda's 15-minute limit and is a batch job. Which service?
> → **AWS Batch**

> A team wants the simplest managed container/web deployment between Beanstalk and Fargate. Which service?
> → **AWS App Runner**

> A developer wants a simplified VPS with bundled pricing. Which service?
> → **Amazon Lightsail**

> Which operations are root-user only?
> → **Closing the account, changing account name/email, changing support plans, enabling IAM access to Billing, restoring an IAM user's deleted permissions**

> An IAM user cannot access billing even with permissions. What must the root user do?
> → **Enable IAM access to Billing**

> How should the root user be protected?
> → **Lock it away behind MFA and use it only for account setup, billing, and root-only operations**

> Which credentials are used for programmatic CLI/API access?
> → **IAM access keys**

> Which credentials are used for console login?
> → **IAM user passwords**

> Which credentials are used to SSH into EC2?
> → **EC2 key pairs**

> Are EC2 key pairs root-only?
> → **No — anyone with EC2 permissions can create them**

> Older material says only root can create CloudFront key pairs for signed URLs. What is current?
> → **Legacy — the modern mechanism is key groups, which require no root access**

> A mobile app must be tested on real physical devices. Which service?
> → **AWS Device Farm**

> A developer connects a GitHub repo and gets automatic build/test/deploy of a web app. Which service?
> → **AWS Amplify Hosting**

> Which file configures the Amplify Hosting build, test, and deployment steps?
> → **`amplify.yml`**

> Which command brings Amplify backend/configuration from AWS to the local machine?
> → **`amplify pull`**

> What is `aws-exports.js`?
> → **A config file with the Amplify app's region, user pool, and identity pool info**

> Which visual development environment simplifies building full-stack Amplify apps with ready-made UI components?
> → **Amplify Studio**

> A team needs vendor-neutral telemetry (traces, metrics, logs) sent to multiple observability backends. Which option?
> → **AWS Distro for OpenTelemetry (ADOT)**

> Which ADOT component collects, processes, and exports telemetry?
> → **The ADOT Collector**

> How do ADOT and X-Ray differ in their basis?
> → **ADOT is OpenTelemetry-based and multi-backend; X-Ray is AWS X-Ray-specific**

> A game must send push notifications, email, and SMS to users and measure engagement. Which service?
> → **Amazon Pinpoint**

> What can Pinpoint track?
> → **Opens/clicks, user engagement, campaigns, audience segments**

> Pinpoint is proposed to synchronize real-time game state between players. Is it correct?
> → **No — use AWS AppSync for real-time data synchronization**

> Which service synchronizes application data in real time?
> → **AWS AppSync**

> Which service protects web applications at the edge (referenced alongside GuardDuty as detection vs protection)?
> → **AWS WAF — web application protection (GuardDuty detects; WAF protects)**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Compliance / config history | **AWS Config** |
| Who made the change | **CloudTrail** |
| Matches template | **CloudFormation Drift Detection** |
| Config rules | **AWS-managed or custom Lambda; detect non-compliance** |
| Config remediation | **Systems Manager Automation** |
| Config coverage | **EC2, S3, IAM, Lambda, RDS, security groups** |
| Big data batch (Spark/Hadoop) | **EMR** |
| EMR distractor reason | **cluster you provision to run frameworks** |
| Data warehouse / BI | **Redshift** |
| Redshift vs Athena | **load into warehouse (repeated BI) vs query S3 in place (ad-hoc)** |
| Analytics compressed | **EMR processes / Kinesis streams / Redshift analyses / DynamoDB stores** |
| In-scope analytics on S3 | **Athena** |
| Aggregate security findings | **Security Hub** |
| Security Hub fixes? | **No — centralizes/prioritizes + standards checks** |
| Security Hub standards | **CIS AWS Foundations, AWS Foundational Best Practices** |
| Deployed CVE scanning | **Inspector** |
| Threat detection | **GuardDuty** |
| GuardDuty sources | **CloudTrail, VPC flow logs, DNS logs** |
| GuardDuty blocks? | **No — detects only** |
| Sensitive data in S3 | **Macie** |
| Best-practice recommendations | **Trusted Advisor** |
| Trusted Advisor categories | **Cost, Performance, Security, Fault Tolerance, Service Limits** |
| Deployed vulnerabilities question | **Inspector** |
| Malicious activity question | **GuardDuty** |
| Sensitive data question | **Macie** |
| Best-practices/cost question | **Trusted Advisor** |
| Self-service with governance | **Service Catalog** |
| Service Catalog unit | **approved CloudFormation products** |
| Third-party / vendor software | **AWS Marketplace** |
| Ready-made serverless app | **SAR** |
| Publish to SAR | **`sam publish`** |
| Build your own serverless | **SAM / CloudFormation (not SAR)** |
| Certificates | **ACM** |
| Auto-renew public cert | **ACM** |
| ACM integrations | **ALB, NLB, API Gateway, CloudFront, Elastic Beanstalk** |
| CloudFront cert region | **`us-east-1`** |
| Uploaded cert, no auto-renew | **IAM certificate store (legacy)** |
| Private/internal certificates | **AWS Private CA** |
| Store server certificates | **ACM + IAM** |
| CodeCommit status | **closed to new customers June 2024; out of scope** |
| CodeCommit Git auth | **SSH keys, HTTPS Git credentials, AWS access keys** |
| CodeCommit trap | **IAM console username/password is NOT Git creds** |
| Connect pipeline to GitHub | **CodeConnections (formerly CodeStar Connections)** |
| Modern pipeline source | **GitHub or S3** |
| Long-running batch job | **AWS Batch** |
| Simplest managed container deploy | **App Runner** |
| Simplified VPS | **Lightsail** |
| Root-only operations | **close account, change name/email, support plans, enable Billing access, restore IAM perms** |
| Billing for IAM user | **root must enable IAM access to Billing** |
| Programmatic access | **IAM access keys** |
| Console login | **IAM password** |
| SSH into EC2 | **EC2 key pair** |
| CloudFront key pair root-only | **legacy — key groups are current** |
| Mobile device testing | **AWS Device Farm** |
| Amplify Hosting | **Git repo → build/test/deploy web app** |
| Amplify build config | **`amplify.yml`** |
| Amplify pull | **bring backend/config from AWS to local** |
| `aws-exports.js` | **Amplify region/user pool/identity pool config** |
| Amplify Studio | **visual full-stack app builder** |
| OpenTelemetry | **ADOT** |
| ADOT component | **ADOT Collector (collect/process/export)** |
| ADOT vs X-Ray | **multi-backend/vendor-neutral vs AWS-specific** |
| Customer engagement campaigns | **Pinpoint** |
| Pinpoint channels | **push, email, SMS** |
| Pinpoint is not | **real-time data synchronization** |
| Real-time data sync | **AppSync** |
| Web application protection | **WAF** |
