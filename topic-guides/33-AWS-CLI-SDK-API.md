## AWS CLI vs AWS API

**AWS API**

The underlying AWS service interface used to perform operations programmatically.

**AWS CLI**

A command-line tool that calls AWS APIs for you.

**AWS SDK**

Programming libraries that call AWS APIs from your application code.

**Relationship**

```plain text
AWS Console
     ↓
AWS API
     ↑
AWS CLI / AWS SDK
```

**Example — EC2**

To change the `DeleteOnTermination` attribute:

- **CLI:** `aws ec2 modify-instance-attribute`
- **API:** `ModifyInstanceAttribute`

Both perform the same underlying EC2 operation.

**DeleteOnTermination**

- `true` → EBS volume is deleted when EC2 is terminated
- `false` → EBS volume is kept

**Exam memory**

> **CLI = command-line interface to AWS APIs**

  **SDK = programming interface to AWS APIs**

  **API = underlying service interface**

### **AWS Glue**

AWS Glue is a fully managed extract, transform, and load (ETL) service that makes it easy for customers to prepare and load their data for analytics. AWS Glue cannot orchestrate a workflow.

### ** AWS Batch**

  AWS Batch runs batch computing jobs on the AWS Cloud. AWS Batch dynamically provisions the optimal quantity and type of compute resources (e.g., CPU or memory optimized instances) based on the volume and specific resource requirements of the batch jobs submitted. AWS Batch cannot orchestrate a workflow.

## AWS CLI and local configuration

**Credentials and profiles**

**`aws configure --profile <name>`** creates a **named credentials profile**, so one machine can work with several AWS accounts or IAM identities.

```plain text
aws configure --profile production
aws s3 ls --profile production
```

Use when: multiple AWS accounts, multiple IAM users, or switching between dev and prod from the same terminal.

- **`-region`** overrides the default Region for a single command, without changing your saved configuration. Use when running one command in another Region.

**`sam local invoke GetUsersFunction --profile <name>`** runs a Lambda locally using a named profile, so the locally running function can reach real AWS resources with the right credentials.

**SAM configuration files**

**`samconfig.toml`** stores SAM CLI settings — build and deploy defaults, including default **`parameter_overrides`**. It exists so you don’t retype the same flags.

- **`-parameter-overrides`** overrides **CloudFormation template parameters** at deploy time, and takes precedence over values in `samconfig.toml`.

```plain text
Template default  <  samconfig.toml  <  --parameter-overrides
```

**Environment variables**

Use them for **application configuration**. **Never store AWS credentials in them** — that is what roles and profiles exist for.

**Credential resolution order** (roughly): explicit parameters → environment variables → the credentials file / profile → container credentials → **instance or task IAM role**. Understanding this order explains why a Lambda or EC2 instance “just works” with no configuration: the role is found at the end of the chain.

**Recall check — CLI**

- **Store credentials for multiple accounts** → `aws configure --profile`
- **Run one command in a different Region** → `-region`
- **Local Lambda needs AWS access** → `sam local invoke --profile`
- **Save SAM CLI settings** → `samconfig.toml`
- **Override CloudFormation template parameters** → `-parameter-overrides` (beats `samconfig.toml`)
- **Application configuration** → environment variables
- **Never put AWS credentials in environment variables** → use roles

---

## EC2 `-dry-run`

**Use:** Check if your IAM user has permission to perform an EC2 action **without actually doing it**.

```bash
aws ec2 terminate-instances \
  --instance-ids i-1234567890abcdef0 \
  --dry-run
```

**Result:**

- `DryRunOperation` → permission **allowed** ✅
- `UnauthorizedOperation` → permission **denied** ❌

**Exam memory:**

> **EC2 permission check → `--dry-run`**

# Question patterns

> An application must call AWS services from its own code. Which interface?
> → **AWS SDK**

> A developer wants to run AWS operations from the terminal. Which tool?
> → **AWS CLI**

> What is the underlying AWS service interface that everything else calls?
> → **The AWS API**

> Which layer do the Console, CLI, and SDK all sit on top of?
> → **The AWS API**

> Change the DeleteOnTermination attribute on an instance from the CLI. Which command?
> → **aws ec2 modify-instance-attribute**

> Which API call corresponds to the CLI command aws ec2 modify-instance-attribute?
> → **ModifyInstanceAttribute**

> Keep the EBS volume when the EC2 instance terminates. Which setting?
> → **DeleteOnTermination = false**

> Delete the EBS volume when the EC2 instance terminates. Which setting?
> → **DeleteOnTermination = true**

> Store credentials for multiple AWS accounts on one machine. What should you use?
> → **aws configure --profile <name>**

> Run a single CLI command in a different Region without changing saved configuration. Which option?
> → **--region**

> Run a Lambda locally with a named profile so it can reach real AWS resources. Which command?
> → **sam local invoke <Function> --profile <name>**

> Save SAM CLI build and deploy defaults, including parameter_overrides. Which file?
> → **samconfig.toml**

> Override CloudFormation template parameters at deploy time. Which option?
> → **--parameter-overrides**

> What is the precedence order for parameter values at deploy?
> → **Template default < samconfig.toml < --parameter-overrides**

> Which wins if a value is set both in samconfig.toml and via --parameter-overrides?
> → **--parameter-overrides**

> Where should application configuration (non-secret) live?
> → **Environment variables**

> A developer puts AWS access keys in environment variables for a release. What is wrong?
> → **You must never store AWS credentials in environment variables — use roles/profiles**

> Why does a Lambda or EC2 instance "just work" with no credentials configured?
> → **The instance/task IAM role is found at the end of the credential resolution chain**

> Give the credential resolution order (roughly).
> → **Explicit parameters → env variables → credentials file/profile → container credentials → instance/task IAM role**

> Check whether an IAM user has permission for an EC2 action without performing it. Which flag?
> → **--dry-run**

> What does a --dry-run result of DryRunOperation mean?
> → **Permission is allowed**

> What does a --dry-run result of UnauthorizedOperation mean?
> → **Permission is denied**

> Where does the AWS CLI store credentials and configuration?
> → **~/.aws/credentials and ~/.aws/config**

> Fully managed ETL service for preparing data for analytics. Which service?
> → **AWS Glue**

> Can AWS Glue orchestrate a workflow?
> → **No**

> Which service runs batch computing jobs and dynamically provisions optimal compute (CPU or memory optimized)?
> → **AWS Batch**

> Can AWS Batch orchestrate a workflow?
> → **No**

> Which authentication methods does the CLI/SDK rely on besides explicit keys?
> → **Named profiles and IAM roles (instance/task roles)**

> A CLI command fails because it targets the wrong Region. Which flag should have been used?
> → **--region**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Terminal operations | **AWS CLI** |
| App code | **AWS SDK** |
| Underlying interface | **AWS API** |
| Console/CLI/SDK call | **The AWS API** |
| Test a change | **--dry-run** |
| DryRunOperation | **Allowed** |
| UnauthorizedOperation | **Denied** |
| Volume on terminate | **DeleteOnTermination** |
| Keep volume | **DeleteOnTermination = false** |
| Delete volume | **DeleteOnTermination = true** |
| Modify attribute | **aws ec2 modify-instance-attribute** |
| Multiple accounts | **aws configure --profile** |
| One-command Region | **--region** |
| Local Lambda + AWS access | **sam local invoke --profile** |
| SAM CLI settings file | **samconfig.toml** |
| Template overrides | **--parameter-overrides** |
| Override precedence | **CLI > samconfig.toml > template default** |
| App configuration | **Environment variables** |
| Credentials in env vars | **Never — use roles** |
| Credential chain end | **Instance/task IAM role** |
| Credentials file | **~/.aws/credentials** |
| Config file | **~/.aws/config** |
| Managed ETL | **AWS Glue** |
| Glue workflow orchestration | **Not supported** |
| Batch compute jobs | **AWS Batch** |
| Batch workflow orchestration | **Not supported** |
| Region override scope | **Single command only** |
| Named profile use case | **Dev/prod switching on one terminal** |
| Instance "just works" | **IAM role at end of chain** |
| Permission check without action | **ec2 --dry-run** |
| ModifyInstanceAttribute | **EC2 API** |
| EBS after termination | **Controlled by DeleteOnTermination** |
| App secrets | **Roles/profiles, never env vars** |
| Profile syntax | **aws configure --profile production** |
