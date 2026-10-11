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
