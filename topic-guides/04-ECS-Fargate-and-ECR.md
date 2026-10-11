## Amazon ECS, Fargate, and ECR

**Mental model:**

```plain text
Code
  ↓
Docker Image
  ↓
ECR (stores images)
  ↓
ECS (runs containers)
  ↓
Fargate (serverless compute for ECS)
```

#### ECR (Elastic Container Registry)

- **Purpose:** Store Docker/container images.
- **Think:** *Docker Hub for AWS.*

#### ECS (Elastic Container Service)

- **Purpose:** Orchestrates and runs Docker containers.
- **Think:** *AWS service that manages your containers.*

The developer terminated the **EC2 instance** that was running ECS.

But ECS may still show the **container instance as a resource in the ECS cluster** because ECS keeps the container-instance registration until it is **deregistered**.

Think:

```plain text
ECS Cluster
    ↓
ECS Container Instance
    ↓
EC2 Instance
```

Terminating the EC2 instance:

```plain text
EC2 ❌ terminated
ECS registration ⚠️ can still exist
```

**What should be done?**

**Deregister the container instance from the ECS cluster.**

If you terminate **a RUNNING ECS **container instance, ECS automatically deregisters it**.**

If you terminate** a STOPPED container instance, **ECS does NOT automatically deregister it.

You can do this through the ECS console or API/CLI.

After deregistration, it will no longer appear as an active container instance in the cluster.

**Important distinction**

**EC2 instance** = the actual virtual machine.

**ECS container instance** = the EC2 instance **registered with an ECS cluster** and managed by the ECS agent.

So:

> **Terminate EC2 ≠ automatically remove the ECS container-instance registration immediately.**

**Exam memory:**

> **EC2 terminated but still appears in ECS cluster → deregister the ECS container instance.**

### ECS Task Placement

**Task placement** = **how ECS decides which EC2 instance should run a new task**. You can control it with **placement strategies** and **placement constraints**.

- **`binpack`** → pack tasks together → **minimize instances/cost**
  - `cpu` or `memory`
- **`spread`** → distribute tasks evenly → **high availability**
  - `instanceId` / AZ
- **`random`** → place tasks **randomly**

**Constraint** = a **rule** that must be satisfied, e.g. `distinctInstance` = don't put two tasks on the same instance.

**Exam memory:**

**Binpack = together**

**Spread = apart**

**Random = random**

#### ECS Cluster Name in CloudFormation

**Problem:** New EC2 instances join the old ECS cluster.

**Cause:** The ECS agent uses `ECS_CLUSTER` to decide which cluster the EC2 instance joins. This is commonly set in EC2 User Data / `/etc/ecs/ecs.config`.

```plain text
ECS_CLUSTER=MainCluster
        ↓
EC2 joins MainCluster
```

If CloudFormation parameter changes to `SecondCluster` but User Data still says:

```plain text
ECS_CLUSTER=MainCluster
```

the new instances still join **MainCluster**.

**Fix:** Pass the CloudFormation cluster parameter into User Data:

```plain text
ECS_CLUSTER=${ClusterName}
```

Then:

```plain text
ClusterName = MainCluster   → ECS_CLUSTER=MainCluster
ClusterName = SecondCluster → ECS_CLUSTER=SecondCluster
```

**Exam memory:**

> **ECS EC2 instance joins cluster based on `ECS_CLUSTER`.**
> **CloudFormation parameter must be passed to the ECS agent configuration.**

### Cluster Query Language (CQL)

**What it is:** ECS expressions used to **filter/group container instances by attributes**.

**Examples:**

```plain text
attribute:ecs.availability-zone == us-east-1a
attribute:ecs.instance-type == t3.large
```

| ECS feature | What it does | Example |
| --- | --- | --- |
| **Cluster Query Language (CQL)** | Finds **which container instances** match attributes | `attribute:ecs.availability-zone == us-east-1a` |
| **Task Placement Constraints** | Says **where a task is allowed/not allowed** to run | `memberOf(attribute:ecs.instance-type == t3.large)` |
| **Task Placement Strategies** | Says **how to distribute tasks** among eligible instances | `spread` across AZs, or `binpack` by memory |
| **Task Groups** | Groups related ECS tasks logically | `serviceA` has several related tasks running together |

### Dynamic Host Port Mapping

**What it is:**

ECS automatically chooses an available **host port** for a container instead of you choosing one manually. This is supported with the **bridge network mode**.

Example:

```plain text
EC2 instance

Task 1: container port 80 → host port 32768
Task 2: container port 80 → host port 32769
Task 3: container port 80 → host port 32770
```

All 3 containers can therefore run on **the same EC2 instance**, even though they all use container port `80`.

For example:

```plain text
EC2:80
  ↓
API A ✅

EC2:80
  ↓
API B ❌ port already occupied
```

That's why ECS dynamic port mapping is useful:

```plain text
EC2
├── :32768 → API A → container :80
├── :32769 → API B → container :80
└── :32770 → API C → container :80
```

#### Fargate

- **Purpose:** Serverless compute engine for ECS (and EKS).
- **Think:** *Run containers without managing EC2 servers.*

**Exam shortcut:**

- **ECR = **Store images
- **ECS = **Run/manage containers
- **Fargate = **No servers to manage
- **DELETING STOPPED** = not deregistered
- **DELETING RUNNING** = deregistered
- **Cluster Query Language** → query/group container instances by attributes
- **Task Placement Constraints** → rules that say where a task may/may not be placed
- **Task Placement Strategies** → decide **how tasks are distributed**, e.g. `spread`, `binpack`
- **Task Groups** → groups of related tasks, not a mechanism for querying instances

**Why they exist**

Containers solve the “works on my machine” problem by packaging the application with its dependencies. But someone still has to decide *where* each container runs, restart it when it dies, and connect it to a load balancer. That is orchestration, and ECS is AWS’s orchestrator. **Fargate** removes the last piece of infrastructure — you no longer manage the hosts either.

```plain text
Task Definition  (the recipe: image, CPU, memory, ports, roles, secrets, logging)
        ↓
     Task  (a running instance of the definition)
        ↓
   Service  (keeps N tasks running, registers them with a load balancer)
        ↓
  Launch type:  EC2 (you manage hosts)  |  Fargate (AWS manages hosts)
```

**Fargate vs ECS on EC2**

|    | **Fargate** | **ECS on EC2** |
| --- | --- | --- |
| Hosts | AWS manages | You manage, patch, scale |
| Billing | Per vCPU and GB requested | Per EC2 instance, whether used or not |
| Control | No host access | Full host access, GPU, custom AMIs |
| Density optimisation | None available | You can pack many tasks per instance |

Fargate is the default answer to “serverless containers” and “no EC2 to manage”. ECS on EC2 wins when you need host-level control or better bin-packing economics.

**Task role vs task execution role — the most-tested container fact**

There are two roles because there are two *actors*, operating at two different times.

```plain text
Task Definition
   ├── executionRoleArn  →  used by the ECS agent / Fargate infrastructure
   │                        BEFORE and WHILE starting the task:
   │                        pull the image from ECR,
   │                        write to CloudWatch Logs,
   │                        fetch values in the `secrets` block
   │
   └── taskRoleArn       →  used by YOUR APPLICATION CODE at runtime:
                            S3, DynamoDB, SQS, SNS, any AWS API
```

The symptom tells you which one is wrong, and this is derivable rather than memorised:

- If the **task never starts** — `CannotPullContainerError`, or no logs appear at all — the failure happened *before your code ran*. That is the **execution role**.
- If the task runs fine and then your code gets **`AccessDenied`** calling S3, the failure happened *inside your code*. That is the **task role**.

|    | Execution role | Task role |
| --- | --- | --- |
| Actor | ECS agent / Fargate | Your application |
| Timing | Task startup | Task runtime |
| Grants | ECR pull, CloudWatch Logs, secrets injection | Your app’s AWS API calls |
| Managed policy | `AmazonECSTaskExecutionRolePolicy` | You write it |
| Failure symptom | Task won’t start, no logs | `AccessDenied` at runtime |

On the **EC2 launch type** there is a third role: the **instance profile**, used by the ECS container agent on the host. Fargate has no instance profile.

An **EC2 instance profile** grants infrastructure-level permissions to the virtual machine itself so the underlying [Amazon ECS](https://aws.amazon.com/ecs) container agent can communicate with AWS. In contrast, an **ECS task role **grants application-level permissions specifically to the containers running inside a single task definition, allowing your code to **access services like S3 or DynamoDB**. \[[1](https://www.youtube.com/watch?v=wpFJekMhYLk&t=654), [2](https://issackpaul95.medium.com/aws-theory-deep-dive-into-amazon-ecs-launch-types-iam-roles-load-balancers-and-data-43088ceaf991), [3](https://www.youtube.com/shorts/VQfRQ9gGYD4)\]

Credentials reach the container through the **task metadata endpoint** (`169.254.170.2`). The AWS SDKs discover it automatically. This is why access keys never need to appear in a task definition — and why any answer suggesting they should is wrong.

**Storage for containers**

Ephemeral task storage disappears when the task stops. For anything durable or shared:

- **EFS** — shared, persistent, multi-AZ, mountable by many tasks at once. This is the Fargate answer.
- **EBS** — single AZ, normally one instance. Wrong answer for “shared across Availability Zones”.

**ECR — the registry**

- Private Docker registry, integrated with IAM for authorisation.
- Login: `aws ecr get-login-password | docker login --username AWS --password-stdin <registry>`. The token is valid **12 hours**.
- **Lifecycle policies** expire old images automatically so the registry doesn’t grow forever.
- **Image scanning** (basic, or enhanced via Inspector) finds CVEs in images.

### ECS Components

| Component | What it does | Think |
| --- | --- | --- |
| **Task definition** | Defines **how containers run**: image, CPU/memory, ports, volumes/EFS, environment variables, IAM roles | **Blueprint** |
| **Service scheduler** | Keeps the desired number of tasks running and replaces failed tasks | **Keeps tasks running** |
| **Container instance** | The **EC2 infrastructure** where ECS tasks run when using EC2 launch type | **The machine** |
| **Container Agent** | Runs on the container instance and communicates with ECS; starts/stops tasks as instructed | **Agent on the machine** |

AWS describes the task definition as the application blueprint, while the service scheduler maintains the desired tasks. The container agent runs on each container instance and handles instructions from ECS.

**Exam shortcut:**

> **Task Definition** → *What/how should the container run?*

  **Service Scheduler** → *How many tasks should stay running?*

  **Container Instance** → *Where does the task run?*

  **Container Agent** → *Who communicates with ECS from the EC2 instance?*

**Recall check — containers**

- **Serverless containers, no EC2 to manage** → Fargate
- **Need host access, GPUs, or custom AMIs** → ECS on EC2
- **Need Kubernetes** → EKS
- **`CannotPullContainerError`** → task **execution** role
- **No container logs in CloudWatch** → task **execution** role
- **App gets `AccessDenied` calling S3** → task **role**
- **Secrets injected from the task definition** → task **execution** role
- **Never store credentials in a task definition** → use roles; credentials arrive via the metadata endpoint
- **Persistent shared storage across AZs for Fargate** → **EFS**
- **Temporary container storage** → ephemeral task storage
- **Stop old images accumulating** → ECR lifecycle policy
- **Find vulnerabilities in container images** → ECR image scanning / Inspector

---

# Question patterns

> A team needs to store and version private Docker container images inside AWS, integrated with IAM. Which service?
> → **Amazon ECR (Elastic Container Registry)**

> A service orchestrates and runs Docker containers across a cluster of hosts. Which service?
> → **Amazon ECS (Elastic Container Service)**

> A developer containerises a Fargate application and is charged only for the vCPU and memory the tasks request. Which launch type?
> → **Fargate — serverless compute for containers, per vCPU and GB requested**

> A team needs serverless containers with no EC2 hosts to manage. Which launch type?
> → **Fargate**

> A workload needs host access, GPUs, custom AMIs, or denser bin-packing economics. Which launch type?
> → **ECS on EC2**

> A team needs Kubernetes rather than the ECS orchestrator. Which service?
> → **Amazon EKS**

> A developer terminates the EC2 instance hosting an ECS task, but the container instance still appears in the cluster. Why?
> → **ECS keeps the container-instance registration until it is deregistered**

> An EC2 instance running ECS was terminated while in the STOPPED state and still shows as an active container instance. What must be done?
> → **Deregister the container instance from the ECS cluster (ECS auto-deregisters only RUNNING container instances)**

> Is terminating an EC2 instance the same as removing the ECS container-instance registration?
> → **No — ECS keeps the registration until it is deregistered; only a RUNNING instance is auto-deregistered**

> A team wants to minimise the number of EC2 instances and cost by packing tasks tightly. Which placement strategy?
> → **binpack (by cpu or memory)**

> A team wants tasks distributed evenly across instances or AZs for high availability. Which placement strategy?
> → **spread**

> Placement should be arbitrary with no ordering requirement. Which strategy?
> → **random**

> A rule must prevent two tasks from running on the same instance. Which mechanism?
> → **A placement constraint (distinctInstance)**

> New EC2 instances keep joining the old ECS cluster after a CloudFormation parameter change. What is the cause?
> → **The ECS agent uses ECS_CLUSTER in EC2 user data / /etc/ecs/ecs.config, which was not updated**

> How do you fix new instances joining the wrong ECS cluster in CloudFormation?
> → **Pass the CloudFormation cluster parameter into user data (ECS_CLUSTER=${ClusterName})**

> A team must filter or group container instances by attributes such as AZ or instance type. Which ECS feature?
> → **Cluster Query Language (CQL), e.g. attribute:ecs.availability-zone == us-east-1a**

> Which ECS mechanism states where a task is allowed or not allowed to run?
> → **Task placement constraints (e.g. memberOf(attribute:ecs.instance-type == t3.large))**

> Which ECS mechanism decides how tasks are distributed among eligible instances?
> → **Task placement strategies (spread, binpack, random)**

> Several tasks must each expose container port 80 on the same EC2 instance behind an ALB. What solves the port conflict?
> → **Dynamic host port mapping — ECS assigns an available host port per task (bridge network mode)**

> Which network mode is required for dynamic host port mapping?
> → **bridge**

> In dynamic host port mapping, how do three tasks on one instance avoid colliding on container port 80?
> → **ECS maps each to a different host port (e.g. 32768, 32769, 32770)**

> An ECS task fails with CannotPullContainerError and no logs appear at all. Which role must grant permission?
> → **The task execution role (it pulls the image and writes logs before your code runs)**

> Which role gives the running application permission to call AWS services like S3 or DynamoDB?
> → **The task role**

> A task starts fine and logs appear, but the application gets AccessDenied calling S3. Which role is missing permissions?
> → **The task role (runtime permissions for your code)**

> Which managed policy is typically attached to the ECS task execution role?
> → **AmazonECSTaskExecutionRolePolicy**

> Which role is used by the ECS agent / Fargate to inject values from the secrets block?
> → **The task execution role**

> On the EC2 launch type, which additional role does the host's ECS container agent use?
> → **The EC2 instance profile — Fargate has no instance profile**

> How do credentials reach a container, and what endpoint is used?
> → **Through the task metadata endpoint 169.254.170.2, which the AWS SDKs discover automatically**

> Should access keys ever be stored in an ECS task definition?
> → **No — use IAM roles; credentials arrive via the task metadata endpoint**

> Fargate tasks need shared, persistent storage across Availability Zones. Which service?
> → **Amazon EFS (EBS is single-AZ and normally one instance — the wrong answer for shared)**

> What happens to ephemeral task storage when a task stops?
> → **It disappears — use EFS for anything durable or shared**

> How long is an ECR login token valid?
> → **12 hours (aws ecr get-login-password | docker login --username AWS --password-stdin <registry>)**

> A team must stop old images accumulating in ECR forever. Which feature?
> → **ECR lifecycle policies**

> A team must find CVEs in container images. Which feature?
> → **ECR image scanning (basic, or enhanced via Amazon Inspector)**

> Which ECS component defines how containers run — image, CPU/memory, ports, volumes, environment variables, IAM roles?
> → **The task definition (the blueprint)**

> Which ECS component keeps the desired number of tasks running and replaces failed tasks?
> → **The service scheduler**

> Which ECS component is the EC2 infrastructure where tasks run on the EC2 launch type?
> → **The container instance (the machine)**

> Which ECS component runs on the container instance and communicates with ECS to start/stop tasks?
> → **The container agent**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Store images | **ECR (Docker Hub for AWS)** |
| Run/manage containers | **ECS** |
| Serverless containers | **Fargate** |
| Kubernetes | **EKS** |
| Host access / GPU / custom AMI | **ECS on EC2** |
| Fargate billing | **Per vCPU and GB requested** |
| EC2 launch billing | **Per EC2 instance, used or not** |
| Terminated EC2 still in cluster | **Deregister the container instance** |
| Deleted STOPPED instance | **Not auto-deregistered** |
| Deleted RUNNING instance | **Auto-deregistered** |
| Pack tasks together | **binpack (cpu or memory)** |
| Distribute tasks apart | **spread (instanceId / AZ)** |
| Place tasks arbitrarily | **random** |
| Rule for where a task may run | **Placement constraint (distinctInstance)** |
| Query container instances by attributes | **Cluster Query Language (CQL)** |
| How tasks are distributed | **Placement strategies** |
| Groups of related tasks | **Task groups** |
| Cluster join config | **ECS_CLUSTER in user data / /etc/ecs/ecs.config** |
| CloudFormation cluster fix | **Pass parameter into user data (ECS_CLUSTER=${ClusterName})** |
| Port conflicts on one host | **Dynamic host port mapping** |
| Dynamic port network mode | **bridge** |
| Pull image | **Task execution role** |
| Write logs / inject secrets at startup | **Task execution role** |
| Execution role managed policy | **AmazonECSTaskExecutionRolePolicy** |
| App permissions | **Task role** |
| CannotPullContainerError / no logs | **Task execution role** |
| AccessDenied at runtime | **Task role** |
| Host agent role (EC2 launch type) | **EC2 instance profile** |
| Credentials in container | **Task metadata endpoint 169.254.170.2** |
| Credentials in task definition | **Never — use roles** |
| Persistent shared storage (Fargate) | **EFS** |
| Single-AZ block storage | **EBS (wrong for shared across AZs)** |
| Temporary container storage | **Ephemeral task storage (lost on stop)** |
| ECR login token | **Valid 12 hours** |
| Stop old images accumulating | **ECR lifecycle policy** |
| Find image vulnerabilities | **ECR scanning / Inspector** |
| Container blueprint | **Task definition** |
| Keeps tasks running | **Service scheduler** |
| The machine (EC2 launch type) | **Container instance** |
| Talks to ECS from the host | **Container agent** |
