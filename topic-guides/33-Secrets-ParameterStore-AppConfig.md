## Secrets Manager, Parameter Store, and AppConfig
**Why there are three**
They look overlapping but answer different questions.
- **Does it need to rotate automatically?** → Secrets Manager
- **Is it just a value my application reads?** → Parameter Store
- **Is it something I want to change gradually and roll back safely?** → AppConfig
**AWS Secrets Manager**
Stores passwords, database credentials, API keys, and tokens, encrypted with **KMS**.
The distinguishing feature is **automatic rotation**: Secrets Manager can invoke a rotation Lambda on a schedule to generate a new credential and update both the secret and the target system. It ships **built-in rotation support for RDS, Aurora, Redshift, and DocumentDB**, which is why “database credentials that must rotate automatically” is always Secrets Manager and never Parameter Store.
Also: cross-account access via resource policy, and versioning with staging labels (`AWSCURRENT`, `AWSPREVIOUS`) so a rotation can be rolled back.
### **AWS Systems Manager Parameter Store**
Stores configuration values and simple secrets in a hierarchical path structure (`/myapp/prod/db-host`).
**Parameter types:**
- **String** — plain text
- **StringList** — comma-separated values
- **SecureString** — encrypted with KMS
**No built-in automatic rotation.** Standard tier is free, which makes it the right default for ordinary configuration and for secrets that don’t need rotation.
**Standard vs Advanced tier** — a limit worth knowing because it decides whether a value fits at all:
|    | **Standard** | **Advanced** |
| --- | --- | --- |
| Parameters per account per Region | **10,000** | **100,000** |
| Max value size | **4 KB** | **8 KB** |
| Parameter policies (expiration, no-change notification) | ❌ | ✅ |
| Cost | **Free** | Paid per parameter |
Supports versioning (every change creates a new version, and you can reference a specific one), **hierarchical paths** — `GetParametersByPath` fetches a whole prefix such as `/myapp/prod/` in one call, which is the efficient way to load all of an application’s configuration — labels for promoting values between environments, and change notification via EventBridge.
#### SSM Parameter Policies(Advanced tier only**)**
**`NoChangeNotification`** → Notifies when a parameter has not changed for a specified time.
**`ExpirationNotification`** → Notifies before a parameter is going to expire.
**`Expiration`** → Automatically expires/deletes the parameter at the specified time.
All three are parameter policies and require the Advanced tier.
**AWS AppConfig**
Also part of Systems Manager, but aimed at a different problem: **changing application behaviour safely at runtime**, without redeploying.
```plain text
Application
  └── Environment (dev / prod)
        └── Configuration Profile   (hosted, S3, or Parameter Store)
              └── Deployment        (strategy + validators + monitors)
```
What makes it distinct:
- **Validators** — a JSON Schema or a Lambda function checks the configuration **before** it deploys. A malformed config is rejected rather than breaking production.
- **Deployment strategies** — roll the change out linearly or exponentially, with a **bake time**.
- **Monitors** — CloudWatch alarms during the bake period trigger **automatic rollback**.
- **Lambda extension / agent** caches the configuration locally so you aren’t making an API call per invocation.
Uses: **feature flags**, operational toggles, allow-lists, tuning limits without a deploy.
**The comparison**
|    | **Secrets Manager** | **Parameter Store** | **AppConfig** | **KMS** | **IAM DB Auth** |
| --- | --- | --- | --- | --- | --- |
| Stores secrets | ✅ | ✅ (`SecureString`) | ❌ (config) | ❌ keys only | ❌ no password exists |
| Encryption | ✅ KMS | ✅ KMS | — | ✅ | IAM tokens |
| **Automatic rotation** | ✅ | ❌ | ❌ | ✅ key rotation | N/A |
| Validation before deploy | ❌ | ❌ | ✅ | — | — |
| Gradual rollout + rollback | ❌ | ❌ | ✅ | — | — |
| Cost | Paid per secret | Standard tier free | Paid | Per key + requests | Free |
**Recall check — secrets and config**
- **Database credentials + automatic rotation** → **Secrets Manager**
- **Encrypted configuration or secret, no rotation needed** → **Parameter Store ****`SecureString`**
- **Plain configuration value** → Parameter Store `String`
- **Multiple values in one parameter** → `StringList`
- **Feature flag / change behaviour without redeploying** → **AppConfig**
- **Validate configuration and roll back automatically** → **AppConfig**
- **Passwordless database login** → **IAM DB authentication**
- **Need encryption keys themselves** → **KMS**
- **Cache config in Lambda without an API call per invocation** → AppConfig / Parameter Store Lambda extension
- **Env vars over 4 KB** → Parameter Store / Secrets Manager / AppConfig
---
