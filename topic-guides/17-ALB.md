## Elastic Load Balancing (Application Load Balancer)
**Why it exists**
One server is a single point of failure and a capacity ceiling. A load balancer spreads requests across many targets, removes unhealthy ones, and terminates TLS in one place. It also lets your application servers live in **private subnets** — the only thing exposed to the internet is the load balancer.
```plain text
Users (Internet)
       ↓
Application Load Balancer  (public subnets)
       ↓
Target Group
       ↓
Private EC2 instances / ECS tasks / Lambda
```
That separation is the security answer: no direct access to servers, cleaner architecture, and the instances need no public IPs.
ALBs can route traffic to different Target Groups based on **URL Path, Hostname, HTTP Headers, and Query Strings.**
**How the pieces fit**
**Listener** — watches a port (80, 443) and holds the rules that decide where traffic goes.
**Rules** — evaluated in priority order. Route by **path** (`/api/*` → API servers, `/images/*` → image servers) or **host** (`api.example.com` → API servers). This is why one ALB can front several applications.
**Target group** — the set of destinations. Targets can be EC2 instances, ECS tasks, IP addresses, or **Lambda functions**.
**Health checks** — the ALB probes each target periodically and only sends traffic to healthy ones. This is how a failing instance is removed without human intervention.
ALB works at **Layer 7** (HTTP/HTTPS), which is what makes content-based routing possible. **NLB** works at Layer 4 (TCP/UDP), is faster, and provides a **static IP** — that’s when you choose it over ALB.
### **Cross-zone load balancing**
**Application Load Balancer (ALB):** Cross-zone load balancing is always turned on at the load balancer level and cannot be disabled there, though you can override it on individual target groups.
**Network Load Balancer (NLB):** The feature is disabled by default, meaning each load balancer node distributes traffic only to targets in its own availability zone. You must turn it on manually.
**Gateway Load Balancer (GWLB):** It is disabled by default and can be enabled to route traffic across alternative zones if local targets fail.
**Classic Load Balancer (CLB):** It is disabled by default if configured via API or CLI, but enabled by default if using the management console
Without it, traffic is split evenly **between Availability Zones** first, then divided among the instances in each zone. Uneven instance counts then produce very uneven per-instance load:
```plain text
Without cross-zone:
AZ1: 2 instances ← 50% of traffic → 25% each
AZ2: 8 instances ← 50% of traffic → 6.25% each

With cross-zone:
10 instances total, 100% of traffic → 10% each
```
Enable it whenever AZ instance counts can differ.
**Observability features**
**Request tracing** — the ALB adds an **`X-Amzn-Trace-Id`** header to every request, so a single request can be followed across services. This is also how X-Ray traces are correlated.
**Access logs** — detailed per-request records delivered to **S3**. Use them to analyse traffic and troubleshoot HTTP errors. (Athena is the usual way to query them.)
**Sticky sessions (session affinity)**
A target-group attribute that pins a user to the same target using a cookie, so server-local session state keeps working.
It’s the zero-code option and therefore a plausible-looking answer, but it loses whenever the question mentions scalability or instance failure — when that instance dies, its users lose their sessions. The correct answer in those cases is shared session storage in **ElastiCache Redis**. The full comparison sits with ElastiCache in Part 2, since the decision is really about *where session state lives*, not about the load balancer.
**Reserved cookie names:** For ALB application-based cookies, you cannot use `AWSALB`, `AWSALBAPP`, or `AWSALBTG` because these names are reserved by the load balancer
**Server Name Indication (SNI)** allows ALB and NLB to use multiple SSL/TLS certificates on a single listener.
**Exam memory**
- **SNI = Multiple certificates on one listener**
- ALB/NLB choose the correct certificate based on the requested hostname.
-
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
**ALB:**
The ALB knows the **EC2 instance + dynamically assigned port** for each task and sends traffic to the correct task.
**Exam memory:**
> **Dynamic host port mapping → multiple ECS tasks on the same EC2 instance.**
> **ECS + dynamic host ports → ALB supports this.**
### Elastic Load Balancer DNS and Static IP
| Load Balancer | DNS Name | Static IP |
| --- | --- | --- |
| **Application Load Balancer (ALB)** | ✅ Yes | ❌ No |
| **Network Load Balancer (NLB)** | ✅ Yes | ✅ Yes (Static IP / Elastic IP support) |
| **Gateway Load Balancer (GWLB)** | ✅ Yes | ❌ No |
**Host-based routing examples:**
> [api.sample.com](http://api.sample.com/)     → API servers<br>[shop.sample.com](http://shop.sample.com/)    → Shop servers<br>[admin.sample.com](http://admin.sample.com/)   → Admin servers
**Problem**
- EC2 sees the **ALB IP**, not the real user IP.
**Solution**
- Use the **X-Forwarded-For header**.
- ALB puts the original client IP in this header.
**Exam memory**
- **ALB → X-Forwarded-For → Client IP**
- All Elastic Load Balancers provide a **DNS name**.
- Only **NLB** supports **static IP addresses**.
**Recall check — ALB**
- **HTTP/HTTPS load balancing with routing rules** → ALB
- **Path-based or host-based routing** → ALB
- **Static IP + TCP/UDP + highest performance** → NLB
- **Distribute traffic to Lambda** → ALB can use Lambda as a target
- **Where traffic should go** → target group
- **Traffic rules on a port** → listener
- **Remove unhealthy instances automatically** → health checks
- **Uneven load across AZs with different instance counts** → enable cross-zone load balancing
- **Trace a request across services** → `X-Amzn-Trace-Id` (ALB request tracing)
- **Analyse ALB requests / troubleshoot HTTP errors** → ALB access logs in S3
- **Keep servers off the internet** → public ALB, private instancess
---
