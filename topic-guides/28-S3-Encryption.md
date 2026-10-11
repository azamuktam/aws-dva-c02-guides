## S3 Encryption

**Why there are four options**

They differ on exactly one axis: **who holds and controls the key.** Once you see that, choosing is mechanical.

**SSE-S3 — S3-managed keys**

AWS creates, stores, and rotates the key. You do nothing. This is the default for new buckets.

```plain text
x-amz-server-side-encryption: AES256
```

Use when the requirement is simply “data must be encrypted at rest” with no key control needed.

**SSE-KMS — AWS KMS keys**

The object is encrypted using **envelope encryption** with a KMS key (see the KMS section). What you gain over SSE-S3:

- **Key control** — your own key, your own rotation schedule, the ability to disable it
- **Audit** — every decrypt is logged in **CloudTrail**, so you can see who read what
- **Access control** — the KMS key policy is a second, independent gate on the data

```plain text
x-amz-server-side-encryption: aws:kms
```

Optionally specify which key:

```plain text
x-amz-server-side-encryption-aws-kms-key-id: <key ARN>
```

If no key ID is supplied, S3 uses the AWS managed key `aws/s3`.

Note the operational consequence: because every object read requires a KMS `Decrypt` call, a very high-throughput workload can hit KMS request limits. **S3 Bucket Keys** reduce those calls substantially.

**SSE-C — customer-provided keys**

You send the key with **every request**. S3 performs the encryption but **does not store your key** — so if you lose it, the data is unrecoverable.

Required headers:

```plain text
x-amz-server-side-encryption-customer-algorithm: AES256
x-amz-server-side-encryption-customer-key:        <base64 256-bit key>
x-amz-server-side-encryption-customer-key-MD5:    <base64 MD5 of the key>
```

Requires HTTPS, since the key travels in the request.

**Client-side encryption**

You encrypt before uploading. S3 never sees plaintext and never sees your key. Maximum control, maximum responsibility — you own key management, rotation, and the encryption process.

| Type | Key management | Header / marker |
| --- | --- | --- |
| **SSE-S3** | AWS manages everything | `AES256` |
| **SSE-KMS** | KMS key, you control policy and rotation | `aws:kms` |
| **SSE-C** | You supply the key on every request | Customer key headers |
| **Client-side** | You encrypt before upload | None — S3 sees ciphertext |

**Enforcing encryption**

Encryption being *available* doesn’t mean it’s *used*. To require it, use an **S3 bucket policy with a `Deny`** that rejects any `PutObject` lacking the correct header.

The canonical question: *“Every upload must use SSE-S3, and uploads without encryption must be rejected.”*

✅ **`PutObject` with `x-amz-server-side-encryption: AES256`, plus a bucket policy that denies requests without that header.**

The mechanism to understand: a bucket policy can inspect request headers via conditions, so “deny unless the header is present and correct” is expressible as policy rather than requiring application discipline.

**Recall check — S3 encryption**

- **Simple encryption at rest, no key control needed** → **SSE-S3** (`AES256`)
- **Need audit and control over encryption keys** → **SSE-KMS** (`aws:kms`)
- **Customer provides the encryption key** → **SSE-C**
- **Customer encrypts before upload** → **client-side encryption**
- **Require all uploads to be encrypted** → **bucket policy `Deny` on the missing header**
- **Who can see which user decrypted an object** → SSE-KMS + CloudTrail
- **High-throughput SSE-KMS hitting KMS limits** → S3 Bucket Keys
- **Lose an SSE-C key** → the data is unrecoverable

**CORS (Cross-Origin Resource Sharing):** Allows a web application from one domain to make requests to resources on a different domain, when the server explicitly allows it:

  You have a website that loads files from an S3 bucket. When you try the URL of the files directly in your Chrome browser it works, but when you try to visit a website with a different domain that tries to load these files it doesn't. What's the problem?

---

# Question patterns

> A new bucket must encrypt data at rest with no key management overhead. Which option and header?
> → **SSE-S3, `x-amz-server-side-encryption: AES256`**

> What is the default server-side encryption for newly created S3 buckets?
> → **SSE-S3 (AES-256), AWS-managed keys**

> Every upload must be encrypted and any unencrypted upload rejected. How is that enforced?
> → **An S3 bucket policy with a `Deny` that rejects `PutObject` lacking the encryption header**

> How can a bucket policy know whether a request carried the encryption header?
> → **It evaluates request headers using conditions, so "deny unless present" is expressible as policy**

> Which server-side encryption option requires the request header `aws:kms`?
> → **SSE-KMS**

> Key usage must be auditable so you can see who decrypted an object. Which option?
> → **SSE-KMS (every decrypt is logged in CloudTrail)**

> Beyond encryption, what two capabilities does SSE-KMS add over SSE-S3?
> → **Key control (your key, rotation schedule, disable) and audit via CloudTrail**

> The KMS key policy acts as what kind of control over the data?
> → **A second, independent access-control gate**

> No KMS key ID is specified for SSE-KMS. Which key does S3 use?
> → **The AWS-managed key `aws/s3`**

> You must choose a specific customer-managed KMS key for an upload. Which header?
> → **`x-amz-server-side-encryption-aws-kms-key-id: <key ARN>`**

> A high-throughput bucket using SSE-KMS is hitting KMS request limits. What reduces the calls?
> → **S3 Bucket Keys**

> Which encryption option keeps the customer's key out of AWS, so S3 never stores it?
> → **SSE-C (customer-provided keys)**

> A client must provide the encryption key on every request. Which option?
> → **SSE-C**

> Which header names the SSE-C algorithm?
> → **`x-amz-server-side-encryption-customer-algorithm: AES256`**

> What is the required format of an SSE-C key?
> → **A base64-encoded 256-bit key**

> Which SSE-C header carries the integrity check for the key?
> → **`x-amz-server-side-encryption-customer-key-MD5` (base64 MD5 of the key)**

> Why must SSE-C always use HTTPS?
> → **The key travels in the request**

> An SSE-C key is lost. What happens to the data?
> → **It is unrecoverable — S3 does not store the key**

> An application must encrypt data before it ever reaches S3, with no key exposure to AWS. Which option?
> → **Client-side encryption**

> Which encryption option means S3 only ever sees ciphertext and no key?
> → **Client-side encryption**

> For SSE-S3, who creates, stores, and rotates the key?
> → **AWS, entirely**

> A requirement says "data must be encrypted at rest" with no key control needed. Which minimal option?
> → **SSE-S3**

> A website on a different domain fails to load files from your S3 bucket, but opening the file URL directly works. What is the problem?
> → **The bucket's CORS configuration does not allow the other origin**

> What does CORS allow?
> → **A web application on one origin to request resources on a different origin when the server explicitly allows it**

> Which header marker identifies SSE-S3 in the request?
> → **`x-amz-server-side-encryption: AES256`**

> Which header marker identifies SSE-KMS in the request?
> → **`x-amz-server-side-encryption: aws:kms`**

> Traffic to a bucket must be restricted to encrypted connections. Which policy condition?
> → **`aws:SecureTransport` (deny when false, i.e. HTTP)**

> Which technique underlies SSE-KMS object encryption?
> → **Envelope encryption with a KMS key**

> You need a customer-managed key with a custom rotation schedule. Which option?
> → **SSE-KMS with a customer-managed key**

> Which option requires you to own key management, rotation, and the encryption process entirely?
> → **Client-side encryption**

> Which two encryption options require the customer to supply key material rather than AWS managing it?
> → **SSE-C and client-side encryption**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Default encryption | **SSE-S3** |
| SSE-S3 header | **`AES256`** |
| SSE-KMS header | **`aws:kms`** |
| Audit key usage | **SSE-KMS + CloudTrail** |
| Customer key per request | **SSE-C** |
| Encrypt before upload | **client-side encryption** |
| Reduce KMS cost | **S3 Bucket Key** |
| Enforce HTTPS | **`aws:SecureTransport`** |
| Default KMS key | **`aws/s3`** |
| Choose KMS key | **`...aws-kms-key-id` header** |
| SSE-C transport | **HTTPS required** |
| SSE-C algorithm | **AES256** |
| SSE-C key format | **base64 256-bit** |
| SSE-C integrity | **base64 MD5 of key** |
| Lose SSE-C key | **data unrecoverable** |
| Enforce encryption | **bucket policy `Deny` on missing header** |
| Envelope encryption | **SSE-KMS** |
| Key policy | **second independent gate** |
| S3 sees ciphertext only | **client-side encryption** |
| Direct URL works, site fails | **CORS misconfiguration** |
| CORS meaning | **Cross-Origin Resource Sharing** |
| High-throughput KMS limit | **SSE-KMS** |
| Bucket key benefit | **fewer KMS API calls** |
| Key control + rotation + disable | **SSE-KMS customer-managed key** |
| AWS manages everything | **SSE-S3** |
| Who read what | **SSE-KMS + CloudTrail** |
| Key not stored by S3 | **SSE-C** |
| Second gate on data | **KMS key policy** |
| Key rotates automatically | **SSE-S3 / KMS key rotation** |
| Header inspection in policy | **bucket policy conditions** |
| Four at-rest options | **SSE-S3, SSE-KMS, SSE-C, client-side** |
| Requirement with no key control | **SSE-S3** |
| Customer owns rotation | **client-side encryption** |
