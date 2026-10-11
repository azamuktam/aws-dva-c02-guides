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

| Platform | Daemon |
| --- | --- |
| **EC2** | Install and run it on the instance |
| **ECS** | Run it as a **sidecar container** in the task |
| **Lambda** | **Provided by the Lambda service** — just enable Active tracing |

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

|    | **Annotation** | **Metadata** |
| --- | --- | --- |
| **Indexed** | ✅ **Yes** | ❌ No |
| Searchable in filter expressions | ✅ | ❌ |
| Limit | 50 per trace | Larger, arbitrary JSON |
| Use for | `customerId`, `orderId`, `environment` | Debug payloads, request bodies |

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

- **`X-Amzn-Trace-Id`** is the header that propagates trace context between services. The ALB adds it; the SDK reads and forwards it.
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

| Variable | Simple meaning |
| --- | --- |
| `_X_AMZN_TRACE_ID` | Contains the **X-Ray trace information** for the current invocation |
| `AWS_XRAY_CONTEXT_MISSING` | What X-Ray should do when **trace context is missing** |
| `AWS_XRAY_DAEMON_ADDRESS` | Where the **X-Ray daemon** is listening |
| `AWS_XRAY_TRACING_NAME` | Sets the **service/segment name** |
| `AWS_XRAY_DEBUG_MODE` | Enables **debug-level X-Ray logs** |
| `AWS_XRAY_LOG_LEVEL` | Controls X-Ray **log level** |
| `AWS_XRAY_SDK_ENABLED` | Enables/disables the X-Ray SDK |

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

# Question patterns

> A distributed application has five possible causes for "the API is slow" and no single log shows the whole path. Which service reconstructs the full request path?
> → **AWS X-Ray**

> Which component inside the application code creates trace data and instruments AWS SDK, HTTP, and SQL calls?
> → **The X-Ray SDK**

> Which component receives trace data locally and batches it to the X-Ray service?
> → **The X-Ray daemon**

> On which port and protocol does the X-Ray daemon listen?
> → **UDP port 2000**

> Why does the X-Ray daemon exist between the SDK and the service?
> → **So the application does not block on network calls to X-Ray**

> A developer runs a container on ECS and needs X-Ray. How should the daemon be deployed?
> → **As a sidecar container in the task**

> How is tracing enabled for Lambda?
> → **Enable Active tracing (the Lambda service provides the daemon — no sidecar)**

> Which IAM policy does a Lambda execution role or EC2 instance profile need to send traces?
> → **`AWSXRayDaemonWriteAccess`**

> X-Ray works locally but no traces appear from EC2. What is the most likely cause?
> → **The instance profile lacks `AWSXRayDaemonWriteAccess`**

> In the "works locally, fails on EC2" scenario, what are the three things to check in order?
> → **IAM role with `AWSXRayDaemonWriteAccess`, daemon running, network reachability to X-Ray**

> Why does X-Ray work locally but fail on EC2 if permissions are missing?
> → **Locally the daemon uses your broad personal credentials; on EC2 it uses the instance profile**

> Which X-Ray construct represents the work done by one service for one request?
> → **A segment**

> Which X-Ray construct identifies which downstream call (DynamoDB, HTTP, SQL) was slow?
> → **A subsegment**

> How does a developer filter traces in the console by customer ID?
> → **Use an annotation (indexed/searchable)**

> What is the difference between annotations and metadata with respect to indexing?
> → **Annotations are indexed and searchable; metadata is not indexed**

> What is the limit on annotations per trace?
> → **50 per trace**

> When should metadata be used instead of annotations?
> → **For debug payloads/request bodies you may read once a trace is found (not searchable)**

> Why does the annotation/metadata distinction exist?
> → **Indexing costs money and has limits**

> At high request volume, why doesn't X-Ray trace every request?
> → **Tracing every request is expensive and slow — sampling is used**

> What is X-Ray's default sampling behaviour?
> → **1 request per second, plus 5% of additional requests**

> If 100,000 requests pass with 10% sampling, how many are traced?
> → **10,000**

> A team must increase how much is traced without redeploying code. How?
> → **Change sampling rules in the X-Ray console — the SDK picks them up at runtime**

> Which header propagates trace context between services?
> → **`X-Amzn-Trace-Id`**

> Which component adds the tracing header on the way in?
> → **The ALB (the SDK reads and forwards it)**

> In X-Ray, what is a 4xx classified as?
> → **Error**

> In X-Ray, what is a 5xx classified as?
> → **Fault**

> In X-Ray, what is a 429 classified as?
> → **Throttle**

> What do the service map node colours represent?
> → **Latency, error rate, and fault rate per node (error/fault/throttle)**

> How is X-Ray tracing enabled on API Gateway?
> → **Per stage**

> Which X-Ray API call returns trace summaries and their trace IDs (used to find traces)?
> → **`GetTraceSummaries`**

> Which X-Ray API call uses trace IDs to retrieve the full traces including segments and subsegments?
> → **`BatchGetTraces`**

> Which X-Ray API call returns the service map/graph?
> → **`GetServiceGraph`**

> Which X-Ray API call returns information about a group (name, ARN, filter expression)?
> → **`GetGroup`**

> What is the correct sequence to find and then retrieve full traces?
> → **`GetTraceSummaries` then `BatchGetTraces`**

> What does `ebextensions/xray-daemon.config` do?
> → **An Elastic Beanstalk config file that enables and runs the X-Ray daemon on EC2 instances**

> A developer needs to reduce tracing overhead and cost. What should they configure?
> → **Sampling (rules)**

> Which two Lambda environment variables are used to facilitate communication with X-Ray?
> → **`_X_AMZN_TRACE_ID` and `AWS_XRAY_CONTEXT_MISSING`**

> Which environment variable contains the trace ID and sampling info for the current Lambda invocation?
> → **`_X_AMZN_TRACE_ID`**

> Which environment variable controls what X-Ray does when trace context is missing?
> → **`AWS_XRAY_CONTEXT_MISSING`**

> A Lambda sets `AWS_XRAY_CONTEXT_MISSING` by default to which value?
> → **`LOG_ERROR`**

> What are the common values of `AWS_XRAY_CONTEXT_MISSING`?
> → **`LOG_ERROR`, `RUNTIME_ERROR`, `IGNORE_ERROR`**

> Which environment variable tells the X-Ray SDK where the daemon is listening?
> → **`AWS_XRAY_DAEMON_ADDRESS`**

> Which environment variable sets the service/segment name?
> → **`AWS_XRAY_TRACING_NAME`**

> Which environment variable enables debug-level X-Ray logs?
> → **`AWS_XRAY_DEBUG_MODE`**

> Which environment variable controls the X-Ray SDK log level?
> → **`AWS_XRAY_LOG_LEVEL`**

> Which environment variable enables or disables the X-Ray SDK?
> → **`AWS_XRAY_SDK_ENABLED`**

> Is `AWS_XRAY_DEBUG_MODE` the same as the trace ID?
> → **No — it enables debug logging, it is not the trace ID**

> A developer wants vendor-neutral tracing that can export to multiple backends. Which AWS option?
> → **AWS Distro for OpenTelemetry (ADOT)**

> How do ADOT and X-Ray differ in backend support?
> → **ADOT is OpenTelemetry-based and multi-backend/vendor-neutral; X-Ray is AWS-focused**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Find slow microservice / bottleneck | **X-Ray** |
| Creates trace data | **X-Ray SDK** |
| Sends traces from EC2/ECS | **X-Ray daemon** |
| Daemon port | **UDP 2000 (not TCP)** |
| ECS daemon | **sidecar container** |
| Lambda tracing | **Active tracing + `AWSXRayDaemonWriteAccess`** |
| Works locally, fails on EC2 | **missing IAM role/permissions** |
| Reduce tracing overhead | **sampling** |
| Change tracing without redeploy | **sampling rules in console (runtime)** |
| Default sampling | **1/sec + 5% additional** |
| Filter traces by business value | **annotation (indexed)** |
| Debug data not searched | **metadata** |
| Annotation limit | **50 per trace** |
| Which downstream call slow | **service map + subsegments** |
| Trace header | **`X-Amzn-Trace-Id`** |
| Error | **4xx** |
| Fault | **5xx** |
| Throttle | **429** |
| Segment | **one service's work for one request** |
| Subsegment | **portion of a segment (downstream call)** |
| API Gateway tracing | **per stage** |
| Find trace IDs | **`GetTraceSummaries`** |
| Get full traces | **`BatchGetTraces`** |
| Service map/graph | **`GetServiceGraph`** |
| Group info | **`GetGroup`** |
| Beanstalk daemon config | **`ebextensions/xray-daemon.config`** |
| Trace ID variable | **`_X_AMZN_TRACE_ID`** |
| Missing context variable | **`AWS_XRAY_CONTEXT_MISSING`** |
| Missing context default | **`LOG_ERROR`** |
| Missing context values | **`LOG_ERROR`/`RUNTIME_ERROR`/`IGNORE_ERROR`** |
| Daemon address variable | **`AWS_XRAY_DAEMON_ADDRESS`** |
| Service name variable | **`AWS_XRAY_TRACING_NAME`** |
| Debug logs variable | **`AWS_XRAY_DEBUG_MODE`** |
| Log level variable | **`AWS_XRAY_LOG_LEVEL`** |
| SDK on/off variable | **`AWS_XRAY_SDK_ENABLED`** |
| Trace pieces | **segments / subsegments** |
| Visualisation | **service map** |
| Vendor-neutral tracing | **ADOT (OpenTelemetry)** |
| Multi-backend export | **ADOT / OpenTelemetry** |
