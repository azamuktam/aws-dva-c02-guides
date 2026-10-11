## Amazon SNS

**Why it exists**

A queue delivers each message to one consumer. Sometimes several independent systems each need to know about the same event — billing, analytics, notifications, search indexing. SNS is **publish/subscribe**: publish once, and every subscriber gets a copy, pushed to them rather than polled.

```plain text
Publisher
    ↓
SNS Topic
    ├──→ SQS Queue A       (order processing)
    ├──→ SQS Queue B       (analytics)
    ├──→ Lambda            (notifications)
    ├──→ HTTPS endpoint    (partner webhook)
    └──→ Email / SMS / Mobile push
```

**Protocols:** SQS, Lambda, HTTP/HTTPS, Email, Email-JSON, SMS, mobile push (APNS/FCM), Kinesis Data Firehose.

**The fan-out pattern**

**SNS topic → multiple SQS queues** is the canonical event-distribution pattern, and understanding *why* it beats SNS alone matters.

SNS by itself pushes and retries, but does not store. If a subscriber is down long enough, the message is lost. Putting an SQS queue in front of each consumer gives each one **durability and its own pace**:

```plain text
SNS Topic
   ├──→ SQS Queue A ──→ fast consumer
   └──→ SQS Queue B ──→ slow consumer, catching up at its own speed
```

Each consumer can fail, be redeployed, or fall behind without affecting the others. This is why “several systems must each process the event independently and reliably” is answered with **SNS → SQS**, not SNS alone.

**SNS Message filtering**

A **filter policy** on a subscription means that subscriber receives only matching messages. Default scope is **message attributes**; setting `FilterPolicyScope: MessageBody` also allows matching on the payload.

The important nuance for exam purposes: attribute-based filtering requires the **publisher to set the attributes**. If the requirement is “route on fields inside the payload *without changing the producer*”, EventBridge is the cleaner answer.

**Other points**

- **FIFO topics** provide ordering and deduplication, but can only deliver to **SQS FIFO queues**.
- Max message size **256 KB**.
- **DLQ is configured on the subscription**, not on the topic.
- **Subscription confirmation** — a new HTTP/HTTPS or email subscription starts in **`PendingConfirmation`**. SNS sends a `SubscriptionConfirmation` message containing a token, and the endpoint must call `ConfirmSubscription` (or the recipient must click the emailed link) before any notifications are delivered. This exists so you cannot subscribe someone else’s endpoint without their consent. The exam symptom: *“the topic publishes successfully but the HTTP endpoint receives nothing”* → the subscription was never confirmed. SQS and Lambda subscriptions are auto-confirmed.
- **Delivery retry policies** — many retries for HTTP/S endpoints; AWS-managed for SQS and Lambda.
- Encryption with KMS; a **topic access policy** controls cross-account publishing.

**SNS vs SQS**

|    | **SNS** | **SQS** |
| --- | --- | --- |
| Model | **Push** to subscribers | **Pull** (poll) by consumers |
| Consumers | Many, all get every message | Each message goes to **one** consumer |
| Persistence | Not stored (retries only) | Stored up to 14 days |
| Purpose | Fan-out, notification | Buffering, load levelling |

**Recall check — SNS**

- **One event, several independent systems must each handle it** → **SNS → SQS fan-out**
- **Notify a person (email/SMS)** → SNS
- **Buffer work so a slow consumer isn’t overwhelmed** → SQS
- **Durability plus fan-out** → SNS → SQS queues
- **Only some subscribers should receive certain messages** → SNS filter policy
- **Filter on payload fields without changing the producer** → EventBridge
- **Ordered, deduplicated pub/sub** → SNS FIFO topic → SQS FIFO queue
- **Capture failed SNS deliveries** → subscription DLQ
- **Notify approvers in a pipeline** → SNS

---

# Question patterns

> One event must be delivered to many SQS queues without any consumer polling the publisher. Which pattern?
> → **SNS fan-out to multiple SQS queues**

> Several independent systems must each process the same event reliably, even if one is temporarily down. Which design?
> → **SNS topic → one SQS queue per consumer**

> A billing, analytics and search-indexing service all need the same order event. What should the order service publish to?
> → **a single SNS topic with a subscription per consumer**

> A person must be alerted by email and SMS when a job completes. Which service and feature?
> → **SNS Email and SMS subscriptions**

> A slow consumer keeps getting overwhelmed by bursts. What should sit in front of it?
> → **an SQS queue (buffering / load levelling)**

> Durability plus fan-out is required so no subscriber misses an event. What pattern answers this?
> → **SNS → SQS queues**

> Only some subscribers should receive certain messages, based on attributes. Which feature?
> → **SNS subscription filter policy**

> Filtering must be based on fields inside the message payload, not just attributes, and the publisher can set attributes. Which setting?
> → **FilterPolicyScope: MessageBody**

> Routing must match on fields inside the payload without changing the producer. Which service is the cleaner fit?
> → **Amazon EventBridge**

> Ordered, deduplicated publish/subscribe is required. Which configuration?
> → **SNS FIFO topic → SQS FIFO queue**

> Which destinations can an SNS FIFO topic deliver to?
> → **SQS FIFO queues only**

> Failed SNS deliveries must be captured for later inspection. Where is the dead-letter queue configured?
> → **on the subscription, not on the topic**

> What is the maximum SNS message size?
> → **256 KB**

> A pipeline must notify human approvers to act. Which service?
> → **SNS**

> Which protocols can SNS deliver to?
> → **SQS, Lambda, HTTP/HTTPS, Email, Email-JSON, SMS, mobile push (APNS/FCM), Kinesis Data Firehose**

> Push notifications must reach iOS and Android devices. Which feature?
> → **SNS mobile push (APNS/FCM)**

> A topic publishes successfully but the HTTP endpoint receives nothing. What is the most likely cause?
> → **the subscription was never confirmed (still PendingConfirmation)**

> What status does a new HTTP/HTTPS or email subscription start in?
> → **PendingConfirmation**

> How does an HTTP/HTTPS or email endpoint become active?
> → **call ConfirmSubscription with the token (or click the emailed link)**

> Which SNS subscriptions are auto-confirmed and need no explicit confirmation?
> → **SQS and Lambda subscriptions**

> Which message carries the token an endpoint uses to confirm a subscription?
> → **SubscriptionConfirmation**

> Why does SNS require subscription confirmation at all?
> → **so you cannot subscribe someone else's endpoint without their consent**

> Who must set the message attributes for attribute-based filtering to work?
> → **the publisher**

> Which model does SNS use to reach subscribers: push or pull?
> → **push**

> How does SQS deliver to consumers: push or pull?
> → **pull (polling)**

> Does SNS store messages for later retrieval?
> → **No — it retries only, it does not store**

> How long can an SQS message be retained?
> → **up to 14 days**

> With SNS, how many consumers receive each message?
> → **all subscribers get a copy (many consumers)**

> With SQS, how many consumers receive each message?
> → **exactly one consumer**

> What is SNS's core purpose versus SQS's?
> → **SNS: fan-out / notification; SQS: buffering / load levelling**

> How should a topic be encrypted?
> → **with KMS**

> How is cross-account publishing to an SNS topic controlled?
> → **the topic access policy**

> How do delivery retry policies differ between endpoint types?
> → **many retries for HTTP/S endpoints; AWS-managed for SQS and Lambda**

> What additional guarantees does an SNS FIFO topic provide?
> → **message ordering and deduplication**

> Which SNS feature lets each subscriber filter messages independently?
> → **the subscription filter policy**

> A team wants the highest-throughput, lowest-latency simple fan-out with no payload filtering. SNS or EventBridge?
> → **SNS**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Fan-out | **SNS → SQS** |
| Notify a person (email/SMS) | **SNS** |
| Buffer work / slow consumer | **SQS** |
| Durability plus fan-out | **SNS → SQS queues** |
| Only some subscribers | **SNS filter policy** |
| Filter payload without changing producer | **EventBridge** |
| Ordered, deduplicated pub/sub | **SNS FIFO topic → SQS FIFO queue** |
| Failed deliveries | **subscription DLQ** |
| Notify approvers | **SNS** |
| Model | **push to subscribers** |
| Consumers | **many, all get every message** |
| Persistence | **not stored (retries only)** |
| Purpose | **fan-out / notification** |
| SQS model | **pull (poll)** |
| SQS persistence | **stored up to 14 days** |
| SQS consumers | **one consumer per message** |
| Max message size | **256 KB** |
| DLQ location | **on the subscription** |
| Filter default scope | **message attributes** |
| Filter the payload body | **FilterPolicyScope: MessageBody** |
| Attribute filtering owner | **the publisher** |
| New HTTP/email sub status | **PendingConfirmation** |
| Confirm an endpoint | **ConfirmSubscription** |
| Confirmation message | **SubscriptionConfirmation** |
| Auto-confirmed subscriptions | **SQS and Lambda** |
| Unconfirmed-sub symptom | **publishes, but endpoint receives nothing** |
| FIFO delivers to | **SQS FIFO queues only** |
| FIFO provides | **ordering + deduplication** |
| Encryption | **KMS** |
| Cross-account publishing | **topic access policy** |
| HTTP/S retries | **many** |
| SQS/Lambda retries | **AWS-managed** |
| Protocols | **SQS, Lambda, HTTP/S, Email, Email-JSON, SMS, push, Firehose** |
| Mobile push | **APNS / FCM** |
| Highest-throughput fan-out | **SNS** |
