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
- **STS ****`AssumeRole`** → gets temporary credentials for an IAM role.
- They are **not the same thing**.
**Exam memory**
`NONE` → Public
`AWS_IAM` → AWS IAM
`Lambda Authorizer` → Custom Lambda logic
`Cognito` → User login + token
