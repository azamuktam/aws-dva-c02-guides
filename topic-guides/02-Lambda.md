*PART 1 — COMPUTE: HOW YOUR CODE RUNS*

The exam’s compute question is always the same underneath: *how much of the machine do you want to be responsible for?* Lambda means none. Fargate means the container but not the host. ECS on EC2 means the host too. Elastic Beanstalk means you hand over a code bundle and AWS assembles the whole stack. EC2 means all of it.

Every “which compute service” question is answered by finding the constraint in the scenario that rules the others out — runtime longer than 15 minutes rules out Lambda, needing OS-level control rules out Fargate, and so on.

---

## AWS Lambda

**Why it exists**

Running a server to handle occasional work is wasteful and operationally expensive: you pay for idle time, patch the OS, and scale it yourself. Lambda inverts this — you supply a function, AWS supplies an execution environment on demand, and you pay per millisecond of actual execution. The cost of this trade is that you no longer control the environment’s lifecycle, and almost every Lambda exam question is about a consequence of that loss of control.

**How it works — the execution environment**

This is the single most important mechanism in the exam. Understand it and cold starts, `/tmp` behaviour, connection reuse, and concurrency all follow.

```plain text
Request arrives, no warm environment available
        ↓
COLD START:
  1. Download the deployment package
  2. Start the runtime (Node, Python, JVM...)
  3. Run your INIT code (everything outside the handler)
        ↓
  4. Run the HANDLER
        ↓
Environment is FROZEN, not destroyed
        ↓
Next request arrives soon → environment is THAWED
        ↓
WARM START: only step 4 runs again
```

Three consequences, all testable:

**Anything outside the handler runs once per environment, not once per request.** So a database connection or SDK client created outside the handler is reused across many invocations. This is the standard performance answer.

```python
# Outside the handler — runs ONCE per environment (init phase)
db = create_connection()
s3 = boto3.client("s3")

def lambda_handler(event, context):
    # Runs on EVERY invocation
    return db.query(event["id"])
```

**`/tmp` persists across invocations in the same environment.** Useful as a scratch cache. But *not* reliable storage — you cannot control which environment handles a request, and environments are eventually discarded. If the question says “persistent” or “shared”, the answer is S3, DynamoDB, or EFS, never `/tmp`.

**Don't memorize it as a cold-start optimization.** It can **reduce work on warm invocations**, but it doesn't prevent or eliminate cold starts.

**Global/static variables leak between requests.** This is why idempotency and request-scoped state matter.

**Why concurrency works the way it does:** each environment handles exactly one request at a time. Ten simultaneous requests need ten environments. That is the whole basis of the concurrency formula.

**Concurrency — deriving the number rather than memorising it**

If each environment serves one request at a time, and each request occupies an environment for the duration of the function, then:

```plain text
Concurrency = requests per second × average duration in seconds
```

Worked example:

```plain text
50 requests/second, 100 seconds execution time

Concurrency = 50 × 100 = 5,000 concurrent executions

Default account limit = 1,000
→ This function WILL be throttled.
```

The default account concurrency limit is **1,000** (a soft limit — raise it via support). When you exceed it, Lambda returns **`TooManyRequestsException` (HTTP 429)** and the CloudWatch **`Throttles`** metric increments.

**Reserved vs Provisioned concurrency — two different problems**

Candidates confuse these constantly because both have “concurrency” in the name. They solve unrelated problems.

**Reserved concurrency** is about *fairness and protection*. The account has a shared pool of 1,000. One runaway function can consume all of it and starve everything else. Reserving carves out a guaranteed slice for a function — and simultaneously **caps** that function at the slice, because a reservation is both a floor and a ceiling.

- Free.
- Set on the function.
- Setting it to **0 disables the function entirely** — a useful kill switch.
- Also the standard answer for *protecting a downstream database*: if only 20 environments can exist, only 20 connections can be opened. (RDS Proxy is the other answer.)

**Provisioned concurrency** is about ***latency***. It keeps N environments already initialised and warm, so requests skip the cold-start path entirely.

- Paid — you’re renting warm capacity.
- Set on a **version or alias**, never on `$LATEST`.
- Can auto-scale on a schedule or via Application Auto Scaling.

**SnapStart** is the free alternative for cold starts: it takes a snapshot of the initialised environment and restores from it. Originally Java-only, now also Python and .NET.

|    | Reserved | Provisioned |
| --- | --- | --- |
| Solves | Starvation, downstream connection limits | **Cold start latency** |
| Cost | Free | Paid |
| Applied to | Function | Version or alias |
| Also acts as | A hard cap | Nothing (it’s a warm floor) |

### Event Source Mapping

Normally, you might imagine:

```plain text
SQS → Lambda
```

But for **poll-based sources**, Lambda doesn't just magically receive the message.

AWS creates a **polling worker** for you:

```plain text
SQS
 ↓
Event Source Mapping
 ↓
Lambda
```

The Event Source Mapping keeps asking:

> **“SQS, do you have new messages?”**

When messages arrive:

```plain text
SQS
 ↓
"Yes, I have 10 messages"
 ↓
Event Source Mapping
 ↓
Lambda gets a batch of 10

```

For your exam, remember these common ones:

```plain text
SQS
Kinesis Data Streams
DynamoDB Streams
MSK
Amazon MQ
DocumentDB
```

These are **poll-based**.

**Destinations vs DLQ**

DLQ came first and only handles failures. Destinations are the newer, richer mechanism.

|    | **Destinations** (preferred) | **DLQ** (legacy) |
| --- | --- | --- |
| Fires on | Success **and** failure | Failure only |
| Targets | SQS, SNS, Lambda, **EventBridge** | SQS, SNS |
| Payload | Request **and** response context | Original event only |

**Success Destinations**

This is another big difference.

DLQ only stores failures.

Destination works for both:

```plain text
SUCCESS
FAILED
```

Example:

```plain text
Lambda
      ↓
Succeeded
      ↓
EventBridge
```

Now another system can continue processing.

---

## Lambda **InvocationType:**

**`RequestResponse`** (default) - Invoke the function synchronously. Keep the connection open until the function returns a response or times out. The API response includes the function response and additional data.

**`Event`** - Invoke the function asynchronously. Send events that fail multiple times to the function's dead-letter queue (if it's configured). The API response only includes a status code.

- **2 retries by default** (configurable 0–2), with backoff.
- `MaximumEventAge`: 60 seconds to 6 hours — after this the event is discarded even if retries remain.
- After retries are exhausted → **on-failure destination** or **DLQ**.

**`DryRun`** - Validate parameter values and verify that the user or role has permission to invoke the function.

## Lambda Container Runtime

**Runtime API** = the interface Lambda uses to communicate with your container and invoke your function.<br>**Custom Runtime** = you provide your own runtime instead of using an AWS-managed runtime. Your runtime must communicate with the **Lambda Runtime API**.

- **AWS Lambda base image** → Runtime Interface Client (RIC) is already included. ✅
- **Your own/OS-only base image** → must include the **Runtime Interface Client (RIC)**. ✅
- **Runtime Interface Emulator (RIE)** → used to **test Lambda container images locally**. ✅
- **RIE is not used in production.** ✅
- **Lambda container images** are stored in **Amazon ECR**. ✅
- Lambda container images must support a **single architecture** (for example, `x86_64` or `arm64`); multi-architecture container images are not supported. ✅

**ZIP vs Container**

|    | ZIP | Container image |
| --- | --- | --- |
| Package | ZIP file | Docker image |
| Stored in | AWS Lambda | Amazon ECR |
| RIC required by you? | No | Yes, for custom/OS-only image |
| RIE | Not relevant | Used for local testing |

**Exam memory:**

> **Container image + Lambda → RIC**
> **Local testing → RIE**
> **AWS base image → RIC already included**
> **Custom/OS-only image → add RIC  (Runtime Interface Client)**

**Custom Runtime** → you control how Lambda initializes and gets invocation events.

#### Lambda supports **two CPU architectures**:`x86_64, arm64`

But **one Lambda function can use only one architecture**.

### Failure Destination

Destination can be:

- **SQS**
- **SNS**
- **EventBridge**
- Another **Lambda**

### Example

```plain text
S3 event
  ↓
Lambda
  ↓
fails after retries
  ↓
SQS (Failure Destination)
```

The failed event is sent to the destination, so you can **inspect or process it late**

**Event source mappings — where most Lambda troubleshooting questions live**

**SQS + Lambda**

**The duplicate-processing problem.** When a consumer receives a message, SQS hides it for the *visibility timeout*. If the function is still working when that timeout expires, the message becomes visible again and a second invocation picks it up. The first invocation then finishes and deletes it — but the work happened twice.

So: **visibility timeout must be greater than or equal to the function timeout.** AWS recommends **6×** to absorb retries. Whenever a question describes duplicate processing with a specific timeout mismatch, this is the answer.

**The whole-batch retry problem.** By default, if any message in a batch fails, the entire batch returns to the queue. The nine that succeeded get reprocessed. The fix is **partial batch response**: set `FunctionResponseTypes: ReportBatchItemFailures` on the event source mapping, and have the function return the IDs of only the failed messages.

Reducing batch size to 1 also solves it, but destroys throughput and multiplies invocation cost — so it loses to `ReportBatchItemFailures` on any “most efficient” question.

Other points:

- Batch size **1–10,000** (a batching window is required above 10).
- `MaximumConcurrency` on the mapping caps how many concurrent pollers the queue creates.
- **Lambda deletes successful messages automatically** — do not delete them in your code.
- `maxReceiveCount` = how many times SQS can deliver a message to a consumer before moving it to the DLQ.

**Kinesis / DynamoDB Streams + Lambda**

These behave differently from SQS because **order matters within a shard**.

- **One concurrent invocation per shard** by default. `ParallelizationFactor` (1–10) allows more, at the cost of strict ordering across the batches.
- Because order must be preserved, **a failing record blocks its shard**. It retries until it expires — the “poison pill”. This is why the exam asks about it. The fixes:
  - `MaximumRetryAttempts`
  - `MaximumRecordAgeInSeconds`
  - `BisectBatchOnFunctionError` — splits the batch repeatedly to isolate the bad record
  - `OnFailure` destination (SQS/SNS)
  - `FunctionResponseTypes: ReportBatchItemFailures`
- `StartingPosition`: `LATEST` (only new records), `TRIM_HORIZON` (everything retained), `AT_TIMESTAMP`.
- **`IteratorAge`** is the metric that tells you the consumer is falling behind. Rising `IteratorAge` = lag.

**1. What is a shard?**

A **shard** is like a lane on a highway.

Example:

```plain text
Kinesis Stream

Shard 1:
A → B → C → D → E

Shard 2:
F → G → H → I
```

Records inside **one shard have an order**.

AWS promises:

```plain text
A must be processed before B
B must be processed before C
```

---

**2. One Lambda invocation per shard**

By default:

```plain text
Shard 1
A B C D E
    |
    ↓
 Lambda worker
```

Only one Lambda invocation reads that shard at a time.

Why?

Because if Lambda did:

```plain text
Lambda 1: process A
Lambda 2: process C
Lambda 3: process B
```

Order could break.

---

**3. What happens if one record fails?**

Example:

```plain text
Shard 1:

A ✅
B ✅
C ❌
D ?
E ?
```

Lambda cannot continue.

Why?

Maybe D depends on C.

So:

```plain text
A ✅
B ✅
C ❌
      ↓
Shard blocked
      ↓
Retry C
      ↓
Retry C
      ↓
Retry C
```

This failed record is called a:

**Poison pill**

Meaning:

> One bad record stops the whole shard.

---

**4. How do we fix poison pills?**

**MaximumRetryAttempts**

Limit retries.

Example:

```plain text
Retry C
Retry C
Retry C

3 attempts reached

Skip C
Continue with D
```

---

**MaximumRecordAgeInSeconds**

Give a record a time limit.

Example:

```plain text
Record C arrived:

12:00

Maximum age = 1 hour

13:00:
Stop retrying C
```

Useful because old data may no longer matter.

---

**BisectBatchOnFunctionError**

This is very important.

Suppose Lambda receives:

```plain text
[A, B, C, D, E, F, G, H]
```

C fails.

Instead of retrying all 8:

```plain text
[A,B,C,D] [E,F,G,H]
```

Then:

```plain text
[A,B,C,D] fails
        ↓
[A,B] [C,D]
        ↓
[C,D] fails
        ↓
[C] [D]
        ↓
Found C
```

It is a binary search for the bad record.

---

**OnFailure Destination**

After retries are exhausted:

```plain text
Bad record
     ↓
SQS
     ↓
Manual investigation
```

Instead of losing the data.

**Versions and aliases**

`$LATEST` is mutable — it changes every time you upload code. That makes it unsafe for clients to point at. Publishing creates an **immutable numbered version**. An **alias** is a named pointer to a version, and it is the stable ARN clients should use.

```plain text
$LATEST  (mutable working copy)
    |
  Publish
    ↓
Version 1   Version 2   Version 3      ← immutable snapshots
                 ↑
            alias "prod"               ← clients point HERE
```

Why this matters:

- **Rollback becomes instant** — move the alias back one version. No redeploy.
- **Aliases support weighted routing** (90% → v2, 10% → v3). This is the machinery CodeDeploy manipulates to perform canary and linear Lambda deployments. Understanding this makes the whole CodeDeploy-for-Lambda section obvious.
- An alias **cannot point to another alias**.
- Versions capture **configuration as well as code**, so changing an environment variable and publishing produces a new version.
- Provisioned concurrency attaches to a version or alias, not `$LATEST`.
- **AWS Lambda natively supports the following programming languages, EXCEPT C++**

**Lambda in a VPC**

By default Lambda runs in an AWS-managed network with internet access but no route into your VPC. Attaching it to your VPC gives it access to private resources (RDS, ElastiCache, internal ALBs) — and, because it is now inside a private subnet, it **loses default internet access**.

```plain text
Lambda in private subnet
        ↓
   NAT Gateway   (in a public subnet)
        ↓
Internet Gateway
        ↓
    Internet
```

- **ENIs** are created automatically by Lambda inside your subnets. You never create or manage them. If a question offers “configure the ENI” as a fix, it is wrong.
- The function’s **security group** must allow outbound traffic.
- For AWS services specifically, a **VPC endpoint** avoids the NAT gateway cost entirely (Gateway endpoint for S3 and DynamoDB; Interface endpoint for most others).
- Only attach Lambda to a VPC when it genuinely needs private resources — it adds configuration surface for no benefit otherwise.
- Subnets, route tables, gateways, security groups and NACLs are covered in full under **VPC, Security Groups, and NACLs in Part 4**.

**Layers, extensions, function URLs**

**Layers** are a `.zip` of shared code or libraries, mounted at `/opt`. They exist to stop you duplicating the same dependencies in every function’s package.

- Max **5** per function; they count toward the **250 MB** unzipped limit.
- Versioned and shareable across accounts.

Difference:

|    | Layer | `/tmp` |
| --- | --- | --- |
| Purpose | Provide dependencies/code | Temporary files |
| Location | `/opt` | `/tmp` |
| Writable? | ❌ No | ✅ Yes |
| Shared? | Shared by Lambda functions | Only this execution environment |
| Permanent? | Versioned and reusable | Temporary |

---

**Example**

You create a layer:

```plain text
my-layer.zip
 └── python/
      └── image_processing.py
```

Lambda sees:

```plain text
/opt/python/image_processing.py
```

Your code:

```plain text
fromimage_processingimportresize
```

**must use the Node.js driver for PostgreSQL**

The PostgreSQL driver (`pg`) is an **external dependency**. AWS Lambda does **not** include it by default.

So you need to **package the dependency with your function** (or put it in a Lambda Layer).

### Correct approach

For a normal deployment package:

```plain text
my-function/
├── index.js
├── package.json
└── node_modules/
    └── pg/
```

Then zip **everything**:

```plain text
function.zip
├── index.js
├── package.json
└── node_modules/
```

Upload the ZIP to Lambda.

---

**If the question specifically asks "How do you bundle your Lambda function?"**

The expected answer is:

> **Install the PostgreSQL (`pg`) package into `node_modules`, then ZIP your function code together with the `node_modules` directory and upload it to Lambda.**

No `/tmp` involved.

But if `resize()` needs to process a 500MB image:

```plain text
S3 image
   ↓
/tmp/image.jpg
   ↓
process
   ↓
S3 result
```

Then `/tmp` is used.

#### Lambda layers = a separate box for dependencies

Instead of:

```plain text
Lambda ZIP
├── your code
├── axios
├── lodash
└── other libraries
```

You make:

```plain text
Lambda ZIP
└── your code

Layer
├── axios
├── lodash
└── other libraries
```

Lambda loads the **Layer together with your function**, so your function can use those libraries. AWS specifically describes layers as a way to package dependencies separately and reduce the size of deployment packages. AWS extracts the layer into `/opt` when the function starts

`/opt` is **provided by Layers**; `/tmp` is **writable temporary storage for the function**.

### Why use it?

Imagine **10 Lambda functions all use `axios`**:

```plain text
Without Layer:
Function 1 → axios
Function 2 → axios
Function 3 → axios
...
Function 10 → axios
```

With a Layer:

```plain text
             ┌── Function 1
             ├── Function 2
Layer ───────┼── Function 3
   axios     ├── ...
             └── Function 10
```

You keep the dependency **separate and reusable**.

### For your exam

> **Lambda ZIP too big because of dependencies → put dependencies in a Lambda Layer.**

**Extension = helper for Lambda.**

- **Internal** → inside Lambda runtime
- **External** → separate process
- Usually installed as a **Layer**
- Can help with **monitoring, secrets, configuration**

**AppConfig / Parameter Store extension:**

Gets the value once → **keeps it locally** → Lambda reuses it.

**Function URLs** give a function a dedicated HTTPS endpoint with no API Gateway in front.

- Auth is `AWS_IAM` or `NONE`. Built-in CORS. Supports response streaming.
- **No** usage plans, API keys, request validation, or WAF. If the question needs any of those, the answer is API Gateway.

**Environment variables and secrets**

Environment variables are encrypted at rest by default with the AWS managed key `aws/lambda`. That protects the stored value — but anyone with console access to the function can still *read* it.

If the requirement is that other developers must not see the value, use a **customer managed KMS key** plus the console **encryption helpers**, and decrypt in code with the KMS SDK.

If the requirement involves **rotation**, environment variables are the wrong tool entirely → Secrets Manager.

The total size of all environment variables shouldn't exceed** 4 KB. **There is** no limit **on the number of variables

**Deployment package options**

| Option | When |
| --- | --- |
| Inline console editor | Trivial functions, \< 3 MB |
| `.zip` direct upload | Up to 50 MB zipped |
| `.zip` via S3 | Larger zips (250 MB unzipped limit still applies) |
| **Container image** | Up to **10 GB** — the answer when the package is genuinely large (ML models, heavy native dependencies) |

Best practices, and the reason for each:

- **Separate the handler from your core logic.** Makes the logic unit-testable without Lambda.
- **Use the execution context** — initialise clients and connections outside the handler.
- **Use environment variables** for operational parameters instead of hardcoding.
- **Control and minimise dependencies.** Every megabyte is download time on every cold start.
- **Avoid recursive code.** A function that invokes itself can run away and generate a very large bill. AWS treats this as an anti-pattern.

**Numbers**

| Limit | Value |
| --- | --- |
| Timeout | default **3 s**, max **900 s (15 min)** |
| Memory | **128 MB – 10,240 MB** (1 MB increments) |
| CPU | Scales **with memory**; \~1 vCPU around 1,769 MB |
| Package, zipped direct upload | **50 MB** |
| Package, unzipped (incl. layers) | **250 MB** |
| Container image | **10 GB** |
| `/tmp` | **512 MB** default, configurable to **10,240 MB** |
| Environment variables | **4 KB** total |
| Layers | **5** per function |
| Default account concurrency | **1,000** (soft) |
| Invocation payload | **6 MB** sync, **256 KB** async |

**The memory–CPU link is the most-used optimisation fact on the exam.** CPU is not configurable directly; it is allocated in proportion to memory. So a CPU-bound function that doesn’t need more memory still gets faster when you give it more memory — and often gets *cheaper*, because billing is memory × duration and the duration drops faster than the memory rises.

```plain text
512 MB  → 10 seconds
2048 MB →  3 seconds     ← 4× memory, 3.3× faster, roughly same or lower cost
```

**Idempotency — why it’s a Lambda topic at all**

Retries are built into the platform: async invocations retry twice, SQS redelivers on visibility timeout expiry, API Gateway clients retry on timeouts, networks drop responses. So **your function will be invoked more than once with the same event**, and it must not double-charge a card or double-ship an order.

The pattern:

```plain text
Client sends a unique request ID
        ↓
Lambda: conditional write of that ID to DynamoDB
        ↓
Write succeeded  → first time → do the work
Write failed     → already processed → return the previous result
```

**DynamoDB is the right store** because it offers a single atomic “check and write” operation (`ConditionExpression` with `attribute_not_exists`), scales with Lambda’s concurrency, and has single-digit millisecond latency.

| Store | Suitable? | Why |
| --- | --- | --- |
| **DynamoDB** | ✅ Best | Atomic conditional writes, scales, low latency |
| RDS | ⚠️ Possible | Works, but connection limits fight Lambda’s scaling |
| ElastiCache | ❌ No | A cache can evict entries; you’d lose the record |
| S3 | ❌ No | No atomic conditional check, too slow for a per-request gate |

Note the distinction from **atomic counters**: `SET count = count + 1` is safe under concurrency but is *not idempotent* — a retry increments twice. Use it for view counts, not for money.

**Lambda@Edge**

**Mental Model**

**Lambda@Edge = Run Lambda at CloudFront edge locations (close to users), not in one AWS Region.**

```plain text
User (Japan)
      ↓
CloudFront Edge (Tokyo)
      ↓
Lambda@Edge runs here
      ↓
CloudFront
      ↓
Origin (S3 / ALB / EC2)
```

Instead of sending every request to your origin, CloudFront can execute code at the nearest edge location.

---

## Common Use Cases

- Redirect users (HTTP → HTTPS)
- URL rewrites
- Authentication/authorization
- Add or remove HTTP headers
- Personalize content by country, language, or device
- A/B testing

### CloudFront Functions vs Lambda@Edge

|    | **CloudFront Functions** | **Lambda@Edge** |
| --- | --- | --- |
| Best for | Simple, lightweight logic | Complex logic |
| Runtime | JavaScript | Node.js / Python |
| Startup/performance | **Very fast** | Slower than Functions |
| Execution time | Very short | Longer-running |
| Cost | **Cheaper** | More expensive |
| Use cases | Headers, URL redirects, JWT validation, redirects | Complex authentication, image processing, external/API calls |
| Trigger | Viewer Request / Viewer Response | Viewer Request / Viewer Response / Origin Request / Origin Response |
| Access to request | Basic | More capabilities |
| Network calls | **No** | **Yes** |
| Operational complexity | **Lower** | Higher |

**For your question**

JWT validation is relatively simple:

**Client → CloudFront Function (Viewer Request) → validate JWT →**

- Invalid → reject immediately
- Valid → continue to ALB → Fargate

So **CloudFront Functions** is the better choice.

### Lambda Authorizer — very simple

A **Lambda Authorizer** is a Lambda function that **checks who is calling your API and whether they are allowed**.

```plain text
User
 ↓
API Gateway
 ↓
Lambda Authorizer
 ↓
ALLOW or DENY
 ↓
Your Lambda
```

**Example**

User sends:

```plain text
Authorization: Bearer ABC123
```

API Gateway asks the Lambda Authorizer:

> “Is this token valid?”

Authorizer says:

```plain text
✅ ALLOW → request continues
❌ DENY → request blocked
```

**When to use it**

Use **Lambda Authorizer** when you have **custom authentication/authorization**, such as:

- OAuth
- SAML
- custom bearer token
- custom authorization rules

---

## Request Flow

```plain text
Viewer
   ↓
CloudFront
   ↓
Lambda@Edge
   ↓
Origin
```

**How Lambda fails — the diagnostic table**

| Symptom | Cause | Fix |
| --- | --- | --- |
| Duplicate SQS processing | Visibility timeout \< function timeout | Raise visibility timeout to ≥ 6× function timeout |
| Whole SQS batch retried on one failure | Default batch behaviour | `ReportBatchItemFailures` |
| Stream processing frozen forever | Poison-pill record blocking the shard | `MaximumRetryAttempts` / `BisectBatchOnFunctionError` / on-failure destination |
| Rising `IteratorAge` | Consumer slower than producer | More shards, higher `ParallelizationFactor`, faster function |
| **429 / `TooManyRequestsException`** | Concurrency limit reached | Raise account limit, or set reserved concurrency |
| Slow first request | Cold start | Provisioned concurrency, SnapStart, smaller package |
| Function is CPU-bound and slow | CPU tied to memory | **Increase memory** |
| Cannot reach RDS | Not in the VPC | Attach the function to the VPC |
| In the VPC, cannot reach the internet | No route out of the private subnet | **NAT gateway** (or VPC endpoint) |
| **No logs in CloudWatch** | Execution role lacks log permissions | Attach `AWSLambdaBasicExecutionRole` |
| Async failures vanishing | No failure handling configured | On-failure destination or DLQ |
| Needs \> 15 minutes | Hard platform limit | Fargate, AWS Batch, or Step Functions |
| Needs shared persistent files | `/tmp` is not shared or durable | Mount **EFS** |
| Runaway invocations and cost | Recursive invocation | Redesign; recursion is an anti-pattern |

**Recall check — Lambda**

- **Improve Lambda performance** → initialise SDK clients and connections **outside the handler**
- An **execution context** is the environment where a Lambda function runs. AWS creates it for your function and can **reuse it for later invocations**.
- **Need persistent storage** → S3 / DynamoDB / EFS, never the execution context
- **Function is CPU-heavy** → **increase memory** (CPU scales with memory)
- **Eliminate cold starts** → provisioned concurrency (or SnapStart, free, for Java/Python/.NET)
- **Must always have capacity / must not starve other functions** → reserved concurrency
- **Turn a function off without deleting it** → reserved concurrency = 0
- **Too many DB connections from Lambda** → reserved concurrency, or RDS Proxy
- **429 / `Throttles` metric** → concurrency limit reached
- **Provisioned concurrency on `$LATEST`** → not allowed; use a version or alias
- **Clients must not break on deploy** → point them at an **alias**
- **Instant rollback** → move the alias to the previous version
- **Send 10% of traffic to new code** → alias weighted routing
- **Async invocation retries** → **2** by default
- **Sync invocation failure** → the **caller** retries; Lambda does not
- **Async failures must not be lost** → on-failure destination (or DLQ)
- **Need success notifications too** → destinations, not DLQ
- **Need EventBridge as a failure target** → destinations only
- **Duplicate SQS processing** → visibility timeout \< function timeout
- **One bad message retries the whole batch** → `ReportBatchItemFailures`
- **Stream stuck forever** → poison pill; retry limits / bisect / on-failure destination
- **Rising `IteratorAge`** → consumer falling behind
- **Process a stream from the beginning** → `TRIM_HORIZON`
- **Share libraries between functions** → Layers
- **Reduce package size** → Layers
- **Cache Parameter Store / AppConfig values** → Lambda extension
- **Simplest HTTPS endpoint, no keys or throttling** → Function URL
- **Need API keys, usage plans, authorizers, WAF** → API Gateway
- **Env vars must be unreadable by other developers** → customer managed KMS key + encryption helpers
- **Credentials needing rotation** → Secrets Manager, not env vars
- **Package too large** → S3 upload, layers, or container image (10 GB)
- **Lambda in VPC cannot reach internet** → **NAT Gateway**
- **Lambda needs RDS** → attach to the VPC
- **ENI** → created automatically; never the fix for internet access
- **ENI is needed for VPC connectivity.**<br>
- **Job longer than 15 minutes** → not Lambda
- **Duplicate API requests must not double-charge** → idempotency with DynamoDB conditional write
- **Store processed request IDs** → DynamoDB
- **Atomic “check and save”** → DynamoDB conditional write
- **Simple + fast + cheap + before origin → CloudFront Functions**
- **Complex + needs more capabilities/network access → Lambda@Edge**
- **Custom token → Lambda Authorizer ✅**
- **Custom Runtime** → you control how Lambda initializes and gets invocation events. ✅

---

# Question patterns

> A web tier invokes a Lambda function 50 times per second and each invocation runs for 100 seconds. How many concurrent executions are required, and what happens?
> → **5,000 (50 × 100); the default account limit of 1,000 is exceeded so the function is throttled with TooManyRequestsException (HTTP 429)**

> A Lambda function returns the error TooManyRequestsException and the CloudWatch Throttles metric is climbing. What is the cause?
> → **The account concurrency limit (default 1,000) has been reached**

> A function opens a database connection outside the handler and reuses it across invocations. Why does this work?
> → **Anything outside the handler runs once per execution environment (init phase) and the environment is reused for later invocations**

> A function writes a scratch file to /tmp and a later invocation on a different environment cannot find it. Why?
> → **/tmp is per-execution-environment and environments are eventually discarded, so it is neither durable nor shared**

> An application needs storage shared reliably across many concurrent Lambda invocations. What should be used?
> → **S3, DynamoDB, or EFS — never /tmp or the execution context**

> A Lambda function must run for 20 minutes on every execution. Which service should be used instead?
> → **Move off Lambda (ECS/Fargate, AWS Batch, Step Functions, or EC2) — Lambda's maximum timeout is 15 minutes (900 s)**

> A team must guarantee a critical function always has capacity and cannot be starved by another function's runaway scaling. Which feature?
> → **Reserved concurrency (a guaranteed slice that also caps the function)**

> A developer must instantly disable a Lambda function without deleting it. What is the simplest way?
> → **Set reserved concurrency to 0**

> A Lambda function opens too many connections and overloads an RDS database. Which two solutions help?
> → **Reserved concurrency (caps concurrent environments/connections) or RDS Proxy**

> A team wants the first invocation of a Java, Python, or .NET function to skip cold-start work. Which free feature?
> → **Lambda SnapStart (restores from a snapshot of the initialised environment)**

> A team needs pre-initialised warm environments so requests never hit the cold-start path. Which feature, and where is it configured?
> → **Provisioned concurrency, set on a version or alias — never on $LATEST**

> A developer tries to enable provisioned concurrency on $LATEST and it fails. Why?
> → **Provisioned concurrency can only be attached to a published version or an alias, not $LATEST**

> Clients must not break when new code is deployed. What should the clients point at?
> → **A Lambda alias (a stable named pointer to an immutable version)**

> A developer must roll back a bad deployment instantly without redeploying. What is the fastest method?
> → **Move the alias back to the previous version**

> A team wants to send 10% of traffic to a new version and 90% to the old. Which Lambda feature?
> → **Alias weighted routing (the mechanism CodeDeploy manipulates for canary/linear deployments)**

> Can a Lambda alias point to another alias?
> → **No — an alias points to a version, never to another alias**

> A change to an environment variable must be captured in a deployable snapshot. What does publishing produce?
> → **A new immutable numbered version — versions capture configuration as well as code**

> An asynchronous Lambda invocation fails. How many times does Lambda retry by default, and what is the range?
> → **2 retries by default, configurable 0–2, with backoff**

> An asynchronous event is more than 6 hours old and has retries remaining. What happens?
> → **It is discarded — MaximumEventAge accepts only 60 seconds to 6 hours**

> Async invocations fail repeatedly and the events must not be lost. What should be configured?
> → **An on-failure destination (SQS/SNS/EventBridge/Lambda) or a DLQ**

> A team needs notifications for both successful and failed invocations with full request/response context. Which mechanism?
> → **Destinations (DLQ only handles failures and stores the original event only)**

> A team needs to route failed async invocations to EventBridge. Which mechanism supports that?
> → **Destinations — DLQ targets are limited to SQS and SNS**

> A Lambda function processes SQS messages and the same message is processed twice. What is the cause?
> → **The visibility timeout is shorter than the function timeout; the message becomes visible again while still being processed**

> What visibility timeout should be set relative to the function timeout to absorb retries?
> → **At least 6× the function timeout (and never less than the function timeout)**

> One bad message in an SQS batch causes all the successful messages to be reprocessed. What is the most efficient fix?
> → **Partial batch response: set FunctionResponseTypes to ReportBatchItemFailures and return only the failed message IDs**

> Why is reducing SQS batch size to 1 not the preferred fix for whole-batch retries?
> → **It destroys throughput and multiplies invocation cost; ReportBatchItemFailures is more efficient**

> What is the valid SQS event source mapping batch size range, and when is a batching window required?
> → **1–10,000 messages; a batching window is required above 10**

> A single Event Source Mapping must limit how many concurrent pollers the queue creates. Which setting?
> → **MaximumConcurrency on the event source mapping**

> Should application code delete successfully processed SQS messages?
> → **No — Lambda deletes successful messages automatically**

> What does maxReceiveCount control for an SQS-driven Lambda?
> → **How many times SQS delivers a message to a consumer before moving it to the DLQ**

> A Kinesis-driven Lambda consumer must preserve strict order within a shard. What does that imply about concurrency?
> → **One concurrent invocation per shard by default; ParallelizationFactor (1–10) increases it at the cost of strict ordering**

> One failing record blocks an entire Kinesis shard forever. What is this called and what are the fixes?
> → **A poison pill; fix with MaximumRetryAttempts, MaximumRecordAgeInSeconds, BisectBatchOnFunctionError, an OnFailure destination, or ReportBatchItemFailures**

> A developer wants to isolate a bad stream record by repeatedly halving the failing batch. Which setting?
> → **BisectBatchOnFunctionError**

> A stream consumer is falling behind the producer. Which metric shows this?
> → **IteratorAge (rising IteratorAge = lag)**

> A stream consumer should process all retained records from the beginning. Which StartingPosition?
> → **TRIM_HORIZON (LATEST = only new records; AT_TIMESTAMP = from a time)**

> Which source types are poll-based and use an event source mapping?
> → **SQS, Kinesis Data Streams, DynamoDB Streams, MSK, Amazon MQ, DocumentDB**

> A team wants to share common libraries across many Lambda functions and shrink each deployment package. Which feature?
> → **Lambda Layers (a .zip mounted at /opt, max 5 per function, counted in the 250 MB unzipped limit)**

> Where does Lambda mount a layer, and is it writable?
> → **/opt — it is read-only; only /tmp is writable temporary storage**

> A Lambda container image built on a custom or OS-only base image fails to run. What must be included?
> → **The Runtime Interface Client (RIC)**

> What is the Runtime Interface Emulator (RIE) used for, and is it used in production?
> → **Testing Lambda container images locally; it is not used in production**

> A team builds a multi-architecture container image for Lambda. What is the problem?
> → **Lambda container images must support a single architecture (x86_64 or arm64); multi-architecture images are not supported**

> Which two CPU architectures does Lambda support, and how many can one function use?
> → **x86_64 and arm64 — but one function can use only one architecture**

> A function returns an AccessDenied error calling S3, but the task starts and logs fine. Which role is misconfigured for ECS (analogous Lambda case)?
> → **The runtime role (task role) used by application code, not the startup/execution role**

> A serverless app must expose a function as an HTTPS endpoint with no API Gateway, and only needs AWS_IAM or NONE auth. Which feature?
> → **Lambda Function URLs (built-in CORS, supports response streaming)**

> A requirement includes API keys, usage plans, request validation, or WAF. Which option must be used instead of a Function URL?
> → **API Gateway — Function URLs support none of those**

> Environment variables must be unreadable to other developers with console access. What is required?
> → **A customer managed KMS key plus the console encryption helpers, decrypting in code with the KMS SDK (default aws/lambda key alone lets console users read the value)**

> Credentials must be rotated automatically. Which service should be used instead of environment variables?
> → **AWS Secrets Manager**

> What is the total size limit for all Lambda environment variables, and is there a limit on the number of variables?
> → **4 KB total; there is no limit on the number of variables**

> A deployment package includes a large ML model. Which packaging option supports up to 10 GB?
> → **A container image (stored in Amazon ECR); direct .zip upload is 50 MB and unzipped packages cap at 250 MB**

> A CPU-bound function is slow but does not need more memory. What single change most likely speeds it up, and could it also reduce cost?
> → **Increase memory — CPU scales with memory (~1 vCPU near 1,769 MB), and billing is memory × duration so a faster run can cost the same or less**

> What are the Lambda memory and /tmp limits?
> → **Memory 128 MB–10,240 MB in 1 MB increments; /tmp defaults to 512 MB and configures up to 10,240 MB**

> What are the synchronous and asynchronous invocation payload limits?
> → **6 MB synchronous, 256 KB asynchronous**

> A retry must not double-charge a customer. What idempotency pattern should be used?
> → **Client sends a unique request ID; Lambda does a DynamoDB conditional write (attribute_not_exists) and returns the previous result if the write fails**

> Why is DynamoDB the recommended store for Lambda idempotency, rather than ElastiCache or S3?
> → **It offers an atomic conditional write, scales with Lambda's concurrency, and has single-digit millisecond latency (a cache can evict; S3 has no atomic conditional check)**

> Why is an atomic counter (SET count = count + 1) not idempotent?
> → **A retry increments it twice; use it for view counts, not for money**

> Requests should be validated with simple logic at the edge before reaching the origin. CloudFront Functions or Lambda@Edge?
> → **CloudFront Functions — simple, very fast, cheaper, JavaScript, but no network calls**

> A edge function must make external network/API calls and run longer logic. Which should be used?
> → **Lambda@Edge (Node.js/Python, supports origin triggers and network calls)**

> A custom bearer token or OAuth/SAML flow must gate an API. Which API Gateway feature?
> → **A Lambda Authorizer (returns ALLOW or DENY based on a custom token)**

> A function has no logs in CloudWatch. What is the likely cause and fix?
> → **The execution role lacks log permissions; attach AWSLambdaBasicExecutionRole**

> A Lambda function is attached to a VPC's private subnet and now cannot reach the internet. What is the standard fix?
> → **Add a NAT Gateway in a public subnet (or use a VPC endpoint for AWS services specifically)**

> A question proposes "configure the ENI" to fix a Lambda internet-access issue. Why is that wrong?
> → **Lambda creates and manages the ENIs automatically; you never create or manage them**

> Lambda must reach RDS in private subnets. What must be configured?
> → **Attach the function to the VPC (subnets + a security group allowing outbound traffic); the execution role needs VPC permissions (AWSLambdaVPCAccessExecutionRole)**

> A function invokes itself and generates an unexpectedly large bill. What is this and how is it treated?
> → **Recursive invocation — an anti-pattern; it must be redesigned**

> Which AWS Lambda languages are natively supported, and which notable language is not?
> → **Java, Python, Node.js, .NET, Ruby, Go, and others are supported; C++ is not supported natively**

> A Lambda function must process a job that takes 60 minutes. What is the correct architecture?
> → **Not Lambda — use Fargate, AWS Batch, Step Functions, or EC2 (15-minute hard limit)**

> A function needs a private resource (RDS/ElastiCache/internal ALB). Should it be attached to a VPC?
> → **Yes, only when it genuinely needs private resources; otherwise attaching adds configuration surface for no benefit**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Max timeout | **15 minutes (900 s); default 3 s** |
| Concurrency formula | **requests/s × average duration in seconds** |
| Default concurrency limit | **1,000 (soft limit)** |
| Throttle symptom | **TooManyRequestsException / HTTP 429; Throttles metric** |
| Reserved concurrency | **Guaranteed slice + hard cap; free; set on the function** |
| Disable a function | **Reserved concurrency = 0** |
| Too many DB connections | **Reserved concurrency or RDS Proxy** |
| Cold-start latency fix | **Provisioned concurrency (paid) on a version/alias** |
| Free cold-start fix | **SnapStart (Java, Python, .NET)** |
| Provisioned on $LATEST | **Not allowed — use a version or alias** |
| Code outside handler | **Runs once per environment (init phase); reused** |
| Execution context | **Environment AWS creates and reuses for later invocations** |
| Handler best practice | **Separate handler from core logic for unit testing** |
| Persistent storage | **S3 / DynamoDB / EFS — never /tmp or execution context** |
| /tmp | **512 MB default, configurable to 10,240 MB; per environment, not durable or shared** |
| Shared files across invocations | **EFS** |
| CPU-heavy function | **Increase memory (CPU scales with memory)** |
| 1 vCPU point | **~1,769 MB** |
| Memory range | **128 MB – 10,240 MB (1 MB increments)** |
| Async retries | **2 by default (configurable 0–2)** |
| MaximumEventAge | **60 s – 6 h** |
| Async failures not lost | **On-failure destination (or DLQ)** |
| DLQ targets | **SQS, SNS** |
| Destination targets | **SQS, SNS, Lambda, EventBridge** |
| Success notifications | **Destinations, not DLQ** |
| Payload limit | **6 MB sync / 256 KB async** |
| Sync failure handling | **The caller retries; Lambda does not** |
| SQS duplicate processing | **Visibility timeout < function timeout** |
| Visibility timeout rule | **≥ 6× function timeout** |
| Whole SQS batch retried | **ReportBatchItemFailures (partial batch response)** |
| SQS batch size | **1–10,000; batching window required above 10** |
| Store processed request IDs | **DynamoDB** |
| Atomic "check and save" | **DynamoDB conditional write (attribute_not_exists)** |
| Atomic counter not idempotent | **Retry double-increments; use for views, not money** |
| SQS message deletion | **Lambda deletes successful messages automatically** |
| maxReceiveCount | **Deliveries before an SQS message moves to the DLQ** |
| Stream concurrency | **One invocation per shard; ParallelizationFactor 1–10** |
| Stream stuck forever | **Poison pill; max retries / bisect / on-failure destination** |
| Bisect bad record | **BisectBatchOnFunctionError** |
| Stream from the beginning | **TRIM_HORIZON** |
| Rising IteratorAge | **Consumer falling behind producer** |
| Share libraries / shrink package | **Lambda Layers** |
| Layer location / limit | **/opt; max 5 per function; counts in 250 MB unzipped** |
| Reduce package size | **Layers, S3 upload, or container image (10 GB)** |
| Package sizes | **50 MB zipped direct, 250 MB unzipped, 10 GB container** |
| Custom Runtime | **You control initialisation and how Lambda gets events** |
| Container image with custom/OS-only base | **Add the Runtime Interface Client (RIC)** |
| AWS base image | **RIC already included** |
| Local image testing | **Runtime Interface Emulator (RIE), not for production** |
| Container images stored | **Amazon ECR** |
| Lambda architectures | **x86_64, arm64 — one function uses only one** |
| Simplest HTTPS endpoint | **Function URL (AWS_IAM or NONE, built-in CORS)** |
| Need API keys / usage plans / WAF | **API Gateway** |
| Env vars unreadable by others | **Customer managed KMS key + encryption helpers** |
| Credentials requiring rotation | **Secrets Manager, not env vars** |
| Env vars total size | **4 KB (no limit on count)** |
| Lambda in VPC cannot reach internet | **NAT Gateway (or VPC endpoint for AWS services)** |
| ENI | **Created automatically by Lambda; never the fix** |
| Lambda needs RDS | **Attach the function to the VPC** |
| No CloudWatch logs | **Execution role lacks AWSLambdaBasicExecutionRole** |
| Job longer than 15 minutes | **Not Lambda (Fargate, Batch, Step Functions, EC2)** |
| Recursive invocation | **Anti-pattern; runaway invocations and cost** |
| Simple + fast + cheap at edge | **CloudFront Functions** |
| Complex edge logic with network access | **Lambda@Edge** |
| Custom token / OAuth / SAML gate | **Lambda Authorizer** |
| Async destination triggers | **Success and failure** |
| EventBridge as failure target | **Destinations only, not DLQ** |
| Unsupported native language | **C++** |
