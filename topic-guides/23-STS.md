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

| API | Use |
| --- | --- |
| **`AssumeRole`** | Same-account or **cross-account** role assumption |
| **`AssumeRoleWithWebIdentity`** | OIDC / social login (Google, Facebook, Cognito) |
| **`AssumeRoleWithSAML`** | Enterprise identity provider (Active Directory, SAML 2.0) |
| **`GetSessionToken`** | MFA-protected temporary credentials for an IAM user |
| **`GetFederationToken`** | Temporary credentials for a non-IAM user |
| **`GetCallerIdentity`** | “Who am I?” — requires no permissions, invaluable for debugging |

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
- **`AccessDenied` assuming a role** → check **both** policies
- **Third-party vendor needs access to your account** → **`ExternalId`**
- **Longest session** → 12 hours (1 hour with role chaining)
- **Users signing in with Google/Facebook and needing AWS access** → Cognito identity pool (which calls `AssumeRoleWithWebIdentity` for you)
- **Enterprise Active Directory federation** → `AssumeRoleWithSAML`
- **MFA-protected temporary credentials for an IAM user** → `GetSessionToken`
- **Temporary credentials for a non-IAM user** → `GetFederationToken`
- **Temporary credentials always include a `SessionToken`**
- **Which identity am I using?** → `GetCallerIdentity`
- **Invalidate all existing sessions** → deny by `aws:TokenIssueTime`

---

# Question patterns

> What does AWS Security Token Service issue?
> → **Temporary AWS credentials**

> What four items make up a set of temporary credentials?
> → **Access key ID, secret access key, session token, and expiration time**

> Which API assumes a role in the same account or a different account?
> → **AssumeRole**

> Which API handles OIDC / social login (Google, Facebook)?
> → **AssumeRoleWithWebIdentity**

> Which API handles enterprise federation with Active Directory / SAML 2.0?
> → **AssumeRoleWithSAML**

> Which API returns MFA-protected temporary credentials for an IAM user?
> → **GetSessionToken**

> Which API returns temporary credentials for a non-IAM user?
> → **GetFederationToken**

> Which API answers "Who am I?" and requires no permissions?
> → **GetCallerIdentity**

> Account A must assume a role in Account B. Which two things must be configured?
> → **Account B's role trust policy naming A's principal, and A's identity policy allowing sts:AssumeRole**

> Only the role's trust policy is configured but Account A still gets an error. What error?
> → **AccessDenied — both sides must agree**

> What is the default maximum session duration for AssumeRole?
> → **1 hour**

> What is the longest configurable AssumeRole session duration?
> → **12 hours**

> What is the minimum session duration?
> → **15 minutes**

> Role A assumes Role B (role chaining). What is the maximum session duration?
> → **1 hour, regardless of settings**

> What protects a third-party SaaS vendor's role from the confused deputy problem?
> → **A unique ExternalId required in the trust policy**

> Can a session policy passed at assume time widen a role's permissions?
> → **No — it can only narrow them**

> Issued temporary credentials cannot be deleted. How do you revoke them?
> → **Attach an inline policy to the role denying all actions where aws:TokenIssueTime is earlier than now**

> Is STS global or regional?
> → **Global, but it exposes regional endpoints**

> Why use the regional STS endpoint?
> → **Lower latency and no single global dependency**

> Users sign in with Google/Facebook and need AWS access. What makes the web-identity call for you?
> → **A Cognito identity pool (calls AssumeRoleWithWebIdentity)**

> An EC2 application uses an IAM role for S3 access. Which service actually issues the credentials?
> → **AWS STS**

> Which side of cross-account access names the external principal?
> → **The role owner's trust policy (Account B)**

> What must always be present in temporary credentials that static keys lack?
> → **A session token**

> Which setting on the role bounds the session duration you can request?
> → **The role's max session duration**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Temporary credentials | **STS** |
| Credential set | **Access key ID + secret key + session token + expiration** |
| Same/cross-account role | **AssumeRole** |
| Web identity (OIDC/social) | **AssumeRoleWithWebIdentity** |
| SAML/AD federation | **AssumeRoleWithSAML** |
| MFA temp creds (IAM user) | **GetSessionToken** |
| Non-IAM user creds | **GetFederationToken** |
| Who am I | **GetCallerIdentity** |
| GetCallerIdentity permissions | **None required** |
| Cross-account requirements | **Trust policy (owner) + identity policy (caller)** |
| AccessDenied assuming a role | **Check both policies** |
| Confused deputy | **ExternalId** |
| Min session | **15 minutes** |
| Default session | **1 hour** |
| Max session | **12 hours** |
| Role chaining max | **1 hour** |
| Session policy | **Narrows only, never widens** |
| Revoke sessions | **Deny by aws:TokenIssueTime** |
| STS locality | **Global with regional endpoints** |
| Regional endpoint benefit | **Lower latency / no global dependency** |
| Social login + AWS access | **Cognito identity pool** |
| Always in temp creds | **SessionToken** |
| Trust policy owner side | **Role owner account (Account B)** |
| Caller side | **Identity policy with sts:AssumeRole** |
| Middleman issuing creds | **STS** |
| Bounds requested duration | **The role's max session duration** |
