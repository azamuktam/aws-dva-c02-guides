## DynamoDB Streams
**What it is:** A record of **changes made to a DynamoDB table**.
When an item is **created, updated, or deleted**, DynamoDB Streams records that change for a limited time.
**Common use:** Trigger a Lambda function when the table changes.
```plain text
DynamoDB table
      ↓  item created / updated / deleted
DynamoDB Stream    (ordered, per partition key, 24-hour retention)
      ↓
Lambda (event source mapping)  or  KCL + Kinesis Adapter
      ↓
Process the change
```
**What to know**
- **Retention: 24 hours.**
- Event names: **`INSERT`****, ****`MODIFY`****, ****`REMOVE`**.
- **View types** (`StreamViewType`) — choose based on what your consumer needs:
	- `KEYS_ONLY` — just the keys
	- `NEW_IMAGE` — the item after the change
	- `OLD_IMAGE` — the item before the change
	- `NEW_AND_OLD_IMAGES` — both, which is what you need to compute *what changed*
- Ordering is preserved **per partition key**, which is why stream consumers are one-invocation-per-shard by default (see the Lambda event source mapping section).
- **Kinesis Data Streams for DynamoDB** is a separate option when 24 hours isn’t enough: up to 365 days of retention and more consumers.
**Common uses:** notify on data change, maintain an OpenSearch index, replicate to another system, audit trail, trigger workflows, archive TTL-expired items.
**Recall check**
- **Item created/updated/deleted → run code** → DynamoDB Streams + Lambda
- **Need the value before AND after** → `NEW_AND_OLD_IMAGES`
- **Streams retention** → 24 hours
- **Need longer retention or replay** → Kinesis Data Streams for DynamoDB
---
