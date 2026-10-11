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

You normally **do not edit the `CodeUri` yourself**. The generated packaged template contains the S3 location.

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

# Question patterns

> Serverless infrastructure as code with far less boilerplate than raw CloudFormation. Which service?
> → **AWS SAM (Serverless Application Model)**

> Which `Transform` line identifies a SAM template?
> → **`Transform: 'AWS::Serverless-2016-10-31'`**

> Is `Transform` a template section or a parameter?
> → **A template section (not a parameter)**

> What does a SAM template compile into before deployment?
> → **A standard CloudFormation template, which then creates the resources**

> Which SAM resource type expands into a Lambda function plus its role, permissions, and event sources?
> → **`AWS::Serverless::Function`**

> Which SAM resource type expands into API Gateway?
> → **`AWS::Serverless::Api`**

> Which SAM resource type expands into an API Gateway HTTP API?
> → **`AWS::Serverless::HttpApi`**

> Which SAM resource type expands into a DynamoDB table?
> → **`AWS::Serverless::SimpleTable`**

> Which SAM resource type expands into a Step Functions state machine?
> → **`AWS::Serverless::StateMachine`**

> Which SAM resource type expands into a Lambda layer?
> → **`AWS::Serverless::LayerVersion`**

> Which SAM resource type embeds a nested serverless application?
> → **`AWS::Serverless::Application`**

> You want to test a single Lambda function locally in Docker. Which command?
> → **`sam local invoke`**

> You want to run API Gateway plus Lambda locally end to end. Which command?
> → **`sam local start-api`**

> You want a local Lambda endpoint for SDK integration testing. Which command?
> → **`sam local start-lambda`**

> Which command builds the application and resolves dependencies before deploying?
> → **`sam build`**

> Which command checks that the SAM template is well-formed?
> → **`sam validate`**

> Which command uploads artifacts to S3 and produces a deployable template?
> → **`sam package`**

> Which command creates or updates the CloudFormation stack?
> → **`sam deploy`**

> Which command shares a reusable layer or application via the Serverless Application Repository?
> → **`sam publish`**

> Which command tails CloudWatch Logs for a function?
> → **`sam logs`**

> Which command performs rapid iterative deploys during development?
> → **`sam sync`**

> Which command creates a new application from a template?
> → **`sam init`**

> What does `CodeUri` tell SAM?
> → **Where the Lambda source code is (e.g. a local folder)**

> Local Lambda code must become deployable. What is the flow?
> → **Package the local code, upload it to S3, and generate a packaged template with the S3 location**

> After packaging, which service deploys the packaged template?
> → **`aws cloudformation deploy` (or `sam deploy`)**

> Canary or gradual Lambda deployments are configured declaratively with SAM using what?
> → **Built-in CodeDeploy integration**

> What are SAM globals used for?
> → **Sharing settings across multiple functions**

> A workload needs YAML/JSON templates for any AWS resource. Which service?
> → **CloudFormation**

> Infrastructure as code written in a programming language that synthesises to CloudFormation. Which tool?
> → **AWS CDK**

> You want to deploy an application where AWS assembles the infrastructure for you. Which service?
> → **Elastic Beanstalk**

> Which statement best describes SAM's relationship to CloudFormation?
> → **SAM is a serverless-optimised layer on top of CloudFormation — everything about stacks, parameters, outputs, and rollback still applies**

> You normally do not edit `CodeUri` after packaging. Why?
> → **The generated packaged template already contains the S3 location of the uploaded code**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Transform | **`AWS::Serverless-2016-10-31`** |
| Compiles to | **CloudFormation** |
| Local single Lambda | **`sam local invoke`** |
| Local REST API | **`sam local start-api`** |
| Local SDK endpoint | **`sam local start-lambda`** |
| Build | **`sam build`** |
| Validate | **`sam validate`** |
| Package | **`sam package`** |
| Deploy | **`sam deploy`** |
| Publish to SAR | **`sam publish`** |
| Tail logs | **`sam logs`** |
| Rapid deploy | **`sam sync`** |
| New project | **`sam init`** |
| Source location | **`CodeUri`** |
| Function type | **`AWS::Serverless::Function`** |
| API type | **`AWS::Serverless::Api`** |
| HTTP API type | **`AWS::Serverless::HttpApi`** |
| Table type | **`AWS::Serverless::SimpleTable`** |
| State machine type | **`AWS::Serverless::StateMachine`** |
| Layer type | **`AWS::Serverless::LayerVersion`** |
| Nested app type | **`AWS::Serverless::Application`** |
| Shared settings | **globals** |
| Canary deploy | **CodeDeploy integration** |
| Underneath | **CloudFormation** |
| Transform kind | **template section, not parameter** |
| Packaging | **local code → upload to S3 → packaged template** |
| Deploy step | **creates/updates the CloudFormation stack** |
| Tracing | **SAM CLI integrates X-Ray** |
| General IaC | **CloudFormation (YAML/JSON)** |
| Programming-language IaC | **CDK** |
| App deploy, AWS builds infra | **Elastic Beanstalk** |
| Edit CodeUri after package | **no — packaged template holds the S3 location** |
