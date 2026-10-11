## RDS Encryption and SQL Server Encryption
### RDS Storage Encryption
**What it is:** Encrypts **RDS data at rest** using **AWS KMS**.
**How it works:**
```plain text
Database data
   ↓
RDS encryption
   ↓
Encrypted storage
```
RDS automatically encrypts data before it is written to the underlying storage and decrypts it when the database reads it.
**Also protects:**
- Automated backups
- Read replicas
- Snapshots
**Key point:** The application does not need to encrypt/decrypt the data itself.
**Exam memory:**
> **RDS encryption → AWS KMS → encryption at rest**
---
### Transparent Data Encryption (TDE)
**What it is:** A **SQL Server database-engine feature** that transparently encrypts database files/data at rest.
**How it works:**
```plain text
SQL Server
   ↓
TDE encrypts database data
   ↓
Storage
```
When SQL Server reads the data, TDE transparently decrypts it.
**Key point:** The application does not need to change its SQL queries or encryption logic.
**Exam memory:**
> **SQL Server + transparent database encryption → TDE**
---
### RDS Encryption vs TDE
<table header-row="true">
<tr>
<td></td>
<td>RDS Encryption</td>
<td>TDE</td>
</tr>
<tr>
<td>Encryption level</td>
<td>AWS/RDS storage</td>
<td>SQL Server database engine</td>
</tr>
<tr>
<td>Key management</td>
<td>AWS KMS</td>
<td>SQL Server/TDE</td>
</tr>
<tr>
<td>Encrypts at rest</td>
<td>✅</td>
<td>✅</td>
</tr>
<tr>
<td>Automatic encrypt/decrypt</td>
<td>✅</td>
<td>✅</td>
</tr>
<tr>
<td>Application changes</td>
<td>No</td>
<td>No</td>
</tr>
<tr>
<td>Main exam clue</td>
<td>**KMS / RDS encryption**</td>
<td>**SQL Server / TDE**</td>
</tr>
</table>
**Important distinction**
Both can encrypt data before it is stored and decrypt it when accessed.
> **RDS encryption = AWS-level storage encryption**
> **TDE = SQL Server-level transparent database encryption**
