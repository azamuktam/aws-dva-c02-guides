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
|    | RDS Encryption | TDE |
| --- | --- | --- |
| Encryption level | AWS/RDS storage | SQL Server database engine |
| Key management | AWS KMS | SQL Server/TDE |
| Encrypts at rest | ✅ | ✅ |
| Automatic encrypt/decrypt | ✅ | ✅ |
| Application changes | No | No |
| Main exam clue | **KMS / RDS encryption** | **SQL Server / TDE** |
**Important distinction**
Both can encrypt data before it is stored and decrypt it when accessed.
> **RDS encryption = AWS-level storage encryption**
> **TDE = SQL Server-level transparent database encryption**
