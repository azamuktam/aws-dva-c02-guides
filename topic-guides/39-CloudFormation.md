# PART 5 — DEPLOYMENT: GETTING CODE INTO PRODUCTION
**24% of the exam.** Two halves: **describing infrastructure as code** (CloudFormation, SAM, CDK) and **moving code through a pipeline** (CodePipeline, CodeBuild, CodeDeploy, CodeArtifact).
The unifying idea behind infrastructure as code: a template is a **desired state**, and the service’s job is to make reality match it. That explains why CloudFormation can roll back, why drift detection exists, and why SAM and CDK both compile down to CloudFormation.
---

## AWS CloudFormation
**Mental model:**
AWS CloudFormation = **Infrastructure as Code (IaC)**.
You write a template describing the AWS resources you want, and CloudFormation **creates and manages those resources as a stack**.
```plain text
CloudFormation Template
        ↓
     Stack
        ↓
EC2 + S3 + IAM + RDS + VPC ...
```
**Why it exists**
Clicking resources together in the console is unrepeatable and undocumented. A CloudFormation template is a declarative description of the resources you want. CloudFormation creates them, tracks them as a **stack**, updates them when the template changes, and — crucially — **deletes them all when the stack is deleted**. That lifecycle coupling is the source of several exam questions.
**Template sections**
<table header-row="true">
<tr>
<td>Section</td>
<td>What it is</td>
<td>Example</td>
</tr>
<tr>
<td>**`AWSTemplateFormatVersion`**</td>
<td>Template version. Almost always the same value</td>
<td>`"2010-09-09"`</td>
</tr>
<tr>
<td>**`Description`**</td>
<td>Human-readable notes</td>
<td>`"Creates a web server"`</td>
</tr>
<tr>
<td>**`Metadata`**</td>
<td>Extra information for humans or tools. **No resources come from it**</td>
<td>Author, console UI hints</td>
</tr>
<tr>
<td>**`Parameters`**</td>
<td>Inputs supplied at deploy time</td>
<td>`InstanceType = t3.micro`</td>
</tr>
<tr>
<td>**`Mappings`**</td>
<td>Static lookup table: “if X, use Y”</td>
<td>`us-east-1 → ami-12345`</td>
</tr>
<tr>
<td>**`Conditions`**</td>
<td>Decide *whether* to create something</td>
<td>Create RDS only in Production</td>
</tr>
<tr>
<td>**`Transform`**</td>
<td>Preprocess the template — mostly **AWS SAM**</td>
<td>Expand SAM resources into CloudFormation</td>
</tr>
<tr>
<td>**`Resources`** ⭐</td>
<td>**The only required section.** What to create</td>
<td>EC2, S3, Lambda, DynamoDB</td>
</tr>
<tr>
<td>**`Outputs`**</td>
<td>Values returned after deployment</td>
<td>EC2 public IP, bucket name</td>
</tr>
</table>
**Where conditions can be applied** — a detail that gets asked directly:
<table header-row="true">
<tr>
<td>Section</td>
<td>Purpose</td>
<td>`Condition` allowed?</td>
</tr>
<tr>
<td>Parameters</td>
<td>User inputs</td>
<td>❌ No</td>
</tr>
<tr>
<td>Mappings</td>
<td>Static lookup values</td>
<td>❌ No</td>
</tr>
<tr>
<td>Conditions</td>
<td>Define the logic itself</td>
<td>N/A</td>
</tr>
<tr>
<td>**Resources**</td>
<td>Create resources</td>
<td>✅ **Yes**</td>
</tr>
<tr>
<td>**Outputs**</td>
<td>Return values</td>
<td>✅ **Yes**</td>
</tr>
</table>
**Parameters vs Mappings vs Parameter Store**
All three supply values, and the difference is *who knows the value and when*.
- **Parameters** — the **deployer** supplies it at deploy time. “Which instance type do you want?”
- **Mappings** — the **template** already contains it, selected by a key. “The AMI ID for this Region.”
- **SSM Parameter Store** — the value lives **outside** the template and can change independently. Dynamic configuration.
**Mappings vs Conditions:** a mapping *returns a value*; a condition *decides whether something exists*.
**`Fn::FindInMap`**** — the three-argument lookup**
```yaml
Mappings:
RegionMap:            # MapName
us-east-1:          # TopLevelKey
AMI: ami-12345    # SecondLevelKey → Value
us-west-2:
AMI: ami-67890
```
Retrieve it:
```yaml
!FindInMap [RegionMap, us-east-1, AMI]
```
```plain text
MapName → TopLevelKey → SecondLevelKey → Value
```
**Always three arguments.** Mappings are static, defined in the template, and unsuitable for secrets or frequently changing values.
**Intrinsic functions**
<table header-row="true">
<tr>
<td>Function</td>
<td>Returns</td>
</tr>
<tr>
<td>**`!Ref`**</td>
<td>The value of a parameter, or the primary identifier of a resource</td>
</tr>
<tr>
<td>**`!GetAtt`**</td>
<td>A specific **attribute** of a resource (`!GetAtt LoadBalancer.DNSName`)</td>
</tr>
<tr>
<td>**`!Sub`**</td>
<td>A string with variables substituted (`!Sub "${AWS::StackName}-bucket"`)</td>
</tr>
<tr>
<td>**`!FindInMap`**</td>
<td>A value from a mapping</td>
</tr>
<tr>
<td>**`!ImportValue`**</td>
<td>An exported output from **another stack**</td>
</tr>
<tr>
<td>**`!GetAZs`**</td>
<td>The Availability Zones in a Region</td>
</tr>
<tr>
<td>**`!Join`** / **`!Select`** / **`!Split`**</td>
<td>String and list manipulation</td>
</tr>
<tr>
<td>**`!If`** / **`!Equals`** / **`!Not`**</td>
<td>Condition logic</td>
</tr>
</table>
**`!Ref`** returns the resource's default/reference value, while **`!GetAtt`** returns a specific attribute of the resource, such as its ARN or DNS name
**`!Sub`** → **substitutes variables into a string**.
```plain text
!Sub"arn:aws:s3:::${MyBucket}"
```
Here `${MyBucket}` is replaced with the value returned by `!Ref MyBucket`.
### CloudFormation Parameters
**Parameter:** A value provided when creating/updating a CloudFormation stack. It makes the template reusable.
**Common types:**
- **String** → text value
- **Number** → numeric value
- **CommaDelimitedList** → comma-separated values
- **AWS-specific types** → existing AWS resources, e.g. `AWS::EC2::KeyPair::KeyName`
#### Fixed list of allowed EC2vchoices → `AllowedValues`
```plain text
AllowedValues:
  - t3.micro
  - t3.small
  - t3.medium
```
#### CloudFormation Template Upload
When you upload a **local template file** to CloudFormation, CloudFormation **automatically uploads it to an S3 bucket in your AWS account**, then uses that S3 copy to create the stack.
**Exam memory:** `Local template → S3 → CloudFormation Stack`
**Cross-stack references — Outputs, Export, ImportValue**
Export the **Security Group ID** from Template A using `Outputs → Export`, then import it in Template B using `Fn::ImportValue` and attach it to the EC2 instance.
**Exam memory:** **Stack A → Export → Stack B → ImportValue**
Splitting infrastructure across stacks (network stack, application stack) requires passing values between them.
```yaml
# Stack A
Outputs:
ELBDNSName:
Value: !GetAtt LoadBalancer.DNSName
Export:
Name: ELBDNSName
```
```yaml
# Stack B
Fn::ImportValue: ELBDNSName
```
**The rule that gets tested: export names must be unique within a Region.**
```plain text
✅ Allowed — same name in different Regions:
   us-east-1: Export ELBDNSName
   us-east-2: Export ELBDNSName

❌ Not allowed — same name twice in one Region:
   us-east-2, Application1: Export ELBDNSName
   us-east-2, Application2: Export ELBDNSName
```
A consequence worth knowing: a stack whose output is imported by another stack **cannot delete that output or be deleted** while the import exists. That coupling is why some teams prefer Parameter Store over exports.
**CloudFormation StackSets**
A single stack lives in one account and one Region. A global company needs the same infrastructure in many.
```plain text
Without StackSets:
   Region A → create stack manually
   Region B → create stack manually
   Region C → create stack manually

With StackSets:
   Template
      ↓
   StackSet
   /    |    \
Stack  Stack  Stack
 US     EU    Asia
```
Create a StackSet from a template, choose target **accounts** and **Regions**, and CloudFormation creates, updates, or deletes the stacks everywhere.
```plain text
StackSet: LoadTesting
Regions: us-east-1, eu-west-1, ap-southeast-1
→ creates one stack in each
```
Supports self-managed permissions (explicit IAM roles) or service-managed via AWS Organizations, plus rollout controls for concurrency and failure tolerance.
The exam preference: **if StackSets can do it, do not write custom Lambda automation.**
**Drift detection**
Because a stack represents a desired state, someone changing a resource by hand in the console breaks that assumption silently. **Drift detection** compares **actual resource configuration** against the template.
- Run **Detect Stack Drift** on the stack or an individual resource.
- Reports `IN_SYNC`, `MODIFIED`, `DELETED`, or `NOT_CHECKED`.
- Triggered by: a resource changed outside CloudFormation.
Drift answers a narrow, specific question, and it is easy to confuse with two neighbours:
<table header-row="true">
<tr>
<td>Tool</td>
<td>The question it answers</td>
</tr>
<tr>
<td>**CloudFormation Drift Detection**</td>
<td>*“Does this resource still match my template?”*</td>
</tr>
<tr>
<td>**AWS Config**</td>
<td>*“Is this resource configured according to my organisation’s rules?”*</td>
</tr>
<tr>
<td>**CloudTrail**</td>
<td>*“Who changed it, when, and from where?”*</td>
</tr>
</table>
Worked example — someone manually edits a Lambda IAM role that CloudFormation created:
- Is it different from the template? → **Drift Detection**
- Does it violate a security rule? → **AWS Config**
- Who changed it and when? → **CloudTrail**
For the exam, remember these AWS services/tools that use **CloudFormation** to create resources:
- **AWS SAM** → uses CloudFormation
- **AWS CDK** → creates CloudFormation templates
- **AWS Service Catalog** → uses CloudFormation templates
- **Elastic Beanstalk** → uses CloudFormation to create resources
- **CloudFormation StackSets** → deploys CloudFormation stacks across accounts/Regions
### CloudFormation Pseudo Parameters
**What it is:** Built-in variables that provide AWS information such as the **Account ID, Region, and Stack Name** without hardcoding values. 
**Common ones:**
- `AWS::AccountId` → AWS account ID
- `AWS::Region` → current Region
- `AWS::StackName` → stack name
- `AWS::NoValue` → removes a property when used with `Fn::If`
- `AWS::Partition` → `aws`, `aws-cn`, etc.
**Exam memory:** **Pseudo parameters = AWS-provided values that make templates reusable across accounts and Regions.** 
**Other CloudFormation features**
- **Change sets** — preview exactly what an update will do before executing it. The safe way to update production.
- **Stack service role** — lets CloudFormation use a specific IAM role to create, update, or delete stack resources on your behalf, with only the permissions you give that role.
- **Stack policies** — protect specific resources from being updated.
- **`DeletionPolicy`** — `Retain`, `Snapshot`, or `Delete`. `Retain` on a database keeps the data when the stack is deleted.
- **Rollback** — a failed create or update rolls back automatically by default.
- **Nested stacks** — reuse a template as a resource inside another template.
- **Custom resources** — call a Lambda function to do something CloudFormation cannot express natively. (such as deleting not empty s3)
- **`CreationPolicy`**** and ****`cfn-signal`** — CloudFormation normally considers an EC2 instance “created” as soon as the API call succeeds, which is long before your software has finished installing. A `CreationPolicy` makes CloudFormation **wait for a success signal** instead. The instance’s user data runs the setup, then calls **`cfn-signal`** to report success, and only then does the stack proceed.
```yaml
WebServer:
Type: AWS::EC2::Instance
CreationPolicy:
ResourceSignal:
Count:1
Timeout: PT15M      # wait up to 15 minutes
```
No signal within the timeout → the resource fails and the stack rolls back. This is the answer to *“the stack reports success before the application is actually ready”*.
- **`WaitCondition`**** / ****`WaitConditionHandle`** — the older, more general mechanism for pausing a stack until an external signal arrives. `CreationPolicy` is preferred for EC2 and Auto Scaling; `WaitCondition` is for waiting on something CloudFormation isn’t creating.
- **`UpdatePolicy`** — controls how an Auto Scaling Group is updated (rolling update batch size, pause time, minimum instances in service).
**Helper scripts:**
- `cfn-init`: Use to retrieve and interpret resource metadata, install packages, create files, and start services.
- `cfn-signal`: **report success or failure**. Use to signal with a CreationPolicy or WaitCondition, so you can synchronize other resources in the stack when the prerequisite resource or application is ready.
- `cfn-get-metadata`: Use to retrieve metadata for a resource or path to a specific key.
- `cfn-hup` : detect and react to metadata changes on a running instance
**Recall check — CloudFormation**
- **Infrastructure as code with YAML/JSON templates** → CloudFormation
- **The only required template section** → `Resources`
- **Inputs at deploy time** → `Parameters`
- **Static lookup by key (e.g. AMI per Region)** → `Mappings` + `Fn::FindInMap`
- **`!FindInMap`**** takes three arguments** → MapName, TopLevelKey, SecondLevelKey
- **Decide whether to create something** → `Conditions` (usable in `Resources` and `Outputs` only)
- **`Transform: AWS::Serverless-2016-10-31`** → it’s a **SAM** template
- **Get a resource’s primary identifier** → `!Ref`
- **Get a specific attribute (DNS name, ARN)** → `!GetAtt`
- **Substitute variables into a string** → `!Sub`
- **Use a value from another stack** → `Outputs` + `Export` + `!ImportValue`
- **Export name conflict** → same name twice in the **same Region**
- **Can two Regions use the same export name?** → **Yes**
- **Deploy the same resources to many Regions or accounts** → **StackSets**
- **Preview an update before applying it** → change set
- **Keep the database when the stack is deleted** → `DeletionPolicy: Retain`
- **Resource changed outside CloudFormation** → **Detect Stack Drift**
- **Continuous compliance monitoring** → AWS Config
- **Who made the change** → CloudTrail
- **Do something CloudFormation can’t express** → custom resource (Lambda)
- **CloudFormation automatically uploads it to an S3 bucket in your account and then uses that S3 copy for the stack**.
- CloudFormation **automatically determines resource dependencies and creation order**; use `DependsOn` only when you need to explicitly control the order
- Fixed list of allowed EC2vchoices → `AllowedValues`
---
