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

> Users must be routed to the AWS Region with the lowest measured latency. Which routing policy?
> → **Latency routing**

> The apex domain (example.com) must point to an ALB. Which record type?
> → **Alias record (A/AAAA Alias to the ALB)**

> Traffic must fail over to a standby Region when the primary fails. Which setup?
> → **Failover routing with health checks**

> 20% of traffic goes to version B and 80% to version A. Which policy?
> → **Weighted routing**

> Internal DNS names are needed for resources inside a VPC. Which feature?
> → **Private hosted zones**

> Content must differ by country for language, licensing or compliance reasons. Which policy?
> → **Geolocation routing (based on where the user is)**

> Traffic must be shifted gradually between Regions by biasing by the resources' location. Which policy?
> → **Geoproximity routing (adjustable bias)**

> A simple client-side load spread among up to 8 healthy records at random. Which policy?
> → **Multivalue answer routing**

> A single resource serves the whole domain with no special logic. Which policy?
> → **Simple routing**

> A domain must point at a third-party SaaS hostname such as yourapp.3rdparty.com. Which record type?
> → **CNAME (the target is a hostname, not an AWS resource)**

> A record must map a name to a raw IPv4 address (and IPv6). Which record types?
> → **A for IPv4; AAAA for IPv6**

> Reverse DNS lookup (IP → hostname) for mail servers. Which record type?
> → **PTR**

> Why can't a CNAME be used at the zone apex, and what is used instead?
> → **CNAME is not allowed at the root domain; an Alias record is used**

> Which AWS resources can an Alias record target?
> → **ALB/NLB, CloudFront, API Gateway, S3 website endpoints, Elastic Beanstalk environments, and other Route 53 records**

> What is the cost of Alias queries compared with CNAME?
> → **Alias queries are free**

> How does one decide between an Alias and a CNAME?
> → **Is the target an AWS resource? If yes → Alias; if it's another hostname → CNAME**

> A root domain must point at an AWS service. Which record type, and why not CNAME?
> → **Alias — a CNAME is impossible at the apex**

> In a registrar-to-Route 53 migration, what must be updated at the registrar?
> → **Replace the domain's name servers with the 4 Route 53 NS records**

> How many name server (NS) records does Route 53 give for a hosted zone?
> → **Four**

> A domain is registered at GoDaddy but DNS should be managed by Route 53. What is the distinction?
> → **The registrar (GoDaddy) is not the DNS provider (Route 53); update NS records at the registrar**

> What is the difference between a public and a private hosted zone?
> → **Public = internet-facing domains; Private = DNS for resources inside a VPC**

> Distinguish geolocation from geoproximity in one line.
> → **Geolocation cares where the user is; geoproximity cares where the resources are and lets you bias traffic between them**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Nearest/fastest Region | **latency routing** |
| Active-passive DR | **failover routing + health checks** |
| Route by user's country | **geolocation routing** |
| Bias traffic between Regions | **geoproximity routing** |
| Percentage split / canary / A-B | **weighted routing** |
| Up to 8 healthy records, random | **multivalue answer routing** |
| One destination, no logic | **simple routing** |
| AWS resource target | **Alias** |
| Apex/root to AWS service | **Alias (CNAME impossible)** |
| Third-party hostname | **CNAME** |
| Map to IPv4 | **A record** |
| Map to IPv6 | **AAAA record** |
| Reverse DNS / mail | **PTR record** |
| CNAME restriction | **not allowed at zone apex** |
| Alias query cost | **free** |
| Alias supported targets | **ALB/NLB, CloudFront, API Gateway, S3 website, Beanstalk, other R53 records** |
| Registrar vs DNS provider | **GoDaddy ≠ Route 53** |
| Migration step | **update NS records at registrar** |
| Route 53 NS record count | **4** |
| Internet-facing domain | **public hosted zone** |
| Internal VPC DNS | **private hosted zone** |
| User location | **geolocation** |
| Resource location + bias | **geoproximity** |
| Route 53 unique extras vs plain DNS | **alias records + health/latency/geo/weight routing** |
| Hostname target, not AWS | **CNAME** |
| IP target | **A / AAAA** |
