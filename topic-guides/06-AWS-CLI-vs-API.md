# AWS CLI vs AWS API
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
