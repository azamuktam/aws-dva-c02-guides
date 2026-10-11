## EBS vs EFS

The distinction is disk versus network filesystem.

|    | **Amazon EBS** | **Amazon EFS** |
| --- | --- | --- |
| Linux analogy | A local hard disk (`/dev/xvda`) | An **NFS** mount |
| Storage type | **Block** | **Shared file** |
| Attached to | Normally **one** instance | **Many** instances / tasks simultaneously |
| Availability Zone | **Single AZ** | **Multi-AZ** |
| Typical use | OS disk, databases, high IOPS | Shared files, web content, container storage, home directories |
| Shared by many? | ❌ No (except Multi-Attach for specific workloads) | ✅ Yes |

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

| Storage class | Access frequency | Cost |
| --- | --- | --- |
| **EFS Standard** | Frequent | Higher |
| **EFS Standard-IA** | Infrequent | Lower |
| **EFS Archive** | Rare | Lowest |

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

## EBS Volume Types

**Mental model**

```plain text
SSD → fast, low latency, random I/O
HDD → large sequential workloads, throughput
```

AWS groups EBS volumes into **SSD**, **HDD**, and **previous-generation Magnetic** volumes.

## SSD volumes

| Type | What to remember |
| --- | --- |
| **gp3** | General purpose; **3,000 IOPS + 125 MiB/s included**; IOPS/throughput can be increased independently |
| **gp2** | General purpose; **3 IOPS/GiB**, max **16,000 IOPS** |
| **io1** | High-performance database; provisioned IOPS; max **64,000 IOPS**; **50 IOPS/GiB** |
| **io2** | High-performance database; more durable than io1; max **64,000 IOPS**, or **256,000 with Block Express** |

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

| Type | What to remember |
| --- | --- |
| **st1** | **Throughput Optimized HDD** → large, sequential, frequently accessed data |
| **sc1** | **Cold HDD** → large, sequential, infrequently accessed data; lowest-cost option |

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

# Question patterns

> Storage must be shared by many EC2 instances across multiple Availability Zones at once. Which service?
> → **Amazon EFS (shared file storage, multi-AZ)**

> A database on a single instance needs consistent high IOPS with low latency. Which volume type?
> → **io1/io2 for provisioned IOPS, or gp3 for general purpose**

> Fast temporary scratch space that is lost if the instance stops. Which storage?
> → **Instance store (ephemeral)**

> One volume must be attached to two instances in the same AZ simultaneously. Which feature?
> → **EBS Multi-Attach (io1/io2 only)**

> A Fargate task needs persistent shared storage. Which service?
> → **EFS (the only option; EBS cannot be shared across AZs)**

> A Lambda function needs shared persistent files. Which service?
> → **EFS**

> What is the Linux analogy for EBS versus EFS?
> → **EBS is a local hard disk (/dev/xvda); EFS is an NFS mount**

> What storage type is EBS and what is EFS?
> → **EBS is block; EFS is shared file**

> How do EBS and EFS differ on attachment and AZ scope?
> → **EBS attaches to normally one instance in a single AZ; EFS attaches to many instances/tasks across multiple AZs**

> Which EBS volume is the default general-purpose SSD?
> → **gp3**

> What baseline performance does gp3 include?
> → **3,000 IOPS and 125 MiB/s, independent of volume size**

> What are gp3's maximum IOPS and throughput?
> → **80,000 IOPS and 2,000 MiB/s**

> What is gp2's IOPS-to-size ratio and maximum?
> → **3 IOPS/GiB, max 16,000 IOPS (reached at ~5.3 TiB)**

> What is the key difference between gp2 and gp3?
> → **gp2 IOPS is tied to volume size; gp3 IOPS and throughput are provisioned independently of size**

> A high-performance database needs provisioned IOPS. Which volume types?
> → **io1 / io2 (and io2 is more durable than io1)**

> What is io1's IOPS-to-size ratio and maximum?
> → **50 IOPS/GiB, max 64,000 IOPS**

> A 200 GiB io1 volume is provisioned with 10,000 IOPS. Valid, and what is the ceiling?
> → **Valid — 200 GiB × 50 = 10,000 IOPS maximum; 15,000 would be rejected**

> Extreme IOPS up to 256,000 is required on a supported Nitro instance. Which volume?
> → **io2 Block Express**

> Large, sequential, frequently accessed data such as Big Data or data warehouses. Which volume?
> → **st1 (Throughput Optimized HDD)**

> Large, sequential, infrequently accessed data at the lowest cost. Which volume?
> → **sc1 (Cold HDD)**

> Can st1 or sc1 be used as a boot volume?
> → **No**

> Which volumes are previous-generation magnetic? When to use?
> → **standard — small datasets, infrequent access, performance not important**

> Frequently accessed EFS files with the highest storage cost. Which class?
> → **EFS Standard**

> Infrequently accessed EFS files that carry retrieval/access charges. Which class?
> → **EFS Standard-IA**

> Rarely accessed EFS files for long-term retention at the lowest cost. Which class?
> → **EFS Archive**

> How do EFS files move between storage classes automatically?
> → **EFS Lifecycle Management (based on time since last access)**

> How is an EBS root volume detached, versus a non-root volume?
> → **Root: stop the instance, then detach. Non-root: unmount, then detach**

> What encryption does EBS support?
> → **Both in-flight encryption and encryption at rest using KMS**

> Why is EBS always the wrong answer for "persistent shared storage across AZs"?
> → **It is single-AZ and normally single-attach**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Shared multi-AZ filesystem | **EFS** |
| Single-instance block disk | **EBS** |
| Ephemeral high-speed local | **instance store** |
| Fargate persistent shared storage | **EFS** |
| Lambda shared persistent files | **EFS** |
| EBS AZ scope | **single AZ** |
| EFS AZ scope | **multi-AZ** |
| EBS attachment | **normally one instance** |
| EFS attachment | **many instances / tasks** |
| Multi-attach | **io1/io2 Multi-Attach (same AZ)** |
| Default general purpose SSD | **gp3** |
| gp3 baseline | **3,000 IOPS + 125 MiB/s included** |
| gp3 maximum | **80,000 IOPS / 2,000 MiB/s** |
| gp2 ratio | **3 IOPS/GiB** |
| gp2 maximum | **16,000 IOPS (~5.3 TiB)** |
| gp2 vs gp3 | **gp2 size affects IOPS; gp3 independent** |
| Provisioned IOPS database | **io1 / io2** |
| io1 ratio | **50 IOPS/GiB, max 64,000** |
| io2 durability | **more durable than io1** |
| Extreme IOPS | **io2 Block Express → 256,000 IOPS** |
| Throughput HDD | **st1 (sequential + frequently accessed)** |
| Cold HDD | **sc1 (sequential + rarely accessed, lowest cost)** |
| st1/sc1 boot volume | **No** |
| Legacy magnetic | **standard** |
| SSD use case | **frequent random I/O, low latency** |
| HDD use case | **large sequential I/O** |
| EFS frequent access | **EFS Standard (highest cost)** |
| EFS infrequent access | **EFS Standard-IA (retrieval charges)** |
| EFS lowest cost / long-term | **EFS Archive (retrieval charges)** |
| EFS auto tiering | **Lifecycle Management** |
| Root volume detach | **stop the instance, then detach** |
| Non-root volume detach | **unmount, then detach** |
| EBS encryption | **in-flight + at rest (KMS)** |
| Root volume example device | **/dev/xvda or /dev/sda1** |
| Non-root volume example device | **/dev/xvdf** |
