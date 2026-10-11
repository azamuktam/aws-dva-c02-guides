## Amazon EC2 and Auto Scaling

**Why it exists**

Sometimes you genuinely need a machine: an OS you control, a long-running process, a licence tied to hardware, software that cannot be containerised. EC2 is the raw virtual server. For the Developer exam it appears mostly as *context* — something your code runs on, something a deployment targets — rather than as a subject in its own right. Designing EC2 architecture is explicitly out of scope for the target candidate.

**How the pieces fit**

```plain text
AMI  (the disk image: OS + preinstalled software)
  ↓
Launch Template  (AMI + instance type + security group + IAM role + user data + storage)
  ↓
Auto Scaling Group  (min / desired / max capacity across multiple AZs)
  ↓
EC2 Instances
  ↑
Target Group  ←  Listener  ←  Application Load Balancer  ←  Users
```

**AMI** — a template containing the OS and any preinstalled application. Faster boot, because the software is already there.

**Launch Template** — the blueprint the ASG uses to create identical instances. Contains AMI, instance type, security group, IAM role, user data, storage config.

**Auto Scaling Group** — maintains the desired instance count, distributes across AZs, and **replaces unhealthy instances automatically**. That last behaviour is the answer to “how do I recover from a failed instance without intervention”.

**Scaling policies — choosing by what you know**

The three policies differ in what information you have:

**Target tracking** — you know the *metric value you want*. “Keep average CPU at 50%.” AWS works out how many instances that needs. Simplest and the default recommendation.

**Step scaling** — you know that different severities need different responses. “CPU \> 70% → add 1 instance. CPU \> 90% → add 5.” Use when the reaction should be non-linear.

**Scheduled scaling** — you know the *time*, not the metric. “Scale up at 08:00 every weekday, down at 20:00.” Use for predictable business cycles, including warming capacity *before* traffic arrives rather than reacting after.

**Scaling on a queue — the backlog-per-instance metric**

CPU is a poor scaling metric for workers pulling from a queue, because a worker waiting on I/O has low CPU while the backlog grows. The right metric is how much work each worker is carrying:

```plain text
Backlog per instance = ApproximateNumberOfMessagesVisible ÷ number of workers

1,000 messages ÷ 10 ECS tasks = 100 messages per task
```

Set a target-tracking policy on that custom metric. If the backlog per task rises above target, add tasks.

Note: the **ECS service scheduler** maintains the desired task count and replaces failed tasks — it is *not* a scaling mechanism. Scaling is Application Auto Scaling on top of it.

**EC2 User Data — bootstrapping**

User data is a script passed at launch that runs automatically on **first boot only**, with **root privileges**.

```bash
#!/bin/bash
yum update -y
yum install nginx -y
systemctl start nginx
```

```plain text
EC2 launches → user data script runs → software installed and started
```

It exists so you don’t have to SSH into every new instance. It’s how an Auto Scaling Group produces ready-to-serve instances, and CloudFormation frequently uses it to configure EC2.

**User data vs custom AMI** — the trade-off is flexibility against boot speed. User data means the setup is defined in code and easy to change, but every launch pays the installation time. A custom AMI has everything preinstalled so boot is fast, but rebuilding the AMI is needed for every change. Fast scaling favours the AMI; frequently changing setup favours user data.

**User data vs Systems Manager** — user data runs at launch; Systems Manager Run Command manages instances that are already running.

**EC2 Instance Types**

**Mental Model**

**Choose the instance based on the resource your application needs most.**

| Type | Best For | Think |
| --- | --- | --- |
| **General Purpose (M, T)** | Balanced CPU + RAM | Everyday workloads |
| **Compute Optimized (C)** | High CPU | CPU-heavy apps |
| **Memory Optimized (R, X, High Memory)** | Large RAM | In-memory databases |
| **Storage Optimized (I, D, H)** | Fast local disk I/O | Databases, analytics |
| **Accelerated Computing (P, G, Inf, Trn, F)** | GPU/FPGA/AI accelerators | ML, graphics, inference |
| **HPC Optimized (Hpc)** | High-Performance Computing | Scientific simulations |

**General Purpose**

- Balanced CPU, memory, networking.
- **Examples:** Web servers, application servers, small/medium databases.

**Compute Optimized**

- More CPU power.
- **Examples:** Batch processing, media encoding, gaming servers, compute-intensive workloads.

**Memory Optimized**

- Large RAM.
- **Examples:** Redis, SAP HANA, in-memory databases, analytics.

**Storage Optimized**

- Very fast local SSD/HDD and high IOPS.
- **Examples:** NoSQL databases, Elasticsearch/OpenSearch, data warehousing.

**Accelerated Computing**

- Uses GPUs or specialized accelerators.
- **Examples:** AI/ML training, inference, video rendering, graphics, FPGA workloads.

**HPC Optimized**

- Built specifically for **High-Performance Computing**.
- **Examples:** Scientific simulations, weather modeling, computational chemistry, engineering simulations, large-scale numerical analysis.

**Exam Shortcuts**

- **Need balanced resources?** → **General Purpose**
- **Need more CPU?** → **Compute Optimized**
- **Need more RAM?** → **Memory Optimized**
- **Need fast local storage/IOPS?** → **Storage Optimized**
- **Need GPU or AI accelerator?** → **Accelerated Computing**
- **Question says "High-Performance Computing (HPC)"?** → **HPC Optimized** ✅

**Can change while running:**

- `DeleteOnTermination` → `true/false`
- Some security-group/network settings
- Some instance attributes

**Requires stopping the instance:**

- Changing the **instance type** (`t3.small → t3.large`)
- Changing certain **EBS/root-device configurations**

**Exam memory:**

> **EC2 attributes are not all modifiable while running. Check the specific attribute.**

For your question, **`DeleteOnTermination` → can be changed while running.**

**Storage attached to EC2**

**EBS** — a network-attached block volume. Persistent, survives instance stop/termination (if configured), single AZ, normally one instance. Use for OS disks, databases, anything needing high IOPS.

**Instance store** — physically attached NVMe. Very fast, but **ephemeral** — data is lost when the instance stops. Use for scratch, cache, temporary processing.

**EC2 purchasing options**

Not on the in-scope service list — kept because they appear as distractors. The organising idea: you trade *commitment* or *interruption tolerance* for price.

| Option | The trade | Best for |
| --- | --- | --- |
| **On-Demand** | No commitment, highest price | Unpredictable workloads, testing |
| **Reserved Instance** | Commit to 1 or 3 years of a specific instance | Steady production workloads |
| **Savings Plans** | Commit to a \$/hour spend, flexible on instance type | Long-running with some flexibility |
| **Spot** | Accept interruption, get a large discount | Batch jobs, data processing, CI/CD, testing |
| **Spot Fleet** | A managed collection of Spot across types and AZs | Large flexible workloads at lowest cost |
| **Dedicated Host** | An entire physical server, hardware visibility | Compliance, bring-your-own-licence |
| **Dedicated Instance** | Hardware isolated to your account, less control than a host | Isolation requirements |
| **Capacity Reservation** | Pay to guarantee capacity exists in an AZ | Critical availability |

### EC2 Reserved Instances

**Topic:** **EC2 pricing & capacity**

**Reserved Instance (RI):** Get a **discount** for long-running EC2 workloads.

**Regional RI**

- Discount across the Region
- **Does NOT reserve capacity**

**Zonal RI**

- Discount in a specific AZ
- **Reserves EC2 capacity in that AZ** ✅

**Exam memory:**

> **Need capacity reservation + RI → Zonal RI**
> **Need flexibility across AZs → Regional RI**

**Don't confuse:** **EC2 Capacity Reservation** is a separate feature specifically designed to reserve capacity; it does not provide an RI-style billing discount.

### Reserved Instance billing

> **1 RI gives up to 3,600 seconds (1 hour) of discounted instance usage per clock-hour.**

Example:

```plain text
1 RI
3 matching instances × 1 hour
        ↓
1 hour → RI pricing
2 hours → On-Demand pricing
```

**Important:** The 3,600 seconds can be shared across multiple instances, but the **total RI-covered usage cannot exceed 1 hour per clock-hour**.

**Exam memory:**

> **1 RI = max 1 instance-hour of discounted usage per hour.**

**EC2 Free Tier (the billing trap)**

A new AWS account (under 12 months) gets **750 hours/month of `t2.micro`** free. When a question asks “how much were you charged for 35 seconds of `t2.micro` on a new account”, the answer is **\$0** — the question is testing Free Tier awareness, not per-second billing granularity. Don’t compute the arithmetic.

## EC2 Auto Scaling Target Tracking

**Supported metrics**

- `ASGAverageCPUUtilization`
- `ASGAverageNetworkOut`
- `ALBRequestCountPerTarget`

**Not supported**

- `ApproximateNumberOfMessagesVisible` → SQS queue depth

**Exam memory**

> **Target Tracking → CPU, NetworkOut, ALB requests/target**
> **SQS messages → not a predefined Target Tracking metric**

**Recall check — EC2**

| Purpose | Command |
| --- | --- |
| Enable detailed monitoring on **existing** instance | `aws ec2 monitor-instances --instance-ids i-1234567890abcdef0` |
| Disable detailed monitoring | `aws ec2 unmonitor-instances --instance-ids i-1234567890abcdef0` |
| Launch new instance with detailed monitoring | `aws ec2 run-instances --image-id ami-09092360 --monitoring Enabled=true` |
| Check instance monitoring state | `aws ec2 describe-instances --instance-ids i-1234567890abcdef0` |

- **Need a virtual machine / full OS control** → EC2
- **Automatically add and remove instances** → Auto Scaling Group
- **Create identical instances** → Launch Template
- **Replace a failed instance automatically** → Auto Scaling Group
- **Keep CPU at a target percentage** → target tracking
- **Different responses at different thresholds** → step scaling
- **Scale before a known daily peak** → scheduled scaling
- **Scale workers on queue depth** → backlog per instance + target tracking
- **Install software automatically at launch** → user data
- **Bootstrap an instance / runs once at first boot** → user data
- **Fastest boot with software preinstalled** → custom AMI
- **Manage already-running servers** → Systems Manager
- **Persistent disk for EC2** → EBS
- **Temporary high-speed storage** → instance store
- **Cheapest compute, can tolerate interruption** → Spot
- **Many Spot instances managed automatically** → Spot Fleet
- **Entire physical server** → Dedicated Host
- **Guaranteed capacity in an AZ** → Capacity Reservation
- **New account + `t2.micro` + within limits** → cost is \$0

---

## EC2 Key Pairs Across Regions

**Key point**

> **EC2 key pairs are Regional.** A key pair created in one Region cannot directly be used in another Region.

**Reuse the same SSH key across Regions**

```plain text
Same private key
      │
      └── Same public key
            ├── Import → Region A
            ├── Import → Region B
            └── Import → Region C
```

- Keep the **private key** securely.
- Import the corresponding **public key** into each AWS Region.
- AWS stores the public key; **AWS does not store your private key**.
- If the private key is lost, AWS **cannot recover it**.

**Exam memory**

> **EC2 key pair = Regional**
> **Same key across Regions → import the same public key into each Region**

### SAML = Security Assertion Markup Language

It is a **standard for letting one system prove a user's identity to another system**.

Simple example:

```plain text
You work at a company
      ↓
Company login system
      ↓
SAML message
      ↓
AWS
      ↓
You get access
```

So you **log in once with your company's account**, and AWS trusts the company's identity system.

**Real example**

You open AWS and click:

> **Sign in with Company Account**

Your company identity provider says:

> “This is John, and he is allowed to access AWS.”

It sends a **SAML assertion** to AWS.

AWS then gives John access, usually through an IAM role.

### Important terms

**Identity Provider (IdP)**

→ The system that knows who you are, such as your company's login system.

**SAML assertion**

→ The message saying who you are and what access you should get.

**AWS**

→ Trusts the assertion and gives you access.

### Exam memory

> **SAML = standard used for federated login / SSO**

And your previous question:

**LDAP doesn't support SAML → use a custom Identity Broker + STS.**

# Question patterns

> An application can tolerate interruptions and must minimise EC2 cost. Which purchasing option?
> → **Spot Instances**

> How should an EC2 application obtain credentials to call S3 without long-term access keys?
> → **An instance profile attached to an IAM role — temporary credentials via the metadata service**

> An Auto Scaling group must keep average CPU utilisation at 50% regardless of load. Which policy?
> → **Target tracking scaling policy**

> An instance is being terminated but must finish uploading logs first. Which feature allows this?
> → **ASG lifecycle hooks**

> A script must run on every EC2 launch to install the application. Where does it go?
> → **User data**

> A key pair created in us-east-1 must be used to launch instances in eu-west-1. What to do?
> → **Import the same public key into eu-west-1**

> A developer must change whether an EBS volume is deleted when its instance terminates. Which call?
> → **aws ec2 modify-instance-attribute (DeleteOnTermination)**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Credentials on EC2 | **instance profile (role)** |
| Metadata IP | **169.254.169.254** |
| Cheapest interruptible | **Spot** |
| Steady 1–3 year workload | **Reserved** |
| Scale on a metric | **target tracking** |
| Pause before termination | **lifecycle hook** |
| Startup script | **user data** |
| Key pair across regions | **import the public key** |
| Ephemeral local disk | **instance store** |
