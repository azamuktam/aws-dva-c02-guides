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

**This is why a message is redelivered on failure without any retry logic: crashing simply means never calling `DeleteMessage`.** Elegant, and the source of the most common SQS bug.

**The bug:** if processing takes longer than the visibility timeout, the message reappears while the first consumer is still working. A second consumer starts on it. The work happens twice.

**The rule that follows:** **visibility timeout ≥ processing time.** With Lambda, that means visibility timeout ≥ function timeout, and AWS recommends **6×** to absorb retries. If a consumer needs more time mid-processing, it can call **`ChangeMessageVisibility`** to extend the hold.

**Standard vs FIFO**

|    | **Standard** | **FIFO** |
| --- | --- | --- |
| Throughput | Effectively unlimited | 300 msg/s, **3,000** with batching (high-throughput FIFO: much higher) |
| Ordering | Best-effort | **Strict, within a message group** |
| Delivery | **At least once** (duplicates possible) | **Exactly once** (deduplicated) |
| Name requirement | — | Must end in `.fifo` |

**The queue type cannot be changed after creation.** If you need the other type, create a new queue. This is a `CreateQueue` decision.

FIFO mechanics:

- **`MessageGroupId`** — the ordering scope. Messages in the same group are strictly ordered; different groups process in parallel. This is how FIFO scales at all: use many groups. You can have as many consumers as “MessageGroupId” for you SQS FIFO queues
- **`MessageDeduplicationId`** — SQS discards duplicates with the same ID within a **5-minute** deduplication window. **Content-based deduplication** hashes the body instead of requiring an explicit ID.

|    | **MessageDeduplicationId** | **Content-based deduplication** |
| --- | --- | --- |
| ID created by | **You** | **SQS** |
| Based on | Any ID you provide | Hash of the **message body** |
| Need to provide ID? | ✅ Yes | ❌ No |
| Purpose | Decide yourself which messages are duplicates | Automatically detect identical message bodies |

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

| API | Effect |
| --- | --- |
| **`DeleteQueue`** | Deletes the queue, all messages, and the configuration. **Irreversible.** AWS recommends waiting \~60 seconds before recreating a queue with the same name |
| **`PurgeQueue`** | Deletes all messages, **keeps the queue** |
| **`RemovePermission`** | Removes an access permission from the queue policy |

So: cleaning up after a test and never needing the queue again → `DeleteQueue`. Clearing bad messages but keeping the queue for future use → `PurgeQueue`.

**Numbers**

| Setting | Value |
| --- | --- |
| Max message size | **1 MiB** (SQS Extended Client Library + S3 for up to 2 GB) |
| Message retention | Default **4 days**, range 60 s – **14 days** |
| Visibility timeout | Default **30 s**, range 0 – **12 hours** |
| Long polling wait | 0 – **20 s** |
| Delay | 0 – **15 minutes** |
| Batch APIs (`SendMessageBatch` etc.) | **10 messages** |
| Message attributes | **10** per message, and they count toward the 256 KB limit |
| Messages per `ReceiveMessage` | **10** |

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

# Question patterns

> Component B is slow or down and it's making component A slow or fail. What breaks the coupling?
> → **SQS — A writes a message and moves on; B reads when it can**

> What is the default visibility timeout?
> → **30 seconds (range 0 to 12 hours)**

> A message is processed twice because a second consumer picked it up mid-work. What is the cause?
> → **The visibility timeout was shorter than the processing time, so the message became visible again**

> A consumer needs more time while still processing. Which call extends the hold?
> → **ChangeMessageVisibility**

> What is the rule relating visibility timeout to processing time (and with Lambda)?
> → **Visibility timeout ≥ processing time; with Lambda, visibility timeout ≥ function timeout, and AWS recommends 6× to absorb retries**

> Can a queue's type (Standard vs FIFO) be changed after creation? What must you do?
> → **No — in-flight the type cannot change; create a new queue (a CreateQueue decision)**

> Messages must be processed exactly once and in order. Which queue type and what is required?
> → **SQS FIFO with MessageGroupId + MessageDeduplicationId**

> Maximum throughput with a standard queue versus FIFO?
> → **Standard is effectively unlimited; FIFO is 300 msg/s, or 3,000 with batching (much higher for high-throughput FIFO)**

> How does FIFO scale to parallelism while keeping order?
> → **Use many MessageGroupId values — same group is strictly ordered, different groups process in parallel**

> How long is the FIFO deduplication window?
> → **5 minutes**

> Who creates the deduplication ID under content-based deduplication, and what is it based on?
> → **SQS; it hashes the message body (no explicit ID needed)**

> Under MessageDeduplicationId, who provides the ID?
> → **You provide any ID you choose**

> Bad messages keep failing and block the queue. Which feature isolates them?
> → **Dead-letter queue (DLQ) via a redrive policy with maxReceiveCount**

> What does the redrive policy set?
> → **maxReceiveCount — after that many receives without deletion, SQS moves the message to the DLQ**

> A standard queue's DLQ can be standard or FIFO — true or false?
> → **False — DLQ type must match the source queue**

> What is the DLQ type-matching rule?
> → **FIFO → FIFO DLQ; Standard → Standard DLQ; cross-matching is not allowed**

> After fixing a bug, messages must be moved from the DLQ back to the source queue. Which feature?
> → **DLQ redrive**

> Consumers receive many empty responses, wasting requests and money. What reduces this?
> → **Long polling (ReceiveMessageWaitTimeSeconds, up to 20 seconds)**

> What is the maximum long polling wait, and are there downsides?
> → **20 seconds; there is no downside — enable it**

> New messages should be invisible to consumers for 10 minutes after arrival. Which feature?
> → **Delay queues (DelaySeconds, 0–15 minutes, applies queue-wide)**

> Delay only a single message rather than the whole queue. Which option, and on which queue type?
> → **Per-message DelaySeconds — available on Standard queues only**

> A message is larger than the maximum size. What is the solution?
> → **SQS Extended Client Library — payload stored in S3 (up to 2 GB)**

> What is the maximum SQS message size?
> → **1 MiB (256 KB with message attributes counting toward it; Extended Client Library + S3 for up to 2 GB)**

> Messages disappear after about 4 days. Why, and how is it changed?
> → **Default retention is 4 days; raise it up to a maximum of 14 days**

> What is the message retention range?
> → **60 seconds to 14 days**

> The queue and all of its messages and configuration must be removed permanently. Which API?
> → **DeleteQueue (irreversible; wait ~60 seconds before recreating the same name)**

> Messages must be cleared but the queue kept for future use. Which API?
> → **PurgeQueue**

> An access permission must be removed from the queue policy. Which API?
> → **RemovePermission**

> How many messages can a single SendMessageBatch (and ReceiveMessage) handle?
> → **10 messages**

> How many message attributes per message are allowed, and do they count toward the size limit?
> → **10 per message, and they count toward the size limit**

> Is there a limit on stored/backlog messages queued?
> → **No limit on stored messages**

> What is the approximate default maximum of in-flight messages on a standard queue?
> → **~120,000 (received but not yet deleted)**

> What are "in-flight" messages?
> → **Messages received by a consumer but not yet deleted**

> A workflow needs human approval. Why is SQS the wrong answer?
> → **SQS is not for human approval — use CodePipeline manual approval or Step Functions**

> What suffix must a FIFO queue name end with?
> → **.fifo**

> What delivery guarantee does a Standard queue provide?
> → **At least once (duplicates possible)**

> What delivery guarantee does a FIFO queue provide?
> → **Exactly once (deduplicated)**

> What ordering does a Standard queue provide?
> → **Best-effort (not guaranteed)**

> What ordering does a FIFO queue provide?
> → **Strict, within a message group**

> A queue must absorb bursts, survive consumer outages and let each side scale independently. Which service?
> → **SQS**

> Under the hood, what mechanism makes a failed message get redelivered without any retry code?
> → **The consumer simply never calls DeleteMessage; the visibility timeout expires and the message reappears**

> After a consumer successfully processes a message, what must it call to remove it permanently?
> → **DeleteMessage**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Decouple / buffer / absorb bursts | **SQS** |
| Default visibility timeout | **30 seconds** |
| Visibility timeout range | **0 s – 12 hours** |
| Duplicate processing cause | **visibility timeout < processing time** |
| Extend hold mid-processing | **ChangeMessageVisibility** |
| Lambda visibility rule | **≥ function timeout (recommend 6×)** |
| Change queue type after creation | **no — create a new queue** |
| Order + exactly once | **FIFO** |
| Max throughput | **Standard** |
| FIFO throughput | **300/s, 3,000/s batched** |
| Ordering scope in FIFO | **MessageGroupId** |
| Parallelism within FIFO | **many MessageGroupId values** |
| Deduplication window | **5 minutes** |
| Dedup ID creator (content-based) | **SQS (hash of body)** |
| Dedup ID creator (explicit) | **you** |
| Poison messages | **DLQ via maxReceiveCount** |
| DLQ type matching | **must match source queue** |
| Standard queue + FIFO DLQ | **not allowed** |
| Return messages from DLQ | **DLQ redrive** |
| Fewer empty polls | **long polling (20 s)** |
| Long polling max wait | **20 seconds** |
| Delayed visibility (all messages) | **delay queue** |
| Delayed visibility (one message) | **per-message DelaySeconds (Standard only)** |
| Delay range | **0–15 minutes** |
| Max message size | **1 MiB** |
| Larger than 1 MiB | **Extended Client Library + S3 (up to 2 GB)** |
| Default retention | **4 days** |
| Retention range | **60 s – 14 days** |
| Delete queue + contents | **DeleteQueue (irreversible)** |
| Recreate same name wait | **~60 seconds** |
| Clear messages, keep queue | **PurgeQueue** |
| Remove access rights | **RemovePermission** |
| Batch API size | **10 messages** |
| Attributes per message | **10 (count toward size)** |
| Messages per ReceiveMessage | **10** |
| Stored message limit | **none** |
| In-flight max (standard) | **~120,000** |
| In-flight means | **received, not yet deleted** |
| Human approval workflow | **CodePipeline / Step Functions, not SQS** |
| FIFO name suffix | **.fifo** |
| Standard delivery | **at least once** |
| FIFO delivery | **exactly once** |
| Standard ordering | **best-effort** |
| FIFO ordering | **strict within group** |
| Redelivery on failure | **never DeleteMessage → timeout expires** |
| Remove processed message | **DeleteMessage** |
