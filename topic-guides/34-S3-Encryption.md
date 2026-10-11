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
<table header-row="true">
<tr>
<td>Type</td>
<td>Key management</td>
<td>Header / marker</td>
</tr>
<tr>
<td>**SSE-S3**</td>
<td>AWS manages everything</td>
<td>`AES256`</td>
</tr>
<tr>
<td>**SSE-KMS**</td>
<td>KMS key, you control policy and rotation</td>
<td>`aws:kms`</td>
</tr>
<tr>
<td>**SSE-C**</td>
<td>You supply the key on every request</td>
<td>Customer key headers</td>
</tr>
<tr>
<td>**Client-side**</td>
<td>You encrypt before upload</td>
<td>None — S3 sees ciphertext</td>
</tr>
</table>
**Enforcing encryption**
Encryption being *available* doesn’t mean it’s *used*. To require it, use an **S3 bucket policy with a ****`Deny`** that rejects any `PutObject` lacking the correct header.
The canonical question: *“Every upload must use SSE-S3, and uploads without encryption must be rejected.”*
✅ **`PutObject`**** with ****`x-amz-server-side-encryption: AES256`****, plus a bucket policy that denies requests without that header.**
The mechanism to understand: a bucket policy can inspect request headers via conditions, so “deny unless the header is present and correct” is expressible as policy rather than requiring application discipline.
**Recall check — S3 encryption**
- **Simple encryption at rest, no key control needed** → **SSE-S3** (`AES256`)
- **Need audit and control over encryption keys** → **SSE-KMS** (`aws:kms`)
- **Customer provides the encryption key** → **SSE-C**
- **Customer encrypts before upload** → **client-side encryption**
- **Require all uploads to be encrypted** → **bucket policy ****`Deny`**** on the missing header**
- **Who can see which user decrypted an object** → SSE-KMS + CloudTrail
- **High-throughput SSE-KMS hitting KMS limits** → S3 Bucket Keys
- **Lose an SSE-C key** → the data is unrecoverable
**CORS (Cross-Origin Resource Sharing):** Allows a web application from one domain to make requests to resources on a different domain, when the server explicitly allows it:
	You have a website that loads files from an S3 bucket. When you try the URL of the files directly in your Chrome browser it works, but when you try to visit a website with a different domain that tries to load these files it doesn't. What's the problem?
<empty-block/>
---
