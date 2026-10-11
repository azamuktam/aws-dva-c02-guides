## AWS CodeDeploy and `appspec`

**AWS CodeDeploy:** A managed service that **automates deploying applications** to EC2, Lambda, or ECS.

**appspec.yml:** A file on **in the root directory** that tells CodeDeploy **what files to deploy and which scripts to run during deployment**.

**Exam memory:** **CodeDeploy = deploy; appspec.yml = deployment instructions.**

**Why it exists**

Copying files onto servers is the easy part. Doing it without downtime, stopping the old process cleanly, verifying the new version works, and rolling back automatically when it doesn't — that is deployment. CodeDeploy handles the sequencing and the rollback.

The `appspec` file is where you declare **what to deploy** and **what to run at each stage of the deployment**.

**Compute platforms and file locations**

| Platform | File | Format | Agent required? |
| --- | --- | --- | --- |
| **EC2 / On-premises** | `appspec.yml` at the **root of the bundle** | YAML only | ✅ **CodeDeploy agent** |
| **Lambda** | `appspec.yml` or `.json` | YAML or JSON | ❌ |
| **ECS** | `appspec.yml` or `.json` | YAML or JSON | ❌ |

#### ECS CodeDeploy AppSpec — `resources`

For an **ECS deployment**, the `resources` section needs to identify:

| Property | Purpose |
| --- | --- |
| **`TaskDefinition`** | The ECS task definition to deploy |
| **`ContainerName`** | Container receiving the new task definition |
| **`ContainerPort`** | Port exposed by that container |

**EC2 / On-premises lifecycle hooks**

For **EC2 and on-premises**, the key component is the **CodeDeploy Agent**.

The order mirrors what a deployment physically has to do: stop the old thing, put the new files down, start it, check it worked.

```plain text
ApplicationStop
      ↓
DownloadBundle       ← agent only, no script allowed
      ↓
BeforeInstall
      ↓
Install              ← agent only, no script allowed
      ↓
AfterInstall
      ↓
ApplicationStart
      ↓
ValidateService      ← last chance to fail the deployment
```

With a load balancer (blue/green, or in-place behind an LB), traffic control hooks are added:

```plain text
BeforeBlockTraffic → BlockTraffic → AfterBlockTraffic
        ... the deployment steps above ...
BeforeAllowTraffic → AllowTraffic → AfterAllowTraffic
```

**The `ApplicationStop` subtlety, which is a real exam question:** `ApplicationStop` runs from the **previously deployed revision**, because the new files aren't down yet. So a broken `ApplicationStop` script **blocks all future deployments** until you fix it in the deployed revision or force a new one. If a question describes deployments failing at the very first hook after a bad release, this is why.

**Lambda hooks — only two**

```plain text
BeforeAllowTraffic     ← new version is deployed, NO production traffic yet
        ↓
   AllowTraffic        ← alias weights shift (canary / linear / all-at-once)
        ↓
AfterAllowTraffic      ← traffic is live
```

- Lambda hooks are **Lambda functions**, not shell scripts, and each must report back with **`PutLifecycleEventHookExecutionStatus`** (`Succeeded` or `Failed`). A hook that never reports causes the deployment to hang and then fail.
- **`ValidateService` does not exist for Lambda deployments.** It is one of the most reliable distractors on the exam.

So: *"run integration tests against the new version before any production traffic is shifted"* → **`BeforeAllowTraffic`**.

**ECS hooks**

```plain text
BeforeInstall
      ↓
AfterInstall
      ↓
AfterAllowTestTraffic     ← test listener only, no production traffic
      ↓
BeforeAllowTraffic
      ↓
AfterAllowTraffic
```

ECS deployments with CodeDeploy are **always blue/green** and require an ALB with a production listener, plus optionally a test listener — which is what `AfterAllowTestTraffic` validates against.

**Two separate settings — don't mix them up**

| Setting | Question it answers | Values |
| --- | --- | --- |
| **Deployment type** | *Where* does the new version run? | **In-place** (EC2 / on-prem only) or **Blue/green** |
| **Deployment configuration** | *How fast* does the switch happen? | EC2: `OneAtATime` / `HalfAtATime` / `AllAtOnce` · Lambda & ECS: `AllAtOnce` / `Canary` / `Linear` |

You pick **one of each**. Blue/green is not a speed; canary is not a place.

**Deployment configurations**

| Platform | Configurations |
| --- | --- |
| **EC2 / On-premises** | `CodeDeployDefault.OneAtATime`, `HalfAtATime`, `AllAtOnce`, or custom by count/percentage |
| **Lambda / ECS** | `AllAtOnce`, **`Canary`**, **`Linear`** |

Naming pattern: `CodeDeployDefault.LambdaCanary10Percent5Minutes`, `CodeDeployDefault.LambdaLinear10PercentEvery1Minute`.

**The canary/linear distinction:**

- **Canary** — **two** shifts. A small percentage, a wait, then the remainder.

```plain text
10% → wait 5 min → 90%
```

- **Linear** — **equal** increments on a repeating interval.

```plain text
10% → wait → 20% → wait → 30% → ...
```

Both are **deployment configurations**, and they exist **only for Lambda and ECS** — where the deployment type is always blue/green anyway. They set the speed of the shift, not the type:

```plain text
Lambda / ECS  →  deployment type is always Blue/green
                 └── deployment configuration:
                     ├── AllAtOnce
                     ├── Canary
                     └── Linear

EC2 Blue/green →  deployment configuration:
                     └── OneAtATime / HalfAtATime / AllAtOnce
                         (no Canary, no Linear)
```

### CodeDeploy deployment support

| Compute platform | Deployment type |
| --- | --- |
| **EC2 / on-premises** | ✅ In-place |
| **EC2 / on-premises** | ✅ Blue/green |
| **Lambda** | ✅ Blue/green |
| **ECS** | ✅ Blue/green |

So the key thing is:

- **ECS → Blue/green** ✅
- **Lambda → Blue/green** ✅
- **EC2/on-premises → In-place or Blue/green** ✅

**Blue/green** itself means two environments — blue (current) and green (new) — with traffic shifted from one to the other. It minimises downtime and makes rollback trivial: shift back.

**How Lambda traffic shifting actually works**

Worth connecting explicitly, because it ties two sections together:

- **Versions** are immutable snapshots, so the old working version is always still there to roll back to.
- **Aliases** point at a version and support **weighted routing**.
- **CodeDeploy shifts traffic by adjusting the alias weights** according to the canary or linear configuration.

There is no special deployment machinery — it's alias weights, moved on a schedule.

### **In-place vs blue/green (EC2)**

- **In-place** — update the existing instances. Cheaper, but capacity dips during the deployment and rollback means redeploying the old version.
- **Blue/green** — provision new instances, shift the load balancer, terminate the old ones. Requires an ALB. Faster rollback, higher cost during the deploy.

**Automatic rollback**

Triggers: **deployment failure**, or a **CloudWatch alarm** entering ALARM state.

Mechanically, rollback works by **redeploying the previous revision as a new deployment** — it is not an undo. Which is why a broken `ApplicationStop` in the previous revision can complicate rollback too.

**AfterInstall** - You can use this deployment lifecycle event for tasks such as configuring your application or changing file permissions

**ApplicationStart** - You typically use this deployment lifecycle event to restart services that were stopped during ApplicationStop

**AllowTraffic** - During this deployment lifecycle event, internet traffic is allowed to access instances after a deployment. This event is reserved for the AWS **CodeDeploy** agent and cannot be used to run scripts

**Recall check — CodeDeploy**

- **`appspec.yml`** → CodeDeploy, at the root
- **Run integration tests before production traffic (Lambda)** → **`BeforeAllowTraffic`**
- **Verify after traffic is live** → `AfterAllowTraffic`
- **Test against a test listener before production (ECS)** → `AfterAllowTestTraffic`
- **Last chance to fail an EC2 deployment, verify the success of the deployment** → `ValidateService`
- **`ValidateService` on a Lambda deployment** → **does not exist** (distractor)
- **Stop the app before files are copied** → `ApplicationStop`
- **New deployments failing immediately after a bad release** → broken `ApplicationStop` in the **previous** revision
- **Lambda hooks are Lambda functions** and must call `PutLifecycleEventHookExecutionStatus`
- **In-place vs blue/green** → deployment **type**
- **AllAtOnce / OneAtATime / HalfAtATime / Canary / Linear** → deployment **configuration**
- **Canary or Linear on an EC2 deployment** → **does not exist** (distractor, same family as `ValidateService` on Lambda)
- **10% now, wait, then the rest** → **Canary**
- **Equal increments on a timer** → **Linear**
- **Two environments, shift traffic, easy rollback** → Blue/green
- **How Lambda traffic shifting works** → alias weighted routing, adjusted by CodeDeploy
- **Keep the old version for rollback** → Lambda versions
- **Stable pointer for clients** → Lambda alias
- **Roll back automatically on errors** → CloudWatch alarm + automatic rollback
- **CodeDeploy agent** → EC2 / on-premises only
- **CodeDeploy agent** communicates **AWS CodeDeploy** using **HTTPS over port 443**
- **Lambda/ECS** → no in-place deployment
- **Lambda** → canary/linear/all-at-once

## CodeDeploy Deployment Configurations

**EC2 / On-Premises** — these move **instances**, not traffic:

- `CodeDeployDefault.AllAtOnce` → all instances
- `CodeDeployDefault.HalfAtATime` → \~50% at a time
- `CodeDeployDefault.OneAtATime` → 1 instance at a time

**Lambda** — these move **traffic** (alias weights):

- `CodeDeployDefault.LambdaCanary10Percent5Minutes` → 10% → wait 5 min → 90%
- `CodeDeployDefault.LambdaCanary10Percent10Minutes` → 10% → wait 10 min → 90%
- `CodeDeployDefault.LambdaCanary10Percent15Minutes` → 10% → wait 15 min → 90%
- `CodeDeployDefault.LambdaCanary10Percent30Minutes` → 10% → wait 30 min → 90%
- `CodeDeployDefault.LambdaLinear10PercentEvery1Minute` → +10% every 1 min
- `CodeDeployDefault.LambdaLinear10PercentEvery2Minutes` → +10% every 2 min
- `CodeDeployDefault.LambdaLinear10PercentEvery3Minutes` → +10% every 3 min
- `CodeDeployDefault.LambdaLinear10PercentEvery10Minutes` → +10% every 10 min
- `CodeDeployDefault.LambdaAllAtOnce` → 100% immediately

**ECS** — same shapes as Lambda, but the ALB listener moves the traffic:

- `CodeDeployDefault.ECSCanary10Percent5Minutes` → 10% → wait 5 min → 90%
- `CodeDeployDefault.ECSCanary10Percent15Minutes` → 10% → wait 15 min → 90%
- `CodeDeployDefault.ECSLinear10PercentEvery1Minutes` → +10% every 1 min
- `CodeDeployDefault.ECSLinear10PercentEvery3Minutes` → +10% every 3 min
- `CodeDeployDefault.ECSAllAtOnce` → 100% immediately

**Exam memory:**<br>EC2 = instances<br>Lambda / ECS = traffic<br>Canary = 2 steps<br>Linear = equal steps<br>AllAtOnce = immediately

All of the above are **configurations**. None of them is a deployment *type* — the type is still in-place or blue/green, chosen separately.

## Deployment Strategies Across Services — Consolidated

Every AWS deployment mechanism is answering the same question: **what happens to the old version while the new one goes live?** There are only four possible answers, and every named strategy is one of them.

**The four underlying shapes**

**1. Replace everything at once.** Fastest, cheapest, brief downtime, highest risk. Fine for dev; almost never the answer for production.

**2. Replace in batches (in place).** No full downtime, but the batch being updated is out of service, so **capacity dips**. Rollback means redeploying the old version over the top.

**3. Add new capacity first, then remove the old.** Capacity never drops. Costs more during the deployment. Rollback is easy because the old thing still exists.

**4. Send a fraction of real traffic to the new version, then the rest.** Everything in (3), plus you find out with 10% of users rather than 100%.

**How each service names them**

| Underlying shape | Elastic Beanstalk | CodeDeploy (EC2) | CodeDeploy (Lambda / ECS) |
| --- | --- | --- | --- |
| Replace all at once | **All at once** | `AllAtOnce` | `AllAtOnce` |
| Batches, capacity dips | **Rolling** | `OneAtATime` / `HalfAtATime` | — |
| New capacity first | **Rolling with additional batch** | Blue/green | Blue/green (implicit) |
| Brand-new environment | **Immutable** | Blue/green | Blue/green (implicit) |
| Fraction of live traffic | **Traffic splitting** | Blue/green + `HalfAtATime` | **Canary** / **Linear** |
| Instant switch back | Blue/green **URL (CNAME) swap** | Blue/green + rollback | Move the **alias** back |

In the CodeDeploy columns, **Blue/green** is the deployment *type*; `AllAtOnce`, `OneAtATime`, `HalfAtATime`, `Canary`, `Linear` are deployment *configurations*. Elastic Beanstalk has no such split — each Beanstalk policy is a single setting.

**The properties that distinguish them, one table**

| Property | All at once | Rolling | Rolling + batch | Immutable / Blue-green | Canary / Traffic splitting |
| --- | --- | --- | --- | --- | --- |
| Downtime | Possible | None | None | **None** | **None** |
| Capacity during deploy | Reduced | **Reduced** | Full | Full (extra cost) | Full |
| New instances created | No | No | Some | **Yes, all** | **Yes, all** |
| EC2 burst balance preserved | Yes | Yes | Mostly | **No** | **No** |
| Rollback speed | Slow (redeploy) | Slow | Slow | **Fast** | **Fast** |
| Cost during deploy | Lowest | Low | Slightly higher | **Highest** | Highest |
| Risk | Highest | Medium | Lower | Lowest | **Lowest** |

**Canary vs Linear, precisely**

- **Canary** — **two** shifts. `10% → wait 5 min → 90%`
- **Linear** — **equal** increments on a repeating timer. `10% → wait → 20% → wait → 30% → ...`

Both are **deployment configurations** for Lambda and ECS, whose deployment type is blue/green either way. See the tree in the CodeDeploy section above.

**Where the mechanism actually lives, per compute type**

| Compute | What physically moves the traffic |
| --- | --- |
| **Lambda** | **Alias weighted routing**, adjusted by CodeDeploy |
| **ECS** | **ALB listener** switching between two target groups |
| **EC2 (blue/green)** | **ALB** switching between two Auto Scaling Groups |
| **Elastic Beanstalk (immutable / traffic splitting)** | A second **Auto Scaling Group**, plus load balancer weights |
| **Elastic Beanstalk (blue/green)** | **Route 53 CNAME swap** between two environments |

Once you see that traffic shifting is always "point the router somewhere else", the strategies stop being names to memorise.

**Recall check — deployment strategies**

- **Fastest, accept downtime** → All at once
- **Balance speed and availability, capacity may dip** → Rolling
- **Keep full capacity during the deployment** → Rolling with additional batch
- **Safest, zero downtime, easy rollback** → Immutable (Beanstalk) / Blue-green (CodeDeploy)
- **Percentage of real users on the new version** → Traffic splitting (Beanstalk) / Canary or Linear (CodeDeploy)
- **10% now, wait, then the rest** → **Canary**
- **Equal increments on a timer** → **Linear**
- **Fastest possible rollback in Beanstalk** → Blue/green **CNAME swap**
- **Fastest possible rollback for Lambda** → move the **alias** back a version
- **EC2 burst balances lost after a deploy** → Immutable / Traffic splitting (new instances)
- **Rollback triggered automatically** → CloudWatch alarm + CodeDeploy automatic rollback

---

# Question patterns

> A managed service that automates deploying applications to EC2, Lambda, or ECS. Which service?
> → **AWS CodeDeploy**

> Where are the deployment instructions (what to deploy and which scripts to run) defined?
> → **appspec.yml**

> Where must appspec.yml be placed for an EC2 deployment?
> → **At the root of the deployment bundle**

> Which compute platform requires the CodeDeploy agent?
> → **EC2 and on-premises**

> Do Lambda and ECS deployments require the CodeDeploy agent?
> → **No**

> Which file formats can appspec use for Lambda and ECS?
> → **YAML or JSON**

> Can appspec be JSON for EC2/on-premises?
> → **No — EC2/on-premises appspec is YAML only**

> For an ECS deployment, which appspec resources property identifies the task definition?
> → **TaskDefinition**

> Which ECS appspec resource property names the container that receives the new task definition?
> → **ContainerName**

> Which ECS appspec resource property is the port exposed by that container?
> → **ContainerPort**

> List the EC2/on-premises lifecycle hooks in order.
> → **ApplicationStop → DownloadBundle → BeforeInstall → Install → AfterInstall → ApplicationStart → ValidateService**

> Which EC2 lifecycle events are reserved for the agent and cannot run scripts?
> → **DownloadBundle and Install**

> Which is the last chance to fail an EC2 deployment?
> → **ValidateService**

> Which hook configures the application or changes file permissions?
> → **AfterInstall**

> Which hook typically restarts services stopped during ApplicationStop?
> → **ApplicationStart**

> Which EC2 hook stops the application before files are copied?
> → **ApplicationStop**

> From which revision does ApplicationStop run?
> → **The previously deployed revision**

> A bad ApplicationStop script causes what?
> → **It blocks all future deployments until fixed in the deployed revision or forced**

> Lambda deployments have how many lifecycle hooks, and which?
> → **Two: BeforeAllowTraffic and AfterAllowTraffic**

> Run integration tests against the new Lambda version before any production traffic. Which hook?
> → **BeforeAllowTraffic**

> Verify behavior after traffic is live on a Lambda deployment. Which hook?
> → **AfterAllowTraffic**

> Which lifecycle hook does not exist for Lambda deployments?
> → **ValidateService (a common distractor)**

> Lambda lifecycle hooks are implemented as what?
> → **Lambda functions, not shell scripts**

> How must a Lambda hook report its result?
> → **PutLifecycleEventHookExecutionStatus (Succeeded or Failed)**

> What happens if a Lambda hook never reports its status?
> → **The deployment hangs and then fails**

> Put the ECS lifecycle hooks in order.
> → **BeforeInstall → AfterInstall → AfterAllowTestTraffic → BeforeAllowTraffic → AfterAllowTraffic**

> Which ECS hook validates against a test listener before production traffic?
> → **AfterAllowTestTraffic**

> ECS deployments with CodeDeploy are always which deployment type?
> → **Blue/green**

> What does ECS blue/green require?
> → **An ALB with a production listener (and optionally a test listener)**

> Which setting answers "where does the new version run"?
> → **Deployment type**

> Which setting answers "how fast does the switch happen"?
> → **Deployment configuration**

> What are the deployment type options?
> → **In-place (EC2/on-prem only) or Blue/green**

> Which deployment configurations apply to EC2/on-premises?
> → **OneAtATime / HalfAtATime / AllAtOnce (or custom by count/percentage)**

> Which deployment configurations apply to Lambda and ECS?
> → **AllAtOnce / Canary / Linear**

> Can you use Canary or Linear on an EC2 deployment?
> → **No — they exist only for Lambda and ECS**

> Can you use an in-place deployment on Lambda?
> → **No — Lambda is always blue/green**

> Describe a Canary deployment configuration.
> → **Two shifts: a small percentage, a wait, then the remainder (e.g., 10% → wait 5 min → 90%)**

> Describe a Linear deployment configuration.
> → **Equal increments on a repeating interval (e.g., 10% → wait → 20% → wait → 30% → ...)**

> What does CodeDeployDefault.HalfAtATime do on EC2?
> → **Deploys roughly 50% of instances at a time (moves instances, not traffic)**

> What does CodeDeployDefault.AllAtOnce do?
> → **Switches everything immediately**

> Give the naming pattern for a Lambda canary configuration.
> → **CodeDeployDefault.LambdaCanary10Percent5Minutes**

> Give the naming pattern for a Lambda linear configuration.
> → **CodeDeployDefault.LambdaLinear10PercentEvery1Minute**

> How does Lambda traffic shifting actually work?
> → **CodeDeploy adjusts alias weighted routing on a schedule according to the canary or linear config**

> What makes the old Lambda version available for rollback?
> → **Versions (immutable snapshots)**

> What stable pointer do clients use that CodeDeploy shifts?
> → **A Lambda alias**

> What triggers automatic rollback?
> → **Deployment failure or a CloudWatch alarm entering ALARM state**

> Mechanically, how does rollback work?
> → **It redeploys the previous revision as a new deployment — it is not an undo**

> What is the difference between in-place and blue/green on EC2?
> → **In-place updates existing instances (capacity dips, slower rollback); blue/green provisions new instances and shifts the load balancer (requires an ALB, fast rollback)**

> Which EC2 deployment type requires an ALB?
> → **Blue/green**

> On which protocol and port does the CodeDeploy agent communicate with CodeDeploy?
> → **HTTPS over port 443**

> What physically moves traffic for a Lambda deployment?
> → **Alias weighted routing, adjusted by CodeDeploy**

> What physically moves traffic for an ECS deployment?
> → **An ALB listener switching between two target groups**

> What physically moves traffic for EC2 blue/green?
> → **An ALB switching between two Auto Scaling Groups**

> Which Beanstalk mechanism uses a second Auto Scaling Group plus load balancer weights?
> → **Immutable / traffic splitting**

> Which Beanstalk blue/green mechanism swaps between two environments?
> → **Route 53 CNAME swap**

> What is the fastest rollback for a Lambda deployment?
> → **Move the alias back a version**

> What is the fastest possible rollback in Elastic Beanstalk?
> → **Blue/green CNAME swap**

> After a deploy, EC2 burst balances are lost. Which strategy caused this?
> → **Immutable / traffic splitting (new instances created)**

> Which strategy has the lowest risk and fastest rollback but the highest cost during deploy?
> → **Immutable (Beanstalk) / Blue/green (CodeDeploy)**

> Which strategy is fastest but risks brief downtime and is fine mostly for dev?
> → **All at once**

> Which strategy balances speed and availability but lets capacity dip in the batch being updated?
> → **Rolling / in-place batches**

> Which strategy keeps full capacity during the deployment by adding a batch first?
> → **Rolling with additional batch**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Deployment instructions | **appspec.yml** |
| appspec location | **Root of the bundle** |
| EC2/on-prem appspec format | **YAML only** |
| Lambda/ECS appspec format | **YAML or JSON** |
| Agent required | **EC2 / on-premises only** |
| Agent protocol | **HTTPS over port 443** |
| ECS resources: task def | **TaskDefinition** |
| ECS resources: container | **ContainerName** |
| ECS resources: port | **ContainerPort** |
| EC2 hook order | **ApplicationStop → DownloadBundle → BeforeInstall → Install → AfterInstall → ApplicationStart → ValidateService** |
| Agent-only hooks | **DownloadBundle, Install** |
| Last chance to fail EC2 | **ValidateService** |
| Configure / chmod | **AfterInstall** |
| Restart services | **ApplicationStart** |
| Stop app before copy | **ApplicationStop** |
| ApplicationStop revision | **Previously deployed revision** |
| Bad ApplicationStop | **Blocks all future deployments** |
| Lambda hooks | **BeforeAllowTraffic, AfterAllowTraffic** |
| Lambda integration tests pre-traffic | **BeforeAllowTraffic** |
| Lambda post-traffic verify | **AfterAllowTraffic** |
| Lambda nonexistent hook | **ValidateService** |
| Lambda hook type | **Lambda functions** |
| Lambda hook report call | **PutLifecycleEventHookExecutionStatus** |
| Hook never reports | **Hangs then fails** |
| ECS hook order | **BeforeInstall → AfterInstall → AfterAllowTestTraffic → BeforeAllowTraffic → AfterAllowTraffic** |
| ECS test listener hook | **AfterAllowTestTraffic** |
| ECS deployment type | **Always blue/green** |
| ECS requirement | **ALB with production listener (optional test listener)** |
| Where new version runs | **Deployment type** |
| How fast it switches | **Deployment configuration** |
| EC2 deployment types | **In-place or Blue/green** |
| EC2 configurations | **OneAtATime / HalfAtATime / AllAtOnce** |
| Lambda/ECS configurations | **AllAtOnce / Canary / Linear** |
| Canary on EC2 | **Does not exist** |
| In-place on Lambda | **Does not exist** |
| Canary | **Two shifts (10% → wait → 90%)** |
| Linear | **Equal increments on a timer** |
| Lambda canary example | **CodeDeployDefault.LambdaCanary10Percent5Minutes** |
| Lambda linear example | **CodeDeployDefault.LambdaLinear10PercentEvery1Minute** |
| Lambda traffic shift mechanism | **Alias weighted routing** |
| Rollback source | **Lambda versions (immutable)** |
| Stable client pointer | **Lambda alias** |
| Automatic rollback trigger | **Failed deployment or CloudWatch ALARM** |
| Rollback mechanism | **Redeploy previous revision (not an undo)** |
| In-place | **Same fleet, capacity dips** |
| Blue/green | **New fleet + traffic shift, needs ALB** |
| ECS traffic mechanism | **ALB listener across two target groups** |
| EC2 blue/green mechanism | **ALB across two Auto Scaling Groups** |
| Beanstalk immutable mechanism | **Second ASG + LB weights** |
| Beanstalk blue/green mechanism | **Route 53 CNAME swap** |
| Fastest Lambda rollback | **Move the alias back** |
| Burst balance lost | **Immutable / traffic splitting** |
| Lowest risk / highest deploy cost | **Immutable / Blue/green** |
| Fastest, risky | **All at once** |
| Capacity dips | **Rolling / in-place batches** |
| Full capacity during deploy | **Rolling with additional batch** |
| Percent of real users on new version | **Canary / Linear / Traffic splitting** |
| Last deployment hook (EC2) | **ValidateService** |
