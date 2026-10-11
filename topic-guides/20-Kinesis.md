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

> A stream must be replayable by multiple consumers in real time, each reading at its own position. Which service?
> → **Kinesis Data Streams**

> Data must be transformed and loaded into S3 or Redshift with no application code. Which service?
> → **Kinesis Data Firehose**

> Several consumers must each read the same records at full throughput, with low latency. Which feature?
> → **enhanced fan-out (dedicated 2 MB/s per consumer, pushed)**

> What decides which shard a record lands on in Kinesis Data Streams?
> → **the partition key (hashed to a shard)**

> Ordering in Kinesis Data Streams is guaranteed at what level?
> → **per shard**

> A team sets the same partition key for nearly all records and gets throttling. What is this called?
> → **a hot shard**

> What error indicates too few shards or a hot shard?
> → **`ProvisionedThroughputExceededException`**

> What is the per-shard write limit?
> → **1 MB/s or 1,000 records/s**

> What is the shared per-shard read limit for standard consumers?
> → **2 MB/s, max 5 `GetRecords` calls/s, shared by all consumers**

> What read throughput does each consumer get with enhanced fan-out?
> → **2 MB/s per consumer**

> What is the maximum Kinesis record size?
> → **1 MB**

> What is the default retention of a Kinesis data stream, and the maximum?
> → **24 hours default, up to 365 days**

> How do you reprocess data from days ago?
> → **extend retention (up to 365 days)**

> What are the two Kinesis Data Streams capacity modes?
> → **provisioned (you manage shards) and on-demand (auto-scales)**

> How do you scale a stream out or in?
> → **resharding — split to scale out, merge to scale in**

> Which operation divides a hot shard into two and doubles its capacity?
> → **`SplitShard`**

> Which operation combines two adjacent shards into one, decreasing shard count?
> → **`MergeShards` (scale-in / cost cut)**

> Which is a distractor API that does not exist?
> → **`AddShards`**

> Which real API resizes the whole stream uniformly?
> → **`UpdateShardCount`**

> How is strict ordering achieved when writing to Kinesis?
> → **same shard + `PutRecord` + `SequenceNumberForOrdering`**

> Is `PutRecords` used for strict ordering?
> → **No — it is batch ingestion, not strict ordering**

> A DynamoDB table's changes must be processed with Kinesis-style APIs. What do you add?
> → **the Kinesis Adapter over DynamoDB Streams**

> What does DynamoDB Streams record?
> → **table changes — insert, update, delete**

> What does the Kinesis Client Library (KCL) handle for you?
> → **shard discovery, load balancing across workers, and checkpointing**

> How many KCL workers process a single shard?
> → **exactly one worker, with exactly one record processor**

> Will adding more KCL instances beyond the shard count increase throughput?
> → **No**

> Where does KCL store checkpoints?
> → **a DynamoDB table created by the KCL**

> A KCL consumer suddenly gets throttled. What is an easily-missed cause?
> → **insufficient capacity on the DynamoDB checkpoint table**

> What does Kinesis Data Firehose do?
> → **automatically collects streaming data and delivers it to destinations**

> Name Firehose's destinations.
> → **S3, Redshift, OpenSearch, Splunk, generic HTTP endpoints, third-party SaaS**

> How does Firehose buffer before delivering?
> → **by size (e.g. 5 MB) or interval (60–900 s), whichever comes first**

> Why is Firehose "near" real time rather than real time?
> → **it buffers by size or interval before delivering**

> Can Firehose invoke a Lambda to transform records in flight?
> → **Yes**

> Can Firehose convert records to Parquet or ORC?
> → **Yes, via format conversion**

> Does Firehose store data or support replay?
> → **No — once delivered it keeps no copy**

> Which service runs SQL or Flink applications over a stream for real-time analytics?
> → **Managed Service for Apache Flink (formerly Kinesis Data Analytics)**

> What is the latency difference between Data Streams and Firehose?
> → **Data Streams ~200 ms (real time); Firehose 60 s+ (near real time)**

> Which service requires consumer code, Data Streams or Firehose?
> → **Data Streams**

> Which service supports ordering: Firehose or Data Streams?
> → **Data Streams (per partition key); Firehose does not**

> Which SQS feature (not present in Kinesis Data Streams) is limited to FIFO queues for ordering?
> → **SQS FIFO queues**

> A Kinesis consumer is falling behind. Which metric shows the lag?
> → **`GetRecords.IteratorAgeMilliseconds`**

> What is the Kinesis Agent for?
> → **collecting log files from a server/EC2 and continuously sending them to Kinesis**

> Where does the Kinesis Agent run?
> → **on your own server/EC2 instance**

> What is the common destination for the Kinesis Agent?
> → **Kinesis Data Firehose**

> Which built-in sink types does Kinesis Agent for Windows support?
> → **Kinesis Data Streams, Kinesis Data Firehose, CloudWatch, CloudWatch Logs, Local FileSystem**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Replayable stream | **Data Streams** |
| Load to S3/Redshift no code | **Firehose** |
| Shard write limit | **1 MB/s, 1,000 records/s** |
| Shard read (shared) | **2 MB/s, max 5 GetRecords/s** |
| Shard read (enhanced fan-out) | **2 MB/s per consumer** |
| Record size | **1 MB max** |
| Retention | **24 h default, up to 365 days** |
| Ordering | **per shard / partition key** |
| Shard selection | **partition key hashed** |
| Hot shard | **low-cardinality partition key** |
| Throttle error | **`ProvisionedThroughputExceededException`** |
| Capacity modes | **provisioned / on-demand** |
| Scale out | **`SplitShard`** |
| Scale in | **`MergeShards`** |
| Whole-stream resize | **`UpdateShardCount`** |
| Non-existent API | **`AddShards`** |
| Strict ordering write | **`PutRecord` + `SequenceNumberForOrdering`** |
| Batch (non-ordered) write | **`PutRecords`** |
| DynamoDB + Kinesis APIs | **Kinesis Adapter** |
| Consumer library | **KCL** |
| KCL per shard | **one worker, one record processor** |
| KCL checkpoints | **DynamoDB table** |
| KCL throttling | **check the DynamoDB checkpoint table** |
| Full-throughput consumers | **enhanced fan-out** |
| Consumer lag metric | **`GetRecords.IteratorAgeMilliseconds`** |
| Firehose buffering | **5 MB or 60–900 s, whichever first** |
| Firehose transform | **Lambda in flight** |
| Firehose format | **JSON → Parquet/ORC** |
| Firehose storage/replay | **none** |
| Real-time SQL on stream | **Managed Service for Apache Flink** |
| Data Streams latency | **~200 ms** |
| Firehose latency | **near real time (60 s+)** |
| Consumer code required | **Data Streams** |
| Kinesis Agent | **collect logs/files from servers → Kinesis** |
| Agent runs on | **your server / EC2** |
| Agent common destination | **Firehose** |
| Agent Windows sinks | **Data Streams, Firehose, CloudWatch, CloudWatch Logs, Local FileSystem** |
