## AWS Security Token Service (STS)
**What it is:**
STS gives you **temporary AWS credentials**.
Instead of using a permanent Access Key + Secret Key, you can request credentials that work for a **limited time**.
```plain text
Application
    ↓
STS
    ↓
Temporary credentials
    ↓
AWS service
```
Temporary credentials contain:
- **Access key ID**
- **Secret access key**
- **Session token**
- **Expiration time**
When you use `AssumeRole`, AWS STS is the service that issues the temporary credentials
### Example
An EC2 application needs to access S3.
Instead of storing permanent credentials:
```plain text
EC2 → permanent AWS keys ❌
```
You can use an IAM role, and AWS provides temporary credentials:
```plain text
EC2
 ↓
IAM Role
 ↓
STS
 ↓
Temporary credentials
 ↓
S3
```
**Why use STS?**
**Security:** credentials automatically expire.
Common uses:
- IAM roles
- Cross-account access
- Federated users
- Applications that need temporary AWS access
**The APIs**
<table header-row="true">
<tr>
<td>API</td>
<td>Use</td>
</tr>
<tr>
<td>**`AssumeRole`**</td>
<td>Same-account or **cross-account** role assumption</td>
</tr>
<tr>
<td>**`AssumeRoleWithWebIdentity`**</td>
<td>OIDC / social login (Google, Facebook, Cognito)</td>
</tr>
<tr>
<td>**`AssumeRoleWithSAML`**</td>
<td>Enterprise identity provider (Active Directory, SAML 2.0)</td>
</tr>
<tr>
<td>**`GetSessionToken`**</td>
<td>MFA-protected temporary credentials for an IAM user</td>
</tr>
<tr>
<td>**`GetFederationToken`**</td>
<td>Temporary credentials for a non-IAM user</td>
</tr>
<tr>
<td>**`GetCallerIdentity`**</td>
<td>“Who am I?” — requires no permissions, invaluable for debugging</td>
</tr>
</table>
**Cross-account access — the two-sided requirement**
This is the single most testable STS fact, and it follows directly from the trust-policy/permissions-policy split.
**Both sides must agree:**
```plain text
Account B (the role owner)
   Role's TRUST POLICY must name Account A's principal

Account A (the caller)
   Caller's IDENTITY POLICY must allow sts:AssumeRole on that role ARN
```
STS is the **middleman that actually gives Account A temporary credentials for Account B's role**.
```plain text
Account A                         Account B
Developer                         Role
    │                               │
    │  "Can I assume this role?"    │
    └──────────→ STS ───────────────┤
                 │                  │
                 │ checks permissions
                 │ + trust policy
                 ↓
        Temporary credentials
                 ↓
        Account A can now
        act as the role in B
```
Configure only one and you get **`AccessDenied`**. So when a question describes a failing cross-account assume-role, the answer is **check both** — not one or the other.
**Other details**
- **Session duration:** 15 minutes up to the role’s **max session duration** (default 1 hour, configurable to **12 hours**).
- **Role chaining** (role A assumes role B) caps the session at **1 hour** regardless of settings.
- **`ExternalId`** protects against the **confused deputy** problem. When you grant a **third-party** SaaS vendor access to your account, the vendor’s role can be assumed by the vendor for *any* customer. Requiring a unique `ExternalId` in the trust policy ensures your account can only be accessed on your behalf.
- **Session policies** passed at assume time can only **narrow** permissions, never widen them.
- **Revoking access:** you cannot delete issued temporary credentials. Instead, attach an inline policy to the role denying all actions where `aws:TokenIssueTime` is earlier than now — invalidating every existing session at once.
- STS is global but exposes **regional endpoints**; use the regional one to reduce latency and avoid a single global dependency.
**Recall check — STS**
- **Temporary credentials** → STS
- **Cross-account access** → `AssumeRole`, with the trust policy **and** the caller’s policy both configured
- **`AccessDenied`**** assuming a role** → check **both** policies
- **Third-party vendor needs access to your account** → **`ExternalId`**
- **Longest session** → 12 hours (1 hour with role chaining)
- **Users signing in with Google/Facebook and needing AWS access** → Cognito identity pool (which calls `AssumeRoleWithWebIdentity` for you)
- **Enterprise Active Directory federation** → `AssumeRoleWithSAML`
- **MFA-protected temporary credentials for an IAM user** → `GetSessionToken` 
- **Temporary credentials for a non-IAM user** → `GetFederationToken`
- **Temporary credentials always include a ****`SessionToken`**
- **Which identity am I using?** → `GetCallerIdentity`
- **Invalidate all existing sessions** → deny by `aws:TokenIssueTime`
---
