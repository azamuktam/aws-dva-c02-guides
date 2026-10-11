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
- **`Encrypt` API fails on a large file** → use `GenerateDataKey`, **not** a quota increase
- **Audit trail of key usage** → SSE-KMS (CloudTrail)
- **Control and rotate the key yourself** → customer managed key
- **No key management at all** → SSE-S3 / AWS managed key
- **`AccessDenied` despite an IAM Allow** → **check the key policy**
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

# Question patterns

> Which service holds keys in HSMs, never releases key material, and logs use to CloudTrail?
> → **AWS KMS**

> What is the maximum data size the KMS Encrypt API accepts?
> → **4 KB**

> Can the 4 KB Encrypt limit be raised by a quota increase?
> → **No — it is a fixed maximum**

> How do you encrypt data larger than 4 KB with KMS?
> → **Envelope encryption using GenerateDataKey**

> What two things does GenerateDataKey return?
> → **The plaintext data key and the encrypted data key**

> For large data, do you send the data to KMS or encrypt it locally?
> → **Encrypt locally with the plaintext data key — the data never goes to KMS**

> What do you do with the plaintext data key after encrypting?
> → **Discard it from memory**

> What do you store alongside the encrypted data?
> → **The encrypted data key**

> How does the Decrypt API know which key to use?
> → **The ciphertext blob carries the key ID, so you don't name the key**

> Which API returns only the encrypted data key, for a service that will decrypt later?
> → **GenerateDataKeyWithoutPlaintext**

> What library implements the envelope-encryption pattern for you?
> → **The AWS Encryption SDK**

> What is the maximum Lambda environment variable size?
> → **4 KB**

> Large data in Lambda should use which approach?
> → **A file with envelope encryption**

> Which key type is free, shared across accounts, and not visible to you?
> → **AWS owned key**

> Which key type is named aws/s3 or aws/lambda, rotates yearly, and is not editable?
> → **AWS managed key**

> Which key type lets you edit the key policy and rotate on your own schedule?
> → **Customer managed key**

> Is a key policy required on every KMS key?
> → **Yes — every key has one**

> A user has kms:Decrypt in their IAM policy but still gets AccessDenied. What do you check?
> → **The key policy**

> What does the default KMS key policy grant?
> → **Full access to the account root, which lets IAM policies take effect**

> A large S3 upload with SSE-KMS returns AccessDenied. Which permission is likely missing?
> → **kms:Decrypt**

> Which two permissions may an SSE-KMS multipart upload require?
> → **kms:GenerateDataKey and kms:Decrypt**

> You need to switch which key is used without changing application code. Which feature?
> → **A KMS alias (alias/my-key)**

> You need temporary, programmatic permission delegation on a key (e.g. for an AWS service). Which mechanism?
> → **KMS grants**

> What is the default symmetric key algorithm, and does it leave KMS?
> → **AES-256; the key material never leaves KMS**

> Which key type exposes a public key for encryption or signature verification outside AWS?
> → **Asymmetric keys**

> Encrypted data must be usable in another Region. Which key type?
> → **A multi-Region key**

> Why choose SSE-KMS over SSE-S3?
> → **Every KMS API call is logged in CloudTrail (audit trail)**

> How do you encrypt a CloudWatch log group, and what does it cover?
> → **AssociateKmsKey with a symmetric key; it encrypts only newly ingested logs**

> Which service provides dedicated single-tenant HSM hardware that you control?
> → **AWS CloudHSM**

> Which FIPS validation level does CloudHSM meet?
> → **FIPS 140-2 Level 3**

> CloudHSM vs KMS — which is easier and AWS-managed?
> → **KMS**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Encryption keys | **KMS** |
| Encrypt API max | **4 KB (not raisable)** |
| Data > 4 KB | **Envelope encryption / GenerateDataKey** |
| GenerateDataKey returns | **Plaintext + encrypted data key** |
| Encrypt large data | **Locally with the data key** |
| Store with ciphertext | **The encrypted data key** |
| Decrypt finds key via | **Key ID in the ciphertext** |
| Encrypted key only | **GenerateDataKeyWithoutPlaintext** |
| Full pattern implemented | **AWS Encryption SDK** |
| Lambda large data | **File + envelope encryption** |
| Lambda env var max | **4 KB** |
| AWS owned key | **Free, not visible** |
| AWS managed key | **aws/s3, yearly rotation, not editable** |
| Customer managed key | **You edit policy/rotate; monthly + per-request cost** |
| Key policy required | **Yes** |
| AccessDenied despite IAM Allow | **Check the key policy** |
| Default key policy | **Account root full access** |
| SSE-KMS multipart | **kms:GenerateDataKey + kms:Decrypt** |
| Swap keys without code | **Alias** |
| Temporary permission delegation | **Grants** |
| Symmetric | **AES-256, never leaves KMS** |
| Asymmetric | **Public key exposed** |
| Cross-Region data | **Multi-Region key** |
| Audit | **CloudTrail (every KMS API call)** |
| SSE-KMS vs SSE-S3 | **Audit trail** |
| CloudWatch log group | **AssociateKmsKey (symmetric, new logs only)** |
| Dedicated HSM | **CloudHSM** |
| CloudHSM validation | **FIPS 140-2 Level 3** |
| CloudHSM control | **Customer controls HSM and keys** |
| Easy AWS-managed keys | **KMS** |
| Envelope encryption benefit | **Data never crosses network; key never leaves KMS** |
| No key management | **SSE-S3 / AWS managed key** |
| Terminology | **"KMS key", not "CMK"** |
