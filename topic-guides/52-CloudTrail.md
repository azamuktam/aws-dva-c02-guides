## AWS CloudTrail
**Why it exists**
CloudWatch tells you what your application did. CloudTrail tells you what was done **to AWS itself** — every API call, from the console, CLI, SDKs, and AWS services acting on your behalf.
It answers: **who did what, when, and from where.**
```plain text
Any AWS API call
      ↓
CloudTrail
      ↓
Event history (90 days, free)  or  a Trail → S3 (+ optionally CloudWatch Logs)
```
**What to know**
- Records API calls with the identity, timestamp, source IP address, request parameters, and response elements.
- Logs actions from **Console, CLI, SDK, and AWS services**.
- **Event history** covers the last 90 days of management events for free. For events older than 90 days, use **Athena** to analyze **CloudTrail** logs stored in the** S3 bucket**
- **Management events** (control-plane operations, like creating a role) are logged by default. **Data events** (S3 object-level, Lambda invocations) are high-volume and must be explicitly enabled.
- **Insights** detects unusual API activity patterns.
- Multi-Region and organisation-wide trails are supported.
- **Log file validation** proves the logs weren’t tampered with.
- Athena is the usual way to query CloudTrail logs in S3.
Applies to: IAM, EC2, Lambda, S3, CloudFormation, KMS — effectively everything.
**The three-way distinction, one more time**

| Tool | Question |
| --- | --- |
| **CloudTrail** | *“Who made the change?”* |
| **AWS Config** | *“What is the current configuration, and is it compliant?”* |
| **CloudFormation Drift Detection** | *“Does it still match my template?”* |

Worked example — someone changes a Lambda IAM role:
- **CloudTrail** → identifies which user or role changed it, and when
- **AWS Config** → checks whether the role now violates a rule
- **Drift Detection** → checks whether it differs from the stack template
- **CloudTrail Insights** — detects **unusual API call rates or API error rates** by comparing current activity with your normal baseline
**Recall check — CloudTrail**
- **“Who changed this?” / audit logs / API activity** → CloudTrail
- **Investigate changes / security auditing** → CloudTrail
- **Long-term retention of API logs** → create a Trail → S3
- **Alarm on specific API activity** → Trail → CloudWatch Logs → metric filter → alarm
- **S3 object-level or Lambda invocation logging** → enable **data events**
- **Event history without a trail** → last 90 days, management events only
- **Query CloudTrail logs with SQL** → Athena
- **CloudTrail = who. Config = compliance. Drift = matches the template.**
- **CloudTrail Insights** — detects **unusual API call rates or API error rates** by comparing current activity with your normal baseline
