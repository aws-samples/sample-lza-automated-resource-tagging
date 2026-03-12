# Deployment Guide — Per-Account Setup and Testing

This guide walks through deploying the resource tagging automation to a single account using LZA's `cloudFormationStacks` approach, verifying it works, and then expanding to additional accounts.

## Prerequisites

- AWS Control Tower deployed with Landing Zone Accelerator (LZA) v1.5.0+
- Access to the LZA configuration repository (`aws-accelerator-config` in CodeCommit or your configured source)
- AWS CLI configured with management account credentials
- At least one workload account to deploy into
- `git-remote-codecommit` installed if using CodeCommit (`pip install git-remote-codecommit`)

## Step 1: Copy the Template

Copy `resource-tagging-automation.yaml` into your LZA configuration repository:

```
aws-accelerator-config/
├── customizations/
│   └── resource-tagging-automation.yaml    ← copy this file here
└── customizations-config.yaml              ← update in Step 2
```

```bash
cp resource-tagging-automation.yaml aws-accelerator-config/customizations/
```

## Step 2: Configure a Single Account

Edit `aws-accelerator-config/customizations-config.yaml` to target one account. Start with event-driven tagging only (retroactive disabled):

```yaml
customizations:
  cloudFormationStacks:
    - deploymentTargets:
        accounts:
          - MyWorkloadAccount          # ← your account name from accounts-config.yaml
      name: ResourceTaggingAutomation-MyWorkloadAccount
      description: Auto-tagging for MyWorkloadAccount
      regions:
        - us-west-2                    # ← your governed region(s)
      runOrder: 1
      template: customizations/resource-tagging-automation.yaml
      terminationProtection: false
      parameters:
        - name: AutomationTags
          value: '{"BillingCode": "PROD-001", "CostCenter": "CC-PROD", "Department": "Engineering"}'
```

> Replace `MyWorkloadAccount` with the account name exactly as it appears in your `accounts-config.yaml`. Replace the tag values with your own.

## Step 3: Push and Deploy

```bash
cd aws-accelerator-config
git add -A
git commit -m "Add resource tagging automation for single account"
git push origin main
```

Trigger the LZA pipeline:

```bash
aws codepipeline start-pipeline-execution \
  --name AWSAccelerator-Pipeline \
  --region us-west-2
```

Monitor the pipeline in the AWS Console (CodePipeline → AWSAccelerator-Pipeline). The `Customizations` stage deploys the CloudFormation stack to the target account. This typically takes 10–15 minutes.

## Step 4: Verify the Stack Deployed

Assume a role into the target account (or use whatever cross-account access you have):

```bash
# Example using Control Tower execution role
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/AWSControlTowerExecution \
  --role-session-name tagging-test \
  --query 'Credentials' --output json)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r .AccessKeyId)
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r .SecretAccessKey)
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r .SessionToken)
```

Check the stack exists:

```bash
aws cloudformation describe-stacks \
  --stack-name ResourceTaggingAutomation-MyWorkloadAccount \
  --region us-west-2 \
  --query 'Stacks[0].StackStatus'
```

Expected output: `"CREATE_COMPLETE"`

## Step 5: Test Event-Driven Tagging

Create a few test resources in the target account:

```bash
# Security group
aws ec2 create-security-group \
  --group-name test-tagging-sg \
  --description "Test auto-tagging" \
  --region us-west-2

# SQS queue
aws sqs create-queue \
  --queue-name test-tagging-queue \
  --region us-west-2

# SNS topic
aws sns create-topic \
  --name test-tagging-topic \
  --region us-west-2
```

Wait 30–60 seconds for EventBridge to deliver the events and Lambda to process them.

## Step 6: Verify Tags Were Applied

```bash
# Check security group tags
SG_ID=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=test-tagging-sg \
  --region us-west-2 \
  --query 'SecurityGroups[0].GroupId' --output text)

aws ec2 describe-tags \
  --filters "Name=resource-id,Values=$SG_ID" \
  --region us-west-2 \
  --output table

# Check SQS queue tags
QUEUE_URL=$(aws sqs get-queue-url \
  --queue-name test-tagging-queue \
  --region us-west-2 \
  --query 'QueueUrl' --output text)

aws sqs list-queue-tags \
  --queue-url $QUEUE_URL \
  --region us-west-2

# Check SNS topic tags
TOPIC_ARN=$(aws sns list-topics \
  --region us-west-2 \
  --query "Topics[?ends_with(TopicArn, ':test-tagging-topic')].TopicArn" \
  --output text)

aws sns list-tags-for-resource \
  --resource-arn $TOPIC_ARN \
  --region us-west-2
```

Each resource should have:
- Your custom tags (`BillingCode`, `CostCenter`, `Department`)
- `CreatedBy` — the IAM identity that created the resource
- `CreatedDate` — ISO 8601 timestamp

## Step 7: (Optional) Enable Retroactive Tagging

If the account has existing untagged resources, enable the one-time backfill by adding the retroactive parameters:

```yaml
      parameters:
        - name: AutomationTags
          value: '{"BillingCode": "PROD-001", "CostCenter": "CC-PROD", "Department": "Engineering"}'
        - name: EnableRetroactiveTagging
          value: 'true'
        - name: MarkerTagKey
          value: BillingCode
```

> The backfill only runs on stack Create. If the stack already exists, you need to delete it first and redeploy for the backfill to run. Before deleting, note that orphaned CloudWatch Log Groups (`/aws/lambda/resource-tagging-automation-function`) survive stack deletion — delete them manually before redeploying.

```bash
# Delete orphaned log groups (in the target account)
aws logs delete-log-group \
  --log-group-name /aws/lambda/resource-tagging-automation-function \
  --region us-west-2 2>/dev/null

aws logs delete-log-group \
  --log-group-name /aws/lambda/resource-tagging-automation-function-backfill \
  --region us-west-2 2>/dev/null
```

After redeploying, check the stack outputs for backfill results:

```bash
aws cloudformation describe-stacks \
  --stack-name ResourceTaggingAutomation-MyWorkloadAccount \
  --region us-west-2 \
  --query 'Stacks[0].Outputs' \
  --output table
```

Look for:
- `BackfillStatus` — "Retroactive tagging complete"
- `BackfillResourcesTagged` — number of resources tagged
- `BackfillResourcesSkipped` — number already tagged (had the MarkerTagKey)

## Step 8: Clean Up Test Resources

```bash
# Delete test security group
aws ec2 delete-security-group --group-id $SG_ID --region us-west-2

# Delete test SQS queue
aws sqs delete-queue --queue-url $QUEUE_URL --region us-west-2

# Delete test SNS topic
aws sns delete-topic --topic-arn $TOPIC_ARN --region us-west-2
```

## Expanding to Additional Accounts

Once you've validated tagging works for one account, add more entries to `customizations-config.yaml`:

```yaml
customizations:
  cloudFormationStacks:
    # Account 1 (already deployed)
    - deploymentTargets:
        accounts:
          - MyWorkloadAccount
      name: ResourceTaggingAutomation-MyWorkloadAccount
      # ... (existing config)

    # Account 2
    - deploymentTargets:
        accounts:
          - AnotherAccount
      name: ResourceTaggingAutomation-AnotherAccount
      description: Auto-tagging for AnotherAccount
      regions:
        - us-west-2
      runOrder: 1
      template: customizations/resource-tagging-automation.yaml
      terminationProtection: false
      parameters:
        - name: AutomationTags
          value: '{"BillingCode": "ACCT2-001", "CostCenter": "CC-ACCT2", "Department": "Finance"}'
```

Each account can have different tag values. Push, commit, and trigger the pipeline — LZA handles the rest.

For uniform tags across an entire OU, consider the StackSet approach instead. See `sample-config/customizations-config-stackset.yaml` and the main [README](README.md#option-b-per-ou-deployment-cloudformationstacksets).

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Stack fails to create | Check the CloudFormation events tab in the target account. Common cause: orphaned log group from a previous deployment. Delete it and retry. |
| Lambda not invoked after creating resources | Verify the Control Tower organization trail is active: `aws cloudtrail get-trail-status --name aws-controltower-BaselineCloudTrail --region <home-region>`. Must show `IsLogging: true`. |
| Tags missing on a resource | Check Lambda logs in CloudWatch (`/aws/lambda/resource-tagging-automation-function`). The resource type may not be in the supported list, or the IAM role may lack permissions. |
| Retroactive backfill shows 0 tagged | Backfill only runs on stack Create. If the stack was updated (not created), the backfill won't re-run. Delete the stack, clean up log groups, and redeploy. |
| Pipeline not triggered on push | Run `aws codepipeline start-pipeline-execution --name AWSAccelerator-Pipeline --region <region>` to trigger manually. |
| KMS error on backfill log group | The KMS key policy needs a wildcard pattern for the log group ARN. Ensure the `AllowCloudWatchLogsEncrypt` condition uses `/aws/lambda/${LambdaAutoTaggingFunctionName}*` (trailing `*`). |
