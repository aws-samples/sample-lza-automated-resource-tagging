# Security Scanner Findings — Rationale for Accepted Warnings

This document provides rationale for security scanner warnings that have been reviewed and accepted as part of the PCSR process. All findings have been reviewed. Remediable items have been fixed; the remaining warnings are by design or false positives.

## Scanner: Checkov

### CKV_AWS_116 — Lambda DLQ

**Status:** Remediated

Added SQS dead-letter queue with KMS encryption, queue policy scoped to the Lambda function ARN, and CloudWatch alarm on message count.

### CKV_AWS_117 — Lambda inside VPC

**Status:** Accepted — By Design

This Lambda function only calls AWS service APIs (Resource Groups Tagging API, EC2 CreateTags, etc.) over HTTPS. It does not access any VPC resources (databases, internal services, etc.). Deploying it inside a VPC would:
- Require a NAT Gateway (~$32/month per AZ) for no security benefit
- Increase cold start latency
- Add operational complexity (subnet management, ENI limits)

The function's blast radius is limited by least-privilege IAM (tagging-only permissions) and reserved concurrency (10).

### CKV_AWS_173 — Lambda environment variable encryption

**Status:** Remediated

Added `KmsKeyArn` property on the Lambda function pointing to the shared KMS encryption key. Environment variables are now encrypted at rest with the customer-managed CMK.

### CKV_AWS_18 — S3 access logging

**Status:** Accepted — By Design

This S3 bucket stores CloudTrail logs. Enabling S3 access logging on a CloudTrail log bucket creates a circular logging dependency. The bucket is protected by:
- KMS encryption (customer-managed key with auto-rotation)
- S3 bucket key enabled
- Public access block (all four settings)
- Versioning enabled
- HTTPS-only bucket policy (denies `aws:SecureTransport: false`)
- Bucket policy restricts writes to CloudTrail service with `aws:SourceArn` condition
- 365-day lifecycle expiration

## Scanner: cfn-guard

### CFN_NO_EXPLICIT_RESOURCE_NAMES — CloudWatch Alarm has explicit name

**Status:** Accepted — By Design

The `AlarmName` is derived from a CloudFormation parameter via `!Sub "${LambdaAutoTaggingFunctionName}-dlq-alarm"`, not a hardcoded literal. This makes the alarm name predictable and identifiable across accounts, which is important for operational monitoring in a multi-account LZA deployment. Stack updates are not impacted because the name is parameter-driven.

### CLOUD_TRAIL_CLOUD_WATCH_LOGS_ENABLED — CloudTrail missing CloudWatch Logs export

**Status:** Accepted — By Design

This trail exists solely to feed EventBridge for real-time resource tagging. CloudWatch Logs export is not needed because:
- The trail's purpose is event delivery to EventBridge, not log analysis
- CloudTrail logs are already stored in an encrypted S3 bucket with log file validation
- Adding CloudWatch Logs export would require an additional IAM role (CloudTrail → CloudWatch Logs delivery) and increase cost
- If the deploying organization requires CloudTrail → CloudWatch Logs integration, it should be configured on the organization-level trail managed by AWS Control Tower, not on this per-account automation trail

### CLOUD_TRAIL_ENCRYPTION_ENABLED — CloudTrail encryption not detected

**Status:** Accepted — False Positive

The trail has `KMSKeyId: !GetAtt EncryptionKey.Arn` which provides KMS encryption. cfn-guard cannot resolve CloudFormation intrinsic functions (`!GetAtt`) and reports the value as "not string." The trail is encrypted with a customer-managed KMS key that has automatic annual rotation.

### CLOUDTRAIL_S3_DATAEVENTS_ENABLED — CloudTrail missing S3 data event selectors

**Status:** Accepted — By Design

This trail captures management events (API calls like `RunInstances`, `CreateBucket`, etc.) to trigger resource tagging. S3 data events (object-level operations like `GetObject`, `PutObject`) are not needed for this use case and would:
- Significantly increase CloudTrail costs (data events are billed per 100,000 events)
- Generate high event volume with no benefit to the tagging automation
- Potentially cause Lambda throttling from excessive invocations

### CLOUDWATCH_ALARM_ACTION_CHECK — CloudWatch Alarm missing actions

**Status:** Accepted — By Design

The DLQ alarm is created without `AlarmActions`, `OKActions`, or `InsufficientDataActions` because the notification target (e.g., SNS topic) is deployment-specific. Deployers should configure alarm actions based on their organization's notification preferences. The alarm still provides value by:
- Surfacing in the CloudWatch Alarms console for manual monitoring
- Being queryable via CloudWatch APIs and CLI
- Integrating with AWS Security Hub and other monitoring tools that poll alarm state

### IAM_NO_INLINE_POLICY_CHECK — IAM role uses inline policy

**Status:** Accepted — By Design

The inline policy is appropriate here because it references stack-specific resources using CloudFormation intrinsic functions (`!GetAtt DeadLetterQueue.Arn`, `!Ref AWS::AccountId`). A managed policy cannot reference resources within the same stack. The inline policy follows least-privilege with explicit action allowlists and `aws:PrincipalAccount` conditions on all statements.

### IAM_POLICYDOCUMENT_NO_WILDCARD_RESOURCE — IAM wildcard resource

**Status:** Accepted — By Design

Tagging APIs (`tag:TagResources`, `ec2:CreateTags`, service-specific tag actions) require `Resource: '*'` because:
- Resources are tagged at creation time, before the ARN is known to the policy
- The Resource Groups Tagging API operates across all resource types
- AWS does not support resource-level permissions for most tagging actions

Mitigated by:
- `aws:PrincipalAccount` condition on all statements (scopes to deploying account only)
- Explicit action allowlists (no wildcard actions)
- Reserved concurrent executions limiting parallel execution

### LAMBDA_INSIDE_VPC — Lambda not in VPC

**Status:** Accepted — By Design

Same rationale as Checkov CKV_AWS_117 above.

### S3_BUCKET_DEFAULT_LOCK_ENABLED — S3 Object Lock not enabled

**Status:** Accepted — By Design

S3 Object Lock provides WORM (Write Once Read Many) compliance, which is not required for this use case. CloudTrail log integrity is ensured by:
- CloudTrail log file validation (digest files detect tampering)
- KMS encryption at rest
- Bucket policy restricting writes to CloudTrail service only
- Versioning enabled

Object Lock would prevent lifecycle expiration from cleaning up old logs and add operational complexity without meaningful security benefit for a tagging automation trail.

### S3_BUCKET_LOGGING_ENABLED — S3 access logging not enabled

**Status:** Accepted — By Design

Same rationale as Checkov CKV_AWS_18 above. Enabling access logging on a CloudTrail log bucket creates a circular logging dependency.

### S3_BUCKET_NO_PUBLIC_RW_ACL — S3 missing AccessControl property

**Status:** Accepted — False Positive

The bucket uses `PublicAccessBlockConfiguration` with all four settings enabled (`BlockPublicAcls`, `BlockPublicPolicy`, `IgnorePublicAcls`, `RestrictPublicBuckets`), which is the modern and recommended approach. The legacy `AccessControl` property is not needed when `PublicAccessBlockConfiguration` is set. AWS recommends using `PublicAccessBlockConfiguration` over ACLs.

### S3_BUCKET_REPLICATION_ENABLED — S3 cross-region replication not enabled

**Status:** Accepted — By Design

Cross-region replication is not needed for a per-account tagging automation trail bucket because:
- The logs are operational data for the tagging automation, not compliance archives
- Logs expire after 365 days via lifecycle policy
- Replication would double storage costs across all deployed accounts
- If cross-region durability is required, it should be configured on the organization-level CloudTrail managed by AWS Control Tower

### S3_BUCKET_SSL_REQUESTS_ONLY — S3 TLS enforcement not detected

**Status:** Accepted — False Positive

The bucket policy includes a `DenyUnencryptedTransport` statement that denies all `s3:*` actions when `aws:SecureTransport` is `false`. cfn-guard cannot fully parse the bucket policy's condition structure and reports this as non-compliant. The HTTPS-only enforcement is implemented and functional.
