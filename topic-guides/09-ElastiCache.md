## Amazon ElastiCache

**Why it exists**

A database query that returns the same answer a thousand times an hour is a thousand wasted queries. ElastiCache keeps hot data in memory, sub-millisecond away. It also solves a second problem that matters for scaling web apps: **where to keep session state** when any request can land on any server.

**Caching strategies — the trade is always freshness against work**

**Lazy loading (cache-aside)**

```plain text
Read request
     ↓
Check cache
     ↓
HIT  → return immediately
MISS → read the database → write to cache → return
```

- ✅ Only data that is actually requested gets cached, so the cache stays small and relevant.
- ❌ **Cache-miss penalty**: a miss costs three round trips instead of one, so the first request after a deploy or eviction is slow.
- ❌ Data can be **stale** — the cache doesn’t know the database changed.

**Write-through**

```plain text
Write request
     ↓
Write to the database
     ↓
Write to the cache
```

- ✅ The cache is never stale, and reads are always fast.
- ❌ **Write penalty** on every single write.
- ❌ **Cache churn** — you fill the cache with data nobody ever reads.
- ❌ Doesn’t help data written before the cache existed, so you still need a lazy-load fallback.

**TTL / expiration** bounds how stale anything can get. In practice you combine lazy loading with a TTL, and add write-through for data that must never be stale.

Mapping symptoms to strategies:

- *“Users see outdated data”* → stale cache → add TTL or switch to write-through
- *“First request is always slow”* → cache-miss penalty of lazy loading
- *“The cache is full of data nobody reads”* → write-through churn

#### ElastiCache Cluster

**Mental model**

Think of an **ElastiCache cluster** as a **cache system** made up of one or more cache servers (**nodes**) that work together. Your application sees it as a single cache, while AWS manages the underlying nodes.

```plain text
Application
      │
      ▼
+----------------------+
| ElastiCache Cluster  |
|                      |
|  Node 1              |
|  Node 2              |
|  Node 3              |
+----------------------+
      │
      ▼
Database
```

**What is a cluster?**

- A cluster is a **logical group of one or more cache nodes**.
- When creating a cluster, you choose the cache engine (Valkey/Redis OSS or Memcached).
- AWS manages the nodes, networking, scaling, and monitoring.

**What is a node?**

A **node** is a single cache server.

Each node has:

- CPU
- RAM (stores cached data)
- Network endpoint

```plain text
Cluster
├── Node 1
├── Node 2
└── Node 3
```

The node is the smallest building block of ElastiCache.

**Why use multiple nodes?**

Multiple nodes provide:

- More memory
- Higher throughput
- Better availability
- Easier scaling

Example:

```plain text
Need 30 GB cache

Cluster
├── Node 1 (10 GB)
├── Node 2 (10 GB)
└── Node 3 (10 GB)
```

Instead of buying one huge server, AWS lets you spread the cache across multiple nodes.

**Redis (Valkey) cluster**

Redis/Valkey clusters can contain **shards**.

A shard contains:

- 1 Primary (read/write)
- 0–5 Replicas (read-only)

```plain text
Cluster
│
├── Shard 1
│     ├── Primary
│     └── Replica
│
└── Shard 2
      ├── Primary
      └── Replica
```

When **cluster mode is enabled**, data is automatically partitioned across shards.

**Memcached cluster**

Memcached does not have primary/replica architecture.

It simply consists of multiple nodes.

```plain text
Cluster
├── Node 1
├── Node 2
├── Node 3
└── Node 4
```

Data is distributed across the nodes.

**Cluster hierarchy**

```plain text
ElastiCache Cluster
        │
        ├── Node
        ├── Node
        └── Node
```

Redis (cluster mode enabled):

```plain text
Cluster
    │
    ├── Shard
    │      ├── Primary
    │      └── Replica
    │
    └── Shard
           ├── Primary
           └── Replica
```

**Exam keywords**

- **Node** = One cache server.
- **Cluster** = One or more nodes.
- **Shard** = A partition of Redis/Valkey data.
- **Primary** = Handles reads and writes.
- **Replica** = Read-only copy used for high availability and read scaling.
- **Cluster mode enabled** = Data is split across multiple shards.
- **Memcached** = Multiple nodes, no primary or replica.

**Exam memory**

- **Node = One server**
- **Cluster = Collection of nodes**
- **Redis/Valkey → Cluster → Shards → Primary + Replicas**
- **Memcached → Cluster → Nodes only**
- **More nodes = More memory + Better scalability + Higher availability**

**Redis vs Memcached**

| Feature | **Redis** | **Memcached** |
| --- | --- | --- |
| Persistence / snapshots | ✅ | ❌ |
| Replication + Multi-AZ failover | ✅ | ❌ |
| Data structures (lists, sets, sorted sets, pub/sub) | ✅ | ❌ key-value only |
| Multi-threaded | ❌ | ✅ |
| Scale out | Cluster mode (sharding) | Add nodes, client-side sharding |
| Backup and restore | ✅ | ❌ |
| Best for | **Sessions**, leaderboards, HA caching, pub/sub | Simple, large, disposable cache |

The reason **Redis wins session questions** is durability: a session lost to a node failure logs a user out. Memcached has no replication or persistence, so a node loss loses those sessions. Redis has both.

**Session management — the full picture**

The problem: with multiple application servers behind a load balancer, a user’s second request may hit a different server than the first. If session state lives on the server, that user appears logged out.

| Approach | Session storage | Assessment |
| --- | --- | --- |
| **ElastiCache Redis** | ✅ Central, shared, replicated | **The correct answer.** Any server can serve any request |
| **ElastiCache Memcached** | ⚠️ Central but not durable | Fine for disposable cache, risky for sessions |
| **Sticky sessions (session affinity)** | ✅ On one specific server | Works, but **less scalable** — and when that instance dies, its users lose their sessions |
| **CloudFront** | ❌ None | A CDN caches content; it does not store application session state |

**Sticky sessions** are worth understanding rather than dismissing: the load balancer pins a user to one instance with a cookie. It’s the zero-code option and appears as a plausible answer. It loses whenever the question mentions scalability, instance failure, or “don’t store sessions on the instances”.

**Recall check — ElastiCache**

- **Shared session storage after migrating to multiple servers** → ElastiCache Redis
- **Cache only, simplest, multi-threaded** → Memcached
- **Avoid storing sessions on EC2** → Redis
- **Leaderboard / sorted set** → Redis
- **CDN for static content** → CloudFront (not session storage)
- **Stale cached data** → add TTL, or write-through
- **First request slow** → lazy-loading cache-miss penalty
- **Cache full of unread data** → write-through churn
- **Cache specifically for DynamoDB** → DAX

---

# Question patterns

> After migrating to multiple application servers behind a load balancer, users appear logged out on their second request. What should store the session state?
> → **ElastiCache Redis (central, shared, replicated)**

> A simple horizontally scaled cache with no persistence requirement and multi-threading is all that is needed. Which engine?
> → **Memcached**

> Which engine should store sessions so they survive a node failure?
> → **Redis — it has replication and persistence; Memcached does not**

> A leaderboard needs sorted-set data structures. Which engine?
> → **Redis**

> Static content must be cached at the edge. Which service?
> → **CloudFront (it does not store application session state)**

> Users see outdated data from the cache. What fixes it?
> → **Add a TTL, or switch to write-through**

> The first request after a deploy is always slow. What is the cause?
> → **The lazy-loading (cache-aside) cache-miss penalty**

> The cache is full of data nobody ever reads. Which strategy caused it?
> → **Write-through cache churn**

> DynamoDB reads must be accelerated with no application code changes. Which service?
> → **DAX — not ElastiCache**

> Which caching strategy rebuilds the cache after a failure instead of serving stale data?
> → **Lazy loading (cache-aside)**

> Which caching strategy updates the cache on every write so reads are never stale?
> → **Write-through**

> What is the main downside of write-through caching?
> → **A write penalty on every write, plus cache churn and the need for a lazy-load fallback**

> What is a shard?
> → **A partition of Redis/Valkey data**

> What composes a Redis/Valkey shard?
> → **1 primary (read/write) and 0–5 replicas (read-only)**

> What happens when cluster mode is enabled on Redis/Valkey?
> → **Data is automatically partitioned across shards**

> Does Memcached have primary and replica nodes?
> → **No — it is just multiple nodes with data distributed across them**

> What is an ElastiCache node?
> → **A single cache server with CPU, RAM, and a network endpoint**

> What is an ElastiCache cluster?
> → **A logical group of one or more nodes that the application sees as a single cache**

> What do multiple nodes provide?
> → **More memory, higher throughput, better availability, and easier scaling**

> How do Redis and Memcached scale out?
> → **Redis by cluster mode (sharding); Memcached by adding nodes with client-side sharding**

> Which engine supports backup and restore, snapshots, and persistence?
> → **Redis**

> Which engine is multi-threaded?
> → **Memcached**

> Which engine supports pub/sub and rich data structures like lists, sets, and sorted sets?
> → **Redis**

> Which engine supports replication and Multi-AZ failover?
> → **Redis**

> A load balancer pins a user to one instance with a cookie. What is this, and why is it a weak answer?
> → **Sticky sessions — less scalable, and users lose their sessions when that instance dies**

> Which approach is the correct answer for scalable session management?
> → **ElastiCache Redis — any server can serve any request**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Shared sessions | **ElastiCache Redis** |
| DynamoDB no-code-change cache | **DAX** |
| No persistence, simple, multi-threaded | **Memcached** |
| Persistence / snapshots / backup | **Redis only** |
| Replication + Multi-AZ failover | **Redis only** |
| Data structures, pub/sub, sorted sets | **Redis only** |
| Sticky sessions downside | **less scalable; instance failure loses sessions** |
| Avoid stale data | **TTL or write-through** |
| Rebuild after failure | **lazy loading (cache-aside)** |
| First request slow | **lazy-loading cache-miss penalty** |
| Cache full of unread data | **write-through churn** |
| Write-through | **updates cache on every write** |
| Node | **one cache server (CPU, RAM, endpoint)** |
| Cluster | **one or more nodes** |
| Shard | **a partition of Redis/Valkey data** |
| Redis shard contents | **1 primary + 0–5 replicas** |
| Cluster mode enabled | **data partitioned across shards** |
| Memcached architecture | **nodes only, no primary/replica** |
| More nodes | **more memory + throughput + availability + scaling** |
| Redis scale out | **cluster mode (sharding)** |
| Memcached scale out | **add nodes, client-side sharding** |
| Leaderboard / sorted set | **Redis** |
| CDN for static content | **CloudFront (not session storage)** |
| Cache specifically for DynamoDB | **DAX** |
| Reduce database reads generally | **ElastiCache / DAX** |
| Cache user sessions | **ElastiCache Redis, not DAX** |
| In-memory latency | **sub-millisecond** |
| Session lost to node failure | **logs the user out (why Redis wins)** |
| Engines offered | **Valkey/Redis OSS, Memcached** |
| Cache-miss cost | **three round trips instead of one** |
