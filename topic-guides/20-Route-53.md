## Amazon Route 53
**Why it exists**
DNS translates names into destinations. Route 53 is AWS’s DNS service, and it does two things beyond ordinary DNS: it has **native awareness of AWS resources** (alias records), and it can make **routing decisions** based on health, latency, geography, or weight.
**Record types — and the one restriction that matters**
<table header-row="true">
<tr>
<td>Record</td>
<td>Points to</td>
<td>Example</td>
</tr>
<tr>
<td>**A**</td>
<td>An IPv4 address</td>
<td>`example.com → 192.0.2.1`</td>
</tr>
<tr>
<td>**AAAA**</td>
<td>An IPv6 address</td>
<td>`example.com → 2001:db8::1`</td>
</tr>
<tr>
<td>**CNAME**</td>
<td>**Another hostname**</td>
<td>`www.mydomain.com → yourapp.3rdparty.com`</td>
</tr>
<tr>
<td>**Alias**</td>
<td>**An AWS resource**</td>
<td>`example.com → ALB`</td>
</tr>
<tr>
<td>**PTR**</td>
<td>Reverse lookup, IP → hostname</td>
<td>Mail servers, reverse DNS</td>
</tr>
</table>
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
<table header-row="true">
<tr>
<td>Policy</td>
<td>Routes based on</td>
<td>Use when</td>
</tr>
<tr>
<td>**Simple**</td>
<td>Nothing — one destination</td>
<td>A single resource serves the domain</td>
</tr>
<tr>
<td>**Failover**</td>
<td>Health checks</td>
<td>Active-passive disaster recovery</td>
</tr>
<tr>
<td>**Geolocation**</td>
<td>The **user’s** location</td>
<td>Content must differ by country (language, licensing, compliance)</td>
</tr>
<tr>
<td>**Geoproximity**</td>
<td>The **resource’s** location, with an adjustable bias</td>
<td>Shift traffic between Regions gradually</td>
</tr>
<tr>
<td>**Latency**</td>
<td>Measured network latency</td>
<td>Resources in several Regions, users should get the fastest</td>
</tr>
<tr>
<td>**Multivalue answer**</td>
<td>Random selection among healthy records</td>
<td>Simple client-side load spreading, up to **8** healthy records</td>
</tr>
<tr>
<td>**Weighted**</td>
<td>Percentages you assign</td>
<td>Canary releases, A/B testing, gradual migration</td>
</tr>
</table>
Note the geolocation/geoproximity distinction: **geolocation** cares where the *user* is; **geoproximity** cares where the *resources* are and lets you bias traffic between them.
**Recall check — Route 53**
- **Point a domain at an AWS resource (ALB, CloudFront, API Gateway, S3 website)** → **Alias**
- **Root domain to an AWS service** → **Alias** (CNAME is not allowed at the apex)
- **`www`**** to a third-party hostname** → **CNAME**
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
