## AWS Key Management Service (KMS)
**Why it exists**
Encryption is only as good as key management. KMS holds keys in hardware security modules, never releases the key material, enforces access with policies, and logs every use to CloudTrail. Your application asks KMS to perform cryptographic operations rather than holding keys itself.
**Envelope encryption — the mechanism to actually understand**
**A common misconception is that you send data to KMS and it sends the encrypted data back. That is not how it works for anything of size.** The `Encrypt` API accepts a maximum of **4 KB**, and it is not a quota you can raise.
Everything larger — S3 objects, EBS volumes, Lambda environment variables — uses **envelope encryption**:
```plain text
1. Call GenerateDataKey
        ↓
2. KMS returns TWO things:
     • the plaintext data key
     • the encrypted data key (a ciphertext blob)
        ↓
3. Encrypt your data LOCALLY with the plaintext data key
        ↓
4. Discard the plaintext data key from memory
        ↓
5. Store the encrypted data key alongside the encrypted data
```
Decryption reverses it:
```plain text
1. Read the encrypted data key
2. Call KMS Decrypt on it (small — well under 4 KB)
3. Receive the plaintext data key
4. Decrypt your data locally
```
Why this design: the large data never crosses the network to KMS, throughput is not limited by KMS, and the key material still never leaves KMS. It is also why `Decrypt` does not need you to name the key — the ciphertext blob carries the key ID.
### Lambda + Large Encrypted Data
- **Envelope + environment variable** → 4 KB Lambda limit
- **KMS direct + environment variable** → 4 KB Lambda limit + KMS direct encryption max 4 KB
- **Envelope + file** → suitable for large data
- **KMS direct + file** → direct KMS encryption is for small data
**Memory:** Large data → **file + envelope encryption**
**Related APIs:**
- `GenerateDataKey` → plaintext key + encrypted key for a service that will encrypt now
- `GenerateDataKeyWithoutPlaintext` → encrypted key only, for a service that will decrypt later
- The **AWS Encryption SDK** implements the whole pattern for you
**Key types**
| Type | Managed by | Rotation | Key policy | Cost |
| --- | --- | --- | --- | --- |
| **AWS owned** | AWS, shared across accounts | AWS | Not visible | Free |
| **AWS managed** (`aws/s3`, `aws/lambda`) | AWS, one per account per service | Automatic, yearly | Not editable | Free |
| **Customer managed** | **You** | Optional, yearly or on demand | **You edit it** | Monthly + per-request |
Choose a customer managed key when you need to control the policy, audit usage in detail, rotate on your own schedule, or disable/delete the key.

**Key policies — the part that surprises people**
**Every KMS key has a key policy, and it is required.** Unlike S3, where IAM permissions alone can grant access, a KMS key with a restrictive key policy denies everyone — **IAM permissions alone are not enough**. The default key policy grants the account root full access, which is what allows IAM policies to take effect at all.
So: *“the user has **`kms:Decrypt`** in their IAM policy and still gets AccessDenied”* → **check the key policy.**
🔑 **KMS key policy = **policy attached to the KMS key that controls who can access/use that key.
### S3 Multipart Upload + SSE-KMS
Mental model
**Large file → Multipart Upload → extra KMS permissions**
Why?
Multipart uploads with **SSE-KMS** may require:
- `kms:GenerateDataKey`
- `kms:Decrypt`
Example
Small file → works ✅
Large file → multipart → missing `kms:Decrypt` → **Access Denied** ❌

**Other features:**
- **Aliases** (`alias/my-key`) let you point at a different key without changing application code.
- **Grants** are temporary, programmatic permission delegations, used by AWS services on your behalf.
- AWS KMS supports both** symmetric **and** asymmetric **KMS keys**.**
- **Symmetric** keys (default, AES-256) never leave KMS. **Asymmetric** keys expose a public key for encrypting or verifying signatures outside AWS.
- **Multi-Region keys** replicate key material to other Regions — required if encrypted data must be usable in another Region.
- **Every KMS API call is logged in CloudTrail.** This is the main practical reason to choose SSE-KMS over SSE-S3.
- **Encrypting CloudWatch Log Group:** Use `AssociateKmsKey` to attach a **symmetric KMS key**; it encrypts only **newly ingested logs**
**Recall check — KMS**
- **Need encryption keys** → KMS
- **Data larger than 4 KB** → envelope encryption / `GenerateDataKey`
- **`Encrypt`**** API fails on a large file** → use `GenerateDataKey`, **not** a quota increase
- **Audit trail of key usage** → SSE-KMS (CloudTrail)
- **Control and rotate the key yourself** → customer managed key
- **No key management at all** → SSE-S3 / AWS managed key
- **`AccessDenied`**** despite an IAM Allow** → **check the key policy**
- **Change keys without changing code** → alias
- **Encrypted data must work in another Region** → multi-Region key
- **Say “KMS key”, not “CMK”**
- **Large S3 + SSE-KMS + Access Denied** → check kms:Decrypt.
---
### AWS CloudHSM
**Mental model**
**CloudHSM = AWS provides dedicated physical HSM hardware, but YOU control it.**
Used when you need **strong control over encryption keys and HSMs**, often for strict security/compliance requirements.
**What it is**
- Dedicated **Hardware Security Module (HSM)**
- Single-tenant hardware
- You control the HSM and keys
- Supports cryptographic operations such as **RSA**
- FIPS 140-2 Level 3 validated
**Use**
Use CloudHSM when you need:
- **Dedicated HSM hardware**
- **Exclusive control** over cryptographic keys
- Existing/on-premises HSM integration
- Compliance requirements requiring customer control
**CloudHSM vs KMS**
|    | CloudHSM | KMS |
| --- | --- | --- |
| Hardware | Dedicated HSM | AWS-managed HSM infrastructure |
| Control | **High customer control** | AWS manages infrastructure |
| Key management | Customer controlled | AWS managed |
| Ease of use | More complex | Easier |
| Typical use | Strict compliance / custom crypto | General AWS encryption |
**Exam memory**
**“Dedicated HSM” → CloudHSM**
**“Exclusive control of HSM/keys” → CloudHSM**
**“Easy AWS-managed encryption keys” → KMS**
