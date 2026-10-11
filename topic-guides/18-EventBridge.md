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

# Question patterns

> An application must react to AWS service events such as EC2 instance state changes. Which service and bus?
> → **Amazon EventBridge on the default bus**

> A serverless cron job must run a Lambda every hour. Which feature?
> → **EventBridge Scheduler (cron/rate expression)**

> After a bug fix, last Tuesday's events must be reprocessed exactly as they flowed. Which feature?
> → **EventBridge archive + replay**

> Multiple consumers each need the same event, filtered by its content. Which service?
> → **EventBridge — rule-based routing with event pattern matching**

> Events must be routed based on nested fields inside the payload. Which service?
> → **EventBridge (event pattern)**

> New consumers must be added without any change to the producer. Which service?
> → **EventBridge**

> A third-party SaaS provider such as Datadog or Zendesk must publish events into your account. Which bus?
> → **a partner event bus**

> An external HTTP/SaaS API must be called with managed authentication when an event occurs. Which feature?
> → **EventBridge API destinations**

> One source must feed one target with light filtering and enrichment, and no Lambda glue code. Which feature?
> → **EventBridge Pipes**

> A DynamoDB stream must be connected to an API with a small transformation, without writing a function. Which feature?
> → **EventBridge Pipes**

> Simple fan-out with the lowest latency is all that is needed. Which service?
> → **SNS**

> Buffering and load levelling are needed. Which service?
> → **SQS**

> Which bus automatically receives events from AWS services?
> → **the default bus**

> Which bus is used for your own application's events?
> → **a custom bus**

> How many targets can a single EventBridge rule send to?
> → **up to 5**

> A rule is triggered by which two kinds of thing?
> → **an event pattern or a schedule**

> Which matching operators does an event pattern support?
> → **prefix, suffix, numeric comparison, anything-but, exists, IP matching, wildcards**

> What does the schema registry and discovery do?
> → **infers schemas from events on a bus and generates code bindings**

> What is the input transformer for?
> → **reshaping the event before it reaches the target**

> Which messaging service in the stack can archive and replay events?
> → **EventBridge (nothing else can replay)**

> Does EventBridge support ordering?
> → **No**

> Which service natively receives AWS service events?
> → **EventBridge**

> What is the core difference between EventBridge and SNS?
> → **EventBridge is content-based routing where consumers declare interest; SNS is publisher-decided fan-out**

> Give an example of an EventBridge rate expression.
> → **rate(5 minutes)**

> Give an example of an EventBridge cron expression.
> → **cron(0 12 * * ? *)**

> What did EventBridge replace for scheduled rules?
> → **CloudWatch Events scheduled rules (now EventBridge schedule / Scheduler)**

> In the post-office analogy, what plays the role of the post office?
> → **EventBridge**

> At what granularity are retry policies and DLQs configured?
> → **per target**

> What two capabilities do API destinations provide for external calls?
> → **managed authentication and rate limiting**

> What stages make up the flow inside EventBridge Pipes?
> → **source → filter → enrich → target**

> What does EventBridge Scheduler add beyond simple rules?
> → **one-time and recurring schedules with time zones**

> Does an EventBridge event pattern match nested fields in the payload?
> → **Yes (for example, fields under detail)**

> Which service should be used when ordering is required: EventBridge or SNS FIFO?
> → **SNS FIFO**

> In EventBridge, who declares what they care about?
> → **the consumers, via rules**

# Pocket card

| Keyword | Answer |
| --- | --- |
| AWS service events | **EventBridge (default bus)** |
| Add consumers, no producer change | **EventBridge** |
| React to an AWS API call / state change | **EventBridge default bus** |
| Scheduled / cron trigger | **EventBridge Scheduler** |
| Replay past events | **archive + replay** |
| SaaS events | **partner event bus** |
| Call an external API | **API destination** |
| Source → target with filtering/enrichment, no code | **EventBridge Pipes** |
| Simple fan-out, lowest latency | **SNS** |
| Buffering / load levelling | **SQS** |
| Model | **event router** |
| Bus types | **default, custom, partner** |
| Own application events | **custom bus** |
| Rule triggers | **event pattern or schedule** |
| Pattern operators | **prefix, suffix, numeric, anything-but, exists, IP, wildcards** |
| Nested payload matching | **yes (event pattern)** |
| Targets per rule | **up to 5** |
| rate example | **rate(5 minutes)** |
| cron example | **cron(0 12 * * ? *)** |
| Input transformer | **reshape the event** |
| Pipes stages | **source → filter → enrich → target** |
| DLQ / retry granularity | **per target** |
| Scheduler supports | **one-time + recurring + time zones** |
| API destination auth | **managed auth + rate limiting** |
| Replay capability | **EventBridge only** |
| Schema registry | **EventBridge only** |
| Scheduling | **EventBridge (SNS/SQS no)** |
| Ordering | **EventBridge has none** |
| Throughput / latency | **SNS highest / lowest latency** |
| AWS service events natively | **EventBridge** |
| Post-office role | **EventBridge** |
| Content-based routing | **distinguishes EventBridge from SNS** |
| Event sources | **AWS services, your apps, SaaS partners** |
| Distinguished by | **payload content matching vs SNS attributes** |
| Replaced by EventBridge schedule | **CloudWatch Events scheduled rule** |
