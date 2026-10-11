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
