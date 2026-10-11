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
<table header-row="true">
<tr>
<td>SAM type</td>
<td>Expands into</td>
</tr>
<tr>
<td>**`AWS::Serverless::Function`**</td>
<td>Lambda function (+ role, permissions, event sources)</td>
</tr>
<tr>
<td>**`AWS::Serverless::Api`**</td>
<td>API Gateway</td>
</tr>
<tr>
<td>**`AWS::Serverless::SimpleTable`**</td>
<td>DynamoDB table</td>
</tr>
<tr>
<td>**`AWS::Serverless::StateMachine`**</td>
<td>Step Functions state machine</td>
</tr>
<tr>
<td>**`AWS::Serverless::HttpApi`**</td>
<td>API Gateway HTTP API</td>
</tr>
<tr>
<td>**`AWS::Serverless::LayerVersion`**</td>
<td>Lambda layer</td>
</tr>
<tr>
<td>**`AWS::Serverless::Application`**</td>
<td>A nested serverless application</td>
</tr>
<tr>
<td></td>
<td></td>
</tr>
</table>
SAM also has **globals** (shared settings across functions) and built-in support for **gradual deployment** via CodeDeploy, which is how canary Lambda deployments are configured declaratively.
**The CLI commands**
<table header-row="true">
<tr>
<td>Command</td>
<td>Does</td>
<td>When you use it</td>
</tr>
<tr>
<td>**`sam init`**</td>
<td>Create a new application from a template</td>
<td>Starting a project</td>
</tr>
<tr>
<td>**`sam validate`**</td>
<td>Validate the SAM template</td>
<td>Check the template is well-formed</td>
</tr>
<tr>
<td>**`sam build`**</td>
<td>Build the application and resolve dependencies</td>
<td>Before deploying</td>
</tr>
<tr>
<td>**`sam local invoke`**</td>
<td>Run **one Lambda** locally in Docker</td>
<td>Test a single function</td>
</tr>
<tr>
<td>**`sam local start-api`**</td>
<td>Run **API Gateway + Lambda** locally</td>
<td>Test a REST API end to end</td>
</tr>
<tr>
<td>**`sam local start-lambda`**</td>
<td>Local Lambda endpoint for SDK calls</td>
<td>Integration testing</td>
</tr>
<tr>
<td>**`sam package`**</td>
<td>Upload artifacts to S3 and produce a deployable template</td>
<td>Packaging step</td>
</tr>
<tr>
<td>**`sam deploy`**</td>
<td>Create or update the CloudFormation stack</td>
<td>Deploy</td>
</tr>
<tr>
<td>**`sam publish`**</td>
<td>Publish a layer or an app to the Serverless Application Repository</td>
<td>Share something reusable</td>
</tr>
<tr>
<td>**`sam logs`**</td>
<td>Tail CloudWatch Logs for a function</td>
<td>Debugging</td>
</tr>
<tr>
<td>**`sam sync`**</td>
<td>Rapid iterative deploy during development</td>
<td>Inner development loop</td>
</tr>
</table>
**SAM vs CloudFormation vs CDK vs Beanstalk**
<table header-row="true">
<tr>
<td></td>
<td>Difference</td>
</tr>
<tr>
<td>**SAM**</td>
<td>IaC optimised for **serverless**; still CloudFormation underneath</td>
</tr>
<tr>
<td>**CloudFormation**</td>
<td>General-purpose IaC in **YAML/JSON**</td>
</tr>
<tr>
<td>**CDK**</td>
<td>IaC in a **programming language**, synthesises to CloudFormation</td>
</tr>
<tr>
<td>**Elastic Beanstalk**</td>
<td>Deploys an **application**; AWS assembles the infrastructure</td>
</tr>
</table>
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
