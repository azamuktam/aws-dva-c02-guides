# PART 2 — DATA: WHERE STATE LIVES

## AWS Database & Storage Access Patterns
| Service | Think of it as | Best when... | Example |
| --- | --- | --- | --- |
| **DynamoDB** | Fast NoSQL database | You need **key-based access, huge scale, flexible schema** | Get order by `orderId` |
| **RDS** | Traditional SQL database | You need **SQL, joins, relationships, transactions** | `JOIN customers + orders` |
| **Aurora** | High-performance SQL database | You need **relational SQL + higher performance/availability** | Large e-commerce DB |
| **Redshift** | Data warehouse | You need **analytics over huge datasets**, not normal application transactions | Analyze billions of sales records |
| **ElastiCache** | General-purpose cache | You need **very fast repeated reads** | Cache popular products |
| **DAX** | DynamoDB cache | You need **microsecond reads specifically for DynamoDB** | Frequently read DynamoDB items |
| **S3** | Object/file storage | You store **files/objects** | Images, videos, PDFs |
| **EBS** | EC2 disk | One EC2 instance needs **block storage** | Database disk / application disk |
| **EFS** | Shared filesystem | Multiple EC2s/containers need to **access the same files** | Shared `/uploads` folder |
AWS categorizes DynamoDB as a key-value/document database, RDS/Aurora as relational databases, ElastiCache as an in-memory cache, and Redshift as a data warehouse.
For storage, AWS describes **EBS as block storage attached to EC2** and **EFS as shared file storage for multiple compute resources**.
### The mental shortcut
```plain text
Need normal application DATABASE?
        ↓
   ┌────┴────┐
   ↓         ↓
DynamoDB   RDS/Aurora
NoSQL      SQL

Need ANALYTICS?
        ↓
     Redshift

Need FAST CACHE?
        ↓
   ┌────┴────┐
   ↓         ↓
ElastiCache  DAX
general      DynamoDB

Need FILE STORAGE?
        ↓
   ┌────┼────┐
   ↓    ↓    ↓
  S3   EBS   EFS
files  disk  shared files
```
**Important distinction:** DAX is specifically a DynamoDB caching layer, while ElastiCache is a general-purpose in-memory caching service.
This version is much better for your exam because when you see **RDS / Aurora / DynamoDB / Redshift / ElastiCache / DAX / S3 / EBS / EFS** as answer choices, you can immediately ask: **“What kind of data/access does the question describe?”**
---
