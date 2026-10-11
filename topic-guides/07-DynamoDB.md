## Amazon DynamoDB

Amazon DynamoDB is serverless You **do not provision or manage EC2 instances**. AWS automatically handles everything.

**Why it exists**

A relational database scales by getting a bigger machine, and its query planner can make any query — which means any query can also be slow. DynamoDB makes the opposite trade: it will only answer questions you’ve designed for, but it answers them in single-digit milliseconds at any scale, on a machine size you never think about.

The whole design follows from **partitioning**. Data is spread across partitions by hashing the partition key. Every operation must therefore identify a partition, which is why you can `Query` by partition key cheaply and why scanning the whole table is expensive.

**This one fact explains almost every DynamoDB exam answer**: capacity units, hot partitions, why LSIs can’t be added later, why GSIs are eventually consistent, why filters don’t save money.

**Structure**

```plain text
Table
 └── Item                    (max 400 KB)
      ├── Partition key      (required — determines which partition)
      ├── Sort key           (optional — orders items within the partition)
      └── Attributes         (schemaless beyond the keys)
```

Partition key alone = a simple key. Partition key + sort key = a composite key, and items sharing a partition key are stored together, sorted.

**Capacity units — derive them, don’t memorise them**

Reads and writes are metered in fixed-size blocks. Only four facts are needed:

| Operation | Cost |
| --- | --- |
| **1 WCU** | one write of up to **1 KB** per second |
| **1 RCU** | one **strongly consistent** read of up to **4 KB** per second |
| **1 RCU** | **two eventually consistent** reads of up to 4 KB per second |
| **Transactional** | **2×** the normal cost, read or write |

**Always round item size UP to the next whole block.** A 6 KB item is two 4 KB read blocks, not one and a half.

**Why reads are cheaper than writes per byte:** a write must be replicated to multiple copies before acknowledgement; a read can be served from one. And **why eventual consistency halves the cost:** DynamoDB replicates to three copies, so an eventually consistent read can be served by any one of them, while a strongly consistent read must consult the leader.

Worked example — reads:

```plain text
100 reads/second, item = 6 KB, strongly consistent

6 KB → round up → 8 KB → 2 blocks of 4 KB
2 RCU per read × 100 reads = 200 RCU
```

Same, eventually consistent:

```plain text
2 blocks ÷ 2 = 1 RCU per read
1 × 100 = 100 RCU
```

Worked example — writes:

```plain text
50 writes/second, item = 3 KB

3 KB → 3 blocks of 1 KB
3 WCU per write × 50 = 150 WCU
```

**Query vs Scan — and why filters don’t help**

**Query** requires the partition key with `=`, optionally narrows with a sort key condition (`begins_with`, `between`, `<`, `>`), and reads only the matching partition. Efficient.

**Scan** reads **every item in the table**. It cannot use partitioning at all, so it consumes capacity proportional to the table size, not the result size.

**`FilterExpression` is the trap, and the mechanism explains why:**

```plain text
Query / Scan
     ↓
Read items from disk    ← CAPACITY IS CONSUMED HERE
     ↓
Apply FilterExpression
     ↓
Return what survived
```

The filter runs **after** the read, in the DynamoDB service, before the response is sent. It reduces what you *see*, never what you *pay for*. Any answer claiming a filter reduces consumed capacity is wrong.

**`ProjectionExpression`** chooses which attributes come back. It reduces network transfer but **not** consumed capacity, because capacity is billed on the full item size read from disk.

If you need to query by an attribute that isn’t the partition key, the correct answer is never “Scan with a filter” — it’s a **GSI**.

**Parallel scan** (`Segment` + `TotalSegments`) splits a Scan across workers when you genuinely must read the whole table faster. It uses *more* capacity, not less.

`ScanIndexForward`Controls the **sort-key order** when using `Query`.

- `true` (default) → **ascending** → old → new
- `false` → **descending** → new → old

**Example:** To get the **latest item first**:

`ScanIndexForward: false` + `Limit: 1`

**Pagination — the 1 MB ceiling**

Any single `Query` or `Scan` response is capped at **1 MB**. When more data exists, the response includes **`LastEvaluatedKey`**, which you pass back as **`ExclusiveStartKey`**.

```plain text
Request 1 → up to 1 MB of items + LastEvaluatedKey
                                        ↓
Request 2 (ExclusiveStartKey = that key) → next page
                                        ↓
No LastEvaluatedKey in the response → you have everything
```

The subtlety worth understanding: the 1 MB cap applies **before** `FilterExpression`. So a heavily filtered query can legitimately return **zero items while still including a `LastEvaluatedKey`**. That is not an error and not an empty result — you must keep paging. Developers who assume “no items means done” silently lose data, and the exam knows it.

`Limit` caps the item *count*; the 1 MB cap still applies on top of it.

## Mental Model

| Option | What it controls |
| --- | --- |
| `--max-items` | ✅ Total items returned by the CLI |
| `--page-size` | Number of items requested per API call |
| `--limit` | Number of items DynamoDB evaluates per Scan request |
| `--starting-token` | Resume pagination from a previous response |

### AWS Exam Tip

If the question says:

> **"paginate the results"** and **"minimize the number of items returned in the CLI's output"**

the keyword is **CLI output**, so the correct answer is:

> ✅ **`--max-items`**

**Secondary indexes — why LSI and GSI differ**

Both let you query by something other than the table’s partition key. The difference comes from *where the index physically lives*.

An **LSI** shares the table’s partition key, so its data lives in the same partitions as the table. Because the partitioning is identical, the index can be read with strong consistency — but it also means the index must exist when the partitions are created, which is why an **LSI can only be created at table creation time and can never be deleted**.

A **GSI** uses a different partition key, so it is a **separate, independently partitioned structure** that DynamoDB keeps in sync asynchronously. That asynchronous replication is exactly why **GSI reads are only ever eventually consistent** — and why a GSI can be added or removed at any time.

|    | **LSI** (Local) | **GSI** (Global) |
| --- | --- | --- |
| Created | **Table creation ONLY** | **Any time** |
| Deleted | ❌ Never | ✅ Yes |
| Partition key | **Same as the table** | **Any attribute** |
| Sort key | Different, required | Optional |
| Max per table | **5** | **20** (soft limit) |
| Throughput | Shares the table’s | **Its own RCU/WCU** |
| Strongly consistent reads | ✅ Yes | ❌ **No — eventual only** |
| Size limit | 10 GB per partition key value | None |

**The GSI throttling chain** — a favourite troubleshooting question. A GSI has its own capacity. **Independent amount of RCU and WCU**

Every write to the base table also writes to the GSI. If the GSI is out of capacity, DynamoDB cannot complete the replicated write, so **writes to the base table start failing too**. The symptom is “base table writes are throttled but the table has plenty of capacity”; the cause is the GSI.

**Projections** — an index only contains the attributes you project into it: `KEYS_ONLY`, `INCLUDE` (named extras), or `ALL`. Reading an attribute that isn’t projected forces a fetch back to the base table, costing extra capacity and latency. Under-projecting to save storage and then paying for fetches is a real trap.

**Writes under concurrency**

Three mechanisms, for three different requirements.

**Conditional writes** — attach a `ConditionExpression` to `PutItem`, `UpdateItem`, or `DeleteItem`. The write happens only if the condition is true, evaluated atomically on the server. Failure raises **`ConditionalCheckFailedException`**.

Functions available: `attribute_not_exists`, `attribute_exists`, `attribute_type`, `begins_with`, `contains`, `size`.

Insert-only-if-absent:

```plain text
ConditionExpression: attribute_not_exists(orderId)
```

**Optimistic locking** — the standard fix for *lost updates*. Two writers read the same item, both modify it, and the second overwrites the first’s change silently. A version attribute makes the overwrite detectable:

```plain text
Read item → version = 4
     ↓
UpdateItem
   SET data = :new, version = :5
   ConditionExpression: version = :4
     ↓
Someone else already bumped it to 5?
     ↓
ConditionalCheckFailedException → re-read, re-apply, retry
```

Note what is *not* the answer: increasing capacity does nothing for lost updates, because the problem is logical, not throughput.

**Atomic counters** — `SET viewCount = viewCount + :inc` is always applied to the current server-side value, so concurrent increments never lose each other.

But it is **not idempotent**: a retried request increments twice. So it is right for view counts and likes, and wrong for anything where an accidental double-application matters (money, inventory). For those, use a conditional write keyed on a request ID.

**Batch and transaction APIs**

| API | Limits | Behaviour |
| --- | --- | --- |
| **`BatchGetItem`** | 100 items / 16 MB | Parallel reads, possibly across tables. Returns `UnprocessedKeys`. |
| **`BatchWriteItem`** | 25 items / 16 MB | `PutItem` + `DeleteItem` only. **No `UpdateItem`. No conditions.** Returns `UnprocessedItems`. |
| **`TransactWriteItems`** | 100 items / 4 MB | All-or-nothing ACID. Supports `ConditionCheck`. **2× WCU.** |
| **`TransactGetItems`** | 100 items / 4 MB | Consistent snapshot. **2× RCU.** |

**Batch operations are not atomic.** They are a convenience for issuing many independent requests in one call, so individual items can fail while others succeed. Those failures come back in **`UnprocessedItems`** / **`UnprocessedKeys`**, and **it is your job to retry them with exponential backoff**. Code that ignores the field appears to work and silently loses data — a well-worn exam scenario.

Transactions *are* atomic, which is why they cost double: DynamoDB must do a two-phase commit across partitions.

**TTL**

Add a numeric attribute holding a **Unix epoch timestamp in seconds**, then enable TTL on that attribute name. DynamoDB deletes items whose timestamp has passed.

Why it’s useful and how it behaves:

- **Free** — TTL deletions consume **no WCU**. This is the cheapest way to expire data.
- **Not instantaneous** — deletion happens **within roughly 48 hours** of expiry. Expired items can still be returned by reads in the meantime, so filter them in your query if correctness depends on it.
- TTL deletions appear in **DynamoDB Streams as `REMOVE` records**, which is how you archive expiring data to S3 with a Lambda.
- Must be **seconds**, not milliseconds. Milliseconds is both a common real-world bug and a plausible-looking wrong answer.

**Capacity modes and throttling**

**Provisioned** — you set RCU/WCU. Cheapest for steady, predictable traffic. **Auto scaling** adjusts within a min/max band toward a target utilisation percentage. **Reserved capacity** gives a 1- or 3-year discount.

**On-demand** — pay per request, scales instantly, no planning. Right for unpredictable or spiky traffic and for new applications where you don’t know the pattern yet. Roughly 5–7× the cost of well-tuned provisioned capacity, which is the trade.

You can switch modes once every 24 hours.

**Diagnosing `ProvisionedThroughputExceededException`**

| Cause | How you recognise it | Fix |
| --- | --- | --- |
| Total capacity genuinely too low | Consumed ≈ provisioned | Raise capacity, enable auto scaling, or switch to on-demand |
| **Hot partition** | **Throttled while average utilisation is low** | Fix the partition key — higher cardinality, add a suffix |
| GSI out of capacity | Base table writes throttled for no apparent reason | Raise the **GSI’s** capacity |
| Read-heavy burst | Reads dominate, same items repeatedly | **DAX** or ElastiCache |
| Client sending faster than the table can absorb | Bursty client | **Exponential backoff with jitter** (the AWS SDKs do this automatically) |

The “throttled while average utilisation is low” signature is the key insight: capacity is provisioned per *table* but consumed per *partition*. If all your traffic hits one partition key, that partition’s share of the capacity is the real ceiling.

**Burst capacity** is the other reason a table sometimes absorbs more than it was provisioned for. DynamoDB retains **up to 5 minutes (300 seconds) of unused read and write capacity** and spends it on short spikes. This is why a table can survive a brief burst above its provisioned rate and then start throttling when the saved-up capacity runs out — the failure looks sudden and unexplained, but the burst reserve was simply exhausted. It is a best-effort reserve, so you cannot design around it.

**Adaptive capacity** shifts capacity toward busy partitions automatically and can isolate frequently accessed items. It reduces hot-key throttling but does not eliminate it, so “rely on adaptive capacity” is not a fix for a badly chosen key.

**A good partition key** has high cardinality and even access distribution. `userId` is usually good; `country` or `status` is usually terrible.

**Everything else, one line each**

| Feature | What to know |
| --- | --- |
| **Max item size** | **400 KB.** Store large objects in S3 and keep the S3 key in the item |
| **PartiQL** | SQL-like syntax over DynamoDB (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) |
| **Global tables** | Multi-Region, active-active replication; requires Streams |
| **Encryption at rest** | Always on; AWS owned key by default, or AWS managed / customer managed |
| **PITR** | Point-in-time recovery — restore to any second in the last **35 days** |
| **On-demand backup** | Full backup, no performance impact |
| **`ReturnConsumedCapacity`** | Ask the API what a request cost — the practical way to debug throttling |
| **Reserved words** | Use `ExpressionAttributeNames` (`#name`) when an attribute name collides with a reserved word |

### DynamoDB Write/Read Response Parameters

### `ReturnConsumedCapacity`

Tells DynamoDB to return **how much capacity the operation used**.

| Value | Returns |
| --- | --- |
| `NONE` | Nothing |
| `TOTAL` | Total capacity consumed |
| `INDEXES` | Total capacity **+ table/index breakdown** |

**Exam memory:**

```plain text
Need capacity → ReturnConsumedCapacity
Table + indexes breakdown → INDEXES
Only total → TOTAL
```

### `ReturnValues`

Tells DynamoDB **which item values to return after an operation**.

Common values:

| Value | Meaning |
| --- | --- |
| `NONE` | Return nothing |
| `ALL_OLD` | Return the entire item **before** the write |
| `UPDATED_OLD` | Return attributes that were changed, **before** the write |
| `ALL_NEW` | Return the entire item **after** the write |
| `UPDATED_NEW` | Return attributes that were changed, **after** the write |

**Recall check — DynamoDB**

- **NoSQL, low latency, high throughput, serverless database** → DynamoDB
- **WCU block = 1 KB. RCU block = 4 KB.**
- **Eventually consistent = half the RCUs; transactional = double**
- **Always round item size UP**
- **Efficient lookup** → Query, never Scan
- **`FilterExpression` does NOT reduce consumed capacity**
- **`ProjectionExpression` does NOT reduce consumed capacity**
- **Need to query by a non-key attribute** → GSI
- **Scan too slow** → parallel scan (`Segment` / `TotalSegments`)
- **Fewer results than expected + `LastEvaluatedKey`** → paginate with `ExclusiveStartKey`
- **Empty page but `LastEvaluatedKey` present** → keep paging, not a bug
- **Production table needs a new query key** → GSI (LSI is impossible after creation)
- **Need strongly consistent reads on an index** → LSI
- **Same partition key, different sort key** → LSI
- **Different partition key** → GSI
- **Base table writes throttled for no clear reason** → check GSI capacity
- **GSI reads** → always eventually consistent
- **Attribute missing from index results** → not projected
- **Prevent lost updates** → `ConditionExpression` on a version attribute (optimistic locking)
- **Insert only if not present** → `attribute_not_exists`
- **Concurrent counter increments** → atomic counter (`SET x = x + :v`)
- **Exactly-once increment** → conditional write on a request ID, not an atomic counter
- **Do NOT answer “increase capacity” for a lost-update problem**
- **Items silently missing after a batch write** → unhandled `UnprocessedItems`
- **All-or-nothing across items or tables** → `TransactWriteItems`
- **`BatchWriteItem` cannot update** → Put and Delete only, no conditions
- **Cheapest bulk write** → `BatchWriteItem` (transactions cost 2×)
- **Auto-delete expired data, free** → TTL (**epoch seconds**, up to 48 h delay)
- **Archive expiring items** → TTL + Streams + Lambda → S3
- **Unpredictable traffic, no capacity planning** → on-demand
- **Steady predictable traffic, lowest cost** → provisioned + auto scaling
- **Throttled while average utilisation is low** → hot partition / bad partition key
- **Good partition key** → high cardinality, evenly distributed
- **Retry throttled requests** → exponential backoff with jitter
- **Read-heavy, microsecond latency** → DAX
- **Item over 400 KB** → object in S3, pointer in DynamoDB
- **SQL syntax on DynamoDB** → PartiQL
- **Multi-Region active-active** → global tables
- **Restore to any second in the last 35 days** → PITR
- **Debug what a request cost**  capacity usage → `ReturnConsumedCapacity`
- `ReturnValues ->` Tells DynamoDB **which item values to return after an operation**.

---

## DynamoDB Export to Amazon S3

**Mental model**

> **DynamoDB Export to S3 = move a snapshot or changes from DynamoDB into S3 without consuming DynamoDB read capacity.**

It is a **fully managed DynamoDB feature**. You don't need to write code to scan the table and copy every item.

```plain text
DynamoDB Table
      ↓
Export to S3
      ↓
Amazon S3
      ↓
Athena / Glue / EMR / download / other processing
```

**What can you export?**

**Full export**

Exports a **snapshot of the entire table** at a chosen point in time within the PITR window.

```plain text
Table at 10:00
   ↓
Full Export
   ↓
Entire table → S3
```

Useful for:

- Backup-like copies in S3
- Analytics
- Migration
- Downloading/processing data outside DynamoDB

**Incremental export**

Exports **changes during a specific period**.

```plain text
10:00 ───────── 11:00
        ↓
Only changes → S3
```

Useful when you don't want to export the entire table again.

**Important requirements**

**PITR must be enabled.**

DynamoDB uses the PITR window as the source for the export.

**Export is asynchronous.**

You start the export and DynamoDB performs it in the background.

**It does NOT consume RCUs.**

This is important for exams:

> DynamoDB Export → S3 **does not consume read capacity units and does not impact table performance/availability.**

**Output format**

You can export as:

- **DynamoDB JSON**
- **Amazon Ion**

The exported files and manifest files are placed in your S3 bucket.

**Where can the S3 bucket be?**

The destination bucket can be:

- In the **same AWS account**
- In a **different AWS account**
- In a **different AWS Region**

You need the appropriate S3/IAM permissions.

#### Encryption

The export can use:

- **SSE-S3**
- **SSE-KMS**

If using a KMS key, the KMS key must be in the **same Region as the destination S3 bucket**.

#### Export vs On-Demand Backup

This is a common exam trap.

|    | **Export to S3** | **On-demand backup** |
| --- | --- | --- |
| Destination | **S3** | DynamoDB-managed backup |
| Purpose | Get data into S3 | Backup/restore DynamoDB |
| Can download S3 data? | **Yes** | No |
| Uses RCUs? | **No** | No |
| PITR required? | **Yes** | No |

So if the question says:

> **"Get DynamoDB data into S3"**

think:

> **Export to S3**

If it says:

> **"Backup DynamoDB and restore it later"**

think:

> **On-demand backup / PITR**.

## Other ways to get DynamoDB data into S3

These can appear in older DVA questions.

**AWS Glue**

Managed **ETL/data integration service**. It can use the DynamoDB export connector to export data to S3 and then process it.

**Amazon EMR + Hive**

**Big-data processing platform**. Hive can work with DynamoDB data and S3.

**AWS Data Pipeline**

Older **data movement/scheduling service** that can move DynamoDB data to S3.

For a modern question specifically asking for DynamoDB → S3, prefer the native **DynamoDB Export to S3** feature.

## Export vs Streams

Don't confuse these.

**Export to S3**

> Bulk/snapshot or periodic changes → S3

**DynamoDB Streams**

> Near-real-time item changes → Lambda/Kinesis/etc.

```plain text
Export:
DynamoDB ──────→ S3
       bulk / incremental

Streams:
DynamoDB ──────→ Stream ──────→ Lambda
                    near-real-time
```

If you need **near-real-time change capture**, Streams is generally the appropriate mechanism. If you don't need near-real-time updates, incremental export can export changes between two points in time.

**Exam memory**

> **DynamoDB → S3 = Export to S3**
> **Whole table = Full export**
> **Only changes = Incremental export**
> **PITR required**
> **No RCU consumed**
> **Asynchronous**
> **DynamoDB JSON or Amazon Ion**
> **Need analytics after export → S3 + Athena/Glue/EMR**
> **Near-real-time changes → DynamoDB Streams**
> **Backup/restore DynamoDB → On-demand backup/PITR**
> **On-demand backup ≠ S3 export**

The biggest exam trap is simply:

**Backup is for DynamoDB recovery. Export is for getting DynamoDB data into S3.**

## DynamoDB Streams

**What it is:** A record of **changes made to a DynamoDB table**.

When an item is **created, updated, or deleted**, DynamoDB Streams records that change for a limited time.

**Common use:** Trigger a Lambda function when the table changes.

```plain text
DynamoDB table
      ↓  item created / updated / deleted
DynamoDB Stream    (ordered, per partition key, 24-hour retention)
      ↓
Lambda (event source mapping)  or  KCL + Kinesis Adapter
      ↓
Process the change
```

**What to know**

- **Retention: 24 hours.**
- Event names: **`INSERT`, `MODIFY`, `REMOVE`**.
- **View types** (`StreamViewType`) — choose based on what your consumer needs:
  - `KEYS_ONLY` — just the keys
  - `NEW_IMAGE` — the item after the change
  - `OLD_IMAGE` — the item before the change
  - `NEW_AND_OLD_IMAGES` — both, which is what you need to compute *what changed*
- Ordering is preserved **per partition key**, which is why stream consumers are one-invocation-per-shard by default (see the Lambda event source mapping section).
- **Kinesis Data Streams for DynamoDB** is a separate option when 24 hours isn’t enough: up to 365 days of retention and more consumers.

**Common uses:** notify on data change, maintain an OpenSearch index, replicate to another system, audit trail, trigger workflows, archive TTL-expired items.

**Recall check**

- **Item created/updated/deleted → run code** → DynamoDB Streams + Lambda
- **Need the value before AND after** → `NEW_AND_OLD_IMAGES`
- **Streams retention** → 24 hours
- **Need longer retention or replay** → Kinesis Data Streams for DynamoDB

---

## DynamoDB Accelerator (DAX)

**Why it exists**

DynamoDB is fast (single-digit milliseconds) but not *memory* fast. For read-heavy workloads hitting the same items repeatedly, an in-memory cache gets you to microseconds. DAX’s specific value is that it is **API-compatible with the DynamoDB SDK**, so adopting it is a client configuration change rather than a rewrite — which is precisely why it beats ElastiCache in DynamoDB caching questions.

**How it works**

- Runs as a **cluster inside your VPC**, in front of the table.
- Two caches: an **item cache** (for `GetItem`/`BatchGetItem`) and a **query cache** (for `Query`/`Scan`).
- **Write-through**: writes go to DynamoDB and update the cache together, so the cache doesn’t serve stale data it just wrote.
- Default TTL 5 minutes.
- **Strongly consistent reads bypass DAX entirely** — they have to, because a cache cannot guarantee it holds the leader’s latest value. So DAX gives you nothing on strongly consistent reads.

| Cache | Use it for |
| --- | --- |
| **DAX** | DynamoDB, microsecond reads, minimal code change |
| **ElastiCache Redis** | Anything else — sessions, leaderboards, arbitrary computed results, pub/sub |
| **ElastiCache Memcached** | Simple, large, throwaway key-value cache |

**Recall check**

- **Microsecond DynamoDB reads with almost no code change** → DAX
- **DAX does not help strongly consistent reads**
- **Cache for something other than DynamoDB** → ElastiCache
- **Cache user sessions** → ElastiCache Redis, not DAX
- **Reduce database reads generally** → ElastiCache / DAX

---

# Question patterns

> A 6 KB item is read with strongly consistent reads. How many RCU does one read consume?
> → **2 RCU (6 KB rounds up to 8 KB = two 4 KB blocks)**

> A 3 KB item is written 50 times per second. How many WCU are required?
> → **150 WCU (3 KB = three 1 KB blocks × 50 writes)**

> How many RCU does a 6 KB eventually consistent read consume?
> → **1 RCU (6 KB rounds up to 8 KB = two blocks, then halved for eventual consistency)**

> How does a strongly consistent read change the cost versus an eventually consistent read?
> → **It doubles it — strongly consistent reads cost the full 4 KB-block RCU count**

> What is the maximum item size in DynamoDB?
> → **400 KB (store larger objects in S3 and keep the S3 key in the item)**

> How much data does one WCU cover?
> → **One write of up to 1 KB per second**

> Why is a read cheaper per byte than a write?
> → **A write must be replicated to multiple copies before acknowledgement; a read can be served from one**

> A workload is read-heavy and hits the same items repeatedly, needing microsecond latency with minimal code change. What goes in front?
> → **DAX (strongly consistent reads bypass it entirely)**

> The application needs a new query pattern on a non-key attribute without rewriting the table. Which feature?
> → **Global Secondary Index (GSI)**

> An index must support strongly consistent reads. Which index type?
> → **LSI (a GSI is only ever eventually consistent)**

> An index must query using a different partition key. Which type?
> → **GSI**

> A production table needs a new index added after table creation. Which type is possible?
> → **GSI (an LSI can only be created at table creation and can never be deleted)**

> How many LSIs and GSIs can a table have?
> → **5 LSIs; 20 GSIs (soft limit)**

> Base table writes are throttled even though the table shows spare capacity. What is the likely cause?
> → **The GSI's own capacity is exhausted, which back-pressures the base-table write**

> A GSI query returns fewer attributes than expected. Why?
> → **Those attributes were not projected into the index (KEYS_ONLY/INCLUDE), forcing a fetch to the base table**

> Query results are fewer than expected but include LastEvaluatedKey. What must you do?
> → **Keep paginating — pass LastEvaluatedKey back as ExclusiveStartKey**

> A heavily filtered Query returns zero items but still includes a LastEvaluatedKey. What does that mean?
> → **Not an error — the 1 MB cap applied before the filter; keep paging or you silently lose data**

> What is the maximum size of a single Query or Scan response?
> → **1 MB (before FilterExpression is applied)**

> What does Limit cap in a Scan?
> → **The number of items DynamoDB evaluates; the 1 MB cap still applies on top**

> A CLI scan must minimize the number of items returned in the CLI's output while paginating. Which parameter?
> → **--max-items**

> Which CLI parameter sets how many items are requested per API call?
> → **--page-size**

> Which CLI parameter controls how many items DynamoDB evaluates per Scan request?
> → **--limit**

> Which CLI parameter resumes pagination from a previous response?
> → **--starting-token**

> A filter is added to a scan to reduce results. What happens to consumed capacity?
> → **Nothing — FilterExpression runs after the read, so capacity is unchanged**

> Does ProjectionExpression reduce consumed capacity?
> → **No — it reduces network transfer only; capacity is billed on the full item size read from disk**

> A whole table must be read faster across workers. Which feature, and what does it cost?
> → **Parallel Scan (Segment + TotalSegments) — it uses more capacity, not less**

> You need the most recent item first from a Query. How?
> → **ScanIndexForward: false with Limit: 1 (default ScanIndexForward is true = ascending)**

> What does ScanIndexForward control?
> → **Sort-key order for a Query (true = ascending/old→new, false = descending/new→old)**

> An update must apply only if the item hasn't changed since it was read. Which mechanism?
> → **Conditional writes with optimistic locking (version attribute + ConditionExpression)**

> Which error is raised when a ConditionExpression is not satisfied?
> → **ConditionalCheckFailedException**

> Which condition expression inserts an item only if it does not already exist?
> → **attribute_not_exists(<key>)**

> A page-view counter must never lose concurrent increments. Which technique?
> → **An atomic counter (SET x = x + :inc) — applied to the current server-side value**

> An exactly-once increment is required (e.g., payments). What must you use instead of an atomic counter?
> → **A conditional write keyed on a request ID — atomic counters are not idempotent**

> Items are silently missing after a batch write. What did the code fail to handle?
> → **UnprocessedItems (retry them with exponential backoff)**

> Which DynamoDB API provides all-or-nothing ACID across items or tables?
> → **TransactWriteItems (2× WCU; supports ConditionCheck)**

> Can BatchWriteItem perform updates or use conditions?
> → **No — Put and Delete only, no Update, no conditions**

> What are the item limits for BatchGetItem and BatchWriteItem?
> → **BatchGetItem 100 items / 16 MB; BatchWriteItem 25 items / 16 MB**

> What are the limits for TransactWriteItems and TransactGetItems?
> → **100 items / 4 MB each (2× WCU / 2× RCU; transactions need a two-phase commit)**

> Data must auto-expire at no cost. Which feature, and what timestamp format?
> → **TTL with a Unix epoch timestamp in seconds (not milliseconds; no WCU; up to 48 h delay)**

> Expired or deleted items must be archived to S3. Which combination?
> → **TTL + DynamoDB Streams (REMOVE records) + Lambda → S3**

> Traffic is unpredictable or spiky with no capacity planning. Which capacity mode?
> → **On-demand**

> Traffic is steady and predictable and cost must be minimized. Which mode?
> → **Provisioned with auto scaling**

> How often can you switch between provisioned and on-demand capacity modes?
> → **Once every 24 hours**

> Reads are throttled while average table utilisation is low. What is the cause?
> → **A hot partition from a poorly chosen partition key**

> A table absorbs a brief spike above its provisioned rate and then suddenly throttles. Why?
> → **Burst capacity — up to 300 seconds of unused capacity is saved and spent on the spike**

> What makes a good partition key?
> → **High cardinality with evenly distributed access (userId is good; country/status is bad)**

> How should throttled requests be retried?
> → **Exponential backoff with jitter (the AWS SDKs do this automatically)**

> How do you find out how much capacity a specific request consumed?
> → **ReturnConsumedCapacity (INDEXES gives table + index breakdown; TOTAL gives the total only)**

> Which stream view type gives both the value before and after a change?
> → **NEW_AND_OLD_IMAGES**

> How long are DynamoDB Streams retained, and what are the event names?
> → **24 hours; INSERT, MODIFY, REMOVE**

> You need longer stream retention (up to 365 days) or replay. Which option?
> → **Kinesis Data Streams for DynamoDB**

> DynamoDB data must be exported to S3 without consuming read capacity. Which feature?
> → **Native DynamoDB Export to S3 (requires PITR; asynchronous; consumes no RCU)**

> What are the DynamoDB export output formats?
> → **DynamoDB JSON or Amazon Ion**

> Distinguish export to S3 from on-demand backup.
> → **Export puts downloadable data in S3 (PITR required); on-demand backup is for DynamoDB backup/restore and needs no PITR**

> Data must be in S3 purely to run analytics. What do you use after export?
> → **Athena / Glue / EMR on the exported S3 data**

> Near-real-time change capture across the table is required. Which mechanism?
> → **DynamoDB Streams (export is bulk/snapshot, not near-real-time)**

> A developer needs SQL-like syntax against the table. Which feature?
> → **PartiQL**

> Items must be available in multiple Regions with active-active writes. Which feature?
> → **Global Tables (requires Streams)**

> Restore to any second in the last 35 days is required. Which feature?
> → **Point-in-time recovery (PITR)**

> Why choose DAX over ElastiCache for DynamoDB caching?
> → **DAX is API-compatible with the DynamoDB SDK, so it needs almost no code change**

# Pocket card

| Keyword | Answer |
| --- | --- |
| NoSQL serverless low-latency DB | **DynamoDB** |
| 1 WCU | **1 KB write/s** |
| 1 RCU | **4 KB strongly consistent read/s (or 2 eventually consistent)** |
| Eventual consistency | **half the RCUs** |
| Transactional read/write | **2× the cost** |
| Item size rounding | **always round UP to the next block** |
| Efficient lookup | **Query, never Scan** |
| FilterExpression | **does NOT reduce consumed capacity** |
| ProjectionExpression | **does NOT reduce consumed capacity** |
| Query by non-key attribute | **GSI** |
| Scan too slow | **parallel scan (Segment / TotalSegments)** |
| Fewer results + LastEvaluatedKey | **paginate with ExclusiveStartKey** |
| Empty page but LastEvaluatedKey present | **keep paging — not a bug** |
| New query key on a production table | **GSI (LSI impossible after creation)** |
| Strongly consistent reads on an index | **LSI** |
| Same partition key, different sort key | **LSI** |
| Different partition key | **GSI** |
| LSI limits | **5 per table, created at table creation only, never deleted** |
| GSI limits | **20 (soft limit), own RCU/WCU, add/remove any time** |
| Base table writes throttled for no clear reason | **check GSI capacity** |
| GSI reads | **always eventually consistent** |
| Attribute missing from index results | **not projected** |
| Prevent lost updates | **ConditionExpression on a version attribute (optimistic locking)** |
| Insert only if not present | **attribute_not_exists** |
| Concurrent counter increments | **atomic counter (SET x = x + :v)** |
| Exactly-once increment | **conditional write on a request ID** |
| Lost-update problem | **do NOT answer "increase capacity"** |
| Items silently missing after a batch write | **unhandled UnprocessedItems** |
| All-or-nothing across items/tables | **TransactWriteItems** |
| BatchWriteItem | **Put/Delete only, no Update, no conditions** |
| Cheapest bulk write | **BatchWriteItem (transactions cost 2×)** |
| Auto-delete expired data, free | **TTL (epoch seconds, up to 48 h delay)** |
| Archive expiring items | **TTL + Streams + Lambda → S3** |
| Unpredictable traffic | **on-demand** |
| Steady predictable traffic, lowest cost | **provisioned + auto scaling** |
| Throttled while average utilisation low | **hot partition / bad partition key** |
| Good partition key | **high cardinality, evenly distributed** |
| Retry throttled requests | **exponential backoff with jitter** |
| Read-heavy, microsecond latency | **DAX** |
| Item over 400 KB | **object in S3, pointer in DynamoDB** |
| SQL syntax on DynamoDB | **PartiQL** |
| Multi-Region active-active | **global tables (requires Streams)** |
| Restore to any second in the last 35 days | **PITR** |
| Debug what a request cost | **ReturnConsumedCapacity** |
| ReturnConsumedCapacity INDEXES | **table + index breakdown** |
| ReturnValues | **which item values to return after an operation** |
| 1 MB response cap | **LastEvaluatedKey → ExclusiveStartKey** |
| Query sort-key order | **ScanIndexForward (true = asc, false = desc)** |
| CLI min items in output | **--max-items** |
| CLI items per call | **--page-size** |
| CLI items evaluated per scan | **--limit** |
| CLI resume pagination | **--starting-token** |
| Encryption at rest | **always on (AWS owned/managed, or customer managed key)** |
| On-demand backup | **full backup, no performance impact** |
| Reserved words | **use ExpressionAttributeNames (#name)** |
| Export to S3 requires | **PITR enabled** |
| Export to S3 consumes RCU? | **No, and no impact on table performance** |
| Export output format | **DynamoDB JSON or Amazon Ion** |
| Export destination bucket | **same/different account and Region allowed** |
| Export encryption | **SSE-S3 or SSE-KMS (KMS key in the destination bucket's Region)** |
| DynamoDB → S3 bulk/snapshot | **Export to S3 (full or incremental)** |
| Whole table export | **full export** |
| Only changes export | **incremental export** |
| Near-real-time changes | **DynamoDB Streams** |
| Backup/restore DynamoDB | **on-demand backup / PITR** |
| Analytics after export | **S3 + Athena/Glue/EMR** |
| On-demand backup vs export | **backup = recovery; export = data into S3** |
| Streams retention | **24 hours** |
| Stream event names | **INSERT, MODIFY, REMOVE** |
| Value before AND after | **NEW_AND_OLD_IMAGES** |
| Stream ordering | **per partition key** |
| Longer stream retention / replay | **Kinesis Data Streams for DynamoDB (up to 365 days)** |
| DAX strongly consistent reads | **bypass DAX entirely** |
| DAX code change | **minimal — DynamoDB SDK-compatible** |
| DAX default TTL | **5 minutes** |
| DAX cache types | **item cache (GetItem/BatchGetItem) + query cache (Query/Scan)** |
| DAX writes | **write-through** |
| Cache user sessions | **ElastiCache Redis, not DAX** |
| Cache anything other than DynamoDB | **ElastiCache** |
| DynamoDB partition key hashing | **every operation must identify a partition** |
| Composite key | **partition key + sort key** |
