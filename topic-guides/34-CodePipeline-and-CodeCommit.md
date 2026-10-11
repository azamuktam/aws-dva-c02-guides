## AWS CodeCommit

**AWS CodeCommit:** A fully managed AWS service for **hosting private Git repositories**, where you can store, version, branch, merge, and collaborate on source code.

**CodeCommit authentication:** It supports:

- **HTTPS + Git credentials** (username/password **generated from IAM**)
- **SSH + SSH keys**
- **HTTPS + AWS CLI credential helper**

**Exam memory:** **CodeCommit = AWS-managed private Git repository.**

## AWS CodePipeline

**Why it exists**

**AWS CodePipeline is primarily a Continuous Delivery (CD) service** that automates the process of building, testing, and deploying code through a release pipeline

It is deliberately not a build tool or a deployment tool — it **calls** those.

```plain text
Source  →  Build  →  (Test)  →  (Approve)  →  Deploy
  ↓         ↓                                    ↓
GitHub    CodeBuild                    CodeDeploy / Beanstalk / ECS / Lambda / CloudFormation
   S3
```

**Structure**

- **Pipeline** → **Stages** → **Actions**. Actions within a stage can run in parallel or in sequence.
- **Source stage** — GitHub, S3, ECR, Bitbucket. (CodeCommit historically; it is no longer on the in-scope list.)
- **Build stage** — CodeBuild, or Jenkins.
- **Deploy stage** — CodeDeploy, Elastic Beanstalk, ECS, Lambda, CloudFormation, S3.
- **Artifacts** pass between stages through an **S3 artifact bucket**. Each action declares input and output artifacts.
- **Cross-region actions** — a pipeline can deploy into multiple Regions. Each Region needs **its own artifact bucket**, because artifacts must be readable in the Region where the action runs; CodePipeline replicates them for you. This is how one pipeline deploys the same release to us-east-1 and eu-west-1.
- **Cross-account actions** — the pipeline account assumes a role in the target account, and the artifact bucket must be encrypted with a **customer managed KMS key** shared with that account. The AWS managed key cannot be shared across accounts, which is why cross-account pipelines specifically require a CMK.
- **Triggers** — a source change (webhook or polling), a schedule, or a manual start. Source events reach the pipeline via EventBridge.
- Integrates with third-party tools: GitHub, Jenkins, and other providers.

**Manual approval**

An action type that **pauses the pipeline** until a person approves or rejects.

```plain text
Build → Test → [Manual Approval] → Deploy to Production
                      ↓
                 SNS notification to reviewers
```

- Configure an **SNS topic** so approvers are notified rather than having to watch the console.
- Approvers need `codepipeline:PutApprovalResult` permission.
- Optionally include a review URL and a comment.

Why this and not something else: SQS stores messages but has no human interface; Step Functions `Wait` pauses for a *duration*, not for a decision. For a release gate inside CI/CD, manual approval is purpose-built.

| Need | Answer |
| --- | --- |
| Human approval before the next stage | **CodePipeline manual approval** |
| Notify the approvers | **SNS** |
| Queue messages between components | **SQS** |
| Orchestrate a workflow, including human callbacks | **Step Functions** (`.waitForTaskToken`) |

**Recall check — CodePipeline**

- **Automate the entire CI/CD workflow** → CodePipeline
- **Connect source, build, and deploy stages** → CodePipeline
- **Compile code or run tests** → CodeBuild
- **Deploy a new application version** → CodeDeploy
- **Pipeline artifact storage** → **S3**
- **Human approval before production** → manual approval action
- **Notify approvers** → SNS
- **CodePipeline orchestrates; it does not build or deploy itself**

---

# Question patterns

> Orchestrate source → build → test → deploy across services. Which service?
> → **AWS CodePipeline**

> Host a private Git repository fully managed within AWS. Which service?
> → **AWS CodeCommit**

> Does CodePipeline compile code or deploy applications itself?
> → **No — it orchestrates and calls build/deploy tools**

> What is the structural hierarchy of a CodePipeline pipeline?
> → **Pipeline → Stages → Actions**

> Within a single stage, how can actions run?
> → **In parallel or in sequence**

> Which sources can a CodePipeline source stage use?
> → **GitHub, S3, ECR, Bitbucket**

> Which build providers can a CodePipeline build stage use?
> → **CodeBuild or Jenkins**

> Which providers can a CodePipeline deploy stage use?
> → **CodeDeploy, Elastic Beanstalk, ECS, Lambda, CloudFormation, S3**

> How do files pass between pipeline stages?
> → **Artifacts stored in an S3 artifact bucket**

> How does each action declare what it consumes and produces?
> → **Input and output artifacts**

> A pipeline deploys into multiple Regions. What must each Region have?
> → **Its own artifact bucket**

> Why does each Region need its own artifact bucket in a cross-region pipeline?
> → **Artifacts must be readable in the Region where the action runs; CodePipeline replicates them**

> What does a cross-account pipeline do to reach the target account?
> → **The pipeline account assumes a role in the target account**

> Which key type must encrypt the artifact bucket for a cross-account pipeline?
> → **A customer managed KMS key shared with the other account**

> Why can't the AWS managed KMS key be used for cross-account pipelines?
> → **The AWS managed key cannot be shared across accounts**

> Which three ways can a pipeline be triggered?
> → **A source change (webhook or polling), a schedule, or a manual start**

> How do source events reach the pipeline?
> → **Via EventBridge**

> A human must approve before the production deployment. Which action type?
> → **CodePipeline manual approval**

> What does a manual approval action do to the pipeline?
> → **Pauses it until a person approves or rejects**

> How do you notify approvers instead of them watching the console?
> → **Configure an SNS topic**

> Which IAM permission must approvers have?
> → **codepipeline:PutApprovalResult**

> What optional details can be attached to a manual approval?
> → **A review URL and a comment**

> Why not use SQS for a release gate?
> → **SQS stores messages but has no human interface**

> Why not use Step Functions Wait for a release gate?
> → **Wait pauses for a duration, not for a decision**

> Which Step Functions feature supports human callbacks in workflows?
> → **.waitForTaskToken**

> Which CodeCommit authentication methods are supported?
> → **HTTPS + Git credentials (generated from IAM), SSH + SSH keys, HTTPS + AWS CLI credential helper**

> Where do CodeCommit HTTPS Git credentials come from?
> → **Generated from IAM**

> Compile code or run unit tests in a pipeline. Which service?
> → **CodeBuild**

> Deploy a new application version. Which service?
> → **CodeDeploy**

> Which service connects source, build, and deploy stages without building or deploying itself?
> → **CodePipeline**

> Is CodeCommit on the current in-scope CodePipeline source list?
> → **No — it is historically used but no longer on the in-scope list (GitHub, S3, ECR, Bitbucket are)**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Orchestration | **CodePipeline** |
| Git in AWS | **CodeCommit** |
| Private Git hosting | **CodeCommit (fully managed)** |
| Pipeline structure | **Pipeline → Stages → Actions** |
| Parallel or sequential | **Actions within a stage** |
| Source providers | **GitHub, S3, ECR, Bitbucket** |
| Build providers | **CodeBuild, Jenkins** |
| Deploy providers | **CodeDeploy, Beanstalk, ECS, Lambda, CloudFormation, S3** |
| Between stages | **S3 artifact bucket** |
| Cross-region | **One artifact bucket per Region** |
| Cross-region replication | **Done by CodePipeline** |
| Cross-account | **Assume role in target account** |
| Cross-account key | **Customer managed KMS key (shared)** |
| AWS managed key | **Cannot be shared across accounts** |
| Triggers | **Source change, schedule, manual** |
| Source events | **EventBridge** |
| Human approval | **Manual approval action** |
| Notify approvers | **SNS** |
| Approver permission | **codepipeline:PutApprovalResult** |
| Approval extras | **Review URL + comment** |
| No human interface | **SQS (not for gates)** |
| Duration, not decision | **Step Functions Wait** |
| Human callbacks | **Step Functions .waitForTaskToken** |
| Compile/test | **CodeBuild** |
| Deploy version | **CodeDeploy** |
| Artifact declaration | **Input/output artifacts** |
| CodeCommit HTTPS auth | **IAM-generated Git credentials** |
| CodeCommit SSH auth | **SSH keys** |
| CodeCommit helper auth | **AWS CLI credential helper** |
| CodePipeline builds/deploys? | **No — it orchestrates only** |
| Webhook vs polling | **Source-change trigger methods** |
| Automatic pipeline start | **Webhook, schedule, or EventBridge** |
| Cross-region deploy example | **us-east-1 + eu-west-1 from one pipeline** |
| Approvers find out | **SNS topic** |
| Pipeline bucket location | **S3** |
