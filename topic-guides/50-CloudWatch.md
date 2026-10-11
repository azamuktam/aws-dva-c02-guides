## AWS CloudWatch 
<table header-row="true">
<tr>
<td>Field</td>
<td>Simple meaning</td>
</tr>
<tr>
<td>**namespace**</td>
<td>Who am I calling? `aws` or `remote`</td>
</tr>
<tr>
<td>**http**</td>
<td>Details about an outgoing HTTP request</td>
</tr>
<tr>
<td>**aws**</td>
<td>Details about the AWS service/resource called</td>
</tr>
<tr>
<td>**error**</td>
<td>An error occurred</td>
</tr>
<tr>
<td>**throttle**</td>
<td>Request was throttled</td>
</tr>
<tr>
<td>**fault**</td>
<td>Server-side failure</td>
</tr>
<tr>
<td>**cause**</td>
<td>What caused the error</td>
</tr>
<tr>
<td>**annotations**</td>
<td>Searchable/indexed key-value data</td>
</tr>
<tr>
<td>**metadata**</td>
<td>Extra data stored for debugging; not indexed</td>
</tr>
<tr>
<td>**subsegments**</td>
<td>Smaller downstream operations inside this subsegment</td>
</tr>
<tr>
<td>**precursor_ids**</td>
<td>IDs of earlier related subsegments</td>
</tr>
</table>
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
<table header-row="true">
<tr>
<td>Service</td>
<td>Metric</td>
<td>What it tells you</td>
</tr>
<tr>
<td>**Lambda**</td>
<td>`Invocations`, `Duration`, `Errors`</td>
<td>Basic health</td>
</tr>
<tr>
<td>**Lambda**</td>
<td>**`Throttles`**</td>
<td>Concurrency limit hit (429)</td>
</tr>
<tr>
<td>**Lambda**</td>
<td>`ConcurrentExecutions`</td>
<td>How much concurrency is in use</td>
</tr>
<tr>
<td>**Lambda**</td>
<td>**`IteratorAge`**</td>
<td>Stream consumer falling behind</td>
</tr>
<tr>
<td>**Lambda**</td>
<td>`DeadLetterErrors`</td>
<td>Failed to write to the DLQ</td>
</tr>
<tr>
<td>**API Gateway**</td>
<td>`4XXError` / `5XXError`</td>
<td>Client vs server errors</td>
</tr>
<tr>
<td>**API Gateway**</td>
<td>**`Latency`** vs **`IntegrationLatency`**</td>
<td>Total time vs backend-only time</td>
</tr>
<tr>
<td>**API Gateway**</td>
<td>`CacheHitCount` / `CacheMissCount`</td>
<td>Cache effectiveness</td>
</tr>
<tr>
<td>**API Gateway**</td>
<td>`Count`</td>
<td>Request volume</td>
</tr>
<tr>
<td>**SQS**</td>
<td>`ApproximateNumberOfMessagesVisible`</td>
<td>Backlog — use this for scaling</td>
</tr>
<tr>
<td>**SQS**</td>
<td>`ApproximateAgeOfOldestMessage`</td>
<td>Consumers are too slow</td>
</tr>
<tr>
<td>**DynamoDB**</td>
<td>`ThrottledRequests`, `ConsumedReadCapacityUnits`</td>
<td>Capacity problems</td>
</tr>
<tr>
<td>**DynamoDB**</td>
<td>`SuccessfulRequestLatency`</td>
<td>Table latency</td>
</tr>
<tr>
<td>**Kinesis**</td>
<td>`GetRecords.IteratorAgeMilliseconds`</td>
<td>Consumer lag</td>
</tr>
<tr>
<td>**Kinesis**</td>
<td>`WriteProvisionedThroughputExceeded`</td>
<td>Hot shard or too few shards</td>
</tr>
<tr>
<td>**ECS**</td>
<td>`CPUUtilization`, `MemoryUtilization`</td>
<td>Service load</td>
</tr>
</table>
**The ****`Latency`**** vs ****`IntegrationLatency`**** pair is worth understanding rather than memorising.** `IntegrationLatency` is the time API Gateway spent waiting for your backend. `Latency` is the total time the client experienced. So:
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
- **Alarm stuck in ****`INSUFFICIENT_DATA`** → no data, or wrong “treat missing data”
- **Reduce alert noise** → composite alarm
- **Roll back a deployment automatically** → alarm + CodeDeploy rollback
- **`Latency`**** high, ****`IntegrationLatency`**** low** → the problem is **API Gateway**, not the backend
- **`IntegrationLatency`**** high** → the **backend** is slow
- **Scale workers on queue depth** → `ApproximateNumberOfMessagesVisible`
- **Queue is short but messages are old** → `ApproximateAgeOfOldestMessage`, consumers too slow
- **`Throttles`**** \> 0 on Lambda** → raise concurrency
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
