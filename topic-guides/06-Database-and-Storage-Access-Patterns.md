*PART 2 — DATA: WHERE STATE LIVES*

## AWS Database & Storage Access Patterns

| Service | Think of it as | Best when... | Example |
| --- | --- | --- | --- |
| **DynamoDB** | Fast NoSQL database | You need **key-based access, huge scale, flexible schema** | Get order by `orderId` |
| **RDS** | Traditional SQL database | You need **SQL, joins, relationships, transactions** | `JOIN customers + orders` |
| **Aurora** | High-performance SQL database | You need **relational SQL + higher performance/availability** | Large e-commerce DB |
| **Redshift** | Data warehouse | You need **analytics over huge datasets**, not normal application transactions | Analyze billions of sales records |
| **ElastiCache** | General-purpose cache | You need **very fast repeated reads** | Cache popular products |
| **DAX** | DynamoDB cache | You need **microsecond reads specifically for DynamoDB** | Frequently read DynamoDB items |
| **S3** | Object/file storage | You store **files/objects** | Images, videos, PDFs |
| **EBS** | EC2 disk | One EC2 instance needs **block storage** | Database disk / application disk |
| **EFS** | Shared filesystem | Multiple EC2s/containers need to **access the same files** | Shared `/uploads` folder |

AWS categorizes DynamoDB as a key-value/document database, RDS/Aurora as relational databases, ElastiCache as an in-memory cache, and Redshift as a data warehouse.

For storage, AWS describes **EBS as block storage attached to EC2** and **EFS as shared file storage for multiple compute resources**.

### The mental shortcut

```plain text
Need normal application DATABASE?
        ↓
   ┌────┴────┐
   ↓         ↓
DynamoDB   RDS/Aurora
NoSQL      SQL

Need ANALYTICS?
        ↓
     Redshift

Need FAST CACHE?
        ↓
   ┌────┴────┐
   ↓         ↓
ElastiCache  DAX
general      DynamoDB

Need FILE STORAGE?
        ↓
   ┌────┼────┐
   ↓    ↓    ↓
  S3   EBS   EFS
files  disk  shared files
```

**Important distinction:** DAX is specifically a DynamoDB caching layer, while ElastiCache is a general-purpose in-memory caching service.

This version is much better for your exam because when you see **RDS / Aurora / DynamoDB / Redshift / ElastiCache / DAX / S3 / EBS / EFS** as answer choices, you can immediately ask: **“What kind of data/access does the question describe?”**

---

# Question patterns

> A serverless application needs a NoSQL key-value/document store with single-digit millisecond responses at huge scale and a flexible schema. Which service?
> → **Amazon DynamoDB**

> An application needs SQL with joins, relationships, and transactions on a managed engine. Which service?
> → **Amazon RDS (or Aurora for higher performance/availability)**

> A team wants MySQL or PostgreSQL compatibility with better performance, auto-scaling storage, and faster failover. Which service?
> → **Amazon Aurora**

> A company must run analytics over billions of historical sales records, separate from its application transactions. Which service?
> → **Amazon Redshift (a data warehouse — a common distractor for the DVA exam)**

> A read-heavy application must be accelerated with a general-purpose in-memory cache for popular items. Which service?
> → **Amazon ElastiCache**

> An application needs microsecond-latency caching specifically in front of a DynamoDB table. Which service?
> → **Amazon DAX (DynamoDB Accelerator)**

> How do DAX and ElastiCache differ?
> → **DAX is a caching layer specifically for DynamoDB; ElastiCache is a general-purpose in-memory cache**

> An application stores images, videos, and PDFs. Which AWS storage service?
> → **Amazon S3 (object/file storage)**

> A single EC2 instance needs a block-storage disk for the OS or a database. Which service?
> → **Amazon EBS (EC2 disk, block storage)**

> Multiple EC2 instances and containers must access the same uploaded files through a shared /uploads folder. Which service?
> → **Amazon EFS (shared file storage for multiple compute resources)**

> According to AWS, how is EBS categorized versus EFS?
> → **EBS is block storage attached to EC2; EFS is shared file storage for multiple compute resources**

> An application needs key-based access at huge scale with a flexible schema — which database category?
> → **NoSQL (DynamoDB), categorized as key-value/document**

> Which AWS database categories do the exam distinguish for RDS/Aurora versus ElastiCache versus Redshift?
> → **RDS/Aurora are relational; ElastiCache is an in-memory cache; Redshift is a data warehouse**

> A question asks for a "general-purpose cache" versus a "DynamoDB cache". Which two services?
> → **ElastiCache (general purpose) and DAX (DynamoDB-specific)**

> Given the choices DynamoDB, RDS, Aurora, Redshift, ElastiCache, DAX, S3, EBS, EFS, what single question should you ask first?
> → **"What kind of data/access does the scenario describe?" (key-value, SQL, analytics, cache, or files)**

> A managed MySQL database must survive an AZ failure with automatic failover. Which RDS feature?
> → **RDS Multi-AZ (a synchronously replicated standby handles automatic failover)**

> A read-heavy RDS workload must be offloaded from the primary database without affecting failover. Which feature?
> → **RDS Read Replicas (asynchronous copies used to scale reads, not for failover)**

> What is the key distinction between RDS Multi-AZ and Read Replicas?
> → **Multi-AZ is for availability/failover (synchronous standby); Read Replicas are for scaling reads (asynchronous, can be promoted manually)**

> A DynamoDB table has unpredictable, spiky traffic and the team does not want to manage capacity. Which capacity mode?
> → **On-demand capacity mode (pay per request); provisioned mode with auto scaling is for more predictable workloads**

> What latency advantage does DAX provide over querying DynamoDB directly?
> → **Microsecond read latency for frequently accessed DynamoDB items versus single-digit millisecond reads from DynamoDB itself**

> An application needs a relational database with SQL and joins for an e-commerce order/customer model. Which service family?
> → **RDS/Aurora (relational), e.g. JOIN customers + orders**

> Which service is the wrong choice for normal per-request application transactions but the right choice for large-scale analytics?
> → **Redshift**

> An Elastic Beanstalk/EC2 application needs a disk that persists independent of the instance and supports high IOPS. Which storage?
> → **Amazon EBS**

> Multiple Fargate tasks and EC2 instances across Availability Zones need a shared, persistent filesystem. Which storage?
> → **Amazon EFS**

> Which storage option is best for storing large unstructured objects accessed over HTTP rather than mounting as a filesystem?
> → **Amazon S3**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Fast NoSQL, key-based, huge scale | **DynamoDB** |
| SQL, joins, relationships, transactions | **RDS** |
| Relational + higher performance/availability | **Aurora** |
| Analytics over huge datasets | **Redshift (DVA distractor)** |
| General-purpose in-memory cache | **ElastiCache** |
| DynamoDB microsecond cache | **DAX** |
| Object/file storage | **S3** |
| EC2 block disk | **EBS** |
| Shared filesystem for many compute | **EFS** |
| DAX vs ElastiCache | **DynamoDB-specific vs general-purpose cache** |
| DynamoDB category | **Key-value/document NoSQL** |
| RDS/Aurora category | **Relational** |
| Redshift category | **Data warehouse** |
| DynamoDB latency | **Single-digit milliseconds** |
| DAX latency | **Microseconds (DynamoDB only)** |
| First question on a data scenario | **What kind of data/access is described?** |
| RDS automatic failover | **Multi-AZ (synchronous standby)** |
| Scale reads on RDS | **Read Replicas (asynchronous)** |
| DynamoDB unpredictable traffic | **On-demand capacity mode** |
| DynamoDB predictable traffic | **Provisioned capacity + auto scaling** |
| EBS scope | **Block storage, attached to EC2 (single AZ)** |
| EFS scope | **Shared file storage, multi-AZ, many compute** |
| S3 use case | **Images, videos, PDFs, unstructured objects** |
| Shared /uploads folder | **EFS** |
| Analytics warehouse vs app DB | **Redshift vs RDS/Aurora** |
