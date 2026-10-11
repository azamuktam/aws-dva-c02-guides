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
