*PART 4 — SECURITY: IDENTITY, CREDENTIALS, AND ENCRYPTION*

**26% of the exam — the second-largest domain, and where borderline candidates lose.** Most people under-study it because it feels like administration rather than development. It isn’t: it’s about how your code proves who it is and protects what it handles.

Three questions run through everything here:

1. **Who is making this call?** (IAM, STS, Cognito)
2. **Is this specific action allowed on this specific resource?** (policies and their evaluation)
3. **How is this data protected, and who holds the key?** (KMS, Secrets Manager, encryption)

One principle dominates: **credentials should not exist at rest.** Roles issue temporary credentials on demand. Any answer involving stored access keys is almost always wrong.

---

## AWS Identity and Access Management (IAM)

**Why it exists**

Every AWS API call must be authenticated and authorised. IAM is the system that decides, for each request, whether the caller is who they claim to be and whether they may do what they’re asking.

**The four primitives**

**User** — a permanent identity for a person or application. Has long-lived credentials (a console password, access keys).

**Group** — a collection of users, so a permission set is defined once and applied to many. Groups cannot contains other groups and cannot be principals in a policy.

**Role** — an identity with permissions but **no permanent credentials**. It is *assumed*, which produces temporary credentials via STS. This is the mechanism that lets a Lambda function, EC2 instance, or ECS task call AWS APIs without any secret being stored anywhere.

**Policy** — a JSON document defining permissions.

|    | Difference |
| --- | --- |
| **User** | Permanent identity, long-lived credentials |
| **Group** | Same permissions for many users |
| **Role** | **Temporary** credentials, assumable by services or people |
| **Policy** | Defines Allow / Deny on actions and resources |

IAM roles and resource-based policies can provide cross-account access **only within the same AWS partition**. They cannot be used to delegate access between, for example, `aws` and `aws-cn`.

**AWS partition → **contains multiple AWS Regions

Example:

- **`aws` partition**
  - `us-east-1`
  - `eu-west-1`
  - `ap-southeast-1`
- **`aws-cn` partition**
  - `cn-north-1`
  - `cn-northwest-1`

**Why roles beat access keys, concretely:** a stored key can be committed to Git, leaked in a log, or copied to a laptop, and it stays valid until someone notices and revokes it. A role’s credentials are generated on demand, expire automatically, and never exist in your code or configuration. This is why “attach an IAM role” beats “store the access key” on every single question where both appear.

**MFA** adds a second factor. Always enable it on the root user.

**Policy types — and what question each answers**

Understanding these as *answers to different questions* makes them easy to keep apart.

**Identity-based policy** — attached to a user, group, or role. Answers: **“What can this identity do?”**

```plain text
IAM User → Policy → Allow s3:GetObject
```

**Resource-based policy** — attached to the resource itself. Answers: **“Who can access this resource?”**

```plain text
S3 Bucket → Bucket Policy → Allow User A
```

Examples: S3 bucket policy, SQS queue policy, SNS topic policy, Lambda resource policy, KMS key policy.

This is what makes **cross-account access** possible without creating users in both accounts: the resource itself names the external principal.

**Trust policy** — a special resource-based policy attached to a **role**. Answers: **“Who is allowed to assume this role?”**

```plain text
EC2 Instance
     ↓ AssumeRole
IAM Role
     ├── Trust policy        →  WHO may assume it
     └── Permissions policy  →  WHAT it can do once assumed
```

For an IAM **role**, there are two different policies:

| Policy | Answers | Type |
| --- | --- | --- |
| **Trust policy** | **Who can assume this role?** | **Resource-based policy** |
| **Permissions policy** | **What can this role do?** | **Identity-based policy** |

**Other policy types**

| Policy | Purpose |
| --- | --- |
| **Permissions policy** | Grants Allow/Deny on actions |
| **Trust policy** | Defines who may assume a role |
| **Permissions boundary** | The **maximum** permissions an identity can ever receive, however many policies are attached |
| **SCP (Service Control Policy)** | Limits what an entire account or OU can do, in AWS Organizations |
| **Session policy** | Passed at assume time; can only narrow the role’s permissions |
| **ACL** | Older, cross-account resource access control (legacy S3) |

A permissions boundary does not grant anything — it caps. An identity’s effective permissions are the *intersection* of its policies and its boundary.

**Policy elements**

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "IpAddress": { "aws:SourceIp": "10.0.0.0/24" }
  }
}
```

**Effect** — `Allow` or `Deny`.

**Action** — the API operation: `s3:GetObject`, `ec2:StartInstances`, `dynamodb:PutItem`. Wildcards allowed (`s3:Get*`).

**Resource** — the ARN affected: `arn:aws:s3:::my-bucket/*`.

**Principal** — who receives the permission. **Used only in resource-based policies**, because in an identity-based policy the principal is implicit (whatever the policy is attached to). Noticing a `Principal` element tells you immediately that you’re looking at a resource-based policy.

```json
"Principal": { "AWS": "arn:aws:iam::123456789012:user/Alice" }
```

**Condition** — extra requirements. Common condition keys: source IP, MFA present, Region, resource tags, time of day, `aws:SourceVpce`, `aws:SecureTransport`.

Use the `aws:SecureTransport` condition key and explicitly **Deny** requests when it is `false`, enforcing HTTPS/SSL for S3 requests.

**`aws:SourceVpce`:** Restricts access to resources so requests are allowed only through a **specific VPC endpoint** identified by its `vpce-...` ID

**Policy variables**

Variables let one policy serve many users, which is how you build per-user isolation without writing a policy per person.

```plain text
bucket-a/user/${aws:username}/
```

resolves per caller:

```plain text
user-x → bucket-a/user/user-x/
user-y → bucket-a/user/user-y/
```

Uses: per-user folders, multi-tenant isolation, tag-based access.

**Policy evaluation logic**

The order is fixed and worth understanding rather than memorising, because it explains a whole class of “the user has Allow but still can’t do it” questions.

```plain text
1. Start with an implicit DENY (default deny)
2. Evaluate all applicable policies
3. Any explicit DENY anywhere  →  DENIED, immediately and permanently
4. An explicit ALLOW with no DENY  →  ALLOWED
5. No ALLOW at all  →  DENIED (default)
```

**Explicit deny always wins.** No amount of Allow can override it, and it does not matter which policy type the Deny came from — identity, resource, boundary, SCP, or session policy.

```plain text
Allow s3:*  +  Deny s3:DeleteObject
        ↓
Can read and write, CANNOT delete
```

Practically, this means: when a permission unexpectedly doesn’t work, look for a Deny — in an SCP, a permissions boundary, or a resource policy — before adding more Allows.

**IAM access to the Billing console**

A quirk worth knowing because it looks like a broken policy. **IAM policies alone do not grant access to Billing and Cost Management.  the AWS account root user must first turn on global billing access **

Symptom: an IAM user has billing permissions in their policy and still cannot open the Billing console.<br>Fix: root enables IAM access to Billing.

**IAM Credential Report:** A downloadable CSV report showing all IAM users and the status of their **passwords, access keys, MFA, and other credentials** for security auditing

**IAM certificate storage (legacy)**

IAM can store uploaded SSL/TLS server certificates. It is a legacy feature for cases where ACM cannot be used, and it does **not** renew certificates automatically.

- **Create and auto-renew certificates** → **ACM**
- **Upload an existing certificate** → ACM import, or IAM (legacy)
- **Private/internal certificates** → **AWS Private CA**

**IAM analysis tools — three different questions**

| Tool | Answers |
| --- | --- |
| **IAM Access Analyzer** | “Which of my resources are shared externally or publicly?” |
| **IAM Policy Simulator** | “Would this policy allow this specific action?” |
| **IAM Access Advisor** | “Which services has this identity actually used?” (for pruning unused permissions) |
| **AWS Config** | “Has this resource’s configuration changed, and is it compliant?” |
| **IAM Credential Report** | A downloadable CSV report showing all IAM users and the status of their **passwords, access keys, MFA, and other credentials** for security auditing |
|    |    |

**Access Analyzer** continuously analyses identity and resource policies and generates findings when something is reachable from outside the account. It covers S3 buckets, KMS keys, IAM roles, SQS/SNS, Secrets Manager, and other supported resources. **It reports; it does not modify permissions.**

**Access Advisor** is how you implement least privilege on an existing identity: see what was never used, then remove it.

**Recall check — IAM**

- **AWS service needs permissions** → IAM **role**
- **Multiple users need the same permissions** → IAM **group**
- **Grant or deny actions** → IAM **policy**
- **the maximum permissions that an IAM user or role can have** → permissions boundary
- **Temporary credentials** → IAM role (via STS)
- **Secure the root account** → enable MFA
- **Follow least privilege** → grant only what’s required
- **“What can this identity do?”** → identity-based policy
- **“Give access directly on a resource” / cross-account** → resource-based policy
- **“Who can assume this role?”** → **trust policy**
- **IAM resource-based policy in a role context** → trust policy
- **Cap the maximum permissions an identity can receive** → permissions boundary
- **Restrict an entire AWS account or OU** → SCP
- **`Principal` element present** → it’s a resource-based policy
- **One policy that works per-user** → policy variables (`${aws:username}`)
- **Explicit Deny always wins**
- **Permission unexpectedly denied** → look for a Deny (SCP, boundary, resource policy)
- **Billing permissions don’t work** → root must enable IAM access to Billing
- **Find public or cross-account access** → IAM Access Analyzer
- **Test whether a policy allows an action** → Policy Simulator
- **Find unused permissions** → Access Advisor
- **Track resource configuration changes** → AWS Config
- **Access Analyzer reports, **it does not fix
- **Enable HTTPS / auto-renew a certificate** → AWS Certificate Manager
- **Upload an existing certificate** → **AWS Certificate Manager**  or **IAM certificate store** (legacy)
- **Private internal certificates** → AWS Private CA
- **Inline Policy** = permissions directly embedded into that identity

# Question patterns

> An application on EC2 calls S3 and no access keys may be stored anywhere. Which mechanism?
> → **An IAM role attached via an instance profile (temporary credentials from STS)**

> A Lambda function must call DynamoDB with no secret in its code. What should you attach?
> → **An IAM role as the function's execution role**

> Several IAM users need the identical permission set. How do you define it once?
> → **Place the users in an IAM group and attach the policy to the group**

> Can an IAM group contain another IAM group?
> → **No — groups cannot contain groups**

> Can an IAM group be named as a Principal in a policy?
> → **No — groups cannot be principals**

> Which IAM identity has permissions but no permanent credentials and must be assumed?
> → **An IAM role**

> A role ARN in the aws partition must be assumed by a principal in aws-cn. Does it work?
> → **No — roles and resource policies only delegate within the same AWS partition**

> What does an AWS partition contain?
> → **Multiple AWS Regions (e.g. aws contains us-east-1, eu-west-1, ap-southeast-1)**

> What second factor should always be enabled on the root user?
> → **MFA**

> A policy is attached to a user, group, or role and answers "What can this identity do?"
> → **An identity-based policy**

> An S3 bucket policy names an external account so it can access the bucket. What type is it?
> → **A resource-based policy**

> Which policy type answers "Who can access this resource?"
> → **A resource-based policy (S3 bucket, SQS, SNS, Lambda, KMS key policy)**

> For an IAM role, which policy answers "Who is allowed to assume this role?"
> → **The trust policy**

> For an IAM role, which policy defines what it can do once assumed?
> → **The permissions policy (identity-based)**

> Is a role's trust policy identity-based or resource-based?
> → **Resource-based**

> What caps the maximum permissions an identity can ever receive?
> → **A permissions boundary**

> An identity has Allow policies and a permissions boundary. What are its effective permissions?
> → **The intersection of its policies and its boundary**

> What limits what an entire account or OU can do in AWS Organizations?
> → **A Service Control Policy (SCP)**

> A session policy is passed at assume time. Can it widen the role's permissions?
> → **No — it can only narrow them**

> Which older, legacy mechanism grants cross-account access to an S3 resource?
> → **An ACL**

> Seeing a "Principal" element in a policy tells you what?
> → **It is a resource-based policy**

> Which policy element names the API operation such as ec2:StartInstances?
> → **Action**

> In which policy type is Principal used?
> → **Only resource-based policies — it is implicit in identity-based policies**

> You must force HTTPS/SSL for S3 requests. Which condition key and effect?
> → **aws:SecureTransport, with an explicit Deny when it is false**

> Access must be allowed only through a specific VPC endpoint. Which condition key?
> → **aws:SourceVpce (the vpce-... ID)**

> One policy should give each user their own folder. Which construct?
> → **Policy variables such as bucket-a/user/${aws:username}/**

> What is the first step of IAM policy evaluation?
> → **Start from an implicit Deny (default deny)**

> A policy explicitly Allows an action but an SCP explicitly Denies it. Result?
> → **Denied — an explicit Deny always wins**

> An IAM user has billing permissions in their policy but cannot open the Billing console. Fix?
> → **The root user must enable IAM access to Billing and Cost Management**

> What produces a downloadable CSV of all IAM users and the status of their passwords, access keys, and MFA?
> → **The IAM Credential Report**

> Which tool finds resources that are shared publicly or cross-account?
> → **IAM Access Analyzer**

> Does IAM Access Analyzer fix the permissions it flags?
> → **No — it reports; it does not modify permissions**

> Which tool answers "Would this policy allow this specific action?"
> → **IAM Policy Simulator**

> Which tool shows which services an identity has actually used, to prune unused permissions?
> → **IAM Access Advisor**

> Which tool answers "Has this resource's configuration changed, and is it compliant?"
> → **AWS Config**

> A certificate must be created and auto-renewed. Which service?
> → **AWS Certificate Manager (ACM)**

> You need private/internal certificates. Which service?
> → **AWS Private CA**

> Does a certificate uploaded to the IAM certificate store auto-renew?
> → **No — the IAM certificate store is legacy and does not auto-renew**

> A permission you expected to be allowed still returns AccessDenied. What should you look for first?
> → **An explicit Deny — in an SCP, a permissions boundary, or a resource policy**

> A policy embedded directly into a single identity rather than attached as a managed document is called?
> → **An inline policy**

# Pocket card

| Keyword | Answer |
| --- | --- |
| AWS service needs permissions | **IAM role** |
| Multiple users, same permissions | **IAM group** |
| Grant or deny actions | **IAM policy** |
| Temporary credentials | **IAM role (via STS)** |
| No permanent credentials | **IAM role** |
| Group contains group | **No** |
| Group as Principal | **No** |
| Permanent identity, long-lived creds | **IAM user** |
| Cross-account (same partition) | **Role + resource-based policy / trust policy** |
| Cross-partition access | **Not possible** |
| Partition | **Group of Regions** |
| Secure the root account | **Enable MFA** |
| "What can this identity do?" | **Identity-based policy** |
| "Who can access this resource?" | **Resource-based policy** |
| "Who can assume this role?" | **Trust policy** |
| Trust policy type | **Resource-based policy** |
| Permissions policy type | **Identity-based policy** |
| Max permissions an identity can receive | **Permissions boundary** |
| Does a boundary grant? | **No — it caps (intersection)** |
| Restrict a whole account/OU | **SCP** |
| Passed at assume time | **Session policy (narrows only)** |
| Principal element present | **It's a resource-based policy** |
| Force HTTPS for S3 | **Deny when aws:SecureTransport is false** |
| Restrict to a VPC endpoint | **aws:SourceVpce** |
| Enforce MFA | **aws:MultiFactorAuthPresent** |
| Per-user folders, one policy | **Policy variables (${aws:username})** |
| Explicit Deny | **Always wins** |
| No Allow at all | **Denied (default deny)** |
| Unexpected AccessDenied | **Look for a Deny (SCP, boundary, resource policy)** |
| Billing console blocked | **Root must enable IAM access to Billing** |
| Credential audit CSV | **IAM Credential Report** |
| Public/cross-account exposure | **IAM Access Analyzer** |
| Access Analyzer behavior | **Reports only, does not fix** |
| Test a policy for an action | **Policy Simulator** |
| Unused permissions | **Access Advisor** |
| Config/compliance tracking | **AWS Config** |
| Auto-renew a certificate | **AWS Certificate Manager** |
| Upload an existing certificate | **ACM import or IAM store (legacy)** |
| Private/internal certificates | **AWS Private CA** |
| IAM certificate store renewal | **None (legacy)** |
| Inline policy | **Permissions embedded in that identity** |
| Resource ARN example | **arn:aws:s3:::my-bucket/\*** |
| Resource-based policy examples | **S3, SQS, SNS, Lambda, KMS key policy** |
| Legacy cross-account access control | **ACL** |
| Roles beat static keys | **On-demand, auto-expiring, never stored** |
| Common condition keys | **Source IP, MFA, Region, tags, time** |
| Credentials at rest | **Should not exist** |
