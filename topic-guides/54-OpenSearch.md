### Amazon OpenSearch Service
**Why it exists.** Neither DynamoDB nor a relational database does full-text search well — ranking, fuzzy matching, faceting, and aggregation over free text are a different problem. OpenSearch is the managed search and log-analytics engine.
**Uses:** full-text search in an application, log analytics dashboards, observability.
**The pattern that appears on the exam:**
```plain text
DynamoDB (source of truth)
      ↓ DynamoDB Streams
   Lambda
      ↓
OpenSearch (search index)
```
Search hits OpenSearch, then retrieves the full item from DynamoDB by key. This keeps DynamoDB as the authoritative store while getting real search.
Also fed by **Kinesis Data Firehose** and by **CloudWatch Logs subscription filters**. **OpenSearch Dashboards** provides visualisation.
**Recall check**
- **Full-text search** → OpenSearch
- **Search over DynamoDB data** → Streams → Lambda → OpenSearch
- **Log analytics dashboards** → OpenSearch (or CloudWatch Logs Insights for ad-hoc queries)
---
