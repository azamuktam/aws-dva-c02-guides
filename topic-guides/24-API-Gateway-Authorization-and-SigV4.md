## API Gateway Authorization Types

**`NONE`**

- No authentication or authorization.
- Anyone who can reach the API can invoke it.
- **Exam keyword:** Public API.

**`AWS_IAM`**

- API Gateway uses **IAM credentials** to authenticate and authorize the caller.
- Caller needs permission for `execute-api:Invoke`.
- Useful when **AWS users/roles** call the API.
- For **cross-account REST API access**, use `AWS_IAM` + an API Gateway **resource policy**.
- **Exam keyword:** IAM role/user, AWS credentials, `execute-api:Invoke`.

**Lambda Authorizer**

- API Gateway calls a Lambda function to decide whether the request is allowed.
- Used for **custom authorization logic**.
- **Exam keyword:** Custom authentication/authorization logic.

API Gateway sends request information to a Lambda function → Lambda decides **Allow / Deny**.

**Types**

| Type | Input to Lambda | Use |
| --- | --- | --- |
| **Token-based** | One token, usually from `Authorization` header | Simple token authorization |
| **Request parameter-based** | Headers, query strings, stage variables, etc. | Custom authorization using multiple request parameters |

**Token-based**

- API Gateway extracts a **single token** from the request.
- Common example: `Authorization: Bearer <token>`
- Lambda validates the token.
- Best when authorization depends mainly on **one token**.

**Request parameter-based**

- Lambda can receive **multiple request parameters**.
- Can use **headers + query strings** and other request information.
- Best for a **custom authorization scheme**.

## Exam memory

**Token → one token**

**Request-based → multiple request parameters**

> **Header + query string parameters → Request parameter-based Lambda Authorizer**

**Cognito User Pool**

- Users authenticate with **Amazon Cognito**.
- Cognito provides a token.
- The client sends the token with the API request.
- Useful for **application users/login systems**.
- **Exam keyword:** User login, Cognito, JWT/token.

**Important distinction**

- **`AWS_IAM`** → API Gateway checks IAM permissions.
- **STS `AssumeRole`** → gets temporary credentials for an IAM role.
- They are **not the same thing**.

**Exam memory**

`NONE` → Public

`AWS_IAM` → AWS IAM

`Lambda Authorizer` → Custom Lambda logic

`Cognito` → User login + token

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

# Question patterns

> An API must be publicly reachable with no authentication at all. Which authorization type?
> → **NONE**

> Which API Gateway authorization uses IAM credentials and execute-api:Invoke?
> → **AWS_IAM**

> A REST API must be callable cross-account using IAM. What else is required?
> → **AWS_IAM plus an API Gateway resource policy**

> Which authorization calls a Lambda function to return Allow/Deny?
> → **A Lambda authorizer**

> A Lambda authorizer receives a single bearer token from the Authorization header. Which type?
> → **Token-based**

> A Lambda authorizer needs headers, query strings, and stage variables. Which type?
> → **Request parameter-based**

> Authorization depends mainly on one token. Which Lambda authorizer type?
> → **Token-based**

> A custom scheme uses a header plus a query string. Which Lambda authorizer type?
> → **Request parameter-based**

> Users log in and the client sends the resulting JWT. Which authorizer?
> → **A Cognito user pool authorizer**

> Is AWS_IAM the same thing as STS AssumeRole?
> → **No — they are different mechanisms**

> What does the AWS_IAM authorization actually check?
> → **The caller's IAM permission for execute-api:Invoke**

> What does STS AssumeRole provide in this context?
> → **Temporary credentials for an IAM role**

> What is AWS Signature Version 4?
> → **A mechanism to sign an AWS API request using AWS credentials (a digital signature)**

> How does a client sign a request to an IAM-authorized API Gateway?
> → **It creates a SigV4 signature over the request**

> Do you usually calculate SigV4 by hand?
> → **No — the AWS SDK or CLI signs the request automatically**

> What does a SigV4 signature include so requests cannot be replayed?
> → **X-Amz-Date (timestamp), credential scope, signed headers, and the signature**

> What inputs are used to calculate a SigV4 signature?
> → **Method, URL, headers, body, credentials, region, service, and timestamp**

> What two things does SigV4 let AWS verify about a request?
> → **That it has not been modified and that it was signed by valid credentials**

> You need per-client rate limits and entitlements. Which API Gateway feature?
> → **Usage plans**

> A client tries to send "Authorization: my-password" to an IAM-authorized API. Correct?
> → **No — with IAM auth the SDK/CLI signs the request with SigV4**

# Pocket card

| Keyword | Answer |
| --- | --- |
| NONE | **Public API** |
| AWS_IAM | **IAM creds + execute-api:Invoke** |
| Cross-account REST API | **AWS_IAM + resource policy** |
| Custom auth logic | **Lambda authorizer** |
| Token-based | **One token (Authorization header)** |
| Request parameter-based | **Multiple request params (headers, query, stage vars)** |
| Cognito user pool authorizer | **Validates the user JWT** |
| AWS_IAM vs STS | **Different things** |
| SigV4 | **AWS request signing mechanism** |
| SigV4 signing | **Automatic via SDK/CLI** |
| Signature fields | **X-Amz-Date, credential scope, signed headers, signature** |
| Signature inputs | **Method, URL, headers, body, creds, region, service, timestamp** |
| Replay protection | **Timestamp (X-Amz-Date)** |
| API Gateway + IAM auth | **SigV4-signed request** |
| Rate/entitlements | **Usage plans** |
| Bearer token | **Token-based authorizer** |
| Header + query string | **Request parameter-based authorizer** |
| User login + token | **Cognito user pool authorizer** |
| execute-api:Invoke | **Permission needed for AWS_IAM** |
| Public API keyword | **NONE** |
| Custom Lambda logic keyword | **Lambda authorizer** |
| IAM role/user keyword | **AWS_IAM** |
| Lambda input (token type) | **Single token** |
| Lambda input (request type) | **Multiple request parameters** |
