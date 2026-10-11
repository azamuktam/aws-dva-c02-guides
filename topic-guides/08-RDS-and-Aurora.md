## Amazon RDS and Aurora

### Amazon RDS (Relational Database Service)

**Mental model**

> AWS manages your database server for you.

- Fully managed **relational database** service.
- AWS handles:
  - Provisioning
  - Patching
  - Backups
  - Monitoring
  - Automatic failover
- You manage:
  - Database schema
  - SQL queries
  - Indexes
  - Performance tuning

### Supported Database Engines

- MySQL
- PostgreSQL
- MariaDB
- Oracle
- Microsoft SQL Server
- IBM Db2

---

## Amazon Aurora

**Mental model**

> Aurora is AWS's cloud-native relational database engine, managed through Amazon RDS.

Aurora is **not just "RDS with better hardware."** It has a different storage architecture designed by AWS for higher performance and availability while remaining compatible with MySQL or PostgreSQL.

Supports only:

- Aurora MySQL
- Aurora PostgreSQL

---

**Architecture Difference**

**RDS**

```plain text
RDS
 │
 ▼
DB instance
 │
 ▼
EBS-backed storage
 │
 ├── Tables
 ├── Indexes
 └── Database files
```

- One database instance owns its storage.
- Read replicas each have their own storage.

---

**Aurora**

```plain text
              Writer
                │
                │
                ▼
        ┌─────────────────┐
        │ Aurora Storage  │
        │                 │
        │   DATA          │
        │   DATA          │
        │   DATA          │
        └─────────────────┘
           ▲           ▲
           │           │
        Reader 1    Reader 2
```

- Compute and storage are separated.
- Writer and all reader instances share the same distributed storage.
- Adding replicas is much faster because no storage copy is required.

### But "one storage" doesn't mean one physical disk

This is important.

Aurora's storage is a **distributed storage system**, automatically replicated across multiple Availability Zones.

So mentally:

```plain text
              Writer
                 │
        ┌────────┴────────┐
        ↓                 ↓
   Aurora distributed storage
    AZ-A ↔ AZ-B ↔ AZ-C
        ↑                 ↑
        └── Readers ──────┘
```

---

**Performance**

**RDS**

- Standard database engines.
- Good for most applications.

**Aurora**

- Designed for up to **6× the throughput** of standard MySQL and PostgreSQL on similar hardware.

---

**Storage**

**RDS**

- Uses Amazon EBS volumes.
- Storage depends on the selected engine and configuration.

**Aurora**

- Automatically grows up to **256 TiB**.
- Distributed, fault-tolerant storage replicated across **3 Availability Zones** (six copies of data).

---

**Read Replicas**

**RDS**

- Supported.
- Engine-level asynchronous replication.
- Each replica has its own storage.

**Aurora**

- Up to **15 Aurora Replicas**.
- All replicas share the same storage.
- Includes a **Reader Endpoint** that automatically load balances read traffic.

Has its **own endpoint**.(Connection string)

---

**High Availability**

**RDS**

- **Multi-AZ** creates a standby instance in another AZ.
- Standby cannot serve read traffic.
- Used only for failover.

**Aurora**

- Storage is already replicated across multiple AZs.
- Aurora Replicas can be promoted automatically if the writer fails.
- Faster failover than standard RDS.

---

**Scaling**

**RDS**

- Vertical scaling (larger instance).
- Read scaling via Read Replicas.

**Aurora**

- Vertical scaling.
- Read scaling with up to 15 replicas.
- Aurora Serverless option for automatic compute scaling.

---

**Compatibility**

**RDS**

Choose when you need:

- Oracle
- SQL Server
- MariaDB
- Db2
- Standard MySQL/PostgreSQL

**Aurora**

Choose when you need:

- MySQL compatibility
- PostgreSQL compatibility
- Higher performance
- Faster failover
- Better scalability

---

**When to Choose**

**Choose RDS if:**

- Standard business applications
- Lower cost
- Need Oracle, SQL Server, MariaDB, or Db2
- Existing applications don't require Aurora's advanced architecture

**Choose Aurora if:**

- High-performance web applications
- SaaS platforms
- High availability
- Many read replicas
- Heavy read/write workloads
- Fast failover is important

---

**Why it exists**

Plenty of applications genuinely need relational data — joins, transactions across tables, an existing SQL codebase. RDS removes the operational work (patching, backups, failover, replication) while leaving you the familiar database engine.

**Engines:** MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, and **Aurora** (AWS’s MySQL/PostgreSQL-compatible engine with a distributed storage layer, faster replication, and Aurora Serverless for variable workloads).

**Read Replicas vs Multi-AZ — different problems, constantly confused**

These look similar (both create a second database) and solve completely unrelated problems. The distinguishing question is: *is the second copy serving traffic?*

**Read Replicas — for read throughput.(Asynchronous replication)**

```plain text
Application
   ├── writes ──→ RDS Primary
   └── reads  ──→ RDS Read Replica(s)
```

The replica is **readable**. Your application must be changed to send reads to the replica endpoint. Asynchronous replication, so replicas can lag slightly. Up to 5 (15 for Aurora). Can be cross-AZ or cross-Region.

Use when: read-heavy workload, too many `SELECT` queries, need to offload reporting.

**Multi-AZ — for availability (Synchronous replication)**

```plain text
RDS Primary  (all traffic)
     ⇅  synchronous replication
RDS Standby  (serves NOTHING — it just waits)
```

The standby is **not readable**. It exists solely to take over via automatic failover (DNS repoints to it) if the primary fails. Synchronous replication, so no data loss.

Use when: high availability, automatic failover, disaster recovery.

**“The standby should serve read traffic” is always wrong.** That is the single most reliable distractor in this area.

**ElastiCache** is the third option for read pressure — it removes queries entirely rather than adding capacity. But it requires application changes to handle cache invalidation, misses, and consistency. So when a question asks for **“least development effort”** to scale reads, **read replicas win**: point reads at a different endpoint, done.

- Used for **High Availability (HA)**.
- Uses **one endpoint (DNS)**.
- During failover, AWS updates the endpoint to the standby.
- **No connection string change** is required.

| Need | Answer | Why |
| --- | --- | --- |
| More read capacity | **Read Replicas** | Readable copies, simplest change |
| Survive an AZ failure | **Multi-AZ** | Automatic failover, standby not readable |
| Eliminate repeated queries entirely | **ElastiCache** | Fastest, but you own the cache logic |
| Least code change for read scaling | **Read Replicas** | Just a second endpoint |

**IAM database authentication**

Instead of a stored password, the application requests a short-lived **IAM authentication token** and uses that to connect.

```plain text
Application (with an IAM role)
        ↓
Request an auth token from RDS
        ↓
Connect using the token instead of a password
```

The value: **there is no password to store, leak, or rotate.** Supported by **MySQL and PostgreSQL** on RDS/Aurora.

Important boundary: it **does not rotate an existing database password** — it removes the need for one. If a question says “the existing password must be rotated automatically”, that’s Secrets Manager, not IAM DB auth.

| Approach | Stores a secret? | Rotation |
| --- | --- | --- |
| **Secrets Manager** | ✅ Yes | ✅ Automatic, with built-in RDS support |
| **Parameter Store `SecureString`** | ✅ Yes | ❌ No |
| **IAM DB authentication** | ❌ **No password exists** | N/A — tokens are short-lived by design |

**RDS proxy: **

- connection pooling and reuse.
- Many application connections → **RDS Proxy** → fewer DB connections.
- Especially useful for **Lambda + RDS/Aurora** because Lambda can create many short-lived connections.

**Recall check — RDS**

- **Need a SQL database, managed** → RDS
- **Read-heavy workload / more `SELECT` capacity** → Read Replicas
- **Automatic failover / high availability** → Multi-AZ
- **“Standby should serve reads”** → **wrong**, standbys serve nothing
- **Least code change to scale reads** → Read Replicas
- **Reduce database load with caching** → ElastiCache
- **Passwordless database login** → IAM DB authentication
- **Existing DB credentials must rotate automatically** → Secrets Manager
- **Too many DB connections → **RDS Proxy**. **
- **Standard RDS monitoring** → **database/instance-level** metrics in CloudWatch, e.g. `CPUUtilization`, `DatabaseConnections`, `FreeStorageSpace`.
- **Enhanced Monitoring** → **OS/process-level** details in CloudWatch Logs, e.g. **`mysqld` process memory**, individual process CPU usage, load average, and disk activity.

RDS **automated backups** are retained for a configurable period:

- **Minimum:** 1 day
- **Maximum:** **35 days**
- Default: **7 days**
- They support **point-in-time recovery (PITR)** during the retention window.

---

## RDS Encryption and SQL Server Encryption

### RDS Storage Encryption

**What it is:** Encrypts **RDS data at rest** using **AWS KMS**.

**How it works:**

```plain text
Database data
   ↓
RDS encryption
   ↓
Encrypted storage
```

RDS automatically encrypts data before it is written to the underlying storage and decrypts it when the database reads it.

**Also protects:**

- Automated backups
- Read replicas
- Snapshots

**Key point:** The application does not need to encrypt/decrypt the data itself.

**Exam memory:**

> **RDS encryption → AWS KMS → encryption at rest**

---

### Transparent Data Encryption (TDE)

**What it is:** A **SQL Server database-engine feature** that transparently encrypts database files/data at rest.

**How it works:**

```plain text
SQL Server
   ↓
TDE encrypts database data
   ↓
Storage
```

When SQL Server reads the data, TDE transparently decrypts it.

**Key point:** The application does not need to change its SQL queries or encryption logic.

**Exam memory:**

> **SQL Server + transparent database encryption → TDE**

---

### RDS Encryption vs TDE

|    | RDS Encryption | TDE |
| --- | --- | --- |
| Encryption level | AWS/RDS storage | SQL Server database engine |
| Key management | AWS KMS | SQL Server/TDE |
| Encrypts at rest | ✅ | ✅ |
| Automatic encrypt/decrypt | ✅ | ✅ |
| Application changes | No | No |
| Main exam clue | **KMS / RDS encryption** | **SQL Server / TDE** |

**Important distinction**

Both can encrypt data before it is stored and decrypt it when accessed.

> **RDS encryption = AWS-level storage encryption**
> **TDE = SQL Server-level transparent database encryption**

# Question patterns

> An RDS database must fail over automatically to a standby in another AZ with no connection-string change. Which feature?
> → **Multi-AZ deployment (synchronous replication; DNS endpoint repoints to the standby)**

> Read queries are overwhelming the primary. What is the simplest way to add read capacity?
> → **Read Replicas (point reads at the replica endpoint, no code logic rewrite; up to 15 with Aurora)**

> Is the Multi-AZ standby instance readable?
> → **No — it serves nothing and exists only for failover**

> Which RDS replication is synchronous and which is asynchronous?
> → **Multi-AZ is synchronous (no data loss); Read Replicas are asynchronous (can lag)**

> How many read replicas can an RDS instance and an Aurora cluster have?
> → **Up to 5 for RDS, up to 15 for Aurora**

> A read-heavy workload must scale with the least development effort. Which option wins?
> → **Read Replicas — just a second endpoint, versus ElastiCache which needs cache logic**

> Repeated identical queries should be eliminated entirely rather than served faster. Which service?
> → **ElastiCache (fastest, but you own cache invalidation, misses, and consistency)**

> The application must log in to the database without a stored password. Which feature?
> → **IAM database authentication (short-lived auth token; supported for MySQL and PostgreSQL on RDS/Aurora)**

> Existing database credentials must be rotated automatically. Which service?
> → **AWS Secrets Manager — IAM DB auth removes the password need but does not rotate an existing one**

> Does Parameter Store SecureString rotate secrets automatically?
> → **No — it stores a secret but has no built-in rotation**

> Many Lambda functions open connections to RDS and exhaust the connection limit. What reduces connection churn?
> → **RDS Proxy (connection pooling and reuse)**

> Which monitoring shows OS/process-level detail such as mysqld process memory and load average?
> → **Enhanced Monitoring (delivered to CloudWatch Logs)**

> Which metrics does standard RDS monitoring give in CloudWatch?
> → **Instance/database-level metrics like CPUUtilization, DatabaseConnections, FreeStorageSpace**

> What are the automated backup retention minimum, maximum, and default?
> → **Minimum 1 day, maximum 35 days, default 7 days**

> Recovery to any second within the retention window is required. Which feature?
> → **Automated backups with point-in-time recovery (PITR)**

> An existing unencrypted RDS database must be encrypted at rest. How?
> → **Snapshot → encrypt the snapshot → restore from the encrypted snapshot**

> Which key management backs RDS storage encryption?
> → **AWS KMS**

> What does RDS storage encryption also protect besides the running database?
> → **Automated backups, read replicas, and snapshots**

> A SQL Server database must encrypt data at rest using the database engine itself, with no query changes. Which mechanism?
> → **Transparent Data Encryption (TDE)**

> What is the difference between RDS encryption and TDE?
> → **RDS encryption is AWS-level KMS storage encryption; TDE is a SQL Server database-engine feature**

> Which database engines are supported by Aurora?
> → **Only Aurora MySQL and Aurora PostgreSQL**

> Across how many AZs and how many copies is Aurora storage replicated?
> → **3 Availability Zones, six copies of the data**

> How large can Aurora storage grow automatically?
> → **Up to 256 TiB**

> How do standard engines compare to Aurora for performance?
> → **Aurora is designed for up to 6× the throughput of standard MySQL/PostgreSQL on similar hardware**

> How is read traffic distributed across Aurora Replicas?
> → **By the Reader Endpoint, which automatically load balances reads**

> How does Aurora failover differ from standard RDS failover?
> → **Aurora promotes a replica automatically and fails over faster than standard RDS**

> What kind of storage underlies an RDS DB instance, and what underlies Aurora?
> → **RDS uses EBS-backed volumes per instance; Aurora uses a shared distributed storage layer across AZs**

> Do Aurora read replicas each have their own storage?
> → **No — all Aurora replicas share the same distributed storage, so adding replicas needs no storage copy**

> Which workloads should choose RDS over Aurora?
> → **Lower cost, standard business apps, or a need for Oracle, SQL Server, MariaDB, or Db2**

> Which option automatically scales compute for unpredictable workloads?
> → **Aurora Serverless**

> A workload needs many read replicas, heavy read/write, and fast failover. Which engine?
> → **Aurora**

> What does AWS manage for you on RDS, and what do you still manage?
> → **AWS manages provisioning, patching, backups, monitoring, and failover; you manage schema, SQL, indexes, and tuning**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Failover / HA | **Multi-AZ (standby, synchronous)** |
| Standby serves reads? | **No — standbys serve nothing** |
| Read scaling | **Read Replicas (asynchronous)** |
| Aurora replicas | **up to 15 (RDS up to 5)** |
| Least code change for read scaling | **Read Replicas** |
| Eliminate repeated queries | **ElastiCache** |
| Passwordless DB login | **IAM DB authentication (MySQL/PostgreSQL)** |
| Rotate existing DB password | **Secrets Manager** |
| Parameter Store SecureString rotation | **No** |
| Connection pooling / churn | **RDS Proxy (esp. Lambda)** |
| Standard RDS monitoring | **CloudWatch instance/database metrics** |
| Enhanced Monitoring | **OS/process-level, to CloudWatch Logs** |
| Automated backup retention | **min 1 day, max 35 days, default 7 days** |
| Point-in-time restore | **automated backups (up to 35 days)** |
| Encrypt existing DB | **snapshot → encrypt → restore** |
| RDS encryption | **AWS KMS (at rest)** |
| RDS encryption also protects | **backups, snapshots, read replicas** |
| SQL Server engine-level encryption | **TDE** |
| RDS encryption vs TDE | **AWS storage vs SQL Server engine** |
| Aurora engines | **Aurora MySQL, Aurora PostgreSQL** |
| Aurora storage replication | **3 AZs, 6 copies** |
| Aurora max storage | **256 TiB** |
| Aurora performance | **up to 6× standard MySQL/PostgreSQL** |
| Aurora read load balancing | **Reader Endpoint** |
| Aurora failover | **faster than RDS; replica promoted automatically** |
| RDS instance storage | **EBS-backed volumes** |
| Aurora storage | **shared distributed, multi-AZ** |
| Aurora replica storage | **shared — no copy needed** |
| RDS engines | **MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2** |
| Aurora Serverless | **automatic compute scaling** |
| Choose Aurora when | **high performance, HA, many replicas, fast failover, heavy read/write** |
| Choose RDS when | **Oracle/SQL Server/MariaDB/Db2, lower cost, standard apps** |
| AWS manages on RDS | **provisioning, patching, backups, monitoring, failover** |
| You manage on RDS | **schema, SQL queries, indexes, tuning** |
| One endpoint during failover | **yes — DNS endpoint updates, no connection change** |
