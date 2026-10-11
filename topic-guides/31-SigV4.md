## SigV4
**AWS Signature Version 4 (SigV4)** is a way to **sign an AWS API request using AWS credentials**.
Think of it as a **digital signature** that proves:
> "This request was created by someone who has these AWS credentials."
### API Gateway example
If API Gateway uses **IAM authorization**:
```plain text
Client
  ↓
AWS credentials
  ↓
Create SigV4 signature
  ↓
API Gateway
  ↓
Check IAM permissions
  ↓
Allow / Deny
```
The client doesn't simply send:
```plain text
Authorization: my-password
```
Instead, the AWS SDK/CLI signs the request automatically.
### What does SigV4 contain?
The signature is calculated from things such as:
- Request method
- URL
- Headers
- Request body
- AWS credentials
- Region
- Service
- Timestamp
This helps AWS verify that the request **hasn't been modified** and was signed by valid credentials.
### Exam memory
> **API Gateway + IAM authorization → SigV4-signed request**
> **SigV4 = AWS request signing mechanism**
You usually **don't manually calculate SigV4**. AWS SDKs and CLI handle it for you.
