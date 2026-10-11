## AWS Step Functions

**Why it exists**

Multi-step processes written as Lambda-calling-Lambda have real problems: no visibility into where a run is, retry logic duplicated in every function, you pay for a function to sit waiting on another, and a failure halfway through leaves inconsistent state.

Step Functions moves the *coordination* out of your code into a declarative state machine. Retries, error handling, branching, parallelism, and long waits become configuration.

**The anti-pattern it replaces:** a chain of Lambdas invoking each other. That is why “orchestrate multiple Lambda functions” is always answered with Step Functions.

**State types**

Workflows are defined in **Amazon States Language (ASL)**, a JSON document.

A **state machine is the entire workflow**; these are **state types inside a state machine**.

| State | Purpose |
| --- | --- |
| **Task** | Do work — Lambda, ECS, DynamoDB, SNS, SQS, or any AWS SDK call |
| **Choice** | Branch on a condition (if/else) |
| **Parallel** | Run several branches simultaneously |
| **Map** | Loop over an array, one iteration per item |
| **Wait** | Pause for a duration or until a timestamp |
| **Pass** | Pass input through, or inject fixed data |
| **Succeed** / **Fail** | Terminate the execution |

**Standard vs Express**

The choice follows from duration and volume.

|    | **Standard** | **Express** |
| --- | --- | --- |
| Max duration | **1 year** | **5 minutes** |
| Execution semantics | **Exactly-once** | **At-least-once** (async) |
| Start rate | \~2,000/s | **100,000+/s** |
| History | Full, visible in the console for 90 days | CloudWatch Logs only |
| Pricing | Per **state transition** | Per **invocation + duration** |
| `.waitForTaskToken` | ✅ | ❌ |
| Best for | Long, auditable, non-idempotent business processes | High-volume, short, idempotent event processing |

Anything longer than 5 minutes, or requiring exactly-once, or requiring human approval, must be **Standard**.

**Error handling — the main reason to use it**

```json
"Retry": [{
  "ErrorEquals": ["States.TaskFailed"],
  "IntervalSeconds": 2,
  "MaxAttempts": 3,
  "BackoffRate": 2.0
}],
"Catch": [{
  "ErrorEquals": ["States.ALL"],
  "Next": "HandleFailure",
  "ResultPath": "$.error"
}]
```

`Retry` runs first; when it is exhausted, `Catch` fires and routes to a handler state.

Predefined errors: `States.ALL`, `States.TaskFailed`, `States.Timeout`, `States.Permissions`, `States.DataLimitExceeded`.

Getting this behaviour without Step Functions means writing retry loops and try/catch into every function — which is exactly the “retry logic without writing code” question.

**Service integration patterns**

| Pattern | Suffix | Behaviour |
| --- | --- | --- |
| **Request Response** | (none) | Call and continue immediately |
| **Run a Job** | `.sync` | Call and **wait for completion** — ECS task, Batch job, EMR step |
| **Wait for Callback** | `.waitForTaskToken` | Pause indefinitely until an external system calls `SendTaskSuccess` |

`.waitForTaskToken` is how **human approval** works: the workflow emits a task token (typically into an email or a queue), then waits. Days later, a person clicks approve, something calls `SendTaskSuccess` with the token, and the workflow resumes. Standard workflows only, because Express caps out at 5 minutes.

### Overall

```plain text
Step Functions
      │
      │ waitForTaskToken
      │
      │ generates token automatically
      │ token = ABC123
      ▼
StartPayment Lambda
      │
      │ receives ABC123
      ▼
Payment Service
      │
      │ ... 10 minutes ...
      ▼
Webhook Lambda
      │
      │ SendTaskSuccess(ABC123)
      ▼
Step Functions
      │
      │ "ABC123 is finished"
      ▼
Next State
```

**Other points**

- **Distributed Map** processes millions of S3 objects with very high parallelism.
- **Activities** are worker-polled tasks, for code running outside AWS.
- Payload between states is capped at **256 KB** — pass an S3 pointer for anything larger.

**Recall check — Step Functions**

- **Orchestrate a multi-step workflow** → Step Functions
- **Chain of Lambdas invoking each other** → replace with Step Functions
- **Workflow longer than 5 minutes** → Standard
- **High-volume short event processing, cheapest** → Express
- **Exactly-once required** → Standard
- **Wait for a human to approve** → `.waitForTaskToken` (Standard only)
- **Wait for an ECS task or Batch job to finish** → `.sync`
- **Retry and error handling without writing code** → `Retry` / `Catch` in ASL
- **Branch on a condition** → Choice state
- **Loop over an array** → Map state
- **Run steps simultaneously** → Parallel state
- **Pause a workflow** → Wait state
- **Payload over 256 KB between states** → store in S3, pass the key

### Step Functions States

**Task** → performs a unit of work, such as calling Lambda or another AWS service.

**Wait** → pauses the workflow until a time/date.

**Choice** → makes a decision and chooses the next path.

**Pass** → passes data through without performing work.

**Succeed** → ends successfully.

**Fail** → ends with failure.

**Exam memory**

> **Task = do work**

  **Wait = wait**

  **Choice = decide**

  **Pass = pass data**

  **Succeed = success**

  **Fail = failure**

---

## AWS Step Functions Activities

### Mental model

**Activity = Step Functions gives a job to your own worker.**

The worker is an application running outside Step Functions, for example on **EC2, ECS, or on-premises**.

```plain text
Step Functions
      ↓
   Activity
      ↓
   Your worker
      ↓
  does the job
      ↓
Success / Failure
```

## How it works

### 1. Create the Activity

You create an Activity, for example:

```plain text
GeneratePDF
```

This gives your worker an **Activity ARN**.

---

### 2. Workflow reaches the Activity

Your Step Functions state says:

```plain text
Run GeneratePDF Activity
```

Step Functions now has a job waiting for a worker.

**Step Functions does NOT automatically run your EC2 command.**

---

### 3. Worker asks for a task

Your worker application must call:

```plain text
GetActivityTask
```

This means:

> “Does Step Functions have a job for me?”

If there is a job, Step Functions gives the worker the task and a **task token**.

---

### 4. Worker does the actual work

For example, the worker receives:

```plain text
customerId = 123
```

Then your worker might execute:

```plain text
generate-pdf customer123
```

The actual command/code is written and controlled by **you**.

---

### 5. Heartbeat

While the worker is processing a long task, **your worker sends**:

```plain text
SendTaskHeartbeat
```

It means:

> **“I'm still working on this task. Don't consider me dead/timed out.”**

It is **not automatically sent by Step Functions**.

Your worker code has to call it.

Example:

```plain text
Worker:
GetActivityTask
      ↓
Start PDF generation
      ↓
SendTaskHeartbeat
      ↓
Still working...
      ↓
SendTaskHeartbeat
      ↓
Finished
      ↓
SendTaskSuccess
```

You normally implement the heartbeat in the worker application, often on a timer.

---

### 6. Task succeeds

When the worker finishes:

```plain text
SendTaskSuccess
```

Step Functions receives the result and continues to the next state.

---

### 7. Task fails

If the worker cannot complete it:

```plain text
SendTaskFailure
```

Step Functions knows the Activity failed and can handle it with its workflow error handling.

---

## APIs

| API | Who calls it? | Purpose |
| --- | --- | --- |
| `CreateActivity` | Developer/setup | Create the Activity |
| `GetActivityTask` | **Worker** | Ask for a job |
| `SendTaskHeartbeat` | **Worker** | “I'm still working” |
| `SendTaskSuccess` | **Worker** | “I finished successfully” |
| `SendTaskFailure` | **Worker** | “I failed” |

## Complete example

Imagine a hospital appointment system needs a worker to generate a complicated insurance report.

```plain text
Step Functions
      ↓
Activity: GenerateInsuranceReport
      ↓
EC2 Worker
      ↓
GetActivityTask
      ↓
"Generate report for patient 123"
      ↓
Worker generates report
      ↓
SendTaskHeartbeat
      ↓
Worker still generating...
      ↓
SendTaskHeartbeat
      ↓
Report finished
      ↓
SendTaskSuccess
      ↓
Step Functions continues
```

### Most important point

**Activity is not a command itself.**

It is a **job/task that Step Functions gives to your external worker**.

Your worker decides **what command/code to execute**.

### Exam memory

**Activity → external worker**

`GetActivityTask` → **get job**

`SendTaskHeartbeat` → **still working**

`SendTaskSuccess` → **done**

`SendTaskFailure` → **failed**

**Heartbeat is sent by the worker, not automatically by Step Functions.**

# Question patterns

> A workflow must orchestrate several Lambda calls with retries and a human approval step. Which service?
> → **AWS Step Functions**

> A chain of Lambdas invoking each other is becoming unmaintainable and hides where a run failed. What should replace it?
> → **Step Functions (orchestrate the state machine externally)**

> A business process can run for days. Standard or Express?
> → **Standard**

> Millions of short, idempotent workflows must run per month at the lowest cost. Which type?
> → **Express workflows**

> Exactly-once execution semantics are required. Which type?
> → **Standard workflows**

> A workflow must pause until a person clicks approve, possibly days later. Which mechanism?
> → **`.waitForTaskToken` callback (Standard only)**

> A workflow must call an ECS task or Batch job and wait for it to finish. Which integration pattern?
> → **Run a Job (`.sync`)**

> Retry and error handling must be added without writing code in every function. Which feature?
> → **`Retry` / `Catch` in the ASL definition**

> A workflow must branch based on a condition. Which state type?
> → **Choice state**

> A workflow must loop over every item of an array. Which state type?
> → **Map state**

> Several steps must run at the same time. Which state type?
> → **Parallel state**

> A workflow must pause for a duration or until a timestamp. Which state type?
> → **Wait state**

> More than 256 KB of data must move between states. What do you do?
> → **store it in S3 and pass the key/pointer**

> In what language are Step Functions workflows defined?
> → **Amazon States Language (ASL), a JSON document**

> Is a "state machine" the same as a "state type"?
> → **No — the state machine is the whole workflow; Task/Choice/etc. are state types inside it**

> Which state type performs work such as calling Lambda, ECS, DynamoDB, SNS or SQS?
> → **Task**

> Which state passes input through or injects fixed data?
> → **Pass**

> Which states terminate the execution?
> → **Succeed and Fail**

> What is the maximum duration of a Standard workflow?
> → **1 year**

> What is the maximum duration of an Express workflow?
> → **5 minutes**

> What execution semantics do Standard workflows provide?
> → **exactly-once**

> What execution semantics do Express workflows provide (async)?
> → **at-least-once**

> What is the approximate start rate of Standard workflows?
> → **~2,000 per second**

> What is the start rate of Express workflows?
> → **100,000+ per second**

> How is Standard execution history exposed?
> → **full history, visible in the console for 90 days**

> How is Express execution history exposed?
> → **CloudWatch Logs only**

> How are Standard workflows priced?
> → **per state transition**

> How are Express workflows priced?
> → **per invocation + duration**

> Which workflow type supports `.waitForTaskToken`?
> → **Standard only**

> When both are present, which runs first: Retry or Catch?
> → **Retry runs first; when exhausted, Catch fires**

> Which predefined error matches any error?
> → **`States.ALL`**

> What does `States.Timeout` represent?
> → **a state exceeded its timeout**

> How does `.waitForTaskToken` implement human approval end to end?
> → **the workflow emits a task token to a human, then an external system calls `SendTaskSuccess` with the token to resume**

> Which integration pattern calls and continues immediately?
> → **Request Response (no suffix)**

> What does Distributed Map do?
> → **processes millions of S3 objects with very high parallelism**

> What are Step Functions Activities for?
> → **worker-polled tasks for code running outside AWS (EC2, ECS, on-premises)**

> What is the mental model for an Activity?
> → **Step Functions gives a job to your own external worker**

> Which API does the worker call to ask for a job?
> → **`GetActivityTask`**

> What does `SendTaskHeartbeat` mean?
> → **"I'm still working — don't time me out"**

> Who sends the heartbeat?
> → **the worker application, not Step Functions automatically**

> Which API reports success?
> → **`SendTaskSuccess`**

> Which API reports failure?
> → **`SendTaskFailure`**

> Which API creates the Activity, and who calls it?
> → **`CreateActivity`, called by the developer/setup tooling**

> What does creating an Activity give your worker?
> → **an Activity ARN**

> Does Step Functions run your EC2 command automatically when a workflow reaches an Activity?
> → **No — your worker must poll for and execute the job itself**

> Why can't Express workflows use `.waitForTaskToken`?
> → **Express has a 5-minute cap, so it cannot wait indefinitely**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Orchestration | **Step Functions** |
| Chain of Lambdas calling each other | **replace with Step Functions** |
| Human approval | **`.waitForTaskToken` callback (Standard only)** |
| Wait for a job (ECS/Batch/EMR) | **`.sync` (Run a Job)** |
| Retries without code | **`Retry` / `Catch` in ASL** |
| Branch | **Choice state** |
| Loop over an array | **Map state** |
| Run steps at once | **Parallel state** |
| Pause | **Wait state** |
| Do work | **Task state** |
| Pass data | **Pass state** |
| Terminate | **Succeed / Fail states** |
| Definition language | **Amazon States Language (JSON)** |
| Standard max duration | **1 year** |
| Express max duration | **5 minutes** |
| Standard semantics | **exactly-once** |
| Express semantics | **at-least-once** |
| Standard start rate | **~2,000/s** |
| Express start rate | **100,000+/s** |
| Standard history | **full, 90 days in console** |
| Express history | **CloudWatch Logs only** |
| Standard pricing | **per state transition** |
| Express pricing | **per invocation + duration** |
| Retry vs Catch order | **Retry first, then Catch** |
| Catch all errors | **`States.ALL`** |
| Predefined errors | **ALL, TaskFailed, Timeout, Permissions, DataLimitExceeded** |
| Request Response | **call and continue** |
| Human-approval API | **`SendTaskSuccess` with the task token** |
| Distributed Map | **millions of S3 objects, high parallelism** |
| Activities | **worker-polled tasks (EC2/ECS/on-prem)** |
| Activity model | **SFN gives a job to your worker** |
| `GetActivityTask` | **worker asks for a job** |
| `SendTaskHeartbeat` | **still working** |
| `SendTaskSuccess` | **done** |
| `SendTaskFailure` | **failed** |
| `CreateActivity` | **developer/setup creates it** |
| Heartbeat sender | **the worker, not automatic** |
| Activity output | **an Activity ARN** |
| Payload between states | **256 KB max** |
| Payload over 256 KB | **store in S3, pass the key** |
| Workflow > 5 min | **Standard** |
| Exactly-once required | **Standard** |
| High-volume, short, cheap | **Express** |
