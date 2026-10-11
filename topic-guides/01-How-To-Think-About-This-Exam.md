# AWS Certified Developer – Associate (DVA-C02)
**Complete Study Reference — Concepts First**
**How this document is organised.** Services are grouped by the role they play in an application, not alphabetically, because that is how the exam presents them: a scenario describes an application, and you have to know which piece is missing or broken.
Each service is written as:
- **Why it exists** — the problem that makes the service necessary. If you know this, you can usually derive the rest.
- **How it works** — the mechanism. Most exam questions are answerable from mechanism alone.
- **The decisions** — why you would pick this over the alternative, with the *reason*, not just a mapping.
- **How it fails** — failure modes and their causal chains. This is Domain 4, and it is also how most Domain 1–3 questions are disguised.
- **Numbers** — limits, where they matter.
- **Recall check** — the compressed keyword→answer pairs. Use these to *verify* you understood, not to substitute for understanding.
**Formatting note.** Headings follow the actual nesting, three levels deep at most:
- **Heading 1** — the part titles (Part 0–8) and *How to study from this*
- **Heading 2** — each service or topic inside a part (AWS Lambda, Amazon DynamoDB, …), plus the 7A/7B dividers and the four domain tables in Part 8
- **Heading 3** — the individual services inside Part 7A and 7B, since those sit one level deeper
Everything below that — **Why it exists**, **How it works**, **Important points**, **Recall check**, and so on — is **bold text**, not a heading. So Notion’s outline shows parts, then services, and stops there instead of drowning in sub-entries.
**Contents**
- **Part 0** — How to think about this exam
- **Part 1 — Compute:** Lambda · EC2 & Auto Scaling · ECS/Fargate/ECR · Elastic Beanstalk
- **Part 2 — Data:** DynamoDB · DynamoDB Streams · DAX · RDS & Aurora · ElastiCache · S3 · EBS vs EFS
- **Part 3 — APIs, events, streams:** API Gateway · ALB · CloudFront · Route 53 · SQS · SNS · EventBridge · Step Functions · Kinesis · AppSync
- **Part 4 — Security:** IAM · STS · Cognito · KMS · Secrets Manager / Parameter Store / AppConfig · S3 encryption · WAF · VPC, security groups & NACLs
- **Part 5 — Deployment:** CloudFormation · SAM · CDK · CLI & local config · CodePipeline · CodeBuild · CodeDeploy · CodeArtifact · deployment strategy comparison
- **Part 6 — Observability:** retries, backoff & idempotency · CloudWatch · X-Ray · CloudTrail
- **Part 7 — The rest:** in-scope but low-frequency (Athena, OpenSearch, Systems Manager, Q Developer, Amplify, CloudShell), then distractors (Config, EMR, Redshift, the security-services family, Service Catalog, Marketplace, SAR, ACM, CodeCommit, root credentials)
- **Part 8** — The reflex list, split by domain weight
- **How to study from this**
---
# PART 0 — HOW TO THINK ABOUT THIS EXAM
**What the exam is actually testing**
**Official structure**
|    |    |
| --- | --- |
| Questions | 65 total — **50 scored**, 15 unscored and unmarked |
| Format | Multiple choice (1 of 4) and multiple response (2+ of 5+) |
| Time | 130 minutes |
| Passing score | **720** on a 100–1,000 scale |
| Scoring model | **Compensatory** — you only need to pass overall, not each domain |
| Cost | 150 USD (varies by country) |
| Validity | 3 years |
| Retake | 14-day wait after a failure, no annual cap |
Unanswered questions score as incorrect and there is no guessing penalty. **Never leave a blank.**
**Domain weights**
| Domain | Weight |
| --- | --- |
| 1. Development with AWS Services | **32%** |
| 2. Security | **26%** |
| 3. Deployment | **24%** |
| 4. Troubleshooting and Optimization | **18%** |
Security being second-largest is the single most useful thing to know about this exam. Most candidates over-prepare Domain 1 and lose the exam in Domain 2.
**The five questions behind almost every question**
Once you stop pattern-matching keywords, you notice the exam only asks a handful of things in different costumes:
**1. “Which service does this job?”** — Tests whether you know what each service is *for*. Answerable from “why it exists.”
**2. “Why is this broken?”** — A symptom is described; you name the cause. Answerable from mechanism. Example: duplicate SQS processing is not a bug in SQS, it is a consequence of visibility timeout being shorter than the work.
**3. “Which is the least effort / cheapest / most secure way?”** — Two answers both work; one is idiomatic. AWS’s idiomatic answer is almost always: use the managed feature instead of writing code, use a role instead of a key, use the purpose-built service instead of a general one.
**4. “What is the correct order / correct name?”** — Lifecycle hooks, buildspec phases, API names. This is the only genuinely memorisation-heavy category, and it is small.
**5. “What does this number have to be?”** — Capacity units, timeouts, limits. Understand *why* the number exists and it becomes derivable rather than memorised.
**Three heuristics that resolve most ties**
**Roles over credentials.** Any answer containing “store access keys”, “hardcode credentials”, or “put the secret in an environment variable” is wrong when a role is available. AWS built roles specifically so credentials never need to exist at rest.
**Managed over custom.** If one answer says “write a Lambda function to do X” and another says “enable feature X”, the feature wins — unless the question explicitly requires custom logic. The exam is partly a test of whether you know what AWS already built for you.
**Least privilege, least effort, least code — in that order.** Security questions prefer the narrowest permission. Everything else prefers the fewest moving parts.
**The scope boundary — what NOT to study**
The official in-scope service list is the authority. These are in scope:
| Category | Services |
| --- | --- |
| **Compute** | EC2, Elastic Beanstalk, Lambda |
| **Containers** | ECR, ECS, EKS |
| **Database** | Aurora, DynamoDB, ElastiCache, RDS |
| **Storage** | EBS, EFS, S3 |
| **Application Integration** | AppSync, EventBridge, SNS, SQS, Step Functions |
| **Analytics** | Athena, Kinesis, OpenSearch Service |
| **Developer Tools** | Amplify, CloudShell, CodeArtifact, CodeBuild, CodeDeploy, CodePipeline, X-Ray, **Amazon Q Developer** |
| **Management & Governance** | AppConfig, CDK, CloudFormation, CloudTrail, CloudWatch, CLI, Systems Manager |
| **Networking & Content Delivery** | API Gateway, CloudFront, ELB, Route 53, VPC |
| **Security, Identity & Compliance** | Cognito, IAM, KMS, Secrets Manager, STS, WAF |
**Job tasks explicitly out of scope for you as a candidate:** designing architectures, designing CI/CD pipelines from scratch, administering IAM users and groups, administering servers and operating systems, designing VPC networking. You need to *use* these, not design them.
**Services in your notes that are NOT on the in-scope list:** EMR, Redshift, AWS Config, Security Hub, Inspector, GuardDuty, Trusted Advisor, Service Catalog, AWS Marketplace, ACM, Serverless Application Repository, and EC2 purchasing options (Spot/RI/Savings Plans/Dedicated Hosts).
I have **kept all of them** in Part 7 of this document, because they still appear as **wrong answers**. Knowing why Inspector is not the answer to a code-review question is worth as much as knowing what Q Developer does. But do not spend study *hours* there.
**What changed recently (so you don’t study stale facts)**
- **December 2024 exam guide revision (v2.1)** added new skills including **Amazon Q Developer** and **event-driven patterns with Amazon EventBridge**.
- **CodeCommit** closed to new customers in 2024 and is no longer on the in-scope list. Assume GitHub or S3 as the source stage.
- **“CMK” / “Customer Master Key”** was renamed **“KMS key”** in 2021.
- **CloudFront key pairs** (root-only) were superseded by **key groups** (no root needed).
- **S3** has had **strong read-after-write consistency** since December 2020. Any answer that adds retries or delays to work around S3 consistency is wrong.
- **SnapStart** now covers Java, Python, and .NET (originally Java only).
- **SNS** can now filter on the message body, not just attributes — though EventBridge is still the answer for routing.
Practice-question banks lag AWS by a year or two. Where a fact has changed, this document flags both versions so you don’t argue with a mock exam.
