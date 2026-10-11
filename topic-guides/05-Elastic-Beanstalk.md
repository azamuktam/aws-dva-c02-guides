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

  A **SQS daemon (`sqsd`)** runs on each worker instance. It polls the queue and **POSTs each message to your application over local HTTP** (`localhost` by default). Your code therefore never talks to SQS directly — it just handles an HTTP request. A `200` response means success and the daemon deletes the message; a non-`200` means failure and the message returns to the queue, eventually landing in the DLQ.

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

**`.ebextensions` — customising the environment**

A folder in your source bundle containing `.config` files (YAML or JSON) that Beanstalk applies during deployment.

Naming convention: `.ebextensions/<something>.config`

Can configure environment variables, EC2 settings, Auto Scaling, load balancer settings, and arbitrary CloudFormation resources.

**The persistent-resource trap**

**Resources created by `.ebextensions` are tied to the environment lifecycle — they are deleted when the environment is deleted or rebuilt.**

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

# Question patterns

> A team must deploy a conventional web application and have AWS assemble EC2, a load balancer, auto scaling, security groups, and CloudWatch with minimal effort. Which service?
> → **AWS Elastic Beanstalk (a Platform as a Service)**

> Is Elastic Beanstalk a black box that hides the underlying resources?
> → **No — it creates ordinary EC2 instances, ASGs, and load balancers in your account that you can see and tune**

> Where are Elastic Beanstalk application versions stored?
> → **In Amazon S3**

> What is the difference between an application, an application version, and an environment in Beanstalk?
> → **The application is the container for versions; a version is a specific uploaded bundle; an environment is the running collection of AWS resources for one version**

> A team needs a lower-cost Beanstalk setup for development and testing. Which environment type?
> → **A single-instance environment (one EC2, no load balancer, no scaling beyond 1)**

> A production Beanstalk workload needs a load balancer, an Auto Scaling group, and multiple AZs. Which environment type?
> → **A load-balanced, scalable environment (High Availability)**

> Are "Single instance" and "High availability" official environment type names in the Beanstalk console?
> → **No — they are configuration presets that create a single-instance or load-balanced scalable environment**

> An Elastic Beanstalk app must process long-running background jobs from a queue. Which setup?
> → **A worker environment driven by SQS**

> In a worker environment, what is sqsd and what does it do?
> → **The SQS daemon running on each worker instance; it polls the queue and POSTs each message to your application over local HTTP (localhost), deleting the message on a 200 and returning it to the queue on a non-200**

> In a worker environment, does application code talk to SQS directly?
> → **No — sqsd polls SQS and delivers each message as a local HTTP request; the code just handles the request**

> A worker returns a non-200 response to sqsd. What happens to the message?
> → **It returns to the queue and eventually lands in the DLQ**

> How are periodic tasks configured in a Beanstalk worker environment?
> → **A cron.yaml file at the root of the application bundle (e.g. schedule "0 2 * * *")**

> Does cron.yaml work in a web server environment?
> → **No — it only works in a worker environment; web server environments ignore it**

> A slow task is blocking web requests. What is the standard Beanstalk fix?
> → **Move the task to a worker environment (web app → SQS → worker) so the web tier stays responsive**

> Which platforms does Elastic Beanstalk support?
> → **Java, .NET, PHP, Node.js, Python, Ruby, Go, and Docker**

> Which deployment methods does Elastic Beanstalk support?
> → **AWS Console, EB CLI, and CodePipeline**

> What are the Elastic Beanstalk environment health colours?
> → **Green (healthy), Yellow (warning), Red (severe)**

> How do you customise the underlying EC2 instances and platform configuration of a Beanstalk environment?
> → **.ebextensions configuration files (.ebextensions/<something>.config, YAML or JSON)**

> A production RDS database is created inside .ebextensions. Why is this dangerous?
> → **Resources created by .ebextensions are tied to the environment lifecycle and are deleted when the environment is deleted or rebuilt**

> Where should a database for a Beanstalk application be created, and why?
> → **Outside Beanstalk as a standalone RDS instance connected via environment variables, so rebuilding the environment does not destroy production data**

> Which temporary resource is acceptable to create inside .ebextensions?
> → **A recreatable one such as an ElastiCache cluster, since losing it only costs a cache warm-up**

> Which file tells Elastic Beanstalk how to run a Docker container?
> → **Dockerrun.aws.json**

> Which file tells Elastic Beanstalk how to start the app?
> → **Procfile**

> Which file excludes files from the deployment bundle?
> → **.ebignore**

> A deployment must have zero downtime, roll out on a fresh Auto Scaling group, and allow the safest rollback. Which policy?
> → **Immutable — it builds a brand-new ASG at full capacity and leaves the old ASG untouched until health checks pass**

> Which deployment policy is the fastest but causes brief downtime and is best for dev/test?
> → **All at once (deploys to every instance simultaneously)**

> Which deployment policy keeps full capacity by launching extra instances first, then removing the old batch?
> → **Rolling with additional batch**

> Which deployment policy deploys in small batches with no full downtime but temporarily reduced capacity?
> → **Rolling**

> Only a small percentage of live users should test the new version. Which Beanstalk policy?
> → **Traffic splitting (Beanstalk's canary)**

> Which policy reintroduces new EC2 instances and therefore loses T-family CPU burst balances?
> → **Immutable and Traffic splitting (both create brand-new instances)**

> A team wants to keep the old environment live and switch URLs when the new one is ready, with the fastest possible rollback. Which approach?
> → **Blue/green via environment clone and CNAME (URL) swap**

> In blue/green Beanstalk, what are the blue and green environments?
> → **Blue is the current live production system; green hosts the updated version for isolated verification and tests**

> What does the EC2 burst balance represent?
> → **The number of saved T-family CPU credits available for temporary bursts above baseline**

> A burstable instance has a low or zero burst balance. What does that mean?
> → **It cannot sustain CPU usage above the baseline for long**

> Which CLI command initialises and configures a Beanstalk application locally?
> → **eb init**

> Which CLI command creates a Beanstalk environment?
> → **eb create**

> Which CLI command deploys your application to a Beanstalk environment?
> → **eb deploy**

> Which API call modifies Elastic Beanstalk application metadata/configuration?
> → **aws elasticbeanstalk update-application**

> How is X-Ray enabled for an Elastic Beanstalk environment?
> → **Via .ebextensions/xray-daemon.config**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Easiest app deployment / PaaS | **Elastic Beanstalk** |
| Application versions stored | **S3** |
| App / version / environment | **Container / uploaded bundle / running resources** |
| Dev/test environment | **Single-instance (no LB, no scaling)** |
| Production environment | **Load-balanced, scalable (HA)** |
| Console presets | **Single instance / High availability (not official type names)** |
| Web requests | **Web server environment** |
| Background jobs from a queue | **Worker environment** |
| Worker queue polling | **sqsd (SQS daemon) over local HTTP** |
| Worker success signal | **HTTP 200 → message deleted** |
| Worker failure signal | **Non-200 → message returns, then DLQ** |
| Periodic tasks | **cron.yaml (worker only)** |
| Long task blocking web tier | **Move to worker environment + SQS** |
| Supported platforms | **Java, .NET, PHP, Node.js, Python, Ruby, Go, Docker** |
| Deployment methods | **Console, EB CLI, CodePipeline** |
| Environment health colours | **Green / Yellow / Red** |
| Customise environment | **.ebextensions/*.config** |
| Persistent DB survival | **Create RDS externally, connect via env vars** |
| .ebextensions lifecycle | **Resources deleted with the environment** |
| Recreatable resource in .ebextensions | **ElastiCache (acceptable)** |
| Docker config file | **Dockerrun.aws.json** |
| How to start the app | **Procfile** |
| Exclude files from deploy | **.ebignore** |
| Environment settings file | **env.yaml** |
| Fastest / cheapest deployment | **All at once (brief downtime)** |
| Balance speed and availability | **Rolling** |
| Keep full capacity | **Rolling with additional batch** |
| Zero downtime, safest rollback, fresh ASG | **Immutable** |
| Canary / percentage of users | **Traffic splitting** |
| Fastest possible rollback | **Blue/green CNAME swap** |
| Blue environment | **Current live production** |
| Green environment | **New version verified in isolation** |
| Burst balances lost | **Immutable / Traffic splitting (new instances)** |
| Burst balance | **Saved CPU credits for temporary bursts** |
| Low burst balance | **Cannot sustain above-baseline CPU long** |
| X-Ray config | **.ebextensions/xray-daemon.config** |
| eb init | **Initialise/configure app locally** |
| eb create | **Create an environment** |
| eb deploy | **Deploy to the environment** |
| update-application | **Modifies EB application metadata** |
