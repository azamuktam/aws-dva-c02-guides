## Amazon Cognito

**User Pool = Login**

**Why it exists**

Two different problems, which is why Cognito has two components that people constantly conflate.

**Problem 1: your application needs users.** Sign-up, sign-in, password reset, MFA, email verification, social login. Building that securely is a lot of work and easy to get wrong.

**Problem 2: those users need to touch AWS directly.** A mobile app uploading photos straight to S3 needs *AWS* credentials — but you cannot ship AWS access keys inside an app on a phone.

**User Pools** solve the first. **Identity Pools** solve the second.

**Cognito User Pools — authentication**

A managed user directory. Handles sign-up, sign-in, password policies and reset, MFA, user profiles, and social/enterprise federation.

```plain text
User
  ↓ sign in
Cognito User Pool
  ↓
JWT tokens
  ↓
Your application / API Gateway
```

Returns three tokens, each with a distinct job:

- **ID token** — contains identity claims about the user (username, email). For *your application* to know who the user is.
- **Access token** — used to authorise API requests. For *the API* to decide what’s permitted.
- **Refresh token** — obtains new ID and access tokens without making the user log in again.

**API Gateway integration:**

```plain text
User
 ↓
Cognito User Pool
 ↓
Login
 ↓
JWT token
 ↓
API Gateway
 ↓
Cognito User Pool Authorizer
 ↓
Token valid?
 ├── YES → request reaches API ✅
 └── NO  → request rejected ❌
```

**Other features:**

- **Hosted UI** — an AWS-provided login page supporting username/password, OAuth, and social providers. Use it when you don’t want to build a login screen.
- **User pool groups** — organise users (Admin, User); group membership appears in the JWT claims, so your application or authorizer can act on role.
- **Lambda triggers** — hook into pre-sign-up, post-confirmation, pre-token-generation, custom auth flows.

**How token validation actually works**

The tokens are standard **JWTs**, signed by the user pool with RS256. Anything validating one does the same four checks:

```plain text
1. Fetch the pool's public keys from
   https://cognito-idp.<region>.amazonaws.com/<poolId>/.well-known/jwks.json
2. Verify the signature against the key identified by the token's `kid`
3. Check `exp` (not expired), `iss` (correct pool), and `aud` / `client_id`
4. Check `token_use` — "id" for an ID token, "access" for an access token
```

A **Cognito authorizer** in API Gateway does all of this for you, which is why it needs no code. A **Lambda authorizer** is the answer when you must do it yourself — a non-Cognito identity provider, or logic that goes beyond the token.

**Claim-based access** — the JWT carries claims, and you authorise on them rather than calling a user database. Group membership appears as **`cognito:groups`**; custom attributes appear as `custom:<name>`. A pre-token-generation Lambda can inject additional claims.

**OAuth 2.0 scopes and resource servers**

A **resource server** in a user pool defines an API and the **custom scopes** it accepts, such as `orders-api/orders.read` and `orders-api/orders.write`. An app client is then configured with the scopes it is permitted to request, and the **access token** carries the granted scopes in its `scope` claim.

The distinction the exam draws:

- **ID token** → *who the user is*. Use it for identity, never for authorising API calls.
- **Access token** → *what the caller may do*, expressed as scopes. This is what an API should check.

Standard scopes are `openid`, `profile`, `email`, `phone`, and `aws.cognito.signin.user.admin`. Supported OAuth flows: authorization code grant (recommended), implicit (legacy), and client credentials for machine-to-machine with no user present.

**Cognito Identity Pools — AWS credentials**

**Identity Pool = AWS access**

An identity pool takes a token from a trusted source and exchanges it for **temporary AWS credentials** attached to an IAM role.

```plain text
User
  ↓ (token from a Cognito user pool, Google, Facebook, SAML, or unauthenticated)
Cognito Identity Pool
  ↓
IAM Role
  ↓
Temporary AWS credentials (via STS)
  ↓
S3 / DynamoDB / any AWS service, directly
```

- Supports **authenticated** and **guest (unauthenticated)** identities with separate roles.
- Role permissions can be scoped per user with **policy variables**, so each user reaches only their own S3 prefix.
- This is what removes AWS access keys from client applications entirely.

**The distinction, stated plainly**

| Feature | **User Pool** | **Identity Pool** |
| --- | --- | --- |
| Purpose | **Authentication** — who is this user? | **Authorization to AWS resources** |
| Returns | **JWT tokens** | **Temporary AWS credentials** |
| Manages users | ✅ Yes | ❌ No |
| Uses IAM roles | ❌ No | ✅ Yes |
| Sign-up / sign-in | ✅ Yes | ❌ No |
| Access S3/DynamoDB directly | ❌ No | ✅ Yes |

**The full pattern for a mobile app uploading to S3 uses both:**

```plain text
1. User signs in           → User Pool → JWT
2. Exchange the JWT        → Identity Pool → temporary AWS credentials
3. Upload directly to S3   using those credentials
```

Neither alone is sufficient: an identity pool has no user directory, and a user pool cannot issue AWS credentials.

**Cognito Sync** is an older service for **syncing application data across** a user’s devices. Not authentication. Largely superseded by **AppSync**.

Use a **Post Authentication Lambda trigger** to capture successful Cognito logins, then have the Lambda write the login information to the **RDS database**

#### # Cognito Adaptive Authentication

**What it is:**

Checks the **risk of a login** and asks for MFA when the login looks suspicious.

**Example:**

Normal login → no MFA

Suspicious login → MFA

### Easy memory

> **User Pool → "Who are you?"** 👤
> **Identity Pool → "What AWS resources can you access?"** 🔑

**Recall check — Cognito**

- **User login / sign-up** → **User Pool**
- **Generate a JWT token** → **User Pool**
- **Secure API Gateway with a JWT, no code** → **Cognito User Pool authorizer**
- **Temporary AWS credentials for app users** → **Identity Pool**
- **Mobile app needs S3 access without embedding AWS keys** → **Identity Pool**
- **Need an IAM role for end users** → **Identity Pool**
- **Sign in AND then access S3 directly** → **User Pool + Identity Pool together**
- **User Pool = authentication. Identity Pool = authorization to AWS resources.**
- **Don’t want to build a login page** → Hosted UI
- **User Pool Authorizer = Check Cognito token before API access**
- **Admin vs normal user permissions** → user pool groups (claims in the JWT)
- **Guest access to AWS resources** → identity pool unauthenticated identities
- **Sync application data across devices** → Cognito Sync (legacy) / AppSync
- **Use Cognito Identity pools to enable trusted third-party authenticated users to access DynamoDB**
- **Risk-based MFA → Adaptive Authentication**

---

# Question patterns

> Users must register and sign in with email/password. Which component?
> → **Cognito user pool**

> Signed-in app users need temporary AWS credentials to use AWS services. Which component?
> → **Cognito identity pool (federated identities)**

> Guest users should access a limited dataset before signing up. Which feature?
> → **Identity pool unauthenticated role**

> A branded login page without building one. Which feature?
> → **Cognito hosted UI**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Directory | **user pool** |
| AWS credentials | **identity pool** |
| Tokens | **ID, access, refresh (JWT)** |
| Federation | **Google/Facebook/SAML/OIDC** |
| MFA | **user pool MFA** |
| Triggers | **Lambda (pre sign-up, post confirmation…)** |
