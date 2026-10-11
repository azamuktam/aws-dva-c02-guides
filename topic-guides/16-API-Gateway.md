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
<table header-row="true">
<tr>
<td></td>
<td>**REST API**</td>
<td>**HTTP API**</td>
<td>**WebSocket API**</td>
</tr>
<tr>
<td>Direction</td>
<td>Request/response</td>
<td>Request/response</td>
<td>**Bidirectional, persistent**</td>
</tr>
<tr>
<td>Cost</td>
<td>Highest</td>
<td>\~70% cheaper</td>
<td>Per message + connection minute</td>
</tr>
<tr>
<td>Latency</td>
<td>Higher</td>
<td>Lower</td>
<td>—</td>
</tr>
<tr>
<td>**Caching**</td>
<td>✅</td>
<td>❌</td>
<td>❌</td>
</tr>
<tr>
<td>**API keys + usage plans**</td>
<td>✅</td>
<td>❌</td>
<td>❌</td>
</tr>
<tr>
<td>**Request validation**</td>
<td>✅</td>
<td>❌</td>
<td>❌</td>
</tr>
<tr>
<td>**Mapping templates (VTL)**</td>
<td>✅</td>
<td>❌</td>
<td>✅</td>
</tr>
<tr>
<td>**WAF**</td>
<td>✅</td>
<td>❌</td>
<td>❌</td>
</tr>
<tr>
<td>Private APIs (VPC only)</td>
<td>✅</td>
<td>❌</td>
<td>❌</td>
</tr>
<tr>
<td>Authorizers</td>
<td>IAM, Lambda, Cognito</td>
<td>IAM, Lambda, **JWT (OIDC/OAuth)**</td>
<td>IAM, Lambda</td>
</tr>
<tr>
<td>Canary deployments</td>
<td>✅</td>
<td>❌</td>
<td>❌</td>
</tr>
<tr>
<td>Best for</td>
<td>Full-featured public APIs</td>
<td>Simple, high-volume Lambda/HTTP proxies</td>
<td>Chat, live dashboards, notifications</td>
</tr>
</table>
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
<table header-row="true">
<tr>
<td>Your table</td>
<td>API Gateway integration type</td>
<td>Meaning</td>
</tr>
<tr>
<td>**Lambda Proxy**</td>
<td>`AWS_PROXY`</td>
<td>Pass request to Lambda; Lambda controls response</td>
</tr>
<tr>
<td>**Lambda Custom (non-proxy)**</td>
<td>`AWS`</td>
<td>Mapping templates transform request/response</td>
</tr>
<tr>
<td>**HTTP Proxy**</td>
<td>`HTTP_PROXY`</td>
<td>Pass request/response through to HTTP backend</td>
</tr>
<tr>
<td>**HTTP Custom**</td>
<td>`HTTP`</td>
<td>Custom request/response mapping</td>
</tr>
<tr>
<td>**AWS Service Integration**</td>
<td>`AWS`</td>
<td>API Gateway calls an AWS service directly</td>
</tr>
<tr>
<td>**Mock**</td>
<td>`MOCK`</td>
<td>No backend; API Gateway returns a configured response</td>
</tr>
</table>
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
<table header-row="true">
<tr>
<td>Situation</td>
<td>Mechanism</td>
</tr>
<tr>
<td>AWS users, roles, or services calling the API</td>
<td>**IAM**</td>
</tr>
<tr>
<td>Custom auth logic, third-party IdP, token you validate yourself</td>
<td>**Lambda authorizer**</td>
</tr>
<tr>
<td>App users signing in, managed by Cognito, JWT</td>
<td>**Cognito User Pool authorizer**</td>
</tr>
<tr>
<td>Identify and meter clients, tiered plans</td>
<td>**API keys + usage plans**</td>
</tr>
</table>
**API keys and usage plans — and why a new key returns 403**
This is a mechanism worth understanding because the question is very specific.
**An API key identifies a client. It does not authorize anything by itself.** All it does is give API Gateway a handle to count and throttle against.
**A usage plan** defines what a client may do: **throttle limits** (rate and burst) and **quota limits** (requests per day/week/month), applied to specific API stages.
The key becomes useful only when **linked** to a usage plan:
```plain text
CreateApiKey          → the key now exists, but grants nothing
CreateUsagePlanKey    → links key → usage plan → the client now has access
```
So: **a newly created API key returning ****`403 Forbidden`**** means it was never associated with a usage plan.** The fix is **`CreateUsagePlanKey`**.
Related API calls and their distinct jobs:
<table header-row="true">
<tr>
<td>API</td>
<td>Does</td>
<td>Does NOT</td>
</tr>
<tr>
<td>**`CreateApiKey`**</td>
<td>Creates a key</td>
<td>Grant access</td>
</tr>
<tr>
<td>**`CreateUsagePlanKey`**</td>
<td>Links a key to a usage plan</td>
<td>—</td>
</tr>
<tr>
<td>**`ImportApiKeys`**</td>
<td>Bulk-imports keys from a file</td>
<td>Associate them with usage plans</td>
</tr>
<tr>
<td>**`UpdateUsagePlan`**</td>
<td>Changes quota/throttle settings</td>
<td>Associate new keys</td>
</tr>
<tr>
<td>**`CreateDeployment`**</td>
<td>Publishes configuration changes to a stage</td>
<td>—</td>
</tr>
</table>
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
<table header-row="true">
<tr>
<td>Code</td>
<td>Meaning</td>
<td>Cause</td>
</tr>
<tr>
<td>**429**</td>
<td>Too Many Requests</td>
<td>Throttling — account limit, stage limit, or usage plan quota</td>
</tr>
<tr>
<td>**502**</td>
<td>Bad Gateway</td>
<td>Lambda returned a **malformed proxy response** (wrong shape), or crashed</td>
</tr>
<tr>
<td>**504**</td>
<td>Gateway Timeout</td>
<td>Integration exceeded the timeout — **29 seconds** by default for REST APIs</td>
</tr>
<tr>
<td>**403**</td>
<td>Forbidden</td>
<td>Authorizer denied, WAF blocked, or **API key not linked to a usage plan**</td>
</tr>
<tr>
<td>**401**</td>
<td>Unauthorized</td>
<td>Missing or invalid authentication</td>
</tr>
<tr>
<td>**503**</td>
<td>Service unavailable</td>
<td>The server/service is currently **unable to handle the request**. No healthy targets behind an **ALB, Backend servers are overloaded**</td>
</tr>
<tr>
<td></td>
<td></td>
<td></td>
</tr>
</table>
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
<empty-block/>
---
<empty-block/>
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
For many cross-origin requests, the browser first sends an **`OPTIONS`**** preflight request**:
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
> **`OPTIONS`**** = preflight request.**
> **CORS does NOT provide authentication or authorization.**
Also remember: **Postman/curl generally won't enforce CORS like a browser does.**
