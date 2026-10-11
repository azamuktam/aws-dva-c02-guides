# PART 2 — DATA: WHERE STATE LIVES

## AWS Database & Storage Access Patterns
<table header-row="true">
<tr>
<td>Service</td>
<td>Think of it as</td>
<td>Best when...</td>
<td>Example</td>
</tr>
<tr>
<td>**DynamoDB**</td>
<td>Fast NoSQL database</td>
<td>You need **key-based access, huge scale, flexible schema**</td>
<td>Get order by `orderId`</td>
</tr>
<tr>
<td>**RDS**</td>
<td>Traditional SQL database</td>
<td>You need **SQL, joins, relationships, transactions**</td>
<td>`JOIN customers + orders`</td>
</tr>
<tr>
<td>**Aurora**</td>
<td>High-performance SQL database</td>
<td>You need **relational SQL + higher performance/availability**</td>
<td>Large e-commerce DB</td>
</tr>
<tr>
<td>**Redshift**</td>
<td>Data warehouse</td>
<td>You need **analytics over huge datasets**, not normal application transactions</td>
<td>Analyze billions of sales records</td>
</tr>
<tr>
<td>**ElastiCache**</td>
<td>General-purpose cache</td>
<td>You need **very fast repeated reads**</td>
<td>Cache popular products</td>
</tr>
<tr>
<td>**DAX**</td>
<td>DynamoDB cache</td>
<td>You need **microsecond reads specifically for DynamoDB**</td>
<td>Frequently read DynamoDB items</td>
</tr>
<tr>
<td>**S3**</td>
<td>Object/file storage</td>
<td>You store **files/objects**</td>
<td>Images, videos, PDFs</td>
</tr>
<tr>
<td>**EBS**</td>
<td>EC2 disk</td>
<td>One EC2 instance needs **block storage**</td>
<td>Database disk / application disk</td>
</tr>
<tr>
<td>**EFS**</td>
<td>Shared filesystem</td>
<td>Multiple EC2s/containers need to **access the same files**</td>
<td>Shared `/uploads` folder</td>
</tr>
</table>
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
