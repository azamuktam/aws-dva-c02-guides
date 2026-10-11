## DynamoDB Accelerator (DAX)
**Why it exists**
DynamoDB is fast (single-digit milliseconds) but not *memory* fast. For read-heavy workloads hitting the same items repeatedly, an in-memory cache gets you to microseconds. DAX’s specific value is that it is **API-compatible with the DynamoDB SDK**, so adopting it is a client configuration change rather than a rewrite — which is precisely why it beats ElastiCache in DynamoDB caching questions.
**How it works**
- Runs as a **cluster inside your VPC**, in front of the table.
- Two caches: an **item cache** (for `GetItem`/`BatchGetItem`) and a **query cache** (for `Query`/`Scan`).
- **Write-through**: writes go to DynamoDB and update the cache together, so the cache doesn’t serve stale data it just wrote.
- Default TTL 5 minutes.
- **Strongly consistent reads bypass DAX entirely** — they have to, because a cache cannot guarantee it holds the leader’s latest value. So DAX gives you nothing on strongly consistent reads.
| Cache | Use it for |
| --- | --- |
| **DAX** | DynamoDB, microsecond reads, minimal code change |
| **ElastiCache Redis** | Anything else — sessions, leaderboards, arbitrary computed results, pub/sub |
| **ElastiCache Memcached** | Simple, large, throwaway key-value cache |
**Recall check**
- **Microsecond DynamoDB reads with almost no code change** → DAX
- **DAX does not help strongly consistent reads**
- **Cache for something other than DynamoDB** → ElastiCache
- **Cache user sessions** → ElastiCache Redis, not DAX
- **Reduce database reads generally** → ElastiCache / DAX
---
