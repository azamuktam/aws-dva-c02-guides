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
- **Encrypted configuration or secret, no rotation needed** → **Parameter Store `SecureString`**
- **Plain configuration value** → Parameter Store `String`
- **Multiple values in one parameter** → `StringList`
- **Feature flag / change behaviour without redeploying** → **AppConfig**
- **Validate configuration and roll back automatically** → **AppConfig**
- **Passwordless database login** → **IAM DB authentication**
- **Need encryption keys themselves** → **KMS**
- **Cache config in Lambda without an API call per invocation** → AppConfig / Parameter Store Lambda extension
- **Env vars over 4 KB** → Parameter Store / Secrets Manager / AppConfig

---

# Question patterns

> A database password for an RDS instance must be rotated automatically on a schedule. Which service?
> → **AWS Secrets Manager (drives a rotation Lambda)**

> Secrets Manager must update both the stored secret and the database credential at the same time. What performs that work?
> → **The rotation Lambda it invokes on a schedule**

> Built-in automatic rotation is required for Aurora, Redshift, and DocumentDB credentials. Which service?
> → **Secrets Manager (built-in rotation for RDS, Aurora, Redshift, DocumentDB)**

> An application just needs to read a plain connection-string value keyed by environment. Which service?
> → **Systems Manager Parameter Store (String)**

> A configuration value is sensitive and must be encrypted with KMS but does not need rotation. Which choice?
> → **Parameter Store `SecureString`**

> Several related values must be stored in a single parameter as a comma-separated list. Which parameter type?
> → **`StringList`**

> A feature flag must be toggled at runtime without redeploying the application. Which service?
> → **AWS AppConfig**

> A configuration change must be validated before deployment and rolled back automatically if alarms fire. Which service?
> → **AWS AppConfig (validators + monitors)**

> You want the maximum number of Parameter Store parameters per account per Region on the free tier. What is the limit?
> → **10,000 (Standard tier)**

> A workload needs up to 100,000 parameters per account per Region. Which tier is required?
> → **Advanced tier**

> A parameter value is 6 KB. Which tier of Parameter Store is needed?
> → **Advanced (Standard caps at 4 KB; Advanced allows 8 KB)**

> What is the maximum Parameter Store value size on the Standard tier?
> → **4 KB**

> What is the maximum Parameter Store value size on the Advanced tier?
> → **8 KB**

> Parameter policies such as expiration and no-change notification are required. Which tier supports them?
> → **Advanced tier only**

> A parameter must automatically delete itself at a specified time. Which parameter policy?
> → **`Expiration`**

> You must be notified before a parameter expires. Which parameter policy?
> → **`ExpirationNotification`**

> You must be notified when a parameter has not changed for a set period. Which parameter policy?
> → **`NoChangeNotification`**

> An application must load all of its configuration under `/myapp/prod/` in a single call. Which API?
> → **`GetParametersByPath`**

> A recently rotated secret must be rolled back to its prior value using versioning. Which labels enable this?
> → **Staging labels `AWSCURRENT` and `AWSPREVIOUS`**

> A Lambda function should read config without making an API call on every invocation. What should it use?
> → **The AppConfig Lambda extension or the Parameter Store Lambda extension (local cache)**

> A secret must be shared with another AWS account. Which mechanism?
> → **A Secrets Manager resource policy (cross-account access)**

> The cost of ordinary, non-rotating configuration is the concern. Which choice is free?
> → **Parameter Store Standard tier**

> An environment variable is larger than 4 KB. Which options can hold it?
> → **Parameter Store (Advanced), Secrets Manager, or AppConfig**

> You must describe the AppConfig object hierarchy. What is it?
> → **Application → Environment → Configuration Profile → Deployment**

> Where can an AppConfig Configuration Profile source its content from?
> → **Hosted configuration, S3, or Parameter Store**

> What checks an AppConfig configuration before it is deployed so a malformed config is rejected?
> → **Validators (JSON Schema or a Lambda function)**

> During the bake period, what can trigger an automatic AppConfig rollback?
> → **CloudWatch alarms configured as monitors**

> You must roll a configuration change out gradually with a pause before full deployment. Which AppConfig feature?
> → **Deployment strategies (linear or exponential) with a bake time**

> Parameter Store changes must generate a notification to another service. Which integration is used?
> → **EventBridge change notification**

> Every edit to a Parameter Store value must be traceable to a distinct revision. Which built-in feature is this?
> → **Versioning (each change creates a new version)**

> A CloudFormation template must pull an encrypted Parameter Store value at deploy time. Which syntax?
> → **A dynamic reference, `{{resolve:ssm-secure:...}}`**

> Passwordless authentication for an RDS database is required. Which mechanism?
> → **IAM database authentication (no password exists)**

> The encryption keys themselves are the asset you must manage. Which service?
> → **AWS KMS**

> Environment-specific values must be promoted between environments using aliases on parameters. Which Parameter Store feature?
> → **Labels (promote values between environments)**

> Which three services can hold secrets or configuration, and which two store secrets?
> → **Secrets Manager, Parameter Store, AppConfig; Secrets Manager and Parameter Store (SecureString) store secrets**

> AppConfig and Parameter Store belong to which parent service?
> → **AWS Systems Manager**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Automatic rotation | **Secrets Manager** |
| Encrypted config, no rotation | **Parameter Store `SecureString`** |
| Plain config value | **Parameter Store `String`** |
| Multiple values in one parameter | **`StringList`** |
| Feature flag | **AppConfig** |
| Validate + auto rollback | **AppConfig (validators + monitors)** |
| Passwordless DB login | **IAM DB authentication** |
| Manage encryption keys | **KMS** |
| Cache config in Lambda | **AppConfig / Parameter Store Lambda extension** |
| Env var over 4 KB | **Parameter Store Advanced / Secrets Manager / AppConfig** |
| Standard parameters per Region | **10,000** |
| Advanced parameters per Region | **100,000** |
| Standard max value size | **4 KB** |
| Advanced max value size | **8 KB** |
| Parameter policies | **Advanced tier only** |
| `NoChangeNotification` | **Notifies when a parameter has not changed** |
| `ExpirationNotification` | **Notifies before a parameter expires** |
| `Expiration` | **Deletes the parameter at a set time** |
| Fetch a whole prefix | **`GetParametersByPath`** |
| Roll back a rotated secret | **Staging labels `AWSPREVIOUS` / `AWSCURRENT`** |
| Built-in rotation targets | **RDS, Aurora, Redshift, DocumentDB** |
| Secret encryption | **KMS** |
| Cross-account secret | **Secrets Manager resource policy** |
| Parameter Store Standard cost | **Free** |
| Parameter Store Advanced cost | **Paid per parameter** |
| Secrets Manager cost | **Paid per secret** |
| Rotation driver | **Rotation Lambda** |
| AppConfig hierarchy | **Application → Environment → Profile → Deployment** |
| AppConfig config sources | **Hosted / S3 / Parameter Store** |
| AppConfig validators | **JSON Schema or Lambda** |
| AppConfig monitors | **CloudWatch alarms → auto rollback** |
| AppConfig gradual rollout | **Deployment strategy + bake time** |
| Parameter change notification | **EventBridge** |
| Parameter versioning | **New version per change** |
| Template secret reference | **`{{resolve:ssm-secure:...}}`** |
| Promote values across envs | **Parameter labels** |
| Part of Systems Manager | **Parameter Store & AppConfig** |
| Config vs secret | **AppConfig stores config, not secrets** |
| Secrets Manager vs Parameter Store | **Rotation vs no rotation** |
