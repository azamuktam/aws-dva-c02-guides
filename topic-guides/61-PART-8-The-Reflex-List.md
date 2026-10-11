# PART 8 — THE REFLEX LIST
Everything above, compressed. **Use this to check yourself, not to learn from.** If you read a row and can’t reconstruct *why* it’s true from the relevant section, that’s the section to go back to.
---
## Domain 1 — Development with AWS Services (32%)
<table header-row="true">
<tr>
<td>Situation</td>
<td>Answer</td>
</tr>
<tr>
<td>Improve Lambda performance</td>
<td>Initialise clients and connections **outside the handler**</td>
</tr>
<tr>
<td>Reuse data between invocations</td>
<td>Execution context</td>
</tr>
<tr>
<td>Need persistent or shared storage from Lambda</td>
<td>S3 / DynamoDB / **EFS** — never `/tmp`</td>
</tr>
<tr>
<td>Lambda is CPU-bound</td>
<td>**Increase memory** (CPU scales with memory)</td>
</tr>
<tr>
<td>Eliminate cold starts</td>
<td>Provisioned concurrency; SnapStart (free)</td>
</tr>
<tr>
<td>Must always have capacity / not starve others</td>
<td>Reserved concurrency</td>
</tr>
<tr>
<td>Disable a function without deleting it</td>
<td>Reserved concurrency = 0</td>
</tr>
<tr>
<td>Too many DB connections from Lambda</td>
<td>Reserved concurrency, or RDS Proxy</td>
</tr>
<tr>
<td>Concurrency needed</td>
<td>requests/sec × duration</td>
</tr>
<tr>
<td>Job longer than 15 minutes</td>
<td>Fargate, Batch, or Step Functions</td>
</tr>
<tr>
<td>Async invocation retries</td>
<td>**2** by default</td>
</tr>
<tr>
<td>Sync invocation failure</td>
<td>The **caller** retries; Lambda does not</td>
</tr>
<tr>
<td>Async failures must not be lost</td>
<td>On-failure **destination** (or DLQ)</td>
</tr>
<tr>
<td>Need success notifications too</td>
<td>Destinations, not DLQ</td>
</tr>
<tr>
<td>Clients must not break on deploy</td>
<td>Point them at an **alias**</td>
</tr>
<tr>
<td>Instant Lambda rollback</td>
<td>Move the alias back a version</td>
</tr>
<tr>
<td>Send 10% of traffic to new code</td>
<td>Alias weighted routing</td>
</tr>
<tr>
<td>Share libraries across functions</td>
<td>**Layers**</td>
</tr>
<tr>
<td>Simplest HTTPS endpoint, no keys</td>
<td>Function URL</td>
</tr>
<tr>
<td>Need API keys, usage plans, WAF</td>
<td>API Gateway</td>
</tr>
<tr>
<td>Duplicate requests must not double-charge</td>
<td>Idempotency: DynamoDB conditional write on a request ID</td>
</tr>
<tr>
<td>RCU / WCU block sizes</td>
<td>**4 KB read, 1 KB write**</td>
</tr>
<tr>
<td>Eventually consistent reads</td>
<td>**Half** the RCUs</td>
</tr>
<tr>
<td>Transactional operations</td>
<td>**2×** the cost</td>
</tr>
<tr>
<td>`FilterExpression` / `ProjectionExpression`</td>
<td>Do **not** reduce consumed capacity</td>
</tr>
<tr>
<td>Query by a non-key attribute</td>
<td>**GSI**</td>
</tr>
<tr>
<td>Fewer results than expected + `LastEvaluatedKey`</td>
<td>Paginate with `ExclusiveStartKey`</td>
</tr>
<tr>
<td>Empty page with `LastEvaluatedKey`</td>
<td>Keep paging — not a bug</td>
</tr>
<tr>
<td>Production table needs a new query key</td>
<td>**GSI** (LSI impossible after creation)</td>
</tr>
<tr>
<td>Strongly consistent reads on an index</td>
<td>**LSI**</td>
</tr>
<tr>
<td>Base table writes throttled for no clear reason</td>
<td>**GSI capacity**</td>
</tr>
<tr>
<td>Prevent lost updates</td>
<td>`ConditionExpression` on a version attribute</td>
</tr>
<tr>
<td>Insert only if absent</td>
<td>`attribute_not_exists`</td>
</tr>
<tr>
<td>Concurrent counter increments</td>
<td>Atomic counter (not idempotent)</td>
</tr>
<tr>
<td>Items missing after `BatchWriteItem`</td>
<td>Unhandled **`UnprocessedItems`**</td>
</tr>
<tr>
<td>All-or-nothing across items</td>
<td>`TransactWriteItems`</td>
</tr>
<tr>
<td>Auto-delete expired items, free</td>
<td>**TTL** (epoch **seconds**, up to 48 h delay)</td>
</tr>
<tr>
<td>Archive expiring items</td>
<td>TTL + Streams + Lambda → S3</td>
</tr>
<tr>
<td>Item over 400 KB</td>
<td>Object in S3, pointer in DynamoDB</td>
</tr>
<tr>
<td>Microsecond DynamoDB reads, minimal code change</td>
<td>**DAX**</td>
</tr>
<tr>
<td>DAX and strongly consistent reads</td>
<td>DAX is bypassed</td>
</tr>
<tr>
<td>Item changed → run code</td>
<td>DynamoDB **Streams** + Lambda</td>
</tr>
<tr>
<td>Need the value before and after</td>
<td>`NEW_AND_OLD_IMAGES`</td>
</tr>
<tr>
<td>Change the format of a Lambda response</td>
<td>**Mapping template (VTL)**</td>
</tr>
<tr>
<td>Different backend per environment</td>
<td>**Stage variables**</td>
</tr>
<tr>
<td>Developers need a new API version</td>
<td>**Create a dev stage**</td>
</tr>
<tr>
<td>API changes not visible</td>
<td>**`CreateDeployment`**</td>
</tr>
<tr>
<td>Reduce Lambda invocations on a read-heavy API</td>
<td>**API Gateway caching**</td>
</tr>
<tr>
<td>Call SQS or DynamoDB with no Lambda</td>
<td>AWS service integration</td>
</tr>
<tr>
<td>One event, several independent consumers</td>
<td>**SNS → SQS fan-out**</td>
</tr>
<tr>
<td>Route on fields inside the payload</td>
<td>**EventBridge**</td>
</tr>
<tr>
<td>Add consumers without changing the producer</td>
<td>**EventBridge**</td>
</tr>
<tr>
<td>Scheduled / cron trigger</td>
<td>**EventBridge schedule**</td>
</tr>
<tr>
<td>Replay past events</td>
<td>**EventBridge archive + replay**</td>
</tr>
<tr>
<td>React to an AWS service event</td>
<td>EventBridge default bus</td>
</tr>
<tr>
<td>Source → filter → enrich → target, no code</td>
<td>**EventBridge Pipes**</td>
</tr>
<tr>
<td>Orchestrate a multi-step workflow</td>
<td>**Step Functions**</td>
</tr>
<tr>
<td>Chain of Lambdas calling each other</td>
<td>Replace with Step Functions</td>
</tr>
<tr>
<td>Workflow longer than 5 minutes</td>
<td>Step Functions **Standard**</td>
</tr>
<tr>
<td>High-volume short workflows, cheapest</td>
<td>Step Functions **Express**</td>
</tr>
<tr>
<td>Wait for human approval in a workflow</td>
<td>`.waitForTaskToken` (Standard)</td>
</tr>
<tr>
<td>Wait for an ECS task or Batch job</td>
<td>`.sync`</td>
</tr>
<tr>
<td>Ordering, replay, many independent consumers</td>
<td>**Kinesis Data Streams**</td>
</tr>
<tr>
<td>Load a stream into S3/Redshift/OpenSearch, no code</td>
<td>**Firehose**</td>
</tr>
<tr>
<td>“Near real time” in the question</td>
<td>Firehose</td>
</tr>
<tr>
<td>One shard = one KCL worker</td>
<td>More instances don’t help</td>
</tr>
<tr>
<td>GraphQL / real-time subscriptions</td>
<td>**AppSync**</td>
</tr>
<tr>
<td>Full-text search</td>
<td>**OpenSearch**</td>
</tr>
<tr>
<td>Search over DynamoDB data</td>
<td>Streams → Lambda → OpenSearch</td>
</tr>
<tr>
<td>SQL on S3</td>
<td>**Athena**</td>
</tr>
<tr>
<td>SQL on one object</td>
<td>**S3 Select**</td>
</tr>
</table>
## Domain 2 — Security (26%)
<table header-row="true">
<tr>
<td>Situation</td>
<td>Answer</td>
</tr>
<tr>
<td>A service needs AWS permissions</td>
<td>IAM **role**, never stored keys</td>
</tr>
<tr>
<td>“Who can assume this role?”</td>
<td>**Trust policy**</td>
</tr>
<tr>
<td>“What can this identity do?”</td>
<td>Identity-based policy</td>
</tr>
<tr>
<td>Cross-account access to a resource</td>
<td>Resource-based policy</td>
</tr>
<tr>
<td>Cap maximum permissions</td>
<td>Permissions boundary</td>
</tr>
<tr>
<td>Restrict an entire account or OU</td>
<td>SCP</td>
</tr>
<tr>
<td>`Principal` element present</td>
<td>It’s a resource-based policy</td>
</tr>
<tr>
<td>One policy that works per user</td>
<td>Policy variables (`${aws:username}`)</td>
</tr>
<tr>
<td>**Explicit Deny always wins**</td>
<td>Look for a Deny before adding Allows</td>
</tr>
<tr>
<td>Billing permissions don’t work</td>
<td>Root must enable IAM access to Billing</td>
</tr>
<tr>
<td>Find public or cross-account access</td>
<td>IAM **Access Analyzer**</td>
</tr>
<tr>
<td>Test whether a policy allows an action</td>
<td>**Policy Simulator**</td>
</tr>
<tr>
<td>Find unused permissions</td>
<td>**Access Advisor**</td>
</tr>
<tr>
<td>Cross-account `AssumeRole` failing</td>
<td>Check **both** the trust policy and the caller’s policy</td>
</tr>
<tr>
<td>Third-party vendor access</td>
<td>**`ExternalId`** (confused deputy)</td>
</tr>
<tr>
<td>Longest STS session</td>
<td>12 hours (1 hour with role chaining)</td>
</tr>
<tr>
<td>Which identity am I</td>
<td>`GetCallerIdentity`</td>
</tr>
<tr>
<td>Invalidate all existing sessions</td>
<td>Deny by `aws:TokenIssueTime`</td>
</tr>
<tr>
<td>User login / sign-up / JWT</td>
<td>**Cognito User Pool**</td>
</tr>
<tr>
<td>Secure API Gateway with a JWT, no code</td>
<td>**Cognito User Pool authorizer**</td>
</tr>
<tr>
<td>Custom auth logic / third-party IdP</td>
<td>**Lambda authorizer**</td>
</tr>
<tr>
<td>AWS users or services calling the API</td>
<td>**IAM authorizer**</td>
</tr>
<tr>
<td>Temporary AWS credentials for app users</td>
<td>**Cognito Identity Pool**</td>
</tr>
<tr>
<td>Mobile app uploads to S3, no keys embedded</td>
<td>**Identity Pool** (after a User Pool sign-in)</td>
</tr>
<tr>
<td>**User Pool = authentication. Identity Pool = AWS access.**</td>
<td>Both together for the full flow</td>
</tr>
<tr>
<td>Encrypt data larger than 4 KB</td>
<td>**Envelope encryption / ****`GenerateDataKey`**</td>
</tr>
<tr>
<td>KMS `Encrypt` fails on a big file</td>
<td>Use `GenerateDataKey`, **not** a quota increase</td>
</tr>
<tr>
<td>`AccessDenied` on KMS despite an IAM Allow</td>
<td>**Check the key policy**</td>
</tr>
<tr>
<td>Audit who decrypted an object</td>
<td>**SSE-KMS** + CloudTrail</td>
</tr>
<tr>
<td>Change keys without changing code</td>
<td>**Alias**</td>
</tr>
<tr>
<td>Encrypted data usable in another Region</td>
<td>Multi-Region key</td>
</tr>
<tr>
<td>Simple encryption at rest</td>
<td>**SSE-S3** (`AES256`)</td>
</tr>
<tr>
<td>Control and audit the key</td>
<td>**SSE-KMS** (`aws:kms`)</td>
</tr>
<tr>
<td>Customer supplies the key each request</td>
<td>**SSE-C**</td>
</tr>
<tr>
<td>Encrypt before upload</td>
<td>Client-side encryption</td>
</tr>
<tr>
<td>Require every upload be encrypted</td>
<td>**Bucket policy ****`Deny`** on the missing header</td>
</tr>
<tr>
<td>Database credentials + automatic rotation</td>
<td>**Secrets Manager**</td>
</tr>
<tr>
<td>Encrypted config, no rotation needed</td>
<td>**Parameter Store ****`SecureString`**</td>
</tr>
<tr>
<td>Feature flag / config change without redeploy</td>
<td>**AppConfig**</td>
</tr>
<tr>
<td>Validate config and roll back automatically</td>
<td>**AppConfig**</td>
</tr>
<tr>
<td>Passwordless database login</td>
<td>**IAM DB authentication**</td>
</tr>
<tr>
<td>Env vars must be unreadable by other developers</td>
<td>Customer managed KMS key + encryption helpers</td>
</tr>
<tr>
<td>Temporary access to a private S3 object</td>
<td>**Presigned URL**</td>
</tr>
<tr>
<td>Presigned URL expires too early</td>
<td>Signed with a role’s temporary credentials</td>
</tr>
<tr>
<td>Private content through CloudFront</td>
<td>**Signed URL / signed cookie** (key groups)</td>
</tr>
<tr>
<td>Many private files at once</td>
<td>Signed **cookie**</td>
</tr>
<tr>
<td>Restrict by IP range</td>
<td>CloudFront signed URL</td>
</tr>
<tr>
<td>Lock the S3 bucket to CloudFront only</td>
<td>**OAC**</td>
</tr>
<tr>
<td>Block SQL injection / XSS</td>
<td>**WAF**</td>
</tr>
<tr>
<td>Block a specific IP address</td>
<td>**NACL** (security groups cannot deny)</td>
</tr>
<tr>
<td>Inbound allowed but the response is blocked</td>
<td>**NACL outbound rules** (stateless)</td>
</tr>
<tr>
<td>Security group = stateful. NACL = stateless.</td>
<td></td>
</tr>
<tr>
<td>See why traffic is blocked</td>
<td>**VPC Flow Logs** (REJECT)</td>
</tr>
<tr>
<td>Private access to S3 or DynamoDB</td>
<td>**Gateway endpoint** (free)</td>
</tr>
<tr>
<td>Private access to other AWS services</td>
<td>**Interface endpoint** (PrivateLink)</td>
</tr>
<tr>
<td>`CannotPullContainerError`</td>
<td>Task **execution** role</td>
</tr>
<tr>
<td>Container app gets `AccessDenied`</td>
<td>Task **role**</td>
</tr>
<tr>
<td>Enable HTTPS / auto-renew certificates</td>
<td>**ACM** (us-east-1 for CloudFront)</td>
</tr>
</table>
## Domain 3 — Deployment (24%)
<table header-row="true">
<tr>
<td>Situation</td>
<td>Answer</td>
</tr>
<tr>
<td>Serverless IaC (Lambda + API GW + DynamoDB)</td>
<td>**SAM**</td>
</tr>
<tr>
<td>`Transform: AWS::Serverless-2016-10-31`</td>
<td>It’s a **SAM** template</td>
</tr>
<tr>
<td>IaC in a programming language</td>
<td>**CDK**</td>
</tr>
<tr>
<td>Generate the CloudFormation template from CDK</td>
<td>`cdk synth`</td>
</tr>
<tr>
<td>See what would change before deploying</td>
<td>`cdk diff` / CloudFormation **change set**</td>
</tr>
<tr>
<td>Test one Lambda locally</td>
<td>`sam local invoke`</td>
</tr>
<tr>
<td>Test a REST API locally</td>
<td>`sam local start-api`</td>
</tr>
<tr>
<td>Upload artifacts to S3</td>
<td>`sam package`</td>
</tr>
<tr>
<td>Only required CloudFormation section</td>
<td>`Resources`</td>
</tr>
<tr>
<td>Static lookup per Region</td>
<td>`Mappings` + **`Fn::FindInMap`** (3 arguments)</td>
</tr>
<tr>
<td>Deploy-time input</td>
<td>`Parameters`</td>
</tr>
<tr>
<td>Decide whether to create something</td>
<td>`Conditions` (Resources and Outputs only)</td>
</tr>
<tr>
<td>Get a resource’s primary identifier</td>
<td>`!Ref`</td>
</tr>
<tr>
<td>Get a specific attribute (ARN, DNS name)</td>
<td>`!GetAtt`</td>
</tr>
<tr>
<td>Use a value from another stack</td>
<td>`Outputs` + `Export` + `!ImportValue`</td>
</tr>
<tr>
<td>Export name conflict</td>
<td>Same name twice in the **same Region**</td>
</tr>
<tr>
<td>Same resources in many Regions/accounts</td>
<td>**StackSets**</td>
</tr>
<tr>
<td>Keep the database when the stack is deleted</td>
<td>`DeletionPolicy: Retain`</td>
</tr>
<tr>
<td>Resource changed outside CloudFormation</td>
<td>**Detect Stack Drift**</td>
</tr>
<tr>
<td>Automate the whole CI/CD workflow</td>
<td>**CodePipeline**</td>
</tr>
<tr>
<td>Pipeline artifact storage</td>
<td>**S3**</td>
</tr>
<tr>
<td>Human approval before production</td>
<td>**Manual approval** action + SNS</td>
</tr>
<tr>
<td>Compile, test, produce artifacts</td>
<td>**CodeBuild**</td>
</tr>
<tr>
<td>Install dependencies</td>
<td>`install` phase</td>
</tr>
<tr>
<td>Set the runtime version</td>
<td>`runtime-versions` in `install`</td>
</tr>
<tr>
<td>Log in to ECR</td>
<td>`pre_build`</td>
</tr>
<tr>
<td>Push image / upload artifacts</td>
<td>`post_build`</td>
</tr>
<tr>
<td>Run cleanup even on failure</td>
<td>`finally`</td>
</tr>
<tr>
<td>Pass a secret to a build</td>
<td>`env/secrets-manager` or `parameter-store`</td>
</tr>
<tr>
<td>Never put secrets in</td>
<td>`env/variables`</td>
</tr>
<tr>
<td>Slow builds redownloading dependencies</td>
<td>`cache`</td>
</tr>
<tr>
<td>Build must reach private RDS</td>
<td>Run CodeBuild in the **VPC**</td>
</tr>
<tr>
<td>Build times out</td>
<td>Default 60 min, max 8 h</td>
</tr>
<tr>
<td>Build Docker images</td>
<td>**Privileged mode**</td>
</tr>
<tr>
<td>Pass a value to the next stage</td>
<td>`exported-variables`</td>
</tr>
<tr>
<td>Run tests before Lambda traffic shifts</td>
<td>**`BeforeAllowTraffic`**</td>
</tr>
<tr>
<td>Verify after traffic is live</td>
<td>`AfterAllowTraffic`</td>
</tr>
<tr>
<td>Test listener before production (ECS)</td>
<td>`AfterAllowTestTraffic`</td>
</tr>
<tr>
<td>Last chance to fail an EC2 deployment</td>
<td>`ValidateService`</td>
</tr>
<tr>
<td>`ValidateService` on Lambda</td>
<td>**Does not exist** (distractor)</td>
</tr>
<tr>
<td>New deployments failing immediately</td>
<td>Broken **`ApplicationStop`** in the **previous** revision</td>
</tr>
<tr>
<td>10% now, wait, then the rest</td>
<td>**Canary**</td>
</tr>
<tr>
<td>Equal increments on a timer</td>
<td>**Linear**</td>
</tr>
<tr>
<td>How Lambda traffic shifting works</td>
<td>**Alias weighted routing**, moved by CodeDeploy</td>
</tr>
<tr>
<td>Roll back automatically on errors</td>
<td>**CloudWatch alarm** + automatic rollback</td>
</tr>
<tr>
<td>CodeDeploy agent required</td>
<td>EC2 / on-premises only</td>
</tr>
<tr>
<td>Fastest Beanstalk deployment</td>
<td>All at once</td>
</tr>
<tr>
<td>Keep full capacity during a Beanstalk deploy</td>
<td>Rolling with additional batch</td>
</tr>
<tr>
<td>Safest, zero downtime, easy rollback</td>
<td>**Immutable**</td>
</tr>
<tr>
<td>Canary in Beanstalk</td>
<td>**Traffic splitting**</td>
</tr>
<tr>
<td>Fastest possible Beanstalk rollback</td>
<td>Blue/green **URL (CNAME) swap**</td>
</tr>
<tr>
<td>EC2 burst balances lost after a deploy</td>
<td>**Immutable / Traffic splitting** (new instances)</td>
</tr>
<tr>
<td>Beanstalk application versions stored in</td>
<td>**S3**</td>
</tr>
<tr>
<td>Customise a Beanstalk environment</td>
<td>**`.ebextensions`**</td>
</tr>
<tr>
<td>Database must survive a Beanstalk rebuild</td>
<td>Create **RDS externally**</td>
</tr>
<tr>
<td>Private npm / pip / Maven repository</td>
<td>**CodeArtifact**</td>
</tr>
<tr>
<td>Cache and proxy public packages</td>
<td>CodeArtifact external connection</td>
</tr>
</table>
## Domain 4 — Troubleshooting and Optimization (18%)
<table header-row="true">
<tr>
<td>Symptom</td>
<td>Cause / answer</td>
</tr>
<tr>
<td>Duplicate SQS processing</td>
<td>Visibility timeout \< processing time</td>
</tr>
<tr>
<td>One bad message retries the whole batch</td>
<td>Enable **`ReportBatchItemFailures`**</td>
</tr>
<tr>
<td>Stream processing frozen forever</td>
<td>Poison pill; retry limits / **`BisectBatchOnFunctionError`** / on-failure destination</td>
</tr>
<tr>
<td>Rising `IteratorAge`</td>
<td>Consumer falling behind</td>
</tr>
<tr>
<td>**429 / ****`Throttles`** on Lambda</td>
<td>Concurrency limit reached</td>
</tr>
<tr>
<td>Lambda produces no logs</td>
<td>Execution role missing `logs:*`</td>
</tr>
<tr>
<td>Lambda in a VPC cannot reach the internet</td>
<td>**NAT gateway** (or VPC endpoint)</td>
</tr>
<tr>
<td>Memory or disk-space metric missing</td>
<td>Install the **CloudWatch agent**</td>
</tr>
<tr>
<td>Alarm when a string appears in logs</td>
<td>**Metric filter** + alarm</td>
</tr>
<tr>
<td>Metric filter shows nothing for old logs</td>
<td>Filters are **not retroactive**</td>
</tr>
<tr>
<td>Logs cost too much</td>
<td>Set log group **retention**</td>
</tr>
<tr>
<td>Alarm stuck in `INSUFFICIENT_DATA`</td>
<td>No data, or wrong “treat missing data”</td>
</tr>
<tr>
<td>`Latency` high, `IntegrationLatency` low</td>
<td>The problem is **API Gateway**, not the backend</td>
</tr>
<tr>
<td>`IntegrationLatency` high</td>
<td>The **backend** is slow</td>
</tr>
<tr>
<td>**429** from API Gateway</td>
<td>Throttling</td>
</tr>
<tr>
<td>**502** from API Gateway</td>
<td>Malformed Lambda proxy response</td>
</tr>
<tr>
<td>**504** from API Gateway</td>
<td>Integration timeout (29 s default)</td>
</tr>
<tr>
<td>**403** from API Gateway</td>
<td>Authorizer denied, WAF, or **unlinked API key**</td>
</tr>
<tr>
<td>New API key returns 403</td>
<td>**`CreateUsagePlanKey`**</td>
</tr>
<tr>
<td>DynamoDB throttled while average usage is low</td>
<td>**Hot partition** / bad partition key</td>
</tr>
<tr>
<td>Filter traces by a business value</td>
<td>**Annotation** (indexed), not metadata</td>
</tr>
<tr>
<td>X-Ray works locally, fails on EC2</td>
<td>**Missing IAM role** (`AWSXRayDaemonWriteAccess`)</td>
</tr>
<tr>
<td>Change tracing volume without redeploying</td>
<td>**Sampling rules** in the console</td>
</tr>
<tr>
<td>X-Ray daemon port</td>
<td>**UDP 2000**</td>
</tr>
<tr>
<td>Which downstream call is slow</td>
<td>Service map + subsegments</td>
</tr>
<tr>
<td>Scale workers on queue depth</td>
<td>`ApproximateNumberOfMessagesVisible`</td>
</tr>
<tr>
<td>Queue short but messages old</td>
<td>`ApproximateAgeOfOldestMessage` — consumers too slow</td>
</tr>
<tr>
<td>Unexplained S3 storage cost</td>
<td>Incomplete multipart uploads → abort lifecycle rule</td>
</tr>
<tr>
<td>S3 event notification never fires</td>
<td>Destination **resource policy**</td>
</tr>
<tr>
<td>Cache returns stale data</td>
<td>Add TTL, or use write-through</td>
</tr>
<tr>
<td>First request always slow</td>
<td>Lazy-loading cache-miss penalty</td>
</tr>
<tr>
<td>Cache full of unread data</td>
<td>Write-through churn</td>
</tr>
<tr>
<td>Reduce Athena cost</td>
<td>Parquet + partitioning + compression + fewer columns</td>
</tr>
<tr>
<td>Read-heavy RDS, least code change</td>
<td>**Read Replicas**</td>
</tr>
<tr>
<td>“Standby should serve reads”</td>
<td>**Wrong** — standbys serve nothing</td>
</tr>
<tr>
<td>Who changed this AWS resource</td>
<td>**CloudTrail**</td>
</tr>
<tr>
<td>Is the resource compliant</td>
<td>**AWS Config**</td>
</tr>
<tr>
<td>Does it match the template</td>
<td>**Drift Detection**</td>
</tr>
</table>
---
