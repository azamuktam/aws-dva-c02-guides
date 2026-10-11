## Amazon CloudFront
**Why it exists**
Distance costs latency. CloudFront caches content at edge locations near users, so a request from Tashkent doesn’t cross an ocean to reach an origin in Virginia. It also absorbs load from the origin, terminates TLS at the edge, and provides a place to run code and security filtering before requests reach you.
**Viewer vs Origin protocol policy**
Two separate hops, two separate settings. This is the whole distinction.
```plain text
Viewer  ──[Viewer Protocol Policy]──▶  CloudFront  ──[Origin Protocol Policy]──▶  Origin
```
**Viewer Protocol Policy** — how users connect to CloudFront. Set in the cache behaviour.
- **HTTP and HTTPS** — both allowed. Use when HTTPS isn’t required.
- **Redirect HTTP to HTTPS** — HTTP requests are redirected. Use to move users to secure connections without breaking old links.
- **HTTPS Only** — HTTP is rejected outright. Strict security requirement.
**Origin Protocol Policy** — how CloudFront connects to the origin (S3, ALB, EC2, API Gateway).
- **HTTP Only** — the origin doesn’t support HTTPS.
- **HTTPS Only** — require a secure connection to the origin.
- **Match Viewer** — use whatever protocol the viewer used.
```plain text
User (HTTP)
    ↓
Viewer Protocol Policy: Redirect HTTP to HTTPS
    ↓
CloudFront (HTTPS)
    ↓
Origin Protocol Policy
    ↓
ALB / S3
```
HTTPS on the default `*.cloudfront.net` domain is provided automatically. A **custom domain** needs an **ACM certificate in us-east-1**.
CloudFront Signed URLs/Cookies should use a** Trusted Key Group **signer instead of an AWS Account (CloudFront key pair) signer.
✅ When you use the root user to manage CloudFront key pairs, you can only have up to** two active **CloudFront key pairs per AWS account
✅ When you create** a signer, the public key **is with CloudFront** and private key is** used to sign a portion of URL
### Why
**CloudFront signed URLs** need a signer. CloudFront supports:
- **Trusted key groups** → recommended modern approach
- **CloudFront key pairs** → legacy approach
For signing:
```plain text
Your application
     ↓
Private key 🔑
     ↓
Creates signed URL
     ↓
CloudFront
     ↓
Uses public key 🔓
to verify signature
```
The **private key must stay with you**. CloudFront has the corresponding **public key** to verify the signature.
**Caching behaviour**
**`Cache-Control: max-age`** on the origin response controls how long CloudFront keeps an object. A longer `max-age` means a **higher cache hit ratio**, fewer origin requests, and lower latency. This is the standard answer to “the website is slow and the same objects are requested repeatedly”.
**Origin failover**
An **origin group** pairs a primary and secondary origin. When the primary returns a configured failure status (`500`, `502`, `503`, `504`, `403`, `404`), CloudFront transparently retries against the secondary.
Trigger signals: 504 errors, origin failure, high availability requirements.
[**Lambda@Edge**](mailto:Lambda@Edge)** and CloudFront Functions**
Running code at the edge means acting on a request **before** it reaches your origin — or even before the cache is consulted.
[**Lambda@Edge**](mailto:Lambda@Edge) runs at four points in the request lifecycle:
| Event | When | Typical use |
| --- | --- | --- |
| **Viewer Request** | Before CloudFront checks its cache | **Authentication/authorisation**, URL rewrite, A/B testing |
| **Origin Request** | Before CloudFront calls the origin | Modify the origin request, change origin |
| **Origin Response** | After the origin responds | Modify the response before caching |
| **Viewer Response** | Before the response reaches the user | Add security headers |
Common uses: auth at the edge, HTTP→HTTPS redirects, header manipulation, URL rewrites, A/B testing, geo-based personalisation.
**Important correction to a common assumption: **[**Lambda@Edge**](mailto:Lambda@Edge)** does not make a website faster by itself.** It is for *executing code closer to users*, not for improving caching. Latency improvements come from caching. If a question asks how to speed up static content, the answer is CloudFront caching, not [Lambda@Edge](mailto:Lambda@Edge).
|    | Runs where | For |
| --- | --- | --- |
| **CloudFront** | Edge | Caching and global delivery |
| **CloudFront Functions** | Edge, very lightweight, sub-millisecond | Simple header/URL manipulation at massive scale |
| [**Lambda@Edge**](mailto:Lambda@Edge) | Regional edge caches | Heavier logic, network access, longer execution |
| **Lambda** | A specific Region | Backend processing |
**Restricting access**
**Signed URLs and signed cookies** provide time-limited access to private content.
```plain text
1. Generate an RSA key pair yourself (openssl)
2. Upload the PUBLIC key to CloudFront
3. Add it to a KEY GROUP
4. Attach the key group as a trusted signer on the cache behaviour
5. Sign URLs or cookies with your PRIVATE key
```
- **Signed URL** = one file. **Signed cookie** = many files, without changing any URLs.
- Signed URLs can additionally restrict by **IP range**, which S3 presigned URLs cannot.
- Signed URLs take precedence over signed cookies. If you use both signed URLs and signed cookies to control access to the same files and a viewer uses a signed URL to request a file
|    | CloudFront Signed URL | S3 Presigned URL |
| --- | --- | --- |
| Entry point | CloudFront edge (cached, fast) | S3 directly |
| Restrict by IP range | ✅ | ❌ |
| Cover many files | ✅ signed cookies | ❌ one object per URL |
| Signed with | Your private key + key group | The signer’s IAM credentials |
**Multiple origins** means one **CloudFront distribution** can get content from **more than one backend/source**.
For your example:
```plain text
                    CloudFront
                   /          \
                  ↓            ↓
                S3             ALB
             static          dynamic
```
So CloudFront has **two origins**:
1. **S3 bucket** → static files like images, CSS, JS
2. **Application Load Balancer** → dynamic application/API requests

**Recall check — CloudFront**
- **Faster global website or API** → CloudFront
- **Force users onto HTTPS** → Viewer Protocol Policy
- **CloudFront must reach the origin securely** → Origin Protocol Policy
- **HTTPS on the default domain** → automatic
- **HTTPS on a custom domain** → ACM certificate in **us-east-1**
- **Slow site, same objects requested repeatedly** → increase `Cache-Control: max-age`
- **504 errors / origin failure / high availability** → origin failover with an origin group
- **Custom logic at edge locations** → [Lambda@Edge](mailto:Lambda@Edge)
- **Authenticate before the request reaches the origin** → [Lambda@Edge](mailto:Lambda@Edge), **Viewer Request**
- **Add security headers to responses** → [Lambda@Edge](mailto:Lambda@Edge), Viewer Response
- **Simple, ultra-high-scale header rewriting** → CloudFront Functions
- **Faster static content** → CloudFront caching, **not** [Lambda@Edge](mailto:Lambda@Edge)
- **Private content through CloudFront** → signed URL / signed cookie
- **Many private files at once** → signed cookie
- **Restrict by IP range** → CloudFront signed URL (S3 presigned cannot)
- **Lock the S3 bucket to CloudFront only** → OAC (formerly OAI)
- **Legacy: only root can create a CloudFront key pair** → superseded by key groups
---
# CloudFront Function — Country Redirect
Imagine the user types:
```plain text
https://tutorialsdojo.com
```
They are in the UK.
CloudFront receives the request **before sending it to your ALB/ECS**.
```plain text
UK user
   ↓
tutorialsdojo.com
   ↓
CloudFront
   ↓
CloudFront Function
   ↓
"User is from GB"
   ↓
302 Redirect
   ↓
tutorialsdojo.com/uk/
```
CloudFront can provide the user's country in the `CloudFront-Viewer-Country` header. AWS has an official example of using this header to perform country-based redirects.
**What does the function actually do?**
Very simple logic:
```plain text
IF country = GB
    redirect to /uk/

IF country = PH
    redirect to /ph/

OTHERWISE
    continue normally
```
So:
```plain text
UK:
tutorialsdojo.com
        ↓
tutorialsdojo.com/uk/

Philippines:
tutorialsdojo.com
        ↓
tutorialsdojo.com/ph/
```
**Why ****`Viewer Request`****?**
**Viewer Request = the moment CloudFront receives the request from the user.**
```plain text
User
 ↓
CloudFront
 ↓  ← Viewer Request
Function checks country
 ↓
ALB
 ↓
ECS
```
We check **before reaching ECS** because there's no reason to send the request all the way to your application just to discover the user's country and redirect them.
CloudFront Functions are designed for lightweight logic at this point in the request flow.
AWS's current documentation notes that CloudFront adds `CloudFront-Viewer-Country` **after the viewer-request event** in the Lambda@Edge example, so for that specific Lambda@Edge pattern AWS uses **origin request**.
However, AWS also provides a **CloudFront Functions viewer-request country redirect example**, with the header made available through the appropriate CloudFront configuration.
**For your exam**
The question's answer is simply:
> **CloudFront Function + ****`CloudFront-Viewer-Country`**** → country-based redirect.**
Don't overthink the header mechanics.
**Mental model:**
> **CloudFront sees user → Function checks country → Function redirects user → ECS never needs to handle that redirect.**
