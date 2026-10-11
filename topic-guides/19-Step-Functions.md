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
# AWS Step Functions Activities
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
# APIs

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

> A workflow must orchestrate Lambda calls, retries and a human approval step. Which service?
> → **AWS Step Functions**

> Long-running tasks run on external EC2 workers that report progress. Which feature?
> → **Step Functions Activities (with heartbeats)**

> Branching, parallel execution and dynamic loops are needed. Which states?
> → **Choice, Parallel and Map states**

> Millions of short workflows per month at the lowest cost. Which type?
> → **Express workflows**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Orchestration | **Step Functions** |
| Human approval | **waitForTaskToken callback** |
| Worker tasks | **activities** |
| Parallel | **Parallel / Map states** |
| Retries | **Retry in the state definition** |
| High-rate cheap | **Express workflows** |
