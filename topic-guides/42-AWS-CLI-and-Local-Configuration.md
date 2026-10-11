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
