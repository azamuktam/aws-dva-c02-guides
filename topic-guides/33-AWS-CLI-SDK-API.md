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

> Verify a CLI change without actually applying it. Which flag?
> → **--dry-run**

> Keep the EBS volume when the instance terminates. Which setting?
> → **DeleteOnTermination = false (modify-instance-attribute)**

> Where does the CLI store credentials and configuration?
> → **~/.aws/credentials and ~/.aws/config (aws configure)**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Terminal | **CLI** |
| App code | **SDK** |
| Underlying | **API** |
| Test a change | **--dry-run** |
| Volume on terminate | **DeleteOnTermination** |
| Config | **aws configure** |
