## Amazon SQS
**Why it exists**
If component A calls component B directly, then B being slow makes A slow, and B being down makes A fail. A queue breaks that coupling: A writes a message and moves on; B reads it whenever it can. The queue absorbs bursts, survives consumer outages, and lets each side scale independently.
**How delivery actually works — visibility timeout**
There is no “lock” on a message. Instead:
```plain text
Consumer calls ReceiveMessage
        ↓
SQS returns the message AND hides it for the visibility timeout
        ↓
Consumer processes it
        ↓
Consumer calls DeleteMessage  →  message is gone for good
        ↓
(If the consumer never deletes it, the timeout expires
 and the message becomes visible to other consumers again)
```
**This is why a message is redelivered on failure without any retry logic: crashing simply means never calling ****`DeleteMessage`****.** Elegant, and the source of the most common SQS bug.
**The bug:** if processing takes longer than the visibility timeout, the message reappears while the first consumer is still working. A second consumer starts on it. The work happens twice.
**The rule that follows:** **visibility timeout ≥ processing time.** With Lambda, that means visibility timeout ≥ function timeout, and AWS recommends **6×** to absorb retries. If a consumer needs more time mid-processing, it can call **`ChangeMessageVisibility`** to extend the hold.
**Standard vs FIFO**
<table header-row="true">
<tr>
<td></td>
<td>**Standard**</td>
<td>**FIFO**</td>
</tr>
<tr>
<td>Throughput</td>
<td>Effectively unlimited</td>
<td>300 msg/s, **3,000** with batching (high-throughput FIFO: much higher)</td>
</tr>
<tr>
<td>Ordering</td>
<td>Best-effort</td>
<td>**Strict, within a message group**</td>
</tr>
<tr>
<td>Delivery</td>
<td>**At least once** (duplicates possible)</td>
<td>**Exactly once** (deduplicated)</td>
</tr>
<tr>
<td>Name requirement</td>
<td>—</td>
<td>Must end in `.fifo`</td>
</tr>
</table>
**The queue type cannot be changed after creation.** If you need the other type, create a new queue. This is a `CreateQueue` decision.
FIFO mechanics:
- **`MessageGroupId`** — the ordering scope. Messages in the same group are strictly ordered; different groups process in parallel. This is how FIFO scales at all: use many groups. You can have as many consumers as “MessageGroupId” for you SQS FIFO queues
- **`MessageDeduplicationId`** — SQS discards duplicates with the same ID within a **5-minute** deduplication window. **Content-based deduplication** hashes the body instead of requiring an explicit ID.
<table header-row="true">
<tr>
<td></td>
<td>**MessageDeduplicationId**</td>
<td>**Content-based deduplication**</td>
</tr>
<tr>
<td>ID created by</td>
<td>**You**</td>
<td>**SQS**</td>
</tr>
<tr>
<td>Based on</td>
<td>Any ID you provide</td>
<td>Hash of the **message body**</td>
</tr>
<tr>
<td>Need to provide ID?</td>
<td>✅ Yes</td>
<td>❌ No</td>
</tr>
<tr>
<td>Purpose</td>
<td>Decide yourself which messages are duplicates</td>
<td>Automatically detect identical message bodies</td>
</tr>
</table>
**Long polling**
`ReceiveMessage` with a wait time of 0 returns immediately, often empty, and you pay for the call. **Long polling** (`ReceiveMessageWaitTimeSeconds`, up to **20 seconds**) holds the request open until a message arrives or the timer expires.
Result: fewer empty responses, lower cost, lower latency. There is no downside — enable it.
**Dead-letter queues**
A **redrive policy** sets `maxReceiveCount`. Once a message has been received that many times without being deleted, SQS moves it to the **DLQ** instead of redelivering forever.
This isolates poison-pill messages so they stop blocking the queue, and gives you somewhere to inspect them.
**The type-matching rule, which is absolute:**
- FIFO queue → DLQ **must** be FIFO
- Standard queue → DLQ **must** be Standard
- ❌ Standard queue cannot use a FIFO DLQ
- ❌ FIFO queue cannot use a Standard DLQ
The common exam trap is the statement *“a standard queue’s DLQ can be either standard or FIFO”* — **false**.
**DLQ redrive** lets you move messages from the DLQ back to the source queue after fixing the bug.
**Delay queues**
`DelaySeconds` (0–15 minutes) postpones delivery of new messages. Queue-level delay applies to every message; **per-message delay is available on Standard queues only**.
**Queue management APIs — three that are easy to confuse**
<table header-row="true">
<tr>
<td>API</td>
<td>Effect</td>
</tr>
<tr>
<td>**`DeleteQueue`**</td>
<td>Deletes the queue, all messages, and the configuration. **Irreversible.** AWS recommends waiting \~60 seconds before recreating a queue with the same name</td>
</tr>
<tr>
<td>**`PurgeQueue`**</td>
<td>Deletes all messages, **keeps the queue**</td>
</tr>
<tr>
<td>**`RemovePermission`**</td>
<td>Removes an access permission from the queue policy</td>
</tr>
</table>
So: cleaning up after a test and never needing the queue again → `DeleteQueue`. Clearing bad messages but keeping the queue for future use → `PurgeQueue`.
**Numbers**
<table header-row="true">
<tr>
<td>Setting</td>
<td>Value</td>
</tr>
<tr>
<td>Max message size</td>
<td>**1 MiB** (SQS Extended Client Library + S3 for up to 2 GB)</td>
</tr>
<tr>
<td>Message retention</td>
<td>Default **4 days**, range 60 s – **14 days**</td>
</tr>
<tr>
<td>Visibility timeout</td>
<td>Default **30 s**, range 0 – **12 hours**</td>
</tr>
<tr>
<td>Long polling wait</td>
<td>0 – **20 s**</td>
</tr>
<tr>
<td>Delay</td>
<td>0 – **15 minutes**</td>
</tr>
<tr>
<td>Batch APIs (`SendMessageBatch` etc.)</td>
<td>**10 messages**</td>
</tr>
<tr>
<td>Message attributes</td>
<td>**10** per message, and they count toward the 256 KB limit</td>
</tr>
<tr>
<td>Messages per `ReceiveMessage`</td>
<td>**10**</td>
</tr>
</table>
- **Stored/backlog messages:** **No limit** on the number of messages a queue can store.
- **In-flight messages:** received by a consumer but **not yet deleted** → approximately **120,000 max** for a standard queue by default.
**Recall check — SQS**
- **Decouple components / buffer work / absorb bursts** → SQS
- **Default visibility timeout** → **30 seconds**
- **Duplicate processing** → visibility timeout shorter than processing time
- **Consumer needs more time mid-processing** → `ChangeMessageVisibility`
- **Can the queue type be changed?** → **No** — create a new queue
- **Strict ordering / no duplicates** → FIFO queue
- **Maximum throughput** → Standard queue
- **Parallelism within FIFO** → many `MessageGroupId` values
- **Deduplication window** → 5 minutes
- **FIFO DLQ** → must be FIFO. **Standard DLQ** → must be Standard
- **Poison messages blocking the queue** → DLQ via `maxReceiveCount`
- **Empty responses wasting money** → enable long polling (20 s)
- **Delay every message** → delay queue. **Delay one message** → per-message `DelaySeconds` (Standard only)
- **Message larger than 256 KB** → SQS Extended Client Library (payload in S3)
- **Messages disappearing after 4 days** → default retention; raise to 14 days
- **Delete the queue and its contents** → `DeleteQueue`
- **Clear messages but keep the queue** → `PurgeQueue`
- **Remove access rights** → `RemovePermission`
- **SQS is not for human approval workflows** → that’s CodePipeline manual approval or Step Functions
- **In-flight messages:** received by a consumer but **not yet deleted** → approximately **120,000 max** for a standard queue by default.
---
