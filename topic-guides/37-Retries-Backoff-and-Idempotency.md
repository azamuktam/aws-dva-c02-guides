*PART 6 — OBSERVABILITY: KNOWING WHAT WENT WRONG*

**Domain 4 is 18%, but observability questions also hide inside the other three domains** — most “why is this broken” questions are really asking which tool would tell you.

Four tools, four different questions:

| Tool | Answers |
| --- | --- |
| **CloudWatch Metrics** | “Is the number too high or too low?” |
| **CloudWatch Logs** | “What did the application say?” |
| **X-Ray** | “Which step in the chain was slow or failed?” |
| **CloudTrail** | “Who made this change to AWS itself?” |

---

## Retries, Backoff, and Idempotency

This is the one genuinely cross-cutting pattern in the exam. It shows up in Domain 1 (writing the client), Domain 3 (deployment retries), and Domain 4 (diagnosing throttling), and it is easy to miss because no single service “owns” it.

**Why retries are unavoidable**

Every distributed call can fail transiently: a throttle, a brief network partition, a leader election in a database, a load balancer draining a target. None of these mean the request was wrong — only that it should be tried again. So AWS APIs are designed on the assumption that clients retry, and a client that treats a `429` or a `503` as fatal is broken.

**Why naive retries make things worse**

If a service is throttling because it is overloaded, and every client immediately retries, the load *increases*. Worse, all the clients that failed at the same moment retry at the same moment, producing a synchronised wave — a **thundering herd**. The queue never drains.

```plain text
Plain retry:      fail → retry immediately → fail → retry → ...
                  (adds load, all clients in lockstep)

Exponential:      wait 1s → 2s → 4s → 8s
                  (load decreases over time, but clients STILL in lockstep)

Exponential
+ jitter:         wait random(0,1)s → random(0,2)s → random(0,4)s
                  (load decreases AND clients spread out)
```

**Exponential backoff** doubles the wait after each failure, so an overloaded service gets breathing room. **Jitter** adds randomness so clients stop retrying in unison. You need both — backoff alone still produces synchronised waves.

**The AWS SDKs already do this.** Every SDK retries throttling and 5xx errors with exponential backoff and jitter by default, configurable via `maxAttempts` and a retry mode (`legacy`, `standard`, `adaptive`). So on the exam, *“the client should implement exponential backoff”* usually means either the code is using raw HTTP calls, or it needs a higher retry count — not that you must write the algorithm.

**What must NOT be retried**

Only **transient** failures. A `400`, a validation error, an `AccessDenied`, or a `ConditionalCheckFailedException` will fail identically forever, and retrying wastes capacity and hides the bug.

| Retry | Don’t retry |
| --- | --- |
| `429` / `ThrottlingException` | `400` / validation errors |
| `ProvisionedThroughputExceededException` | `AccessDenied` (fix permissions) |
| `500` / `503` / `ServiceUnavailable` | `ConditionalCheckFailedException` — re-read first, then retry the *logic* |
| Timeouts and connection resets | `ResourceNotFoundException` |
| `UnprocessedItems` / `UnprocessedKeys` | Malformed requests |

**Why retries force idempotency**

A retry after a timeout is dangerous in a specific way: **you do not know whether the first attempt succeeded.** The request may have been processed and only the response lost. So retrying can duplicate the effect — charge a card twice, ship an order twice.

The pattern:

```plain text
Client sends a unique request ID with every attempt
        ↓
Server: conditional write of that ID to DynamoDB
        ↓
Write succeeded → first time → do the work, store the result
Write failed    → already processed → return the stored result
```

**DynamoDB’s conditional write is the right primitive** because “check and record” happens atomically in one operation, at any scale, in single-digit milliseconds. A cache would evict the record; a relational database fights Lambda’s connection scaling; S3 has no atomic conditional gate.

Note that an **atomic counter** (`SET x = x + 1`) is concurrency-safe but **not idempotent** — a retry increments twice. Safe for view counts; wrong for money.

**Where this appears, service by service**

| Service | The retry/idempotency fact |
| --- | --- |
| **DynamoDB** | `ProvisionedThroughputExceededException` → backoff. **`UnprocessedItems` from `BatchWriteItem` must be retried by you** — ignoring them silently loses data |
| **Kinesis** | `ProvisionedThroughputExceededException` → backoff, then fix shard count or the partition key |
| **SQS** | Redelivery *is* the retry, driven by visibility timeout. Standard queues are **at-least-once**, so consumers must be idempotent |
| **Lambda (async)** | 2 automatic retries, then destination/DLQ. Your handler will see duplicates |
| **Lambda (sync)** | No platform retry — **the caller** retries |
| **Lambda (streams)** | Retries in order, forever, until the record expires — hence `BisectBatchOnFunctionError` |
| **SNS** | Retries per the delivery policy; delivery is at-least-once |
| **Step Functions** | `Retry` with `IntervalSeconds`, `MaxAttempts`, `BackoffRate` — declarative, no code |
| **API Gateway** | Does not retry the integration; clients retry. A `504` may mean the work *did* complete |
| **S3** | Event notifications are at-least-once |
| **CloudFormation / CodeDeploy** | Roll back rather than retry — a failed deploy is not transient |

**Recall check — retries and idempotency**

- **Handle throttled requests** → **exponential backoff with jitter**
- **Why jitter** → prevents a synchronised retry wave (thundering herd)
- **AWS SDKs** → already retry with backoff and jitter by default
- **`UnprocessedItems` after a batch write** → **retry them yourself** with backoff
- **`AccessDenied` or a validation error** → do **not** retry, fix the cause
- **`ConditionalCheckFailedException`** → re-read, re-apply the logic, then retry
- **Duplicate processing must not double-charge** → idempotency with a request ID + DynamoDB conditional write
- **Store processed request IDs** → **DynamoDB**
- **Atomic counter** → concurrency-safe but **not idempotent**
- **Retries without writing code** → Step Functions `Retry` / `Catch`
- **Standard SQS and SNS are at-least-once** → consumers must be idempotent

---

# Question patterns

> A service begins returning throttling errors under heavy load, and every client immediately retries, making the overload worse. Which retry strategy should the developer implement?
> → **Exponential backoff with jitter**

> A team adds exponential backoff but still sees all clients fail and retry in synchronized waves. What is missing?
> → **Jitter (randomness) to spread retries out**

> A client treats a `503 ServiceUnavailable` as a fatal error and stops processing. What is wrong with this approach?
> → **5xx and throttling errors are transient and should be retried**

> A developer writes custom raw HTTP calls to an AWS API and must handle throttling. Which retry behaviour should they add?
> → **Exponential backoff with jitter**

> An application using an AWS SDK is being throttled despite the SDK's built-in retries. What configuration increases retry attempts?
> → **Increase `maxAttempts` (and/or use a retry mode such as standard or adaptive)**

> Which AWS SDK retry configuration options control the retry algorithm?
> → **Retry modes: `legacy`, `standard`, and `adaptive`**

> A request fails with `400` and a validation error. Should the client retry it?
> → **No — validation errors fail identically and waste capacity**

> A call fails with `AccessDenied`. What is the correct action?
> → **Do not retry — fix the permissions**

> A DynamoDB conditional write returns `ConditionalCheckFailedException`. What should the code do?
> → **Re-read the item, re-apply the logic, then retry**

> A request fails with `ResourceNotFoundException`. Should it be retried?
> → **No — do not retry non-transient errors**

> A retry happens after a network timeout where the response was lost. Why is this dangerous without idempotency?
> → **The first attempt may have succeeded, so retrying could duplicate the effect**

> A payment system must not charge a customer twice when a retry occurs. Which pattern prevents this?
> → **Idempotency using a unique request ID recorded with a DynamoDB conditional write**

> Which primitive is the right place to store processed request IDs for idempotency, and why?
> → **DynamoDB conditional write — atomic check-and-record at any scale in single-digit ms**

> Why is a cache a poor choice for storing processed request IDs?
> → **A cache can evict the record, losing the dedup guard**

> Why is a relational database a poor fit for idempotency records behind Lambda?
> → **It fights Lambda's connection scaling**

> Why is S3 a poor choice for the idempotency gate?
> → **S3 has no atomic conditional write**

> An atomic counter (`SET x = x + 1`) is used to detect duplicates. Is it idempotent?
> → **No — a retry increments twice; concurrency-safe but not idempotent**

> For which use case is an atomic counter acceptable?
> → **View counts (not money or anything that must not double-count)**

> A `BatchWriteItem` call returns a non-empty `UnprocessedItems` list. What must the developer do?
> → **Retry the unprocessed items themselves with backoff — ignoring them silently loses data**

> A Kinesis consumer receives `ProvisionedThroughputExceededException`. What is the correct response?
> → **Backoff, then fix the shard count or partition key**

> In SQS, what mechanism actually drives redelivery/retry?
> → **The visibility timeout**

> Why must consumers of a standard SQS queue be idempotent?
> → **Standard queues are at-least-once, so duplicates are possible**

> An SNS topic delivers notifications with what delivery guarantee?
> → **At-least-once delivery**

> An asynchronous Lambda invocation fails. How many automatic retries does the platform perform?
> → **2 automatic retries, then a destination or DLQ**

> A developer worries about duplicate records from async Lambda retries. What should the handler do?
> → **Be idempotent — the handler will see duplicates**

> A synchronous Lambda invocation fails. Who retries?
> → **No platform retry — the caller retries**

> A Lambda consumer of a Kinesis stream keeps failing on one record and blocks the shard. What setting helps?
> → **`BisectBatchOnFunctionError` to isolate the bad record**

> A Lambda stream consumer retries a failing record. How long does it retry?
> → **In order, forever, until the record expires**

> Which service lets a developer add retries declaratively with no code, using `IntervalSeconds`, `MaxAttempts`, and `BackoffRate`?
> → **Step Functions `Retry` (with `Catch`)**

> API Gateway returns a `504` to a client for a call that may have completed. What should the client understand?
> → **API Gateway does not retry the integration; the work may have completed, so retries must be idempotent**

> S3 event notifications are delivered with what guarantee?
> → **At-least-once**

> A CloudFormation or CodeDeploy deployment fails. Should it retry or roll back?
> → **Roll back — a failed deploy is not a transient failure**

> A service throws `ProvisionedThroughputExceededException`. Is this retryable?
> → **Yes — retry with exponential backoff**

> Which of these should NOT be retried: `429`, timeouts, `UnprocessedKeys`, or malformed requests?
> → **Malformed requests (and other non-transient errors)**

> A client needs to survive a leader election and a load balancer draining a target. What should it do?
> → **Retry with exponential backoff with jitter, since these are transient**

> What is the term for all clients failing at once and retrying at once, preventing a queue from draining?
> → **Thundering herd**

> A developer must reduce retry load over time while still letting clients eventually succeed. Which combination is required?
> → **Exponential backoff (load decreases over time) plus jitter (clients spread out)**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Throttling | **exponential backoff + jitter** |
| Jitter | **prevents synchronized retry wave (thundering herd)** |
| Backoff alone | **still synchronized — need jitter too** |
| AWS SDKs default | **retry throttling/5xx with backoff + jitter** |
| Retry config | **`maxAttempts` + retry mode (`legacy`/`standard`/`adaptive`)** |
| `429` / `ThrottlingException` | **retry with backoff** |
| `ProvisionedThroughputExceededException` | **retry with backoff** |
| `500` / `503` | **retry (transient)** |
| Timeouts / connection resets | **retry** |
| `UnprocessedItems` / `UnprocessedKeys` | **retry them yourself** |
| `400` / validation error | **do NOT retry** |
| `AccessDenied` | **do NOT retry — fix permissions** |
| `ConditionalCheckFailedException` | **re-read, re-apply logic, retry** |
| `ResourceNotFoundException` | **do NOT retry** |
| Idempotency key store | **DynamoDB conditional write** |
| Why DynamoDB for idempotency | **atomic check-and-record, any scale, ms latency** |
| Why not cache | **record can be evicted** |
| Why not RDBMS | **fights Lambda connection scaling** |
| Why not S3 | **no atomic conditional gate** |
| Atomic counter | **concurrency-safe but NOT idempotent** |
| Atomic counter safe for | **view counts, not money** |
| BatchWriteItem | **retry `UnprocessedItems` yourself** |
| Kinesis throttle | **backoff then fix shards/partition key** |
| SQS redelivery | **visibility timeout** |
| Standard SQS | **at-least-once → idempotent consumers** |
| SNS delivery | **at-least-once** |
| Lambda async retries | **2, then destination/DLQ** |
| Lambda sync retries | **none — caller retries** |
| Lambda stream retries | **in order, forever, until expiry** |
| Lambda stream fix | **`BisectBatchOnFunctionError`** |
| Declarative retries | **Step Functions `Retry`/`Catch`** |
| Step Functions fields | **`IntervalSeconds`, `MaxAttempts`, `BackoffRate`** |
| API Gateway integration retry | **does not retry — client does** |
| API Gateway `504` | **work may have completed** |
| S3 event notifications | **at-least-once** |
| CloudFormation / CodeDeploy | **roll back, don't retry** |
| Thundering herd | **synchronized retry wave** |
| Distributed call failures | **transient — design clients to retry** |
