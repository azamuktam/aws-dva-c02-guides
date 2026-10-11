# PART 6 — OBSERVABILITY: KNOWING WHAT WENT WRONG
**Domain 4 is 18%, but observability questions also hide inside the other three domains** — most “why is this broken” questions are really asking which tool would tell you.
Four tools, four different questions:
<table header-row="true">
<tr>
<td>Tool</td>
<td>Answers</td>
</tr>
<tr>
<td>**CloudWatch Metrics**</td>
<td>“Is the number too high or too low?”</td>
</tr>
<tr>
<td>**CloudWatch Logs**</td>
<td>“What did the application say?”</td>
</tr>
<tr>
<td>**X-Ray**</td>
<td>“Which step in the chain was slow or failed?”</td>
</tr>
<tr>
<td>**CloudTrail**</td>
<td>“Who made this change to AWS itself?”</td>
</tr>
</table>
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
<table header-row="true">
<tr>
<td>Retry</td>
<td>Don’t retry</td>
</tr>
<tr>
<td>`429` / `ThrottlingException`</td>
<td>`400` / validation errors</td>
</tr>
<tr>
<td>`ProvisionedThroughputExceededException`</td>
<td>`AccessDenied` (fix permissions)</td>
</tr>
<tr>
<td>`500` / `503` / `ServiceUnavailable`</td>
<td>`ConditionalCheckFailedException` — re-read first, then retry the *logic*</td>
</tr>
<tr>
<td>Timeouts and connection resets</td>
<td>`ResourceNotFoundException`</td>
</tr>
<tr>
<td>`UnprocessedItems` / `UnprocessedKeys`</td>
<td>Malformed requests</td>
</tr>
</table>
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
<table header-row="true">
<tr>
<td>Service</td>
<td>The retry/idempotency fact</td>
</tr>
<tr>
<td>**DynamoDB**</td>
<td>`ProvisionedThroughputExceededException` → backoff. **`UnprocessedItems`**** from ****`BatchWriteItem`**** must be retried by you** — ignoring them silently loses data</td>
</tr>
<tr>
<td>**Kinesis**</td>
<td>`ProvisionedThroughputExceededException` → backoff, then fix shard count or the partition key</td>
</tr>
<tr>
<td>**SQS**</td>
<td>Redelivery *is* the retry, driven by visibility timeout. Standard queues are **at-least-once**, so consumers must be idempotent</td>
</tr>
<tr>
<td>**Lambda (async)**</td>
<td>2 automatic retries, then destination/DLQ. Your handler will see duplicates</td>
</tr>
<tr>
<td>**Lambda (sync)**</td>
<td>No platform retry — **the caller** retries</td>
</tr>
<tr>
<td>**Lambda (streams)**</td>
<td>Retries in order, forever, until the record expires — hence `BisectBatchOnFunctionError`</td>
</tr>
<tr>
<td>**SNS**</td>
<td>Retries per the delivery policy; delivery is at-least-once</td>
</tr>
<tr>
<td>**Step Functions**</td>
<td>`Retry` with `IntervalSeconds`, `MaxAttempts`, `BackoffRate` — declarative, no code</td>
</tr>
<tr>
<td>**API Gateway**</td>
<td>Does not retry the integration; clients retry. A `504` may mean the work *did* complete</td>
</tr>
<tr>
<td>**S3**</td>
<td>Event notifications are at-least-once</td>
</tr>
<tr>
<td>**CloudFormation / CodeDeploy**</td>
<td>Roll back rather than retry — a failed deploy is not transient</td>
</tr>
</table>
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
