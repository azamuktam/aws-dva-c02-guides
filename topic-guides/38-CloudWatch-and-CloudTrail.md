## AWS CloudWatch

| Field | Simple meaning |
| --- | --- |
| **namespace** | Who am I calling? `aws` or `remote` |
| **http** | Details about an outgoing HTTP request |
| **aws** | Details about the AWS service/resource called |
| **error** | An error occurred |
| **throttle** | Request was throttled |
| **fault** | Server-side failure |
| **cause** | What caused the error |
| **annotations** | Searchable/indexed key-value data |
| **metadata** | Extra data stored for debugging; not indexed |
| **subsegments** | Smaller downstream operations inside this subsegment |
| **precursor_ids** | IDs of earlier related subsegments |

**Metric:** A measurement tracked over time, such as CPU usage.

**Namespace:** Who am I calling? `aws` or `remote`

**Dimensions:** Identify what the metric belongs to, such as an EC2 instance ID.

**Resolution:**

- **Standard:** 1-minute
- **High resolution:** 1-second

**EC2 monitoring:**

- **Basic monitoring:** Every 5 minutes
- **Detailed monitoring:** Every 1 minute

**\`CloudWatchAgentServerPolicy\`** → For EC2 instances running the CloudWatch Agent. Allows the agent to **publish metrics/logs** to CloudWatch. ✅

**\`CloudWatchAgentAdminPolicy\`** → For **installing/configuring/managing** the CloudWatch Agent, with additional **Systems Manager/Parameter Store** permissions.

**Default EC2 metrics:** CPU, network, disk I/O, and status checks.

**Not available by default:** Memory usage, disk space used, and swap.

**Why?** AWS can monitor the EC2 instance from outside, but it cannot see how the operating system uses its memory or disk space.

**Solution:** Install the **CloudWatch Agent** inside the EC2 instance to collect these OS-level metrics.

**Exam memory:** **Detailed monitoring = more frequent data, NOT more types of metrics.**

**Custom metrics**

- **`PutMetricData`** — an API operation used to **publish custom metric data to CloudWatch**; set `StorageResolution: 1` for **1-second high-resolution metrics**
- Use dimensions to segment (per environment, per customer, per function).
- **Metric math** combines metrics into expressions; **anomaly detection** builds an expected band. Both can drive alarms.

**`SetAlarmState`** — temporarily changes a CloudWatch alarm to `ALARM`, `OK`, or `INSUFFICIENT_DATA` **for testing**, so you can verify that its configured actions (e.g. SNS notification) work. The metric alarm normally returns to its real state after the next evaluation.

**Retention** is tiered automatically: 1-second data for 3 hours, 1-minute for 15 days, 5-minute for 63 days, 1-hour for 15 months.

**Logs**

```plain text
Log Group          e.g. /aws/lambda/my-function
   └── Log Stream  one per execution environment / instance
        └── Log Events
```

**Retention defaults to “Never expire”.** That is a cost trap on every account — logs accumulate forever unless you set retention (1 day to 10 years) on the **log group**.

**Metric filters** — match a pattern in log events and publish a **metric** from it. Since alarms watch metrics and not logs, this is the only way to alarm on log *content*: count `ERROR` lines, then alarm when the count exceeds a threshold.

**Critical property: metric filters are not retroactive.** They apply only to events that arrive after the filter is created. If you create a filter and see nothing, that is expected behaviour, not a misconfiguration.

**Subscription filters** — stream matching log events in near real time to **Lambda, Kinesis Data Streams, Data Firehose, or OpenSearch**. This is how you get logs out of CloudWatch into a search platform or a custom processor.

**Logs Insights** — a purpose-built query language for searching and aggregating logs interactively.

```plain text
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() by bin(5m)
| sort @timestamp desc
| limit 20
```

Interactive and ad-hoc — not a streaming mechanism. Use it to *investigate*; use subscription filters to *pipe*.

**Embedded Metric Format (EMF)** — write specially structured JSON to your logs and CloudWatch automatically extracts metrics from it. For Lambda this is significantly better than `PutMetricData`, because it needs no API call, adds no latency to the invocation, and cannot fail independently of the log write.

**Other:** **export to S3** for cheap long-term archive; **Live Tail** to watch events stream in real time.

**Why Lambda logs go missing:** the function’s **execution role** must allow `logs:CreateLogGroup`, `logs:CreateLogStream`, and `logs:PutLogEvents` — exactly what the managed policy **`AWSLambdaBasicExecutionRole`** grants. A function with no logs at all almost always has a role problem, not a code problem.

**Alarms**

- **States:** `OK`, `ALARM`, `INSUFFICIENT_DATA`.
- **Actions:** SNS notification, Auto Scaling policy, EC2 action (stop / terminate / reboot / recover), Systems Manager action.
- **Configuration:** period, evaluation periods, datapoints to alarm, and **“treat missing data as”** (`missing`, `notBreaching`, `breaching`, `ignore`).
- **Composite alarms** combine alarms with AND/OR to cut noise — e.g. only page someone if latency *and* error rate are both bad.
- Alarms can watch **metric math** expressions and **anomaly detection** bands.
- **Alarms are the trigger for CodeDeploy automatic rollback**, which is the main deployment-domain reason to care about them.

**Dashboards and cross-account observability**

- **Dashboards** are collections of widgets (graphs, numbers, logs tables, alarm status). They can display metrics from **multiple Regions on one dashboard**, which is the only straightforward way to see a multi-Region application at a glance.
- **Cross-account observability** lets a **monitoring account** view metrics, logs, and traces from linked **source accounts** without switching roles. Set up one monitoring account, link the source accounts, and its dashboards and Logs Insights queries span all of them. This is the answer to *“a central team needs to see metrics and logs across every account”* — the alternative, replicating metrics with custom code, is the wrong answer.
- Dashboards can be shared publicly or with specific people via a link, without giving them console access.

An alarm stuck in `INSUFFICIENT_DATA` means no datapoints are arriving — either the metric isn’t being published, or the “treat missing data” setting is wrong for a sparse metric.

**Metrics worth recognising by name**

| Service | Metric | What it tells you |
| --- | --- | --- |
| **Lambda** | `Invocations`, `Duration`, `Errors` | Basic health |
| **Lambda** | **`Throttles`** | Concurrency limit hit (429) |
| **Lambda** | `ConcurrentExecutions` | How much concurrency is in use |
| **Lambda** | **`IteratorAge`** | Stream consumer falling behind |
| **Lambda** | `DeadLetterErrors` | Failed to write to the DLQ |
| **API Gateway** | `4XXError` / `5XXError` | Client vs server errors |
| **API Gateway** | **`Latency`** vs **`IntegrationLatency`** | Total time vs backend-only time |
| **API Gateway** | `CacheHitCount` / `CacheMissCount` | Cache effectiveness |
| **API Gateway** | `Count` | Request volume |
| **SQS** | `ApproximateNumberOfMessagesVisible` | Backlog — use this for scaling |
| **SQS** | `ApproximateAgeOfOldestMessage` | Consumers are too slow |
| **DynamoDB** | `ThrottledRequests`, `ConsumedReadCapacityUnits` | Capacity problems |
| **DynamoDB** | `SuccessfulRequestLatency` | Table latency |
| **Kinesis** | `GetRecords.IteratorAgeMilliseconds` | Consumer lag |
| **Kinesis** | `WriteProvisionedThroughputExceeded` | Hot shard or too few shards |
| **ECS** | `CPUUtilization`, `MemoryUtilization` | Service load |

**The `Latency` vs `IntegrationLatency` pair is worth understanding rather than memorising.** `IntegrationLatency` is the time API Gateway spent waiting for your backend. `Latency` is the total time the client experienced. So:

- Both high → the **backend** is slow.
- `Latency` high, `IntegrationLatency` low → the extra time was spent **in API Gateway** (authorizer, mapping template, throttling queue), not in your code.

That single subtraction turns a vague “the API is slow” scenario into a precise answer.

### CloudWatch: Metric vs Alarm vs Event

- **Metric** → A value measured over time, such as EC2 CPU utilization.
- **Alarm** → Watches a metric and takes action when it crosses a defined threshold.
- **Event** → A notification about something that happened, such as an alarm changing from `OK` to `ALARM`; EventBridge can react to it.

**Simple flow:** **Metric → Alarm detects problem → Event triggers action**. 

**Recall check — CloudWatch**

- **Memory or disk-space metrics missing** → install the **CloudWatch agent** (custom metric)
- **Do NOT answer “enable detailed monitoring”** for a missing memory metric
- **1-minute EC2 metrics** → detailed monitoring
- **1 second** — High-resolution custom metrics in CloudWatch can have a minimum resolution of **1 second**.
- **High-resolution metric:** 1-second resolution
- **High-resolution alarm:** **10, 20, or 30 seconds**
- **Regular alarm:** **60 seconds or any multiple of 60** 
- **Publish an application metric** → `PutMetricData`
- **Publish metrics from Lambda with no API call** → **EMF**
- **Alarm when a string appears in logs** → **metric filter** + alarm
- **Metric filter shows nothing for old logs** → filters are **not retroactive**
- **Search and aggregate logs interactively** → **Logs Insights**
- **Stream logs to another system** → **subscription filter**
- **Logs cost too much / never expire** → set log group **retention**
- **Archive logs cheaply** → export to S3
- **Lambda produces no logs** → execution role missing `logs:*` (`AWSLambdaBasicExecutionRole`)
- **Notify someone on a threshold** → alarm + SNS
- **Alarm stuck in `INSUFFICIENT_DATA`** → no data, or wrong “treat missing data”
- **Reduce alert noise** → composite alarm
- **Roll back a deployment automatically** → alarm + CodeDeploy rollback
- **`Latency` high, `IntegrationLatency` low** → the problem is **API Gateway**, not the backend
- **`IntegrationLatency` high** → the **backend** is slow
- **Scale workers on queue depth** → `ApproximateNumberOfMessagesVisible`
- **Queue is short but messages are old** → `ApproximateAgeOfOldestMessage`, consumers too slow
- **`Throttles` \> 0 on Lambda** → raise concurrency
- **\`CloudWatchAgentServerPolicy\`** → For EC2 instances running the CloudWatch Agent. Allows the agent to **publish metrics/logs** to CloudWatch.
- **\`CloudWatchAgentAdminPolicy\`** → For **installing/configuring/managing** the CloudWatch Agent, with additional **Systems Manager/Parameter Store** permissions.

---

### CloudWatch Alarm Evaluation

**Period** = how long each data point represents.

**Evaluation Periods** = how many recent data points CloudWatch checks.

**Datapoints to Alarm** = how many of those must breach the threshold.

**Example: 3 consecutive breaches**

```plain text
Period = 1 minute
Evaluation Periods = 3
Datapoints to Alarm = 3
```

```plain text
10 min → ❌
11 min → ❌
12 min → ❌
             ↓
           ALARM
```

All **3 of the latest 3** must breach.

**M out of N**

```plain text
Evaluation Periods = N
Datapoints to Alarm = M
```

Example:

```plain text
2 out of 3

Evaluation Periods = 3
Datapoints to Alarm = 2
```

Means **at least 2 of the latest 3** must breach. They **do not have to be consecutive**.

**Period ≠ Evaluation Periods**

```plain text
Period = 3 minutes
```

means **each data point covers 3 minutes**.

It does **not** mean “check 3 data points.”

**High-resolution:** CloudWatch can use 10-, 20-, or 30-second periods, but high-resolution alarms cost more.

**Exam memory:**

> **Period = size of each data point**

  **Evaluation Periods = how many to look at**

  **Datapoints to Alarm = how many must breach**

  **3 consecutive → 3 / 3**

  **2 of 3 → 2 / 3**

## AWS CloudTrail

**Why it exists**

CloudWatch tells you what your application did. CloudTrail tells you what was done **to AWS itself** — every API call, from the console, CLI, SDKs, and AWS services acting on your behalf.

It answers: **who did what, when, and from where.**

```plain text
Any AWS API call
      ↓
CloudTrail
      ↓
Event history (90 days, free)  or  a Trail → S3 (+ optionally CloudWatch Logs)
```

**What to know**

- Records API calls with the identity, timestamp, source IP address, request parameters, and response elements.
- Logs actions from **Console, CLI, SDK, and AWS services**.
- **Event history** covers the last 90 days of management events for free. For events older than 90 days, use **Athena** to analyze **CloudTrail** logs stored in the** S3 bucket**
- **Management events** (control-plane operations, like creating a role) are logged by default. **Data events** (S3 object-level, Lambda invocations) are high-volume and must be explicitly enabled.
- **Insights** detects unusual API activity patterns.
- Multi-Region and organisation-wide trails are supported.
- **Log file validation** proves the logs weren’t tampered with.
- Athena is the usual way to query CloudTrail logs in S3.

Applies to: IAM, EC2, Lambda, S3, CloudFormation, KMS — effectively everything.

**The three-way distinction, one more time**

| Tool | Question |
| --- | --- |
| **CloudTrail** | *“Who made the change?”* |
| **AWS Config** | *“What is the current configuration, and is it compliant?”* |
| **CloudFormation Drift Detection** | *“Does it still match my template?”* |

Worked example — someone changes a Lambda IAM role:

- **CloudTrail** → identifies which user or role changed it, and when
- **AWS Config** → checks whether the role now violates a rule
- **Drift Detection** → checks whether it differs from the stack template
- **CloudTrail Insights** — detects **unusual API call rates or API error rates** by comparing current activity with your normal baseline

**Recall check — CloudTrail**

- **“Who changed this?” / audit logs / API activity** → CloudTrail
- **Investigate changes / security auditing** → CloudTrail
- **Long-term retention of API logs** → create a Trail → S3
- **Alarm on specific API activity** → Trail → CloudWatch Logs → metric filter → alarm
- **S3 object-level or Lambda invocation logging** → enable **data events**
- **Event history without a trail** → last 90 days, management events only
- **Query CloudTrail logs with SQL** → Athena
- **CloudTrail = who. Config = compliance. Drift = matches the template.**
- **CloudTrail Insights** — detects **unusual API call rates or API error rates** by comparing current activity with your normal baseline

# Question patterns

> A developer needs to publish an application-specific metric such as number of orders processed. Which API operation should the code call?
> → **`PutMetricData` (custom metric)**

> A custom metric must have 1-second resolution. What must be set when publishing?
> → **`StorageResolution: 1`**

> A team wants to segment a custom metric per customer and per environment. What should they attach?
> → **Dimensions**

> An alarm must fire when multiple underlying metrics breach together. Which feature builds the alarm expression?
> → **Metric math**

> A developer wants an alarm that fires only when behaviour deviates from an automatically learned expected band. Which feature?
> → **Anomaly detection**

> A team wants to test that an alarm's SNS action works without waiting for a real breach. What can they call?
> → **`SetAlarmState` to temporarily force the alarm to ALARM/OK/INSUFFICIENT_DATA**

> How long is high-resolution (1-second) CloudWatch metric data retained?
> → **3 hours**

> How long is 1-minute CloudWatch metric data retained?
> → **15 days**

> How long is 5-minute CloudWatch metric data retained?
> → **63 days**

> How long is 1-hour CloudWatch metric data retained?
> → **15 months**

> An EC2 instance's memory usage and disk space are missing from CloudWatch. What should the developer do?
> → **Install the CloudWatch Agent to collect OS-level metrics as custom metrics**

> Why are memory and disk-space metrics not available for EC2 by default?
> → **AWS monitors EC2 from outside and cannot see OS-level memory/disk usage**

> A developer is told memory metrics are missing and considers enabling detailed monitoring. Why is that wrong?
> → **Detailed monitoring = more frequent data, not more metric types; use the CloudWatch Agent**

> Basic EC2 monitoring publishes metrics at what interval?
> → **Every 5 minutes**

> Detailed EC2 monitoring publishes metrics at what interval?
> → **Every 1 minute**

> Which IAM managed policy lets the CloudWatch Agent on an EC2 instance publish metrics and logs?
> → **`CloudWatchAgentServerPolicy`**

> Which managed policy is needed to install, configure, and manage the CloudWatch Agent, including Systems Manager/Parameter Store access?
> → **`CloudWatchAgentAdminPolicy`**

> A developer must raise an alarm when a specific string appears in log data. What must they create first?
> → **A metric filter (then an alarm on the emitted metric)**

> A metric filter is created for ERROR lines but returns no data for events written the previous day. Why?
> → **Metric filters are not retroactive — they only apply to events arriving after creation**

> The team needs to stream matching log events in near real time to Lambda, Kinesis Data Streams, Firehose, or OpenSearch. Which feature?
> → **Subscription filters**

> A developer needs to interactively search and aggregate logs with a query language. Which feature?
> → **CloudWatch Logs Insights**

> Is Logs Insights a streaming mechanism or an investigation tool?
> → **Investigation tool — use subscription filters to pipe logs onward**

> Log storage costs keep rising because logs never expire. What should be set, and where?
> → **Set retention (1 day to 10 years) on the log group**

> What is the default retention for a CloudWatch log group?
> → **Never expire**

> A team wants the cheapest long-term archive of logs. What should they do?
> → **Export logs to S3**

> A Lambda function produces no logs at all. What is the most likely cause?
> → **The execution role is missing `logs:*` permissions (AWSLambdaBasicExecutionRole)**

> Which managed policy grants exactly `logs:CreateLogGroup`, `logs:CreateLogStream`, and `logs:PutLogEvents`?
> → **`AWSLambdaBasicExecutionRole`**

> Logs are grouped as which hierarchy?
> → **Log group → log stream → log events**

> A developer wants to publish custom metrics from Lambda without any API call or added latency. Which approach?
> → **Embedded Metric Format (EMF) — write structured JSON to logs**

> Why is EMF better than `PutMetricData` for Lambda metrics?
> → **No API call, no added latency, cannot fail independently of the log write**

> An alarm shows `INSUFFICIENT_DATA`. What does this mean?
> → **No datapoints are arriving, or the "treat missing data" setting mis-handles a sparse metric**

> A metric is published sparsely (only when traffic occurs). Which alarm setting controls how missing points are treated?
> → **"Treat missing data as" (`missing`, `notBreaching`, `breaching`, `ignore`)**

> A team is drowning in alert noise and only wants to page when latency AND error rate are both bad. Which feature?
> → **Composite alarms (AND/OR combinations)**

> Which CloudWatch feature is the trigger for CodeDeploy automatic rollback?
> → **Alarms**

> A developer wants one dashboard showing metrics from multiple Regions. What should they use?
> → **CloudWatch Dashboards (can display multiple Regions on one dashboard)**

> A central team must view metrics, logs, and traces across many AWS accounts without switching roles. Which feature?
> → **CloudWatch cross-account observability (monitoring account + linked source accounts)**

> What is the wrong answer to the central cross-account observability requirement?
> → **Replicating metrics with custom code**

> API Gateway `Latency` is high but `IntegrationLatency` is low. Where is the delay?
> → **In API Gateway (authorizer, mapping template, throttling), not the backend**

> API Gateway `Latency` and `IntegrationLatency` are both high. Where is the problem?
> → **The backend**

> Which API Gateway metric measures backend-only time?
> → **`IntegrationLatency`**

> Lambda `Throttles` is greater than zero. What is happening and what should be done?
> → **Concurrency limit hit (429) — raise concurrency**

> Which Lambda metric indicates a stream consumer is falling behind?
> → **`IteratorAge`**

> Which Lambda metric counts failures to write to the DLQ?
> → **`DeadLetterErrors`**

> A worker fleet should scale on queue depth. Which SQS metric?
> → **`ApproximateNumberOfMessagesVisible`**

> An SQS queue looks short but messages are old. Which metric, and what does it reveal?
> → **`ApproximateAgeOfOldestMessage` — consumers are too slow**

> Which Kinesis metric measures consumer lag?
> → **`GetRecords.IteratorAgeMilliseconds`**

> Which Kinesis metric indicates a hot shard or too few shards?
> → **`WriteProvisionedThroughputExceeded`**

> Which DynamoDB metrics point to capacity problems?
> → **`ThrottledRequests`, `ConsumedReadCapacityUnits`**

> What is the minimum resolution of a high-resolution CloudWatch custom metric?
> → **1 second**

> What period values can a high-resolution alarm use?
> → **10, 20, or 30 seconds**

> What period values can a regular alarm use?
> → **60 seconds or any multiple of 60**

> An alarm uses Period = 1 minute, Evaluation Periods = 3, Datapoints to Alarm = 3. What does this require?
> → **All 3 of the latest 3 data points must breach (3 consecutive)**

> An alarm uses Evaluation Periods = 3 and Datapoints to Alarm = 2. What does this require?
> → **At least 2 of the latest 3 data points must breach — not necessarily consecutive**

> An alarm sets Period = 3 minutes. Does that mean "check 3 data points"?
> → **No — it means each data point covers 3 minutes**

> Which three states can a CloudWatch alarm be in?
> → **`OK`, `ALARM`, `INSUFFICIENT_DATA`**

> Which alarm actions can CloudWatch trigger?
> → **SNS notification, Auto Scaling policy, EC2 action (stop/terminate/reboot/recover), Systems Manager action**

> Which AWS service records who made an API call to AWS, when, and from where?
> → **AWS CloudTrail**

> A security team needs to know which user changed an IAM role. Which service?
> → **CloudTrail**

> How long does CloudTrail Event History retain management events for free, with no trail?
> → **90 days**

> A developer must analyze CloudTrail logs stored in S3 with SQL for events older than 90 days. Which service?
> → **Amazon Athena**

> S3 object-level access and Lambda invocation logging are not captured by default. What must be enabled?
> → **CloudTrail data events**

> Which CloudTrail event type (management vs data) is logged by default?
> → **Management events (control-plane operations)**

> A developer wants to alarm on specific API activity. What is the correct pipeline?
> → **CloudTrail → CloudWatch Logs → metric filter → alarm**

> How can a team retain API audit logs long-term?
> → **Create a Trail that delivers to S3 (optionally CloudWatch Logs)**

> Which CloudTrail feature detects unusual API call rates or error rates against a normal baseline?
> → **CloudTrail Insights**

> Which CloudTrail feature proves log files were not tampered with?
> → **Log file validation**

> Someone changes a Lambda IAM role. Which service identifies who made the change?
> → **CloudTrail**

> Someone changes a Lambda IAM role. Which service checks whether it now violates a rule?
> → **AWS Config**

> Someone changes a Lambda IAM role. Which service checks whether it still matches the CloudFormation template?
> → **CloudFormation Drift Detection**

> Which CloudTrail sources of API activity are logged?
> → **Console, CLI, SDK, and AWS services acting on your behalf**

> What does CloudWatch tell you versus CloudTrail?
> → **CloudWatch = what your application did; CloudTrail = what was done to AWS itself**

> Which tool answers "is the number too high or too low"?
> → **CloudWatch Metrics**

> Which tool answers "what did the application say"?
> → **CloudWatch Logs**

> Which tool answers "which step in the chain was slow or failed"?
> → **X-Ray**

> Which tool answers "who made this change to AWS itself"?
> → **CloudTrail**

> What is the simple flow linking these CloudWatch concepts?
> → **Metric → Alarm detects problem → Event triggers action**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Metric | **a value measured over time (e.g. CPU)** |
| Namespace | **who am I calling — `aws` or `remote`** |
| Dimensions | **identify what the metric belongs to (e.g. instance ID)** |
| Standard resolution | **1 minute** |
| High resolution | **1 second** |
| Basic EC2 monitoring | **every 5 minutes** |
| Detailed EC2 monitoring | **every 1 minute** |
| Detailed monitoring | **more frequent data, NOT more metric types** |
| Missing memory/disk metrics | **install CloudWatch Agent** |
| Wrong answer for memory | **"enable detailed monitoring"** |
| Agent publish policy | **`CloudWatchAgentServerPolicy`** |
| Agent admin policy | **`CloudWatchAgentAdminPolicy`** |
| Publish custom metric | **`PutMetricData`** |
| 1-second custom metric | **`StorageResolution: 1`** |
| Segment metrics | **dimensions** |
| Combine metrics | **metric math** |
| Expected band | **anomaly detection** |
| Force alarm for testing | **`SetAlarmState`** |
| Retention 1s | **3 hours** |
| Retention 1m | **15 days** |
| Retention 5m | **63 days** |
| Retention 1h | **15 months** |
| Log hierarchy | **log group → log stream → log events** |
| Default log retention | **never expire** |
| Log cost fix | **set retention on the log group** |
| Archive logs cheaply | **export to S3** |
| Alarm on log content | **metric filter + alarm** |
| Metric filters | **not retroactive** |
| Stream logs out | **subscription filter** |
| Search/aggregate logs | **Logs Insights** |
| Lambda metrics no API call | **EMF** |
| Lambda missing logs | **execution role lacks `logs:*`** |
| Lambda logs policy | **`AWSLambdaBasicExecutionRole`** |
| Live stream logs | **Live Tail** |
| Alarm states | **`OK`, `ALARM`, `INSUFFICIENT_DATA`** |
| Alarm actions | **SNS, Auto Scaling, EC2 action, Systems Manager** |
| Missing data setting | **`missing`/`notBreaching`/`breaching`/`ignore`** |
| Reduce alert noise | **composite alarm** |
| Auto rollback trigger | **alarm + CodeDeploy** |
| Multi-Region view | **dashboard** |
| Central cross-account view | **cross-account observability (monitoring account)** |
| Alarm `INSUFFICIENT_DATA` | **no data / wrong treat-missing-data** |
| Latency high, IntegrationLatency low | **problem is API Gateway** |
| IntegrationLatency high | **backend is slow** |
| Lambda `Throttles` | **concurrency limit (429)** |
| Lambda `IteratorAge` | **stream consumer behind** |
| Scale on queue depth | **`ApproximateNumberOfMessagesVisible`** |
| Old messages in short queue | **`ApproximateAgeOfOldestMessage`** |
| Kinesis lag | **`GetRecords.IteratorAgeMilliseconds`** |
| High-res alarm periods | **10/20/30 seconds** |
| Regular alarm periods | **60s or multiple of 60** |
| 3 consecutive | **3 / 3** |
| 2 of 3 | **2 / 3** |
| Period | **size of each data point** |
| Evaluation Periods | **how many data points to look at** |
| Datapoints to Alarm | **how many must breach** |
| Who changed it | **CloudTrail** |
| Compliance/config history | **AWS Config** |
| Matches template | **Drift Detection** |
| CloudTrail Event History | **90 days, management events, free** |
| Query CloudTrail in S3 | **Athena** |
| S3 object / Lambda logging | **CloudTrail data events** |
| Unusual API activity | **CloudTrail Insights** |
| Tamper-proof logs | **log file validation** |
| Alarm on API activity | **Trail → CloudWatch Logs → metric filter → alarm** |
| Metric vs Alarm vs Event | **value / watches metric / notification of a change** |
