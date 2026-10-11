# PART 8 — THE REFLEX LIST
Everything above, compressed. **Use this to check yourself, not to learn from.** If you read a row and can’t reconstruct *why* it’s true from the relevant section, that’s the section to go back to.
---
## Domain 1 — Development with AWS Services (32%)

| Situation | Answer |
| --- | --- |
| Improve Lambda performance | Initialise clients and connections **outside the handler** |
| Reuse data between invocations | Execution context |
| Need persistent or shared storage from Lambda | S3 / DynamoDB / **EFS** — never `/tmp` |
| Lambda is CPU-bound | **Increase memory** (CPU scales with memory) |
| Eliminate cold starts | Provisioned concurrency; SnapStart (free) |
| Must always have capacity / not starve others | Reserved concurrency |
| Disable a function without deleting it | Reserved concurrency = 0 |
| Too many DB connections from Lambda | Reserved concurrency, or RDS Proxy |
| Concurrency needed | requests/sec × duration |
| Job longer than 15 minutes | Fargate, Batch, or Step Functions |
| Async invocation retries | **2** by default |
| Sync invocation failure | The **caller** retries; Lambda does not |
| Async failures must not be lost | On-failure **destination** (or DLQ) |
| Need success notifications too | Destinations, not DLQ |
| Clients must not break on deploy | Point them at an **alias** |
| Instant Lambda rollback | Move the alias back a version |
| Send 10% of traffic to new code | Alias weighted routing |
| Share libraries across functions | **Layers** |
| Simplest HTTPS endpoint, no keys | Function URL |
| Need API keys, usage plans, WAF | API Gateway |
| Duplicate requests must not double-charge | Idempotency: DynamoDB conditional write on a request ID |
| RCU / WCU block sizes | **4 KB read, 1 KB write** |
| Eventually consistent reads | **Half** the RCUs |
| Transactional operations | **2×** the cost |
| `FilterExpression` / `ProjectionExpression` | Do **not** reduce consumed capacity |
| Query by a non-key attribute | **GSI** |
| Fewer results than expected + `LastEvaluatedKey` | Paginate with `ExclusiveStartKey` |
| Empty page with `LastEvaluatedKey` | Keep paging — not a bug |
| Production table needs a new query key | **GSI** (LSI impossible after creation) |
| Strongly consistent reads on an index | **LSI** |
| Base table writes throttled for no clear reason | **GSI capacity** |
| Prevent lost updates | `ConditionExpression` on a version attribute |
| Insert only if absent | `attribute_not_exists` |
| Concurrent counter increments | Atomic counter (not idempotent) |
| Items missing after `BatchWriteItem` | Unhandled **`UnprocessedItems`** |
| All-or-nothing across items | `TransactWriteItems` |
| Auto-delete expired items, free | **TTL** (epoch **seconds**, up to 48 h delay) |
| Archive expiring items | TTL + Streams + Lambda → S3 |
| Item over 400 KB | Object in S3, pointer in DynamoDB |
| Microsecond DynamoDB reads, minimal code change | **DAX** |
| DAX and strongly consistent reads | DAX is bypassed |
| Item changed → run code | DynamoDB **Streams** + Lambda |
| Need the value before and after | `NEW_AND_OLD_IMAGES` |
| Change the format of a Lambda response | **Mapping template (VTL)** |
| Different backend per environment | **Stage variables** |
| Developers need a new API version | **Create a dev stage** |
| API changes not visible | **`CreateDeployment`** |
| Reduce Lambda invocations on a read-heavy API | **API Gateway caching** |
| Call SQS or DynamoDB with no Lambda | AWS service integration |
| One event, several independent consumers | **SNS → SQS fan-out** |
| Route on fields inside the payload | **EventBridge** |
| Add consumers without changing the producer | **EventBridge** |
| Scheduled / cron trigger | **EventBridge schedule** |
| Replay past events | **EventBridge archive + replay** |
| React to an AWS service event | EventBridge default bus |
| Source → filter → enrich → target, no code | **EventBridge Pipes** |
| Orchestrate a multi-step workflow | **Step Functions** |
| Chain of Lambdas calling each other | Replace with Step Functions |
| Workflow longer than 5 minutes | Step Functions **Standard** |
| High-volume short workflows, cheapest | Step Functions **Express** |
| Wait for human approval in a workflow | `.waitForTaskToken` (Standard) |
| Wait for an ECS task or Batch job | `.sync` |
| Ordering, replay, many independent consumers | **Kinesis Data Streams** |
| Load a stream into S3/Redshift/OpenSearch, no code | **Firehose** |
| “Near real time” in the question | Firehose |
| One shard = one KCL worker | More instances don’t help |
| GraphQL / real-time subscriptions | **AppSync** |
| Full-text search | **OpenSearch** |
| Search over DynamoDB data | Streams → Lambda → OpenSearch |
| SQL on S3 | **Athena** |
| SQL on one object | **S3 Select** |

## Domain 2 — Security (26%)

| Situation | Answer |
| --- | --- |
| A service needs AWS permissions | IAM **role**, never stored keys |
| “Who can assume this role?” | **Trust policy** |
| “What can this identity do?” | Identity-based policy |
| Cross-account access to a resource | Resource-based policy |
| Cap maximum permissions | Permissions boundary |
| Restrict an entire account or OU | SCP |
| `Principal` element present | It’s a resource-based policy |
| One policy that works per user | Policy variables (`${aws:username}`) |
| **Explicit Deny always wins** | Look for a Deny before adding Allows |
| Billing permissions don’t work | Root must enable IAM access to Billing |
| Find public or cross-account access | IAM **Access Analyzer** |
| Test whether a policy allows an action | **Policy Simulator** |
| Find unused permissions | **Access Advisor** |
| Cross-account `AssumeRole` failing | Check **both** the trust policy and the caller’s policy |
| Third-party vendor access | **`ExternalId`** (confused deputy) |
| Longest STS session | 12 hours (1 hour with role chaining) |
| Which identity am I | `GetCallerIdentity` |
| Invalidate all existing sessions | Deny by `aws:TokenIssueTime` |
| User login / sign-up / JWT | **Cognito User Pool** |
| Secure API Gateway with a JWT, no code | **Cognito User Pool authorizer** |
| Custom auth logic / third-party IdP | **Lambda authorizer** |
| AWS users or services calling the API | **IAM authorizer** |
| Temporary AWS credentials for app users | **Cognito Identity Pool** |
| Mobile app uploads to S3, no keys embedded | **Identity Pool** (after a User Pool sign-in) |
| **User Pool = authentication. Identity Pool = AWS access.** | Both together for the full flow |
| Encrypt data larger than 4 KB | **Envelope encryption / ****`GenerateDataKey`** |
| KMS `Encrypt` fails on a big file | Use `GenerateDataKey`, **not** a quota increase |
| `AccessDenied` on KMS despite an IAM Allow | **Check the key policy** |
| Audit who decrypted an object | **SSE-KMS** + CloudTrail |
| Change keys without changing code | **Alias** |
| Encrypted data usable in another Region | Multi-Region key |
| Simple encryption at rest | **SSE-S3** (`AES256`) |
| Control and audit the key | **SSE-KMS** (`aws:kms`) |
| Customer supplies the key each request | **SSE-C** |
| Encrypt before upload | Client-side encryption |
| Require every upload be encrypted | **Bucket policy ****`Deny`** on the missing header |
| Database credentials + automatic rotation | **Secrets Manager** |
| Encrypted config, no rotation needed | **Parameter Store ****`SecureString`** |
| Feature flag / config change without redeploy | **AppConfig** |
| Validate config and roll back automatically | **AppConfig** |
| Passwordless database login | **IAM DB authentication** |
| Env vars must be unreadable by other developers | Customer managed KMS key + encryption helpers |
| Temporary access to a private S3 object | **Presigned URL** |
| Presigned URL expires too early | Signed with a role’s temporary credentials |
| Private content through CloudFront | **Signed URL / signed cookie** (key groups) |
| Many private files at once | Signed **cookie** |
| Restrict by IP range | CloudFront signed URL |
| Lock the S3 bucket to CloudFront only | **OAC** |
| Block SQL injection / XSS | **WAF** |
| Block a specific IP address | **NACL** (security groups cannot deny) |
| Inbound allowed but the response is blocked | **NACL outbound rules** (stateless) |
| Security group = stateful. NACL = stateless. |    |
| See why traffic is blocked | **VPC Flow Logs** (REJECT) |
| Private access to S3 or DynamoDB | **Gateway endpoint** (free) |
| Private access to other AWS services | **Interface endpoint** (PrivateLink) |
| `CannotPullContainerError` | Task **execution** role |
| Container app gets `AccessDenied` | Task **role** |
| Enable HTTPS / auto-renew certificates | **ACM** (us-east-1 for CloudFront) |

## Domain 3 — Deployment (24%)

| Situation | Answer |
| --- | --- |
| Serverless IaC (Lambda + API GW + DynamoDB) | **SAM** |
| `Transform: AWS::Serverless-2016-10-31` | It’s a **SAM** template |
| IaC in a programming language | **CDK** |
| Generate the CloudFormation template from CDK | `cdk synth` |
| See what would change before deploying | `cdk diff` / CloudFormation **change set** |
| Test one Lambda locally | `sam local invoke` |
| Test a REST API locally | `sam local start-api` |
| Upload artifacts to S3 | `sam package` |
| Only required CloudFormation section | `Resources` |
| Static lookup per Region | `Mappings` + **`Fn::FindInMap`** (3 arguments) |
| Deploy-time input | `Parameters` |
| Decide whether to create something | `Conditions` (Resources and Outputs only) |
| Get a resource’s primary identifier | `!Ref` |
| Get a specific attribute (ARN, DNS name) | `!GetAtt` |
| Use a value from another stack | `Outputs` + `Export` + `!ImportValue` |
| Export name conflict | Same name twice in the **same Region** |
| Same resources in many Regions/accounts | **StackSets** |
| Keep the database when the stack is deleted | `DeletionPolicy: Retain` |
| Resource changed outside CloudFormation | **Detect Stack Drift** |
| Automate the whole CI/CD workflow | **CodePipeline** |
| Pipeline artifact storage | **S3** |
| Human approval before production | **Manual approval** action + SNS |
| Compile, test, produce artifacts | **CodeBuild** |
| Install dependencies | `install` phase |
| Set the runtime version | `runtime-versions` in `install` |
| Log in to ECR | `pre_build` |
| Push image / upload artifacts | `post_build` |
| Run cleanup even on failure | `finally` |
| Pass a secret to a build | `env/secrets-manager` or `parameter-store` |
| Never put secrets in | `env/variables` |
| Slow builds redownloading dependencies | `cache` |
| Build must reach private RDS | Run CodeBuild in the **VPC** |
| Build times out | Default 60 min, max 8 h |
| Build Docker images | **Privileged mode** |
| Pass a value to the next stage | `exported-variables` |
| Run tests before Lambda traffic shifts | **`BeforeAllowTraffic`** |
| Verify after traffic is live | `AfterAllowTraffic` |
| Test listener before production (ECS) | `AfterAllowTestTraffic` |
| Last chance to fail an EC2 deployment | `ValidateService` |
| `ValidateService` on Lambda | **Does not exist** (distractor) |
| New deployments failing immediately | Broken **`ApplicationStop`** in the **previous** revision |
| 10% now, wait, then the rest | **Canary** |
| Equal increments on a timer | **Linear** |
| How Lambda traffic shifting works | **Alias weighted routing**, moved by CodeDeploy |
| Roll back automatically on errors | **CloudWatch alarm** + automatic rollback |
| CodeDeploy agent required | EC2 / on-premises only |
| Fastest Beanstalk deployment | All at once |
| Keep full capacity during a Beanstalk deploy | Rolling with additional batch |
| Safest, zero downtime, easy rollback | **Immutable** |
| Canary in Beanstalk | **Traffic splitting** |
| Fastest possible Beanstalk rollback | Blue/green **URL (CNAME) swap** |
| EC2 burst balances lost after a deploy | **Immutable / Traffic splitting** (new instances) |
| Beanstalk application versions stored in | **S3** |
| Customise a Beanstalk environment | **`.ebextensions`** |
| Database must survive a Beanstalk rebuild | Create **RDS externally** |
| Private npm / pip / Maven repository | **CodeArtifact** |
| Cache and proxy public packages | CodeArtifact external connection |

## Domain 4 — Troubleshooting and Optimization (18%)

| Symptom | Cause / answer |
| --- | --- |
| Duplicate SQS processing | Visibility timeout \< processing time |
| One bad message retries the whole batch | Enable **`ReportBatchItemFailures`** |
| Stream processing frozen forever | Poison pill; retry limits / **`BisectBatchOnFunctionError`** / on-failure destination |
| Rising `IteratorAge` | Consumer falling behind |
| **429 / ****`Throttles`** on Lambda | Concurrency limit reached |
| Lambda produces no logs | Execution role missing `logs:*` |
| Lambda in a VPC cannot reach the internet | **NAT gateway** (or VPC endpoint) |
| Memory or disk-space metric missing | Install the **CloudWatch agent** |
| Alarm when a string appears in logs | **Metric filter** + alarm |
| Metric filter shows nothing for old logs | Filters are **not retroactive** |
| Logs cost too much | Set log group **retention** |
| Alarm stuck in `INSUFFICIENT_DATA` | No data, or wrong “treat missing data” |
| `Latency` high, `IntegrationLatency` low | The problem is **API Gateway**, not the backend |
| `IntegrationLatency` high | The **backend** is slow |
| **429** from API Gateway | Throttling |
| **502** from API Gateway | Malformed Lambda proxy response |
| **504** from API Gateway | Integration timeout (29 s default) |
| **403** from API Gateway | Authorizer denied, WAF, or **unlinked API key** |
| New API key returns 403 | **`CreateUsagePlanKey`** |
| DynamoDB throttled while average usage is low | **Hot partition** / bad partition key |
| Filter traces by a business value | **Annotation** (indexed), not metadata |
| X-Ray works locally, fails on EC2 | **Missing IAM role** (`AWSXRayDaemonWriteAccess`) |
| Change tracing volume without redeploying | **Sampling rules** in the console |
| X-Ray daemon port | **UDP 2000** |
| Which downstream call is slow | Service map + subsegments |
| Scale workers on queue depth | `ApproximateNumberOfMessagesVisible` |
| Queue short but messages old | `ApproximateAgeOfOldestMessage` — consumers too slow |
| Unexplained S3 storage cost | Incomplete multipart uploads → abort lifecycle rule |
| S3 event notification never fires | Destination **resource policy** |
| Cache returns stale data | Add TTL, or use write-through |
| First request always slow | Lazy-loading cache-miss penalty |
| Cache full of unread data | Write-through churn |
| Reduce Athena cost | Parquet + partitioning + compression + fewer columns |
| Read-heavy RDS, least code change | **Read Replicas** |
| “Standby should serve reads” | **Wrong** — standbys serve nothing |
| Who changed this AWS resource | **CloudTrail** |
| Is the resource compliant | **AWS Config** |
| Does it match the template | **Drift Detection** |

---
