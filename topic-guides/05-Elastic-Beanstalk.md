## AWS Elastic Beanstalk

**Why it exists**
Between “here is a function” (Lambda) and “here is a virtual machine” (EC2) sits a common case: a conventional web application that needs EC2 instances, a load balancer, auto scaling, and monitoring wired together correctly. Assembling that yourself is repetitive. Beanstalk takes a code bundle and builds the whole stack for you — **Platform as a Service**.
Crucially, Beanstalk is not a black box: it creates *ordinary* EC2 instances, ASGs, and load balancers in your account, and you can see and tune them.
```plain text
Application code (.zip / .war / Dockerfile)
        ↓
   Elastic Beanstalk
        ↓
EC2 + Auto Scaling Group + Load Balancer + Security Groups + CloudWatch
```
**Concepts**
- **Application** — the container for versions.
- **Application Version** — a specific uploaded bundle. **Stored in S3.**
- **Environment** — the running collection of AWS resources for one version.
In the **Elastic Beanstalk Console**, you'll often see **configuration presets** like:
- **Single instance**
- **High availability**
These are **presets**, not official environment type names.
Under the hood:
- **Single instance preset** → creates a **Single-instance environment**.
- **High availability preset** → creates a **Load-balanced, scalable environment** with recommended production settings.
### AWS Developer Associate exam
Know these two environment types:
- ✅ **Single-instance environment**
  - 1 EC2
  - No Load Balancer
  - No Auto Scaling beyond 1 instance
  - Lower cost
  - Development/testing
- ✅ **Load-balanced, scalable environment (High Availability)**
  - Elastic Load Balancer
  - Auto Scaling Group
  - Multiple EC2 instances
  - Can span multiple Availability Zones
  - Recommended for production
**Environment types:**
| Environment | Use |
| --- | --- |
| **Web server environment** | Handles normal HTTP/web requests |
| **Worker environment** | Processes background jobs from **SQS** |
| **Single-instance environment** | One EC2 instance; mainly for simple/dev workloads |
- **Web Server environment** — handles HTTP requests. `User → Load Balancer → EC2 → Application`
- **Worker environment** — processes background jobs from an **SQS queue**. `SQS → Worker → background processing`
  A **SQS daemon (****`sqsd`****)** runs on each worker instance. It polls the queue and **POSTs each message to your application over local HTTP** (`localhost` by default). Your code therefore never talks to SQS directly — it just handles an HTTP request. A `200` response means success and the daemon deletes the message; a non-`200` means failure and the message returns to the queue, eventually landing in the DLQ.
  **Periodic tasks** are configured with a **`cron.yaml`** file at the root of the application bundle — this is how you get scheduled work in a worker environment without EventBridge:
```yaml
version:1
cron:
-name:"nightly-report"
url:"/generate-report"
schedule:"0 2 * * *"
```
Web server environments ignore `cron.yaml`; it only works in a worker environment.
Splitting a slow task into a worker environment is the standard Beanstalk answer for “long-running background job shouldn’t block the web request”.
**Supported platforms:** Java, .NET, PHP, Node.js, Python, Ruby, Go, Docker.
**Deployment methods:** AWS Console, EB CLI, CodePipeline.
**Environment health:** Green (healthy), Yellow (warning), Red (severe).
**`.ebextensions`**** — customising the environment**
A folder in your source bundle containing `.config` files (YAML or JSON) that Beanstalk applies during deployment.
Naming convention: `.ebextensions/<something>.config`
Can configure environment variables, EC2 settings, Auto Scaling, load balancer settings, and arbitrary CloudFormation resources.
**The persistent-resource trap**
**Resources created by ****`.ebextensions`**** are tied to the environment lifecycle — they are deleted when the environment is deleted or rebuilt.**
That leads directly to the rule:
- **Databases must be created *outside* Beanstalk** (a standalone RDS instance) and connected via environment variables. Otherwise rebuilding the environment destroys your production data.
- **Temporary/recreatable resources** like an ElastiCache cluster are acceptable inside `.ebextensions`, because losing them costs nothing but a cache warm-up.
This isn’t arbitrary — it follows from “the resource dies with the environment”.
| File | Simple meaning |
| --- | --- |
| `env.yaml` | Environment settings/configuration |
| `.ebextensions/*.config` | Configure the underlying EC2 instances |
| `cron.yaml` | Scheduled jobs for **worker environments** |
| `Dockerrun.aws.json` | Tell EB how to run a **Docker container** |
| `Procfile` | Tell EB **how to start the app** |
| `.ebignore` | Files to **exclude from deployment** |
**Beanstalk deployment policies**
These stay here, with the rest of Beanstalk, so the service is in one place. The cross-service comparison against CodeDeploy’s strategies lives in **Part 5**, because that is where the exam pits them against each other.
The five policies trade off speed, downtime, capacity, cost, and rollback safety. Once you see what each one does to the instances, the properties follow.
**All at once** — deploy to every instance simultaneously.
```plain text
Old Old Old  →  New New New
```
Fastest. Brief downtime while all instances restart. Highest risk. Dev/test environments.
**Rolling** — deploy in small batches. Some instances serve the old version while others serve the new one.
```plain text
Old Old Old
New Old Old
New New Old
New New New
```
No full downtime, but **capacity is temporarily reduced** because the batch being updated is out of service. If a batch fails you can be left with unhealthy instances; if those are terminated, EC2 goes `shutting-down` → `terminated`, the ASG replaces them, and the replacements receive the current application version.
**Rolling with additional batch** — launch extra instances *first*, so full capacity is maintained throughout, then remove the old ones.
```plain text
Add:     New New New
Existing: Old Old Old
Then remove the old batch
```
Costs slightly more (extra instances during the deploy) but capacity never drops.
**Immutable** — build a **completely new Auto Scaling Group** at full capacity with the new version. The old ASG is untouched until health checks pass.
```plain text
Old ASG:  EC2 EC2 EC2   (still serving)
New ASG:  EC2 EC2 EC2   (being validated)

Failure → delete the new ASG, old instances untouched
```
Zero downtime, no capacity reduction, **safest rollback**, highest cost during the deploy.
**Traffic splitting** — like immutable, but sends only a *percentage* of live traffic to the new instances. This is Beanstalk’s **canary**.
```plain text
Old version → 90%
New version → 10%
    ↓ (validated)
New version → 100%
```
| Policy | Instances | Downtime | Capacity | Risk |
| --- | --- | --- | --- | --- |
| All at once | Replace all together | Possible | Reduced | Highest |
| Rolling | Replace in batches | Minimal | Reduced temporarily | Medium |
| Rolling + additional batch | Extra instances first | Minimal | **Full** | Lower |
| Immutable | Brand-new ASG | **None** | Full (extra cost) | **Lowest** |
| Traffic splitting | New ASG + weighted traffic | **None** | Full | Lowest + canary |
**Blue/green in Beanstalk** is a separate approach: clone the environment, deploy to the clone, then **swap environment URLs (CNAME swap)**. This gives the fastest possible rollback — swap back — and is what you use when you need to change something Beanstalk can’t do in place.
**Blue Environment:** Acts as the current live production system serving active user requests.**Green Environment:** Hosts the updated software version where final deployment verification and tests run in isolation.
**Burst balance = saved CPU credits.**
Simple example:
```plain text
Use little CPU
     ↓
Save credits 💰
     ↓
Suddenly need lots of CPU
     ↓
Spend credits
     ↓
Temporarily use more CPU
```
So:
> **Burst balance tells you how many CPU credits you have saved for temporary CPU bursts.**
**High balance** → you have more burst capacity.
**Low/zero balance** → you can't sustain CPU usage above the baseline for long.
**The burst balance question**
*“After a test deployment, EC2 burst balances were lost. Which deployment policies can cause this?”*
**Immutable** and **Traffic splitting.**
The reasoning: both create **brand-new EC2 instances**. Burstable instances (T-family) accumulate CPU credits over time. A new instance starts with only its launch credit balance, so all accumulated credits are gone.
Rolling, rolling with additional batch, and all-at-once reuse the existing instances (or replace only some), so they don’t wipe the fleet’s credits the same way.
**Elastic Beanstalk Worker Environment:** Handles long-running/background tasks asynchronously so the web application stays responsive.
**How:**
Web app → **SQS queue** → Worker Environment → process task. Elastic Beanstalk manages the queue and worker processes for you.
**Scaling:** Uses **EC2 Auto Scaling + CloudWatch** to add/remove worker instances based on workload.
**Recall check — Elastic Beanstalk**
| Service | Simple meaning | You manage | Best clue |
| --- | --- | --- | --- |
| **Elastic Beanstalk** | Deploy an application easily; AWS manages the infrastructure | Mostly your app | **Easiest / simplest deployment** |
| **ECS** | AWS container orchestration | Containers/tasks; compute can be EC2 or Fargate | **Run/manage Docker containers** |
| **Fargate** | Serverless compute **for containers** | Containers only, not servers | **No EC2/server management** |
- **Deploy an application without managing infrastructure / PaaS** → Elastic Beanstalk
- **Automatic scaling + load balancing + deployment, minimal effort** → Elastic Beanstalk
- **Need full EC2 control** → EC2
- **Need container orchestration** → ECS/EKS
- **Background processing with SQS** → Worker environment
- **Where are application versions stored** → **S3**
- **Customise the environment** → `.ebextensions`
- **Persistent database that must survive a rebuild** → create RDS **externally**, connect via environment variables
- **Temporary cache that can be recreated** → `.ebextensions` is acceptable
- **Fastest deployment** → All at once
- **Balance speed and availability** → Rolling
- **Keep full capacity during deployment** → Rolling with additional batch
- **Safest, zero downtime, easy rollback** → Immutable
- **Canary in Beanstalk / percentage of users** → Traffic splitting
- **Fastest possible rollback** → Blue/green with URL (CNAME) swap
- **New EC2 instances created** → Immutable / Traffic splitting
- **EC2 burst balances lost** → Immutable / Traffic splitting
- **Cheapest deployment** → All at once
- **Elastic Beanstalk + long-running/background tasks** → Worker Environment + SQS
- Elastic Beanstalk + X-Ray → `.ebextensions/xray-daemon.config`
- **`eb init`** → initialize/configure your Elastic Beanstalk application locally
- **`eb create`** → create an Elastic Beanstalk environment
- **`eb deploy`** → deploy your application to that environment
- **`aws elasticbeanstalk update-application`** → modifies **Elastic Beanstalk application metadata/configuration**.
- **Beanstalk = easiest app deployment**
- **ECS = container orchestration**
- **Fargate = serverless container compute**
