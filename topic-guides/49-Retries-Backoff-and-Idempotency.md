# PART 6 — OBSERVABILITY: KNOWING WHAT WENT WRONG
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
| **DynamoDB** | `ProvisionedThroughputExceededException` → backoff. **`UnprocessedItems`**** from ****`BatchWriteItem`**** must be retried by you** — ignoring them silently loses data |
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
- **`UnprocessedItems`**** after a batch write** → **retry them yourself** with backoff
- **`AccessDenied`**** or a validation error** → do **not** retry, fix the cause
- **`ConditionalCheckFailedException`** → re-read, re-apply the logic, then retry
- **Duplicate processing must not double-charge** → idempotency with a request ID + DynamoDB conditional write
- **Store processed request IDs** → **DynamoDB**
- **Atomic counter** → concurrency-safe but **not idempotent**
- **Retries without writing code** → Step Functions `Retry` / `Catch`
- **Standard SQS and SNS are at-least-once** → consumers must be idempotent
---
