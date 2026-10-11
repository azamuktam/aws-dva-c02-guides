## Amazon Kinesis
Amazon Kinesis is an AWS service for **collecting, processing, and analyzing data that arrives continuously in real time**.
**Why it exists**
A queue is built around *messages consumed and deleted*. A stream is built around an *ordered, retained log* that multiple consumers can read independently, each at its own position — and re-read later. That is what analytics, change-data-capture, and real-time processing need.
**Kinesis Data Streams**
```plain text
Producers (applications, IoT devices, logs, clickstreams)
        ↓
   partition key → hash → shard
        ↓
   Shard 1   Shard 2   Shard 3      ← records retained, ordered within a shard
        ↓
Consumers (Lambda, KCL applications, Firehose, Managed Flink)
   each tracking its own position
```
**Ordering is per shard**, and the **partition key** decides which shard a record lands on. That single design choice explains both the ordering guarantee and the hot-shard failure mode.

| Limit | Value |
| --- | --- |
| Write per shard | **1 MB/s** or **1,000 records/s** |
| Read per shard (shared) | **2 MB/s**, max 5 `GetRecords` calls/s, **shared by all consumers** |
| Read per shard (**enhanced fan-out**) | **2 MB/s per consumer** |
| Record size | 1 MB max |
| Retention | **24 hours** default, up to **365 days** |

- **Capacity modes:** provisioned (you manage shards) or **on-demand** (scales automatically).
- **Resharding:** split a shard to scale out, merge to scale in.
- **Enhanced fan-out** gives each consumer its own dedicated 2 MB/s and *pushes* records rather than being polled, which lowers latency. Use it when several consumers each need full throughput.
- **`ProvisionedThroughputExceededException`** means either too few shards or — more often — a **hot shard** caused by a low-cardinality partition key.
#### DynamoDB Streams + Kinesis Adapter
**Use:** Real-time processing of DynamoDB table changes.
```plain text
DynamoDB
   ↓
DynamoDB Streams
   ↓
Kinesis Adapter
   ↓
Application on EKS
```
- **DynamoDB Streams** → records table changes (insert, update, delete)
- **Kinesis Adapter** → lets applications process DynamoDB Streams using Kinesis-style APIs/KCL
- **EKS** → runs the real-time analytics application
**Exam memory**
> **DynamoDB changes + Kinesis-style processing → Kinesis Adapter**
**Kinesis Client Library (KCL)**
Kinesis Client Library (KCL) is a library that helps applications **read and process records from Kinesis Data Streams**.
A library for building stream consumers that handles the hard parts: **shard discovery, load balancing across workers, and checkpointing**.
- **Each shard is processed by exactly one KCL worker, with exactly one record processor.** So you never need multiple instances to process a single shard — and adding workers beyond the shard count gains nothing.
- Checkpoints are stored in a **DynamoDB table** created by the KCL. That table needs adequate capacity, and it is a genuine and easily-missed source of throttling.
- Supports multiple consumer applications reading the same stream cooperatively.
**Kinesis Data Firehose**
Kinesis Data Firehose is a service that **automatically collects streaming data and delivers it to destinations**.
Firehose exists because most streaming use cases are just “get this data into a store” and don’t need shards, ordering, or replay.
- **Fully managed delivery**, **near real time**, **no shards, no consumer code**.
- Destinations: **S3, Redshift, OpenSearch, Splunk**, generic HTTP endpoints, third-party SaaS.
- **Buffering** by size (e.g. 5 MB) or interval (60–900 s), whichever comes first — this is why it is “near” real time.
- Can invoke a **Lambda** to transform records in flight, and convert JSON to **Parquet/ORC**.
- **No storage and no replay** — once delivered, Firehose has no copy.
**Managed Service for Apache Flink** (formerly Kinesis Data Analytics) runs SQL or Flink applications over a stream for real-time aggregation and analytics.
**Data Streams vs Firehose vs SQS**

|    | **Data Streams** | **Firehose** | **SQS** |
| --- | --- | --- | --- |
| Latency | Real time (\~200 ms) | Near real time (60 s+) | Real time |
| Storage / replay | ✅ up to 365 days (default 1 day) | ❌ | Up to 14 days, no replay |
| Ordering | ✅ per partition key | ❌ | FIFO queues only |
| Many consumers, same data | ✅ | ❌ | ❌ one consumer per message |
| Scaling | Shards or on-demand | Automatic | Automatic |
| Consumer code required | ✅ | ❌ | ✅ |

**Recall check — Kinesis**
- **Real-time streaming, live data, event stream** → Kinesis Data Streams
- **Read from Kinesis with checkpointing and multiple consumers** → KCL
- **One shard is processed by exactly one KCL worker** — more instances don’t help
- **KCL throttling** → check the DynamoDB checkpoint table’s capacity
- **Throttled writes** → add shards, or fix the partition key (hot shard)
- **Several consumers each needing full throughput** → enhanced fan-out
- **Reprocess data from days ago** → extend retention (up to 365 days)
- **Don’t want to manage shards** → on-demand mode
- **Load streaming data into S3/Redshift/OpenSearch with no code** → **Firehose**
- **“Near real time” in the question** → Firehose
- **Convert streaming JSON to Parquet** → Firehose format conversion
- **Need ordering, replay, or independent consumers** → Data Streams
- **Real-time SQL over a stream** → Managed Service for Apache Flink
- **Consumer lag** → `GetRecords.IteratorAgeMilliseconds`
- **`MergeShards`** — combines two *adjacent* shards into one. This *decreases* shard count. It's the scale-in operation, used to cut cost when a stream is over-provisioned.
- **`SplitShard`** ✅ — divides **hot **shard into two, increasing shard count and doubling that shard's capacity.
- **`AddShards`** — doesn't exist. Pure distractor.
- **`UpdateShardCount`** — real, and it does increase shards, but it resizes the *whole stream* uniformly.
- Kinesis strict ordering → same shard + `PutRecord` + `SequenceNumberForOrdering`
- Kinesis `PutRecords` → batch ingestion, not strict ordering
---
## Kinesis Agent
**What it is:**
A small software agent installed on a server/EC2 instance that **collects log files and continuously sends them to Amazon Kinesis**.
```plain text
EC2 / Server
   ↓
Kinesis Agent
   ↓
Kinesis Data Firehose
   ↓
S3 / Redshift / OpenSearch
```
**Use:**
When you have **logs or files on a server** and want to continuously stream them into AWS.
**Example:**
```plain text
/var/log/app.log
      ↓
Kinesis Agent
      ↓
Firehose
      ↓
S3
```
**Important:**
Kinesis Agent is mainly for **collecting and sending data**, not processing it.
### Exam memory
> **Kinesis Agent = collect logs/files from servers → send to Kinesis.**
> **Agent runs on your server/EC2.**
> **Common destination → Kinesis Data Firehose.**
## Kinesis Agent for Windows — Sink Types
**Sink = where Kinesis Agent for Windows sends the collected data.** AWS documents these built-in sink types:
- **Kinesis Data Streams** ✅
- **Kinesis Data Firehose** ✅
- **CloudWatch** ✅
- **CloudWatch Logs** ✅
- **Local FileSystem** ✅
For **Firehose**, the data can then be delivered to destinations such as **S3, Redshift, OpenSearch, and Splunk**.
### Exam memory
> **Agent → Sink → destination**
```plain text
Log file
   ↓
Kinesis Agent
   ↓
Kinesis Firehose
   ↓
S3 / Redshift / OpenSearch / Splunk
```
If the question asks **which sink type is NOT supported**, look for something outside the built-in sink types above.

# Question patterns

> A stream must be replayable by multiple consumers in real time. Which service?
> → **Kinesis Data Streams**

> Data must be transformed and loaded into S3/Redshift with no application code. Which service?
> → **Kinesis Data Firehose**

> A Lambda function must consume every record of every shard. How?
> → **Lambda event source mapping (polls each shard)**

> More throughput is needed. What changes?
> → **Increase the number of shards (resharding)**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Replayable stream | **Data Streams** |
| Load to S3 | **Firehose** |
| Shard in | **1 MB/s, 1000 records/s** |
| Shard out | **2 MB/s** |
| Retention | **up to 365 days (default 24 h)** |
| Consumers | **KCL / Lambda / EC2** |
| SQL on streams | **Kinesis Data Analytics** |
