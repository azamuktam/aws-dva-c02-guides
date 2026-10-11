## Amazon Route 53

**Why it exists**

DNS translates names into destinations. Route 53 is AWS’s DNS service, and it does two things beyond ordinary DNS: it has **native awareness of AWS resources** (alias records), and it can make **routing decisions** based on health, latency, geography, or weight.

**Record types — and the one restriction that matters**

| Record | Points to | Example |
| --- | --- | --- |
| **A** | An IPv4 address | `example.com → 192.0.2.1` |
| **AAAA** | An IPv6 address | `example.com → 2001:db8::1` |
| **CNAME** | **Another hostname** | `www.mydomain.com → yourapp.3rdparty.com` |
| **Alias** | **An AWS resource** | `example.com → ALB` |
| **PTR** | Reverse lookup, IP → hostname | Mail servers, reverse DNS |

**The CNAME restriction:** a CNAME **cannot be used at the zone apex** (the root domain).

```plain text
mydomain.com        ❌ CNAME not allowed
www.mydomain.com    ✅ CNAME allowed
```

**Alias records** are Route 53’s answer to that restriction. They are an AWS extension, they **work at the root domain**, they resolve to AWS resources directly, and **queries are free**.

Alias supports: ALB/NLB, CloudFront, API Gateway, S3 website endpoints, Elastic Beanstalk environments, and other Route 53 records.

So the decision reduces to a single question: **is the target an AWS resource?**

- AWS resource (ALB, CloudFront, API Gateway, S3 website) → **Alias**
- Root domain pointing at an AWS service → **Alias** (a CNAME is impossible)
- Some other hostname, e.g. a third-party SaaS → **CNAME**
- A raw IP address → **A** (or **AAAA** for IPv6)

Worked example:

```plain text
www.mydomain.com  →  yourapp.3rdparty.com
```

✅ **CNAME** — the target is another hostname, not an AWS resource.

**Route 53 Routing policies — choose by the criterion**

| Policy | Routes based on | Use when |
| --- | --- | --- |
| **Simple** | Nothing — one destination | A single resource serves the domain |
| **Failover** | Health checks | Active-passive disaster recovery |
| **Geolocation** | The **user’s** location | Content must differ by country (language, licensing, compliance) |
| **Geoproximity** | The **resource’s** location, with an adjustable bias | Shift traffic between Regions gradually |
| **Latency** | Measured network latency | Resources in several Regions, users should get the fastest |
| **Multivalue answer** | Random selection among healthy records | Simple client-side load spreading, up to **8** healthy records |
| **Weighted** | Percentages you assign | Canary releases, A/B testing, gradual migration |

Note the geolocation/geoproximity distinction: **geolocation** cares where the *user* is; **geoproximity** cares where the *resources* are and lets you bias traffic between them.

**Recall check — Route 53**

- **Point a domain at an AWS resource (ALB, CloudFront, API Gateway, S3 website)** → **Alias**
- **Root domain to an AWS service** → **Alias** (CNAME is not allowed at the apex)
- **`www` to a third-party hostname** → **CNAME**
- **Map to an IP address** → **A** record (**AAAA** for IPv6)
- **Reverse DNS lookup** → **PTR**
- **Active-passive failover** → Failover routing
- **Route by the user’s country** → Geolocation
- **Bias traffic between Regions** → Geoproximity
- **Send users to the fastest Region** → Latency routing
- **Split traffic by percentage / canary** → Weighted routing
- **Up to 8 healthy records at random** → Multivalue answer
- **Alias queries are free and work at the apex**

#### Using Route 53 with a GoDaddy Domain

**Steps**

- Create a **Hosted Zone** in Route 53 for your domain.
- Route 53 provides **4 Name Server (NS)** records.
- In GoDaddy, replace the domain's existing **Name Servers** with the Route 53 NS records.
- After DNS propagation, Route 53 becomes the DNS provider for your domain.

**Exam memory**

- **Domain registrar (GoDaddy)** ≠ **DNS provider (Route 53)**
- **Update the domain's NS records to Route 53 name servers**
- **Public Hosted Zone → Internet-facing domain (GoDaddy, website)**
- **Private Hosted Zone → Internal AWS VPC DNS**

# Question patterns

> Users must be routed to the AWS Region with the lowest latency. Which routing policy?
> → **Latency routing**

> The apex domain (example.com) must point to an ALB. Which record type?
> → **Alias record (A/AAAA Alias to the ALB)**

> Traffic must fail over to a standby region when the primary fails. Which setup?
> → **Failover routing with health checks**

> 20% of traffic goes to version B, 80% to version A. Which policy?
> → **Weighted routing**

> Internal DNS names for resources inside a VPC. Which feature?
> → **Private hosted zones**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Nearest Region | **latency routing** |
| ALB/CloudFront apex | **Alias (not CNAME)** |
| Primary/standby | **failover + health checks** |
| Split traffic | **weighted routing** |
| Geography | **geolocation routing** |
| Internal DNS | **private hosted zone** |
