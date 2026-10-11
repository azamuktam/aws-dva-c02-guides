## AWS SAM (Serverless Application Model)
**Why it exists**
Defining a Lambda function in raw CloudFormation takes a function resource, a role, a log group, a permission, and — for an API — several API Gateway resources. SAM adds shorthand resource types that expand into all of that, plus a local testing toolchain.
**SAM is CloudFormation.** The `Transform` directive tells CloudFormation to expand SAM resources into standard ones before deploying. So everything you know about CloudFormation still applies: stacks, parameters, outputs, rollback.
```plain text
SAM template (YAML)
      ↓  Transform
CloudFormation template
      ↓
AWS resources
```
**The identifying marker:**
```yaml
Transform:'AWS::Serverless-2016-10-31'
```
Seeing that line means the template is a SAM template. Note that `Transform` is a template section, not a parameter.
**SAM resource types**
| SAM type | Expands into |
| --- | --- |
| **`AWS::Serverless::Function`** | Lambda function (+ role, permissions, event sources) |
| **`AWS::Serverless::Api`** | API Gateway |
| **`AWS::Serverless::SimpleTable`** | DynamoDB table |
| **`AWS::Serverless::StateMachine`** | Step Functions state machine |
| **`AWS::Serverless::HttpApi`** | API Gateway HTTP API |
| **`AWS::Serverless::LayerVersion`** | Lambda layer |
| **`AWS::Serverless::Application`** | A nested serverless application |
|    |    |
SAM also has **globals** (shared settings across functions) and built-in support for **gradual deployment** via CodeDeploy, which is how canary Lambda deployments are configured declaratively.
**The CLI commands**
| Command | Does | When you use it |
| --- | --- | --- |
| **`sam init`** | Create a new application from a template | Starting a project |
| **`sam validate`** | Validate the SAM template | Check the template is well-formed |
| **`sam build`** | Build the application and resolve dependencies | Before deploying |
| **`sam local invoke`** | Run **one Lambda** locally in Docker | Test a single function |
| **`sam local start-api`** | Run **API Gateway + Lambda** locally | Test a REST API end to end |
| **`sam local start-lambda`** | Local Lambda endpoint for SDK calls | Integration testing |
| **`sam package`** | Upload artifacts to S3 and produce a deployable template | Packaging step |
| **`sam deploy`** | Create or update the CloudFormation stack | Deploy |
| **`sam publish`** | Publish a layer or an app to the Serverless Application Repository | Share something reusable |
| **`sam logs`** | Tail CloudWatch Logs for a function | Debugging |
| **`sam sync`** | Rapid iterative deploy during development | Inner development loop |
**SAM vs CloudFormation vs CDK vs Beanstalk**
|    | Difference |
| --- | --- |
| **SAM** | IaC optimised for **serverless**; still CloudFormation underneath |
| **CloudFormation** | General-purpose IaC in **YAML/JSON** |
| **CDK** | IaC in a **programming language**, synthesises to CloudFormation |
| **Elastic Beanstalk** | Deploys an **application**; AWS assembles the infrastructure |
### AWS SAM — Lambda Packaging
**Mental model**
SAM lets you describe a Lambda function using local source code. Before CloudFormation can deploy it, the code is packaged and uploaded to **S3**.
**`CodeUri`**
`CodeUri` tells SAM **where the Lambda source code is**.
It can point to a local folder containing the application code.
**Packaging**
`aws cloudformation package`:
**local code → package → upload to S3 → create packaged template**
You normally **do not edit the ****`CodeUri`**** yourself**. The generated packaged template contains the S3 location.
**Deployment**
After packaging, use the **packaged template** with `aws cloudformation deploy`.
**Exam memory**
**Local Lambda code → Package to S3 → Deploy packaged template**
**Recall check — SAM**
- **Lambda + API Gateway + DynamoDB application** → SAM
- **Serverless IaC** → SAM
- **YAML templates for any AWS resource** → CloudFormation
- **`Transform: AWS::Serverless-2016-10-31`** → SAM template
- **SAM templates are CloudFormation templates** with serverless shorthand
- **Test one Lambda locally** → `sam local invoke`
- **Test a REST API locally** → `sam local start-api`
- **Build the application** → `sam build`
- **Validate the template** → `sam validate`
- **Upload artifacts to S3** → `sam package`
- **Deploy or update the stack** → `sam deploy`
- **Publish a reusable layer or app** → `sam publish`
- **SAM always uses CloudFormation underneath**
- **CodeUri** → location of Lambda source code
- **package** → uploads code to S3 and prepares the template for deplou
- **deploy** → creates/updates the CloudFormation stack
---
