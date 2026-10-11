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

> Infrastructure defined in TypeScript/Python and deployed as CloudFormation. Which service?
> → **AWS CDK**

> Preview changes between the deployed stack and the code. Which command?
> → **cdk diff**

> Generate the CloudFormation template from CDK code. Which command?
> → **cdk synth**

# Pocket card

| Keyword | Answer |
| --- | --- |
| IaC in code | **CDK** |
| Preview | **cdk diff** |
| Output | **CloudFormation templates** |
| Test infrastructure | **CDK assertions** |
| Constructs | **L1 (CFN), L2 (curated), L3 (patterns)** |
