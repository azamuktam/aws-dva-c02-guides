## Amazon EventBridge
**What it is**
EventBridge is an **event bus** that receives events from AWS services, applications, or SaaS providers and automatically sends them to the correct target.
**Mental model**
Think of it as a **post office**:
- AWS service = Sends a letter (event)
- EventBridge = Post office
- Lambda, SNS, SQS, Step Functions = Receivers
```plain text
S3 uploads file
      │
      ▼
EventBridge
      │
 ┌────┼────┐
 ▼    ▼    ▼
Lambda SQS SNS
```
**Why it exists**
SNS fans out to whoever subscribed; the publisher decides the topic. EventBridge inverts this: producers emit events describing *what happened*, and **consumers declare what they care about** using rules matched against the event content. Adding a consumer requires no change to the producer at all.
It also does things no other messaging service does: it receives **AWS service events natively**, it can **archive and replay**, and it can act as a **scheduler**.
```plain text
Event source
(AWS service, your application, SaaS partner)
        ↓
   Event Bus
        ↓
     Rule          ← event pattern OR schedule
        ↓
   Target(s)       ← up to 5 per rule
   Lambda / SQS / SNS / Step Functions / ECS task / API destination / ...
```
**Event buses**
- **Default bus** — automatically receives events from AWS services. This is how “run something when an EC2 instance changes state” or “react to an AWS API call” works.
- **Custom bus** — for your own application events.
- **Partner bus** — SaaS providers (Datadog, Shopify, Zendesk) publish directly into your account.
**Rules**
**Event pattern** — JSON matched against the event, **including nested fields in the payload**. Supports prefix, suffix, numeric comparison, `anything-but`, `exists`, IP matching, and wildcards.
```json
{
  "source": ["myapp.orders"],
  "detail-type": ["OrderPlaced"],
  "detail": {
    "amount": [{ "numeric": [">", 500] }],
    "country": ["UZ", "KZ"]
  }
}
```
This content-based matching on the payload is the core capability that distinguishes EventBridge from SNS.
**Schedule** — cron or rate expressions (`rate(5 minutes)`, `cron(0 12 * * ? *)`). This is the replacement for what used to be called a “CloudWatch Events scheduled rule”, and it is the answer to “run a Lambda every night”.
**Everything else it offers**
- **Input transformer** — reshape the event before it reaches the target.
- **Archive and replay** — store matching events and re-deliver them later. **Nothing else in the messaging stack can replay.** The answer to “we need to reprocess last Tuesday’s events”.
- **Schema registry and discovery** — infers schemas from events flowing through a bus and generates code bindings.
- **DLQ and retry policy per target.**
- **API destinations** — call external HTTP/SaaS APIs with managed authentication and rate limiting.
- **EventBridge Pipes** — point-to-point wiring: source → *filter* → *enrich* → target, with no Lambda glue code. The answer to “connect a DynamoDB stream to an API with light transformation, without writing a function”.
- **EventBridge Scheduler** — a dedicated, higher-scale scheduling service supporting one-time and recurring schedules with time zones.
**EventBridge vs SNS vs SQS**

|    | **EventBridge** | **SNS** | **SQS** |
| --- | --- | --- | --- |
| Model | Event **router** | Pub/sub **fan-out** | **Queue** |
| Filtering | On the **event payload**, nested JSON | Message attributes (and body) | None |
| Consumers | Rules and targets, many AWS services | Subscriptions | Pollers |
| Replay / archive | ✅ | ❌ | ❌ |
| Schema registry | ✅ | ❌ | ❌ |
| Scheduling | ✅ | ❌ | Delay only |
| Throughput / latency | Good | **Highest / lowest latency** | Very high |
| Ordering | ❌ | FIFO topics | FIFO queues |
| AWS service events | ✅ Native | ❌ | ❌ |

**Recall check — EventBridge**
- **Route events on fields inside the payload** → EventBridge
- **Add consumers without changing the producer** → EventBridge
- **React to an AWS API call or service state change** → EventBridge (default bus)
- **Scheduled / cron trigger** → EventBridge schedule (or EventBridge Scheduler)
- **Replay past events** → EventBridge archive + replay
- **Third-party SaaS events** → partner event bus
- **Call an external API when an event occurs** → API destination
- **One source to one target with filtering and enrichment, no code** → EventBridge Pipes
- **Simple fan-out, lowest latency** → SNS
- **Buffering and load levelling** → SQS
---
