## EBS vs EFS
The distinction is disk versus network filesystem.
<table header-row="true">
<tr>
<td></td>
<td>**Amazon EBS**</td>
<td>**Amazon EFS**</td>
</tr>
<tr>
<td>Linux analogy</td>
<td>A local hard disk (`/dev/xvda`)</td>
<td>An **NFS** mount</td>
</tr>
<tr>
<td>Storage type</td>
<td>**Block**</td>
<td>**Shared file**</td>
</tr>
<tr>
<td>Attached to</td>
<td>Normally **one** instance</td>
<td>**Many** instances / tasks simultaneously</td>
</tr>
<tr>
<td>Availability Zone</td>
<td>**Single AZ**</td>
<td>**Multi-AZ**</td>
</tr>
<tr>
<td>Typical use</td>
<td>OS disk, databases, high IOPS</td>
<td>Shared files, web content, container storage, home directories</td>
</tr>
<tr>
<td>Shared by many?</td>
<td>❌ No (except Multi-Attach for specific workloads)</td>
<td>✅ Yes</td>
</tr>
</table>
Mental model: **EBS is a hard drive plugged into one server. EFS is a network folder mounted by many servers.**
Because EBS is single-AZ and single-attach, it is always the wrong answer to “persistent shared storage across Availability Zones” — including for **Fargate**, where EFS is the only option.
**Instance store** is a third thing: physically attached, very fast, and **ephemeral**. Lost on stop.
### Root vs non-root EBS volume
**Root volume** = the EBS volume where the EC2 operating system is installed and from which the instance boots.
Example: `/dev/xvda` or `/dev/sda1`
**Non-root volume** = an additional EBS volume attached to the instance for data, logs, databases, files, etc.
Example: `/dev/xvdf`
**Recall check**
- **Disk for one EC2 instance, high IOPS** → EBS
- **Shared filesystem across instances or AZs** → EFS
- **Fargate persistent shared storage** → EFS
- **Lambda needs shared persistent files** → EFS
- **Temporary high-speed scratch** → instance store
- **EBS volumes support both in-flight encryption and encryption at rest using KMS**
- For a **root volume**, stop the instance first and then detach the volume; for a **non-root volume**, only unmount it first and then detach the volume.
## EFS Storage Classes
**EFS Standard**
- Frequently accessed files
- Highest storage cost
- Immediate access
**EFS Standard-IA**
- Infrequently accessed files
- Lower storage cost
- Immediate access, but **retrieval/access charges apply**
- Good for files that are accessed occasionally
**EFS Archive**
- Rarely accessed files
- Lowest storage cost
- Designed for long-term retention
- Retrieval/access charges apply
### Simple comparison
<table header-row="true">
<tr>
<td>Storage class</td>
<td>Access frequency</td>
<td>Cost</td>
</tr>
<tr>
<td>**EFS Standard**</td>
<td>Frequent</td>
<td>Higher</td>
</tr>
<tr>
<td>**EFS Standard-IA**</td>
<td>Infrequent</td>
<td>Lower</td>
</tr>
<tr>
<td>**EFS Archive**</td>
<td>Rare</td>
<td>Lowest</td>
</tr>
</table>
### Important
EFS can use **Lifecycle Management** to automatically move files between storage classes based on how long they've gone without being accessed.
For example:
```plain text
Frequently accessed
       ↓
EFS Standard
       ↓ no access for some time
EFS Standard-IA
       ↓ no access for longer
EFS Archive
```
**Exam memory:**
> **EFS Standard → frequent**
> **EFS Standard-IA → infrequent**
> **EFS Archive → rarely accessed **
# EBS Volume Types
**Mental model**
```plain text
SSD → fast, low latency, random I/O
HDD → large sequential workloads, throughput
```
AWS groups EBS volumes into **SSD**, **HDD**, and **previous-generation Magnetic** volumes. 
## SSD volumes
<table header-row="true">
<tr>
<td>Type</td>
<td>What to remember</td>
</tr>
<tr>
<td>**gp3**</td>
<td>General purpose; **3,000 IOPS + 125 MiB/s included**; IOPS/throughput can be increased independently</td>
</tr>
<tr>
<td>**gp2**</td>
<td>General purpose; **3 IOPS/GiB**, max **16,000 IOPS**</td>
</tr>
<tr>
<td>**io1**</td>
<td>High-performance database; provisioned IOPS; max **64,000 IOPS**; **50 IOPS/GiB**</td>
</tr>
<tr>
<td>**io2**</td>
<td>High-performance database; more durable than io1; max **64,000 IOPS**, or **256,000 with Block Express**</td>
</tr>
</table>
SSD volumes are suited to workloads with frequent reads/writes and random I/O. 
**gp2 — testable calculation**
> **3 IOPS/GiB**
> **16,000 IOPS maximum**
So:
```plain text
16,000 ÷ 3 ≈ 5,334 GiB ≈ 5.3 TiB
```
**5.3 TiB → maximum gp2 IOPS.** 
**gp3**
> **3,000 IOPS + 125 MiB/s included**
Maximum:
> **80,000 IOPS + 2,000 MiB/s**
The important difference:
> **gp2 → volume size affects IOPS**
> **gp3 → IOPS and throughput can be provisioned independently of size.** 
**io1**
> **50 IOPS/GiB**
Example:
```plain text
200 GiB × 50 = 10,000 IOPS max
```
Therefore:
```plain text
200 GiB + 10,000 IOPS → ✅
200 GiB + 15,000 IOPS → ❌
```
**Exam memory:**
> `io1 = provisioned IOPS + 50 IOPS/GiB`
**io2**
Think:
> **io2 = high IOPS + higher durability**
And:
> **io2 Block Express → up to 256,000 IOPS** on supported Nitro instances. 
---
## HDD volumes
<table header-row="true">
<tr>
<td>Type</td>
<td>What to remember</td>
</tr>
<tr>
<td>**st1**</td>
<td>**Throughput Optimized HDD** → large, sequential, frequently accessed data</td>
</tr>
<tr>
<td>**sc1**</td>
<td>**Cold HDD** → large, sequential, infrequently accessed data; lowest-cost option</td>
</tr>
</table>
HDD volumes are optimized for **large sequential I/O**, not small/random I/O. 
**st1**
> **Big sequential data + high throughput**
Examples:
- Big Data
- Data warehouses
- Log processing
**sc1**
> **Cold/infrequently accessed data + low cost**
**Easy distinction:**
```plain text
st1 → sequential + frequently accessed
sc1 → sequential + rarely accessed + cheaper
```
Neither `st1` nor `sc1` can be used as a boot volume. 
---
#### Previous generation
**standard**
**Magnetic HDD / previous-generation volume**
> Small datasets + infrequent access + performance isn't important.
Generally, choose a modern volume type instead for new workloads. 
---
## Multi-Attach
**io1 and io2** support **EBS Multi-Attach**.
This allows one volume to be attached to multiple supported EC2 instances in the same Availability Zone. 
```plain text
EC2 ─┐
EC2 ─┼── io1 / io2
EC2 ─┘
```
---
**Exam shortcut**
```plain text
              EBS
               │
       ┌───────┴───────┐
       │               │
      SSD             HDD
       │               │
   gp2 / gp3        st1 / sc1
   io1 / io2
```
**General purpose → gp3**
**Old general purpose → gp2**
**High IOPS database → io1 / io2**
**Extreme IOPS → io2 Block Express**
**Large sequential data → st1**
**Cold + cheap sequential data → sc1**
**Legacy magnetic → standard**
### Numbers worth memorizing
> **gp2 → 3 IOPS/GiB → 16,000 max → \~5.3 TiB**
> **gp3 → 3,000 IOPS + 125 MiB/s → 80,000 IOPS / 2,000 MiB/s max**
> **io1 → 50 IOPS/GiB → 64,000 max**
> **io2 Block Express → 256,000 IOPS**
That is the core EBS information I'd keep for your DVA practice tests. 
<empty-block/>
