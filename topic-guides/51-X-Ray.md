## AWS X-Ray
Traces a request as it travels through your application and AWS services, helping you find **slow parts, errors, and bottlenecks**
**Why it exists**
In a monolith, a stack trace tells you where time went. In a distributed application — API Gateway to Lambda to DynamoDB to another service — no single log has the whole picture, and “the API is slow” could mean any of five things. X-Ray reconstructs the **full path of one request** across services and shows where the time and the errors were.
**How it works**
```plain text
Application
    |
X-Ray SDK          ←inside your application; collects information about requestd
    |
X-Ray Daemon ← receives that information locally via UDP port 2000 and sends it to AWS
    |
AWS X-Ray Service  ← receives and stores the data, then shows you the trace and service map
```
- **X-Ray SDK** — added to your application code. Creates the trace data, instruments AWS SDK calls, HTTP calls, and SQL queries. **Default behaviour**: 1 request per second, plus 5% of additional requests.
- **X-Ray Daemon** — sits between the SDK and the service, listening on **UDP port 2000**, batching segments and sending them to AWS. It exists so your application doesn’t block on network calls to X-Ray. 
- **X-Ray service** — stores traces and renders the **service map** with latency, error rate, and fault rate per node.
- `ebextensions/xray-daemon.config` is an **Elastic Beanstalk configuration file** that tells Elastic Beanstalk to **enable and run the X-Ray daemon on your EC2 instances** 
Where the daemon runs:
<table header-row="true">
<tr>
<td>Platform</td>
<td>Daemon</td>
</tr>
<tr>
<td>**EC2**</td>
<td>Install and run it on the instance</td>
</tr>
<tr>
<td>**ECS**</td>
<td>Run it as a **sidecar container** in the task</td>
</tr>
<tr>
<td>**Lambda**</td>
<td>**Provided by the Lambda service** — just enable Active tracing</td>
</tr>
</table>
```plain text
ECS Task
  Application Container
        |  UDP 2000
        ↓
  X-Ray Daemon Container (sidecar)
        ↓
  AWS X-Ray
```
**Segments, subsegments, annotations, metadata**
**Segment** — the work done by one service for one request.<br>**Subsegment** — a portion of a segment: a DynamoDB call, an HTTP request, a function you chose to instrument. This is what tells you *which downstream call* was slow.
**Annotations** → extra information/instructions attached to something.
**Metadata** → information that **describes** something.
**Annotations vs metadata** — the distinction exists because indexing costs money and has limits.
<table header-row="true">
<tr>
<td></td>
<td>**Annotation**</td>
<td>**Metadata**</td>
</tr>
<tr>
<td>**Indexed**</td>
<td>✅ **Yes**</td>
<td>❌ No</td>
</tr>
<tr>
<td>Searchable in filter expressions</td>
<td>✅</td>
<td>❌</td>
</tr>
<tr>
<td>Limit</td>
<td>50 per trace</td>
<td>Larger, arbitrary JSON</td>
</tr>
<tr>
<td>Use for</td>
<td>`customerId`, `orderId`, `environment`</td>
<td>Debug payloads, request bodies</td>
</tr>
</table>
So: *“I want to filter traces in the console by customer ID”* → **annotation**, because only indexed fields are searchable. Metadata is for context you’ll read once you’ve found the trace.
**Sampling**
X-Ray does not trace every request — at high volume that would be expensive and slow. **Sampling rules** decide which requests are recorded.
```plain text
100,000 requests
Sampling: 10%
        ↓
10,000 requests traced
```
**Default behaviour**: 1 request per second, plus 5% of additional requests.
**Sampling rules are configured centrally in the X-Ray console and picked up by the SDK at runtime.** That means you can change how much is traced **without redeploying** — which is the answer whenever a question asks how to increase or decrease tracing without a code change.
**Tracing header and error classification**
- <span underline="true">**`X-Amzn-Trace-Id`**</span> is the header that propagates trace context between services. The ALB adds it; the SDK reads and forwards it.
- **Error** = 4xx client error. **Fault** = 5xx server error. **Throttle** = 429. The service map colours nodes by these, so the map alone often identifies the failing component.
**Enabling it — and the classic failure**
Lambda: enable **Active tracing**; the execution role needs **`AWSXRayDaemonWriteAccess`**.<br>API Gateway: enable X-Ray tracing **per stage**.<br>EC2: the instance profile needs `AWSXRayDaemonWriteAccess`, and the daemon must be running.
**The classic exam scenario:** *“X-Ray works when run locally but no traces appear from EC2.”*
Reasoning: locally, the daemon uses your personal credentials, which are broad. On EC2, it uses the **instance profile**. If that role lacks `AWSXRayDaemonWriteAccess`, the daemon runs, receives segments, and silently fails to upload them.
Check, in order:<br>1. Does the EC2 instance have an IAM role with **`AWSXRayDaemonWriteAccess`**?<br>2. Is the X-Ray daemon actually running?<br>3. Does the network configuration permit it to reach X-Ray?
**\`GetGroup\`** → Gets information about an **X-Ray group** (group name, ARN, filter expression). Not for retrieving trace IDs.
**\`GetTraceSummaries\`** → Gets **summaries of traces** and provides their **trace IDs**. Use this to find traces.
**\`BatchGetTraces\`** → Uses **trace IDs** to retrieve the **full traces**, including segments and subsegments.
**\`GetServiceGraph\`** → Gets the **X-Ray service map/graph**, showing how services connect. Not for full traces.
**Recall check — X-Ray**
- **Find a slow microservice / trace a request path / find a bottleneck** → X-Ray
- **Application creates trace data** → X-Ray SDK
- **Send traces from EC2 or ECS** → X-Ray daemon
- **X-Ray daemon port** → **UDP 2000** (not TCP)
- **X-Ray on ECS** → daemon **sidecar container**
- **X-Ray on Lambda** → Active tracing + `AWSXRayDaemonWriteAccess` (no sidecar needed)
- **Works locally, fails on EC2** → **missing IAM role / permissions**
- **Reduce tracing overhead and cost** → sampling
- **Change how much is traced without redeploying** → sampling rules in the console
- **Filter traces by a business value** → annotation (indexed)
- **Attach debug data you don’t need to search** → metadata
- **Find which downstream call is slow** → service map + subsegments
- **Trace header** → `X-Amzn-Trace-Id`
- `ebextensions/xray-daemon.config` is an **Elastic Beanstalk configuration file** that tells Elastic Beanstalk to **enable and run the X-Ray daemon on your EC2 instances**
-  GetTraceSummaries → **find trace IDs**
- BatchGetTraces → **get full traces**
- GetServiceGraph → **service map**
- GetGroup → **group information**
- GetTraceSummaries → BatchGetTraces
---
### Lambda + X-Ray Environment Variables
**Main X-Ray variables to know for the DVA exam:**
<table header-row="true">
<tr>
<td>Variable</td>
<td>Simple meaning</td>
</tr>
<tr>
<td>`_X_AMZN_TRACE_ID`</td>
<td>Contains the **X-Ray trace information** for the current invocation</td>
</tr>
<tr>
<td>`AWS_XRAY_CONTEXT_MISSING`</td>
<td>What X-Ray should do when **trace context is missing**</td>
</tr>
<tr>
<td>`AWS_XRAY_DAEMON_ADDRESS`</td>
<td>Where the **X-Ray daemon** is listening</td>
</tr>
<tr>
<td>`AWS_XRAY_TRACING_NAME`</td>
<td>Sets the **service/segment name**</td>
</tr>
<tr>
<td>`AWS_XRAY_DEBUG_MODE`</td>
<td>Enables **debug-level X-Ray logs**</td>
</tr>
<tr>
<td>`AWS_XRAY_LOG_LEVEL`</td>
<td>Controls X-Ray **log level**</td>
</tr>
<tr>
<td>`AWS_XRAY_SDK_ENABLED`</td>
<td>Enables/disables the X-Ray SDK</td>
</tr>
</table>
AWS documents these as X-Ray SDK/Lambda configuration variables. 
**`_X_AMZN_TRACE_ID`**
Think:
> **"Which X-Ray trace does this Lambda invocation belong to?"**
Contains the tracing information, including the trace ID and sampling information.
**Exam keyword:** **trace ID / tracing header**. 
**`AWS_XRAY_CONTEXT_MISSING`**
Think:
> **"What should X-Ray do if there is no active trace?"**
Common values:
- `LOG_ERROR` → log the error and continue
- `RUNTIME_ERROR` → throw an error
- `IGNORE_ERROR` → ignore it
Lambda sets it to `LOG_ERROR` by default. 
**`AWS_XRAY_DAEMON_ADDRESS`**
Think:
> **"Where is the X-Ray daemon?"**
Tells the X-Ray SDK the daemon's address/port. 
**`AWS_XRAY_TRACING_NAME`**
Think:
> **"What should this service be called in X-Ray?"**
Sets the service/segment name. 
**`AWS_XRAY_DEBUG_MODE`**
Think:
> **"Give me more X-Ray debugging logs."**
`TRUE` enables debug-level SDK logging. **It is not the trace ID.** 
**`AWS_XRAY_LOG_LEVEL`**
Controls the X-Ray SDK log level, such as `debug`, `info`, `warn`, or `error`. 
**`AWS_XRAY_SDK_ENABLED`**
Controls whether the X-Ray SDK is enabled. Setting it to `false` disables the SDK. 
**For your exact TD question:**
> **Which TWO environment variables does Lambda use to facilitate communication with X-Ray?**
Remember:
**`_X_AMZN_TRACE_ID`** → **trace information**
**`AWS_XRAY_CONTEXT_MISSING`** → **what to do when trace context is missing**
Those are the two to select. ([Brainscape](https://www.brainscape.com/flashcards/incorrect-from-test-10300251/packs/17260747?utm_source=chatgpt.com))
**Exam memory**
```plain text
TRACE ID      → _X_AMZN_TRACE_ID
MISSING       → AWS_XRAY_CONTEXT_MISSING
DAEMON        → AWS_XRAY_DAEMON_ADDRESS
SERVICE NAME  → AWS_XRAY_TRACING_NAME
DEBUG LOGS    → AWS_XRAY_DEBUG_MODE
LOG LEVEL     → AWS_XRAY_LOG_LEVEL
SDK ON/OFF    → AWS_XRAY_SDK_ENABLED
```
Note: AWS now also supports **AWS Distro for OpenTelemetry (ADOT)** for Lambda tracing, so don't assume every current Lambda tracing setup uses the X-Ray SDK specifically. 
