## AWS CDK (Cloud Development Kit)

**Why it exists**

YAML has no loops, no functions, no types, no IDE autocomplete, and no unit tests. Infrastructure with real logic — “create one subnet per AZ”, “these twelve microservices are identical except for a name” — becomes verbose and error-prone. CDK lets you express infrastructure in a **real programming language** and generates the CloudFormation template for you.

```plain text
CDK code (TypeScript / Python / Java / C# / Go)
          ↓  cdk synth
CloudFormation template
          ↓  cdk deploy
AWS resources
```

**CDK uses CloudFormation internally** — so you keep stacks, rollback, drift detection, and change sets.

### AWS CDK Workflow

**1. Create the app from a template provided by AWS CDK**

Start a new CDK application using a CDK-provided template.

**2. Add code to the app to create resources within stacks**

Define your AWS infrastructure using CDK code.

**3. Build the app (optional)**

Compile/build the application if the chosen programming language requires it.

**4. Synthesize one or more stacks in the app**

Run `cdk synth` to convert the CDK code into a **CloudFormation template**.

**5. Deploy stack(s) to your AWS account**

Run `cdk deploy` to deploy the generated CloudFormation stacks and create/update the AWS resources.

**Exam memory**

> **CDK code → Build → Synth → CloudFormation → Deploy**

**Concepts**

**Construct** — a reusable cloud component. Three levels:

- **L1** — a direct one-to-one mapping to a CloudFormation resource (`CfnBucket`)
- **L2** — a curated construct with sensible defaults and helper methods (`Bucket`)
- **L3 / patterns** — a whole architecture in one object (`ApplicationLoadBalancedFargateService`)

**Stack** — maps to a CloudFormation stack.

**App** — the top-level container holding one or more stacks.

**Workflow and commands**

```plain text
1. cdk init      → create the application from a template
2. Write code    → add resources inside stacks
3. (Build)       → optional, language-dependent
4. cdk synth     → generate the CloudFormation template
5. cdk deploy    → create/update resources
```

| Command | Does |
| --- | --- |
| **`cdk init`** | Create a CDK application |
| **`cdk synth`** | **Generate the CloudFormation template** |
| **`cdk deploy`** | Create or update AWS resources |
| **`cdk diff`** | Compare the code against what’s deployed, before deploying |
| **`cdk bootstrap`** | Prepare the account/Region for CDK (creates the staging bucket and roles) |
| **`cdk destroy`** | Delete the stack |

**Supported languages:** TypeScript, Python, Java, C#, Go.

**Recall check — CDK**

- **IaC using Python / TypeScript / a real programming language** → CDK
- **CloudFormation generated from code** → CDK
- **`cdk synth`** → generate the CloudFormation template
- **`cdk deploy`** → create or update resources
- **`cdk diff`** → see what would change
- **Reusable cloud component** → construct
- **Maps to a CloudFormation stack** → stack
- **CDK workflow** → init → add resources → synth → deploy
- **CDK uses CloudFormation internally**

---

# Question patterns

> Infrastructure must be defined in TypeScript or Python and deployed as CloudFormation. Which service?
> → **AWS CDK**

> A team wants infrastructure-as-code with loops, functions, types, and IDE autocomplete instead of YAML. Which tool?
> → **AWS CDK**

> Which programming languages does AWS CDK support?
> → **TypeScript, Python, Java, C#, Go**

> Convert CDK code into a CloudFormation template. Which command?
> → **cdk synth**

> Generate the CloudFormation template from CDK code without deploying. Which command?
> → **cdk synth**

> Create or update the AWS resources defined in CDK code. Which command?
> → **cdk deploy**

> Preview the difference between the deployed stack and the current CDK code before deploying. Which command?
> → **cdk diff**

> Create a new CDK application from an AWS-provided template. Which command?
> → **cdk init**

> Prepare an AWS account and Region for CDK by creating the staging bucket and roles. Which command?
> → **cdk bootstrap**

> Delete a CDK stack. Which command?
> → **cdk destroy**

> What is the correct order of the CDK workflow?
> → **init → add resources → (build) → synth → deploy**

> CDK generates what artifact before resources are created?
> → **A CloudFormation template**

> What does CDK use internally to deploy and manage resources?
> → **CloudFormation**

> Which CloudFormation capabilities do you retain when using CDK?
> → **Stacks, rollback, drift detection, and change sets**

> Is the build step in the CDK workflow mandatory?
> → **No — it is optional and language-dependent**

> A reusable cloud component in CDK is called what?
> → **A construct**

> Which construct level maps one-to-one to a CloudFormation resource such as CfnBucket?
> → **L1**

> Which construct level provides curated defaults and helper methods, such as Bucket?
> → **L2**

> Which construct level represents an entire architecture in one object, such as ApplicationLoadBalancedFargateService?
> → **L3 / patterns**

> What is another name for the L3 construct level?
> → **Patterns**

> A CDK object that maps to a CloudFormation stack is called what?
> → **A stack**

> The top-level CDK container that holds one or more stacks is called what?
> → **An app**

> Why does CDK exist instead of writing raw CloudFormation YAML?
> → **YAML has no loops, functions, types, IDE autocomplete, or unit tests**

> Which lets you express logic like "one subnet per AZ" or "twelve identical microservices" cleanly?
> → **AWS CDK (a real programming language)**

> In the CDK pipeline, what is produced by cdk synth?
> → **A CloudFormation template**

> In the CDK pipeline, what step actually creates the AWS resources?
> → **cdk deploy (via the generated CloudFormation stack)**

> How would you unit-test CDK infrastructure code?
> → **CDK assertions**

> Which command shows code-versus-deployed differences before applying a change?
> → **cdk diff**

> Which command compares the code against what is already deployed?
> → **cdk diff**

> A user needs to deploy the same CDK app to a new account and gets a bootstrap-related error. What is the fix?
> → **Run cdk bootstrap to create the staging bucket and roles**

> Which CDK command creates the application skeleton from a template?
> → **cdk init**

> How many stacks can a CDK app contain?
> → **One or more**

> A validation error cites a CfnBucket resource. Which construct level is in use?
> → **L1**

# Pocket card

| Keyword | Answer |
| --- | --- |
| IaC in a real language (Python/TS) | **AWS CDK** |
| CloudFormation generated from code | **AWS CDK** |
| cdk synth | **Generate the CloudFormation template** |
| cdk deploy | **Create/update AWS resources** |
| cdk diff | **Preview changes vs deployed** |
| cdk init | **Create CDK app from template** |
| cdk bootstrap | **Prepare account/Region (staging bucket + roles)** |
| cdk destroy | **Delete the stack** |
| CDK workflow | **init → add resources → synth → deploy** |
| CDK runs on | **CloudFormation internally** |
| Reusable cloud component | **Construct** |
| L1 | **One-to-one CFN resource (CfnBucket)** |
| L2 | **Curated defaults + helpers (Bucket)** |
| L3 / patterns | **Whole architecture (ALB Fargate service)** |
| Maps to CFN stack | **Stack** |
| Top-level container | **App** |
| Supported languages | **TypeScript, Python, Java, C#, Go** |
| Build step | **Optional / language-dependent** |
| Test infrastructure | **CDK assertions** |
| Why not YAML | **No loops, functions, types, autocomplete, tests** |
| Kept from CloudFormation | **Stacks, rollback, drift detection, change sets** |
| cdk synth output | **CloudFormation template** |
| Staging bucket + roles | **Created by cdk bootstrap** |
| One subnet per AZ | **CDK loop (real logic)** |
| Twelve identical services | **CDK (parameterized constructs)** |
| Construct levels | **L1, L2, L3** |
| App contains | **One or more stacks** |
| Before deploying | **cdk diff** |
| Access multiple accounts | **Named profiles (aws configure)** |
| Generated artifact | **CloudFormation template** |
| Preview without applying | **cdk diff (no changes made)** |
| Delete resources | **cdk destroy** |
| Fresh account setup | **cdk bootstrap** |
| Real programming language IaC | **CDK** |
| CloudFormation from code | **CDK** |
