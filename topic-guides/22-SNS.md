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
