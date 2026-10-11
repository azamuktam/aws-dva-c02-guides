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
| **Parameter Store ****`SecureString`** | ✅ Yes | ❌ No |
| **IAM DB authentication** | ❌ **No password exists** | N/A — tokens are short-lived by design |

**RDS proxy: **
- connection pooling and reuse.
- Many application connections → **RDS Proxy** → fewer DB connections.
- Especially useful for **Lambda + RDS/Aurora** because Lambda can create many short-lived connections.
**Recall check — RDS**
- **Need a SQL database, managed** → RDS
- **Read-heavy workload / more ****`SELECT`**** capacity** → Read Replicas
- **Automatic failover / high availability** → Multi-AZ
- **“Standby should serve reads”** → **wrong**, standbys serve nothing
- **Least code change to scale reads** → Read Replicas
- **Reduce database load with caching** → ElastiCache
- **Passwordless database login** → IAM DB authentication
- **Existing DB credentials must rotate automatically** → Secrets Manager
- **Too many DB connections → **RDS Proxy**. **
- **Standard RDS monitoring** → **database/instance-level** metrics in CloudWatch, e.g. `CPUUtilization`, `DatabaseConnections`, `FreeStorageSpace`.
- **Enhanced Monitoring** → **OS/process-level** details in CloudWatch Logs, e.g. **`mysqld`**** process memory**, individual process CPU usage, load average, and disk activity.
RDS **automated backups** are retained for a configurable period:
- **Minimum:** 1 day
- **Maximum:** **35 days**
- Default: **7 days**
- They support **point-in-time recovery (PITR)** during the retention window.
---
