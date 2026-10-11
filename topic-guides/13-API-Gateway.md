## Amazon API Gateway

**Why it exists**

A Lambda function has no URL, no throttling, no authentication, no caching, and no versioning. API Gateway is the managed front door that supplies all of it, so your function only contains business logic.

```plain text
Client
  ↓
API Gateway
  ├── Authorizer        (who are you?)
  ├── Request validation / mapping template
  ├── Throttling / usage plan
  ├── Cache
  ↓
Integration  →  Lambda | HTTP endpoint | AWS service | ALB | EC2
  ↓
Response mapping template
  ↓
Client
```

**The three API types**

|    | **REST API** | **HTTP API** | **WebSocket API** |
| --- | --- | --- | --- |
| Direction | Request/response | Request/response | **Bidirectional, persistent** |
| Cost | Highest | \~70% cheaper | Per message + connection minute |
| Latency | Higher | Lower | — |
| **Caching** | ✅ | ❌ | ❌ |
| **API keys + usage plans** | ✅ | ❌ | ❌ |
| **Request validation** | ✅ | ❌ | ❌ |
| **Mapping templates (VTL)** | ✅ | ❌ | ✅ |
| **WAF** | ✅ | ❌ | ❌ |
| Private APIs (VPC only) | ✅ | ❌ | ❌ |
| Authorizers | IAM, Lambda, Cognito | IAM, Lambda, **JWT (OIDC/OAuth)** | IAM, Lambda |
| Canary deployments | ✅ | ❌ | ❌ |
| Best for | Full-featured public APIs | Simple, high-volume Lambda/HTTP proxies | Chat, live dashboards, notifications |

**REST API is the exam default** — whenever a question mentions caching, API keys, usage plans, request validation, mapping templates, or canary releases, it is a REST API, because HTTP APIs do not have those features.

Choose HTTP API when the question emphasises **lowest cost and latency with no advanced features**.

**WebSocket APIs** use **routes** rather than resources, driven by a route selection expression on the message body, plus three special routes: `$connect`, `$disconnect`, and `$default`. The server pushes messages back with the `@connections` API using the stored connection ID.

### API Gateway: Method Request vs Integration Request

**1. Method Request**

You tell API Gateway:

> "`id` must be provided."

```plain text
Method Request
→ Query parameter: id
→ Required: Yes
```

So this is about **declaring/validating what the client sends**.

**2. Integration Request**

You tell API Gateway:

> "Take the `id` query parameter and pass it to my backend."

```plain text
Client
  ↓
GET /users?id=123
  ↓
API Gateway
  ↓
Integration Request
  ↓
Lambda receives id = 123
```

**Stages and deployments — the mental model that unlocks several questions**

API Gateway does not serve your API configuration directly. Configuration changes are inert until you create a **deployment**, and a deployment is only reachable through a **stage**.

```plain text
API Gateway API
      ↓
  Deployment      (an immutable snapshot of the API configuration)
      ↓
    Stage         (a named environment pointing at a deployment)
      ↓
  Public URL
```

Two direct consequences:

**Changes are invisible until deployed.** If you edit a method, resource, or authorizer and the change doesn’t appear, the answer is **`CreateDeployment`**. This is the most common “my change didn’t take effect” question in the whole exam.

**Stages are how you run multiple versions side by side.**

```plain text
Customers:   https://api.company.com/prod/orders
Developers:  https://api.company.com/dev/orders
```

So when a question asks *“developers need to test a new API version without affecting existing customers”*, the answer is **create a development stage and point developers at it** — not a second API, not a canary. Same API, no duplication, production untouched.

**Stage variables**

Configuration values attached to a stage, referenced in the integration configuration. They let one API definition point at different backends per environment **without redeploying different code**.

```plain text
dev stage  → backend = Lambda-dev
prod stage → backend = Lambda-prod
```

Also usable to select a Lambda **alias**, which is how you wire `dev` stage → `dev` alias → version 3.

**Mapping templates**

Written in **VTL (Velocity Template Language)**, mapping templates transform the request before it reaches the backend, or the response before it reaches the client.

Why they exist: the shape your backend produces is not always the shape your client wants, and changing either can be expensive or impossible (a legacy backend, a published client contract).

Uses: change JSON structure, filter fields out of a response, rename fields, convert query parameters into a body, convert XML to JSON.

**The canonical question:** *“Lambda returns the full dataset, but the API should return only some fields / a different format.”*<br>✅ **API Gateway mapping template.** Not a Lambda code change (the requirement is presentation, not logic), and not stage variables (those select backends, not shapes).

Mapping templates require a **non-proxy (custom) integration** — proxy integration passes everything through untouched by definition.

#### **Integration types**

| Your table | API Gateway integration type | Meaning |
| --- | --- | --- |
| **Lambda Proxy** | `AWS_PROXY` | Pass request to Lambda; Lambda controls response |
| **Lambda Custom (non-proxy)** | `AWS` | Mapping templates transform request/response |
| **HTTP Proxy** | `HTTP_PROXY` | Pass request/response through to HTTP backend |
| **HTTP Custom** | `HTTP` | Custom request/response mapping |
| **AWS Service Integration** | `AWS` | API Gateway calls an AWS service directly |
| **Mock** | `MOCK` | No backend; API Gateway returns a configured response |

The **AWS Service Integration** is worth remembering: “send the request straight to SQS without a Lambda” is a real answer, and it’s the “least code” option when no logic is needed.

**Authorization — four mechanisms for four situations**

Authorization runs **before the backend is invoked**, so a rejected request never costs you a Lambda invocation.

**IAM authorization** — the caller signs the request with **AWS Signature v4**. For callers that already have AWS identities: IAM users, roles, other AWS services, internal applications.

With the caller's **AWS credentials**:

- **Access Key ID** → identifies the caller
- **Secret Access Key** → used to create the SigV4 signature

**Where are they stored?**

Usually in:

```plain text
~/.aws/credentials
```

**Lambda authorizer** — API Gateway invokes *your* function, which inspects the request (a token, a header, anything) and returns an **IAM policy** allowing or denying. For custom logic: a third-party identity provider, a token format you validate yourself, a database lookup, OAuth. Results are cacheable.

**Cognito User Pool authorizer** — API Gateway validates a Cognito-issued **JWT** automatically: signature, expiry, claims. **No code required.** For web and mobile application users.

**API keys + usage plans** — see below. **This is identification, not authentication.**

| Situation | Mechanism |
| --- | --- |
| AWS users, roles, or services calling the API | **IAM** |
| Custom auth logic, third-party IdP, token you validate yourself | **Lambda authorizer** |
| App users signing in, managed by Cognito, JWT | **Cognito User Pool authorizer** |
| Identify and meter clients, tiered plans | **API keys + usage plans** |

**API keys and usage plans — and why a new key returns 403**

This is a mechanism worth understanding because the question is very specific.

**An API key identifies a client. It does not authorize anything by itself.** All it does is give API Gateway a handle to count and throttle against.

**A usage plan** defines what a client may do: **throttle limits** (rate and burst) and **quota limits** (requests per day/week/month), applied to specific API stages.

The key becomes useful only when **linked** to a usage plan:

```plain text
CreateApiKey          → the key now exists, but grants nothing
CreateUsagePlanKey    → links key → usage plan → the client now has access
```

So: **a newly created API key returning `403 Forbidden` means it was never associated with a usage plan.** The fix is **`CreateUsagePlanKey`**.

Related API calls and their distinct jobs:

| API | Does | Does NOT |
| --- | --- | --- |
| **`CreateApiKey`** | Creates a key | Grant access |
| **`CreateUsagePlanKey`** | Links a key to a usage plan | — |
| **`ImportApiKeys`** | Bulk-imports keys from a file | Associate them with usage plans |
| **`UpdateUsagePlan`** | Changes quota/throttle settings | Associate new keys |
| **`CreateDeployment`** | Publishes configuration changes to a stage | — |

**Caching**

API Gateway can cache responses at the stage level, keyed on the request. Repeated identical requests are served from the cache and never reach the backend.

- Reduces backend load, Lambda invocations, and latency simultaneously.
- Best for **read-heavy APIs**, data that changes periodically, and public unauthenticated endpoints.
- Configured per stage; TTL is tunable; specific methods can be excluded.
- **Cache invalidation:** a client can force a cache refresh by sending the header **`Cache-Control: max-age=0`**. That client needs `execute-api:InvalidateCache` permission; otherwise you can choose to return a **403** or to ignore the header. Leaving it unauthenticated lets any caller flush your cache at will — which is exactly the exam twist.
- **Cache key parameters:** by default the cache key is the full request path. You can explicitly designate query strings, headers, or path parameters as **cache key parameters**, so `?userId=1` and `?userId=2` cache separately. Forgetting this is why “different users see each other’s data” happens with caching enabled.
- Cache size ranges 0.5 GB to 237 GB, configured **per stage**. Enabling caching is a stage-level setting; individual methods can override or opt out.

**Require Authorization** controls who can invalidate an API Gateway cache entry.

- Client sends `Cache-Control: max-age=0`.
- API Gateway requires the client to be **authorized**.
- The client needs an IAM policy allowing **`execute-api:InvalidateCache`** for the API resource.
- Without this authorization requirement, **any client** can invalidate the cache.

Watch out for the caching-versus-freshness trade: caching a personalised or fast-changing response serves the wrong data to the wrong user.

**Throttling and errors — the error codes are diagnostic**

Each status code maps to exactly one cause, and knowing them turns troubleshooting questions into lookups:

| Code | Meaning | Cause |
| --- | --- | --- |
| **429** | Too Many Requests | Throttling — account limit, stage limit, or usage plan quota |
| **502** | Bad Gateway | Lambda returned a **malformed proxy response** (wrong shape), or crashed |
| **504** | Gateway Timeout | Integration exceeded the timeout — **29 seconds** by default for REST APIs |
| **403** | Forbidden | Authorizer denied, WAF blocked, or **API key not linked to a usage plan** |
| **401** | Unauthorized | Missing or invalid authentication |
| **503** | Service unavailable | The server/service is currently **unable to handle the request**. No healthy targets behind an **ALB, Backend servers are overloaded** |
|    |    |    |

Default account-level throttle is 10,000 requests/second with a 5,000 burst; stage and method limits sit inside that.

The 502-vs-504 distinction is heavily tested. **502 = the response was wrong. 504 = the response was too slow.**

**Other features worth a line**

- **Request validation** — reject malformed requests at the gateway using a JSON Schema model, before paying for a Lambda invocation.
- **CORS** — configured on the API; requires an `OPTIONS` preflight method (a Mock integration is typical).
- **WAF integration** — attach a Web ACL to a stage for SQL-injection and XSS filtering.
- **Private APIs** — reachable only from inside a VPC via an interface endpoint.
- **SDK generation** — API Gateway can generate client SDKs from your API definition.
- **X-Ray** — enable tracing per stage.
- **Custom domain names** — require an **ACM certificate**; for edge-optimised endpoints the certificate must be in **us-east-1**.

**Recall check — API Gateway**

- **Build a secure, scalable API / serverless API** → API Gateway
- **Change the format of a Lambda response, or filter fields** → **Mapping template (VTL)**
- **Transform JSON / convert request format** → mapping template
- **Different backend per environment** → **stage variables**
- **Developers need a new version without affecting customers** → **create a dev stage**
- **Changes aren’t visible in production** → **`CreateDeployment`**
- **Percentage of production traffic to a new deployment** → canary release on the stage
- **Authenticate API users** → authorizer
- **JWT / Cognito-managed users** → **Cognito User Pool authorizer**
- **Custom authentication logic / third-party IdP / validate a token yourself** → **Lambda authorizer**
- **AWS users, roles, or services calling the API** → **IAM authorizer**
- **Authentication happens before the backend is invoked**
- **Improve API speed / reduce Lambda invocations / read-heavy API** → **API Gateway caching**
- **Limit or meter API usage, subscription tiers, premium users** → **API keys + usage plans**
- **API key exists but new users get 403** → **`CreateUsagePlanKey`**
- **Bulk import API keys** → `ImportApiKeys` (still needs usage plan association)
- **Change quota or throttle limits** → `UpdateUsagePlan`
- **API key is identification, not authorization**
- **429** → throttling
- **502** → malformed Lambda proxy response
- **504** → integration timeout (29 s default)
- **403** → authorizer denied, WAF, or unlinked API key
- **503** → the service/backend is unavailable or has no healthy capacity.
- **Reject bad requests before Lambda runs** → request validation
- **Call SQS or DynamoDB with no Lambda** → AWS service integration
- **Custom domain HTTPS** → ACM certificate (us-east-1 for edge-optimised)
- Real-time + server pushes data to client + no polling → **WebSocket API**
- ELB health check timeout → check **Security Groups, NACLs, port, and health-check** path.
- EC2 SG should allow traffic from the **ALB SG on the health-check/listener port.**
- **Require Authorization** →only clients with `execute-api:InvalidateCache` permission can do it.

---

### API Gateway CORS

**Mental model**

CORS controls whether a **browser** is allowed to call your API from a **different origin**.

```plain text
Frontend
https://app.example.com
        ↓
        ↓ CORS check
        ↓
API Gateway
https://api.example.com
```

These are different **origins**, so the browser enforces CORS.

Can restrict requests coming from unauthorized domains

**What it is**

**CORS = Cross-Origin Resource Sharing**

It is a browser security mechanism that allows an API to explicitly say:

> “Requests from this origin are allowed.”

**Important points**

API Gateway can return headers such as:

```plain text
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET,POST,OPTIONS
Access-Control-Allow-Headers: Content-Type,Authorization
```

For many cross-origin requests, the browser first sends an **`OPTIONS` preflight request**:

```plain text
Browser
   ↓ OPTIONS
API Gateway
   ↓
CORS headers
   ↓
Browser
   ↓ actual GET/POST
API Gateway
```

**Exam memory**

> **CORS is mainly a browser issue.**
> **API Gateway must return the appropriate CORS headers.**
> **`OPTIONS` = preflight request.**
> **CORS does NOT provide authentication or authorization.**

Also remember: **Postman/curl generally won't enforce CORS like a browser does.**

# Question patterns

> A Lambda function needs a URL, authentication, throttling, caching and versioning without writing that logic. What provides it?
> → **API Gateway (the managed front door)**

> Which three API types does API Gateway offer, and what is each designed for?
> → **REST (full-featured), HTTP (cheap/low-latency proxies), WebSocket (bidirectional, persistent)**

> A requirement mentions caching, API keys, usage plans, request validation or canary releases. Which API type must it be?
> → **REST API — HTTP APIs don't have those features**

> The goal is lowest cost and latency with no advanced features for a simple Lambda/HTTP proxy. Which API type?
> → **HTTP API (about 70% cheaper, lower latency)**

> A chat app or live dashboard needs the server to push data to clients without polling. Which API type?
> → **WebSocket API**

> What are the three special routes of a WebSocket API?
> → **$connect, $disconnect, $default**

> How does a WebSocket API server push a message to a client?
> → **The @connections API using the stored connection ID**

> A WebSocket API dispatches by a route selection expression over what?
> → **The message body (routes, not resources)**

> Which authorizer does an HTTP API support for OIDC/OAuth token validation that REST supports differently?
> → **JWT authorizer (REST uses Cognito User Pool authorizer)**

> A method change was made but doesn't appear in production. What is the fix?
> → **CreateDeployment (publishes an immutable snapshot; changes are inert until deployed)**

> Developers must test a new API version without affecting existing customers. What is the answer?
> → **Create a development stage and point developers at it — not a second API or a canary**

> One API definition must point at different backends per environment without redeploying different code. What feature?
> → **Stage variables**

> A dev stage must invoke a specific Lambda version through an alias. How is this wired?
> → **Stage variables select the Lambda alias (dev stage → dev alias)**

> What is the relationship between an API, a deployment and a stage?
> → **Deployment = immutable config snapshot; stage = named environment pointing at a deployment with a public URL**

> Lambda returns the full dataset but the API must return only some fields or a different format. What is the answer?
> → **API Gateway mapping template (VTL) — a presentation problem, not a Lambda code change**

> Mapping templates are written in which language?
> → **VTL (Velocity Template Language)**

> Which integration type is required for mapping templates?
> → **A non-proxy (custom) integration — proxy passes everything through untouched**

> Which integration type corresponds to Lambda Proxy?
> → **AWS_PROXY**

> Which integration type corresponds to a Lambda Custom (non-proxy) integration?
> → **AWS**

> Which integration type passes request/response through to an HTTP backend untouched?
> → **HTTP_PROXY**

> Which integration type returns a configured response with no backend at all?
> → **MOCK**

> A request must go straight to SQS with no Lambda. Which integration type?
> → **AWS service integration (the "least code" option when no logic is needed)**

> Which authorization mechanism signs requests with AWS Signature v4 for IAM users, roles and services?
> → **IAM authorization**

> Custom authentication logic, a third-party identity provider, or a token you validate yourself. Which authorizer?
> → **Lambda authorizer (returns an IAM policy; results cacheable)**

> App users sign in and JWT validation is fully managed. Which authorizer?
> → **Cognito User Pool authorizer (validates signature, expiry, claims — no code)**

> Clients must be identified and metered with tiered plans. Which feature?
> → **API keys + usage plans**

> Is an API key authentication or authorization?
> → **Neither by itself — it is identification only (a handle to count and throttle against)**

> A newly created API key returns 403 Forbidden. What is wrong and what is the fix?
> → **The key was never linked to a usage plan; fix with CreateUsagePlanKey**

> Which API call bulk-imports API keys, and what does it not do?
> → **ImportApiKeys; it does not associate them with usage plans**

> Which API call changes quota/throttle settings rather than associating keys?
> → **UpdateUsagePlan**

> Which API call publishes configuration changes to a stage?
> → **CreateDeployment**

> A client must force an API Gateway cache refresh. Which header and permission are needed?
> → **Cache-Control: max-age=0 plus execute-api:InvalidateCache permission**

> What happens if cache invalidation is left unauthenticated?
> → **Any caller can flush your cache at will — the classic exam twist**

> With caching enabled, different users start seeing each other's data. Why?
> → **Cache key parameters weren't designated, so the shared cache key matched too broadly**

> What is the default API Gateway cache key, and what can you add?
> → **The full request path; you can explicitly designate query strings, headers or path parameters**

> What is the configurable API Gateway cache size range?
> → **0.5 GB to 237 GB, configured per stage**

> At what scope is API Gateway caching configured, and can methods opt out?
> → **At the stage level; individual methods can override or opt out**

> Which error code means throttling (account, stage, or usage plan quota)?
> → **429 Too Many Requests**

> Which error code means the Lambda returned a malformed proxy response (or crashed)?
> → **502 Bad Gateway**

> Which error code means the integration exceeded the timeout, and what is the default?
> → **504 Gateway Timeout; 29 seconds by default for REST APIs**

> Which error code means authorizer denied, WAF blocked, or an API key not linked to a usage plan?
> → **403 Forbidden**

> Which error code means missing or invalid authentication?
> → **401 Unauthorized**

> Which error code means the backend/service is unavailable or has no healthy capacity?
> → **503 Service Unavailable (e.g. no healthy targets behind an ALB)**

> What is the default account-level API Gateway throttle?
> → **10,000 requests/second with a 5,000 burst; stage and method limits sit inside that**

> Malformed requests must be rejected before paying for a Lambda invocation. Which feature?
> → **Request validation using a JSON Schema model**

> Which HTTP method handles the CORS preflight, and what integration is typical?
> → **OPTIONS, typically backed by a Mock integration**

> What does CORS actually protect against, and what does it not provide?
> → **Browser cross-origin enforcement; it provides no authentication or authorization**

> An API must be reachable only from inside a VPC. Which configuration?
> → **Private API + resource policy + interface VPC endpoint**

> A custom domain over HTTPS needs a certificate. Which service, and which Region for edge-optimised endpoints?
> → **ACM certificate; it must be in us-east-1 for edge-optimised endpoints**

> An ALB health check for an API-backed target times out. What should be checked?
> → **Security Groups, NACLs, port, and the health-check path; the EC2 SG must allow the ALB SG on the health-check/listener port**

> A synchronous request takes 40 seconds end to end and times out. Why, and what is the fix?
> → **API Gateway's 29-second hard timeout — move to asynchronous processing (SQS, Step Functions, async invocation)**

> A small percentage of production traffic must go to a new deployment. Which feature?
> → **Canary release on the stage (REST API)**

> Tracing must be enabled for an API. Where is it turned on?
> → **X-Ray, enabled per stage**

> API Gateway can generate client SDKs from what?
> → **Your API definition (SDK generation)**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Managed front door for Lambda | **API Gateway** |
| Caching / API keys / usage plans / validation / canary | **REST API** |
| Cheapest, lowest-latency simple proxy | **HTTP API** |
| Bidirectional, server push, no polling | **WebSocket API** |
| WebSocket special routes | **$connect / $disconnect / $default** |
| Server push over WebSocket | **@connections API + connection ID** |
| HTTP API JWT validation | **JWT authorizer** |
| Change not visible in production | **CreateDeployment** |
| Version side by side with prod untouched | **dev stage** |
| Different backend per environment | **stage variables** |
| Select a Lambda alias per stage | **stage variables** |
| Reshape/filter response fields | **mapping template (VTL)** |
| Mapping template requires | **non-proxy (custom) integration** |
| Mapping template language | **VTL (Velocity Template Language)** |
| Lambda Proxy integration | **AWS_PROXY** |
| Lambda Custom integration | **AWS** |
| HTTP Proxy integration | **HTTP_PROXY** |
| Mock (no backend) | **MOCK** |
| Call SQS/DynamoDB with no Lambda | **AWS service integration** |
| SigV4 signing for AWS identities | **IAM authorization** |
| Custom auth / third-party IdP | **Lambda authorizer** |
| Managed app-user JWT | **Cognito User Pool authorizer** |
| Identification + metering + tiers | **API keys + usage plans** |
| New API key returns 403 | **not linked to a usage plan → CreateUsagePlanKey** |
| Bulk import keys | **ImportApiKeys** |
| Change quota/throttle | **UpdateUsagePlan** |
| Force cache refresh | **Cache-Control: max-age=0 + execute-api:InvalidateCache** |
| Invalidation left open | **anyone can flush the cache** |
| Cache key default | **full request path** |
| Different users see each other's data | **cache key parameters not set** |
| Cache size | **0.5 GB–237 GB, per stage** |
| Caching scope | **stage level (methods can opt out)** |
| 429 | **throttling** |
| 502 | **malformed Lambda proxy response** |
| 504 | **integration timeout (29 s default)** |
| 403 | **authorizer/WAF/unlinked API key** |
| 401 | **missing/invalid authentication** |
| 503 | **service unavailable / no healthy backend** |
| Account-level throttle | **10,000 rps, 5,000 burst** |
| Reject bad requests pre-Lambda | **request validation (JSON Schema)** |
| CORS preflight | **OPTIONS (Mock integration)** |
| CORS provides | **browser policy only, not auth** |
| VPC-only API | **private API + resource policy + VPC endpoint** |
| Custom domain cert | **ACM, us-east-1 for edge-optimised** |
| Percentage traffic to new deployment | **canary release on stage** |
| Tracing | **X-Ray per stage** |
| Auth happens when | **before the backend is invoked** |
| API key is | **identification, not authorization** |
| ELB health-check timeout fix | **check SG, NACL, port, health-check path** |
