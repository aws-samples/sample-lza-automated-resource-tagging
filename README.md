# Automated Resource Tagging for Landing Zone Accelerator on AWS

> This sample adapts the [aws-samples/resource-tagging-automation](https://github.com/aws-samples/resource-tagging-automation) solution for deployment in [AWS Control Tower](https://aws.amazon.com/controltower/) environments using [Landing Zone Accelerator on AWS (LZA)](https://aws.amazon.com/solutions/implementations/landing-zone-accelerator-on-aws/). The core tagging logic (CloudTrail → EventBridge → Lambda) comes from the upstream project; this repo adds LZA-compatible packaging, per-account tag customization via LZA parameters, and governance context for multi-account landing zones.

## Table of Contents

- [Overview](#overview)
- [Tagging Governance on AWS — A Layered Approach](#tagging-governance-on-aws--a-layered-approach)
- [When to Use This Solution](#when-to-use-this-solution)
- [Architecture](#architecture)
- [Supported Resource Types](#supported-resource-types)
- [Auto-Tagged Metadata](#auto-tagged-metadata)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
  - [Option A: Per-Account Deployment (cloudFormationStacks)](#option-a-per-account-deployment-cloudformationstacks)
  - [Option B: Per-OU Deployment (cloudFormationStackSets)](#option-b-per-ou-deployment-cloudformationstacksets)
- [Onboarding New Accounts](#onboarding-new-accounts)
- [Parameters](#parameters)
- [LZA Compatibility Notes](#lza-compatibility-notes)
- [Security Considerations](#security-considerations)
- [Cost](#cost)
- [Troubleshooting](#troubleshooting)
- [Further Considerations](#further-considerations)
- [References](#references)
- [Contributing](#contributing)
- [License](#license)

## Overview

This sample adapts the [aws-samples/resource-tagging-automation](https://github.com/aws-samples/resource-tagging-automation) solution for multi-account AWS environments managed by AWS Control Tower and Landing Zone Accelerator on AWS (LZA). The upstream project provides the core automated tagging pipeline — CloudTrail, EventBridge, and Lambda — that applies predefined tags to newly created AWS resources. This repo wraps that pipeline in an LZA-compatible CloudFormation template and adds:

- **Per-account tag customization** through LZA's `cloudFormationStacks` configuration, enabling unique billing codes, cost centers, and department tags for each account.
- **Per-OU deployment** via `cloudFormationStackSets` for uniform tagging across an entire Organizational Unit.
- **CDK CfnInclude compatibility** so the template deploys cleanly through the LZA pipeline.
- **Governance context** — positioning automated tagging within the broader layered approach to tagging governance on AWS.

> **Important:** This solution is provided as a sample implementation. You must review, evaluate, assess, and approve the solution in compliance with your organization's particular security, tagging, and operational requirements.

> **Security Notice:** This is sample code for educational and demonstration purposes. Before deploying to production:
> - Conduct your own security review and threat modeling
> - Test thoroughly in non-production environments
> - Consult with your security and legal teams to meet organizational requirements
> - Understand that deploying this solution will incur AWS charges

## Tagging Governance on AWS — A Layered Approach

AWS recommends a layered approach to tagging governance, combining [proactive and reactive controls](https://docs.aws.amazon.com/tag-editor/latest/userguide/best-practices-and-strats.html) to ensure consistent tagging across your organization. Each layer addresses a different aspect of the problem:

| Layer | Mechanism | What It Does | Limitation |
|-------|-----------|-------------|------------|
| **Preventive** | [Service Control Policies (SCPs)](https://aws.amazon.com/blogs/mt/implement-aws-resource-tagging-strategy-using-aws-tag-policies-and-service-control-policies-scps/) | Deny resource creation if required tags are missing | Requires the *creator* to supply the correct tags. Must be written per-service/per-action. Can break automation and restore workflows (e.g., RDS snapshot restores) if not carefully scoped. |
| **Standardization** | [Tag Policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_tag-policies.html) | Enforce allowed tag keys, values, and casing across accounts | Reports non-compliance and can block non-compliant values, but does not deny creation of untagged resources on its own. |
| **Detective** | [AWS Config Rules](https://aws.amazon.com/blogs/mt/implementing-automated-and-centralized-tagging-controls-with-aws-config-and-aws-organizations/) | Find non-compliant resources after creation; trigger remediation | Reactive — resources exist in a non-compliant state until detected and remediated. Remediation still requires a Lambda or SSM Automation to apply tags. |
| **Automated** | [EventBridge + Lambda](https://github.com/aws-samples/resource-tagging-automation) | Automatically apply tags to resources at creation time, without requiring action from the creator. This sample adapts the linked upstream solution for LZA deployment. | Only tags resources created *after* deployment. Does not enforce that creators provide their own tags. |

**These layers are complementary, not mutually exclusive.** The right combination depends on your operating model:

- **SCPs + Tag Policies** work best when the resource creator is responsible for providing tags and you want to enforce discipline at creation time.
- **AWS Config** works best for compliance reporting, catching drift, and remediating resources that slip through preventive controls.
- **Automated tagging ([upstream solution](https://github.com/aws-samples/resource-tagging-automation), adapted here for LZA)** works best when the central platform team — not the resource creator — is responsible for ensuring tags exist, and you need tags applied consistently without relying on end-user behavior.

## When to Use This Solution

This sample is designed for a specific operating model: **a central platform team that provisions and manages AWS accounts for end users (agencies, business units, development teams) and needs to guarantee that cost allocation and governance tags are applied to all resources — without requiring action from or access to the end-user accounts.** The [upstream solution](https://github.com/aws-samples/resource-tagging-automation) provides the core tagging automation; this repo packages it for LZA-managed landing zones.

### Ideal Use Cases

- **Managed service providers or central IT teams** (e.g., a state IT agency providing AWS accounts to government agencies) that need to back-charge customers for their AWS spend. The central team controls the tag values (`AgencyCode`, `BillingCode`, `CostCenter`) and cannot rely on end users to apply them correctly.
- **Multi-account environments where end users should not be burdened with the platform team's tagging requirements.** SCPs that deny resource creation without tags create friction — end users don't know (and shouldn't need to know) the platform team's billing codes.
- **Environments where the platform team cannot or should not access end-user accounts.** This solution is deployed via LZA at account vending time and runs locally in each account — no cross-account access required.
- **Metadata tags that the creator cannot provide.** Tags like `CreatedBy` (extracted from CloudTrail identity) and `CreatedDate` cannot be supplied by the creator at resource creation time. This solution adds them automatically.

### When SCPs + AWS Config May Be a Better Fit

- The resource creators are your own infra team and you want to enforce tagging discipline — i.e., the creator *should* be responsible for providing the correct tags.
- You need to deny creation of resources that don't meet tagging requirements (hard enforcement).
- You want to standardize tag values and prevent incorrect values from being used.

### Recommended: Combine Both

For the strongest governance posture, layer this solution with other controls:

```
┌─────────────────────────────────────────────────────────────────┐
│  SCPs (optional)      Deny creation of resources missing tags   │
│  (AWS Organizations)  that end users are expected to provide.   │
│                       Use when creators share tagging burden.   │
├─────────────────────────────────────────────────────────────────┤
│  Tag Policies         Standardize tag keys, values, and casing  │
│  (AWS Organizations)  across all accounts. Non-disruptive.      │
├─────────────────────────────────────────────────────────────────┤
│  This Sample         Auto-apply cost allocation and governance │
│  (EventBridge+Lambda) tags at resource creation. Zero end-user  │
│  via upstream repo    burden. Adds CreatedBy/CreatedDate.       │
├─────────────────────────────────────────────────────────────────┤
│  AWS Config           Detect and report any resources that      │
│  (required-tags rule) slip through. Compliance dashboard.       │
└─────────────────────────────────────────────────────────────────┘
```

## Architecture

```
CloudTrail (captures API events)
    ↓
EventBridge (filters resource creation events)
    ↓
Lambda (applies tags via Resource Groups Tagging API)
```

Each target account gets its own CloudTrail → EventBridge → Lambda stack. Tags are passed as a CloudFormation parameter (`AutomationTags`), allowing each account to have a unique set of tags.

**How it works:**

1. A per-account CloudTrail trail captures API events and writes them to an encrypted S3 bucket.
2. An EventBridge rule filters for resource creation events (e.g., `RunInstances`, `CreateBucket`, `CreateDBInstance`).
3. A Lambda function extracts the resource ARN from the event, then applies the configured tags plus `CreatedBy` and `CreatedDate` metadata using the Resource Groups Tagging API.

## Supported Resource Types

The solution supports 30+ resource types across the following categories:

| Category | Services |
|----------|----------|
| Compute | EC2 (instances, volumes, EIPs, NAT/Internet gateways, VPC endpoints, transit gateways, security groups), Lambda, ECS, SageMaker (10 resource types) |
| Storage | S3, EBS, EFS |
| Database | RDS (instances and clusters), DynamoDB, ElastiCache, Redshift, Redshift Serverless, Neptune |
| Networking | ELB/ALB/NLB, API Gateway (REST and HTTP APIs) |
| Analytics | OpenSearch, MSK, Kinesis Data Streams, Glue (jobs, crawlers, databases) |
| Integration | SNS, SQS, Step Functions, EventBridge (custom event buses) |
| Security | KMS, Secrets Manager |
| Messaging | Amazon MQ |
| Monitoring | CloudWatch Alarms, CloudWatch Log Groups |
| Developer Tools | CodeBuild, CodePipeline |
| Containers | ECR |

## Auto-Tagged Metadata

In addition to your custom tags, every resource automatically receives:

- **CreatedBy** — The IAM principal (user, role, or federated identity) that created the resource, extracted from the CloudTrail event.
- **CreatedDate** — ISO 8601 UTC timestamp of when the resource was created.

## Prerequisites

- AWS Control Tower deployed with Landing Zone Accelerator (LZA) v1.5.0+
- Access to the LZA configuration repository (e.g., `aws-accelerator-config` in CodeCommit or your configured source)
- At least one workload account to deploy the solution into

## Deployment

### Step 1: Copy the template

Copy `resource-tagging-automation.yaml` into your LZA configuration repository:

```
aws-accelerator-config/
├── customizations/
│   └── resource-tagging-automation.yaml   ← copy here
└── customizations-config.yaml             ← update this
```

### Step 2: Configure deployment targets

Choose one of the following deployment options based on your tagging requirements.

### Option A: Per-Account Deployment (cloudFormationStacks)

Use this option when each account needs **unique tags** (e.g., different billing codes per account). Add entries to `customizations-config.yaml`:

```yaml
customizations:
  cloudFormationStacks:
    - deploymentTargets:
        accounts:
          - ProductionAccount           # UPDATE: your account name from accounts-config.yaml
      name: ResourceTaggingAutomation-Production
      description: Auto-tagging for Production
      regions:
        - us-east-1                     # UPDATE: your LZA home region
      runOrder: 1
      template: customizations/resource-tagging-automation.yaml
      terminationProtection: false
      parameters:
        - name: AutomationTags
          value: '{"BillingCode": "PROD-001", "CostCenter": "CC-PROD", "Department": "Engineering"}'
        - name: TrailName
          value: resource-tagging-automation-trail
    - deploymentTargets:
        accounts:
          - StagingAccount              # UPDATE: your account name from accounts-config.yaml
      name: ResourceTaggingAutomation-Staging
      description: Auto-tagging for Staging
      regions:
        - us-east-1                     # UPDATE: your LZA home region
      runOrder: 1
      template: customizations/resource-tagging-automation.yaml
      terminationProtection: false
      parameters:
        - name: AutomationTags
          value: '{"BillingCode": "STG-001", "CostCenter": "CC-STAGING", "Department": "QA"}'
        - name: TrailName
          value: resource-tagging-automation-trail
```

See [`sample-config/customizations-config.yaml`](sample-config/customizations-config.yaml) for a complete example.

### Option B: Per-OU Deployment (cloudFormationStackSets)

Use this option when all accounts in an OU should receive the **same tags**. Simpler configuration but no per-account customization:

```yaml
customizations:
  cloudFormationStackSets:
    - capabilities:
        - CAPABILITY_IAM
      deploymentTargets:
        organizationalUnits:
          - Workloads                   # UPDATE: your OU name from organization-config.yaml
      name: ResourceTaggingAutomation
      description: Auto-tagging for all workload accounts
      regions:
        - us-east-1                     # UPDATE: your LZA home region
      template: customizations/resource-tagging-automation.yaml
      parameters:
        - name: AutomationTags
          value: '{"BillingCode": "WL-001", "CostCenter": "CC-WORKLOADS", "Department": "Platform"}'
        - name: TrailName
          value: resource-tagging-automation-trail
```

See [`sample-config/customizations-config-stackset.yaml`](sample-config/customizations-config-stackset.yaml) for a complete example.

### Step 3: Push and trigger the pipeline

```bash
cd aws-accelerator-config
git add -A
git commit -m "Add resource tagging automation"
git push
aws codepipeline start-pipeline-execution \
  --name AWSAccelerator-Pipeline \
  --region <your-home-region>
```

> **Note:** The LZA pipeline may not auto-trigger on push depending on your configuration. Use the `start-pipeline-execution` command to trigger it manually.

## Onboarding New Accounts

To add tagging automation to a new account:

1. Add the account to `accounts-config.yaml` in your LZA configuration repository.
2. Add a new `cloudFormationStacks` entry (Option A) or ensure the account's OU is covered by a `cloudFormationStackSets` entry (Option B) in `customizations-config.yaml`.
3. Commit, push, and trigger the LZA pipeline.

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `AutomationTags` | `''` (empty) | JSON string of key-value pairs to apply as tags. Example: `{"BillingCode": "PROD-001", "CostCenter": "CC-PROD"}` |
| `LambdaAutoTaggingFunctionName` | `resource-tagging-automation-function` | Name of the Lambda function |
| `EventBridgeRuleName` | `resource-tagging-automation-rules` | Name of the EventBridge rule |
| `TrailName` | `resource-tagging-automation-trail` | Name of the CloudTrail trail |

## LZA Compatibility Notes

This CloudFormation template is designed to be compatible with CDK `CfnInclude`, which LZA uses internally to process custom templates:

- **No `Conditions` block** — CfnInclude has limited support for conditional resources.
- **No `DeletionPolicy: Retain`** — This can cause CDK assembly errors with CfnInclude.
- **Native YAML EventPattern** — The EventBridge rule uses YAML-native lists, not embedded JSON strings.
- **No nested intrinsic functions** — Avoids `!Join`/`!Split`/`!Select` combinations that may not synthesize correctly.

If you modify this template, validate that it still synthesizes through the LZA pipeline before deploying to production.

## Security Considerations

The template includes the following security controls:

- **KMS Encryption**: A single customer-managed KMS key (with automatic annual rotation) encrypts CloudTrail logs (at rest in S3 and in the trail itself), Lambda CloudWatch Logs, Lambda environment variables, and the dead-letter queue. The key policy is scoped to only the specific trail and log group using `aws:SourceArn` and `kms:EncryptionContext` conditions, and to the Lambda service via `kms:ViaService`.
- **S3 Bucket**: KMS encryption (via the shared CMK), public access block, versioning, HTTPS-only deny policy, 365-day lifecycle expiration. Bucket key enabled to reduce KMS API costs.
- **CloudTrail**: Log file validation enabled, multi-region trail, KMS-encrypted.
- **Lambda Logs**: Dedicated CloudWatch Log Group with KMS encryption and 90-day retention, created before the Lambda function to ensure logs are encrypted from the first invocation.
- **Lambda**: Reserved concurrent executions (10) to limit blast radius. Explicit handler dispatch map to prevent code injection. Environment variables encrypted with KMS. Dead-letter queue captures failed invocations.
- **Dead-Letter Queue**: KMS-encrypted SQS queue captures failed Lambda invocations for retry and monitoring. CloudWatch alarm triggers when messages arrive in the DLQ. Queue policy restricts SendMessage to the specific Lambda function ARN.
- **IAM**: Least-privilege role with `aws:PrincipalAccount` condition on all tagging permissions, scoping the role to only operate within the deploying account. DLQ access scoped to the specific queue ARN. Uses `AWSLambdaBasicExecutionRole` managed policy for CloudWatch Logs access.
- **Partition-aware ARNs**: All ARN references use `${AWS::Partition}` (CloudFormation) or runtime partition detection (Lambda) to support AWS, AWS GovCloud, and AWS China regions.

## Cost

You are responsible for the cost of the AWS services used while running this solution. The primary cost drivers per account are:

| AWS Service | Estimated Monthly Cost |
|-------------|----------------------|
| AWS CloudTrail (management events) | ~$2.00 |
| Amazon S3 (trail log storage) | < $1.00 |
| AWS KMS (encryption key + API calls) | ~$1.00 |
| AWS Lambda | < $1.00 (free tier eligible) |
| Amazon CloudWatch Logs | < $1.00 |
| Amazon SQS (dead-letter queue) | < $0.01 (free tier eligible) |
| Amazon CloudWatch Alarms | < $0.10 |
| Amazon EventBridge | No additional charge for AWS service events |
| **Total per account** | **~$3–6/month** |

Costs will vary based on the volume of resource creation events in each account. The estimates above assume a low-to-moderate workload.

**Scaling note:** For large deployments (50–200+ accounts), the primary cost driver is CloudTrail. Each account gets its own trail, and the trail is multi-region by default — meaning it writes digest files for every AWS region (~20+) every hour, even if those regions are unused. For a 100-account deployment, expect ~$300–600/month total. To reduce costs, you can set `IsMultiRegionTrail` to `false` in the template if you deploy the stack to every region in your landing zone, or if resources are only created in the deployed region. You can also reduce the S3 lifecycle `ExpirationInDays` from 365 to a shorter period (e.g., 90 days) since these logs exist to feed EventBridge, not for compliance archival.

## Troubleshooting

| Issue | Resolution |
|-------|-----------|
| Lambda not invoked | Verify the CloudTrail trail is logging (`IsLogging: true`). Check that the EventBridge rule is enabled and the Lambda permission grants `events.amazonaws.com` invoke access. |
| Tags not applied | Check Lambda CloudWatch Logs for errors. Verify the `AutomationTags` parameter is valid JSON. Ensure the IAM role has the required tagging permissions for the resource type. |
| CDK assembly errors | Ensure the template has no `Conditions` block, no `DeletionPolicy: Retain`, and no nested intrinsic functions. See [LZA Compatibility Notes](#lza-compatibility-notes). |
| Stack name conflicts | Each `cloudFormationStacks` entry must have a unique `name`. If redeploying, delete the existing stack first or use a different name. |
| Pipeline not triggered | The LZA pipeline may not auto-trigger on CodeCommit push. Run `aws codepipeline start-pipeline-execution --name AWSAccelerator-Pipeline --region <your-region>` to trigger manually. |

## Further Considerations

- **Existing resources**: This solution only tags resources created *after* deployment. To tag existing resources, consider using [AWS Resource Explorer](https://aws.amazon.com/resourceexplorer/) with [Tag Editor](https://docs.aws.amazon.com/ARG/latest/userguide/tag-editor.html).
- **Tag policies**: For tag standardization and compliance reporting, combine this solution with [AWS Organizations Tag Policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_tag-policies.html) configured in your LZA `organization-config.yaml`. Tag Policies enforce allowed values and casing without disrupting resource creation.
- **Multi-region**: The CloudTrail trail is configured as multi-region by default (`IsMultiRegionTrail: true`). This means a single-region deployment captures events from all regions, but the Lambda and EventBridge rule only exist in the deployed region — so only resources created in that region get tagged. For multi-region tagging, deploy the stack to each region in your landing zone. If you deploy to every region, consider setting `IsMultiRegionTrail` to `false` in the template to avoid redundant cross-region digest files, which reduces S3 write volume and cost. For landing zones restricted to a small number of regions (e.g., 4 US regions), the multi-region overhead is minimal.
- **Control Tower CloudTrail**: If AWS Control Tower is managing an organization-level CloudTrail trail, the per-account trail in this solution is additive. This is by design — the per-account trail feeds EventBridge in the target account. There is no conflict, but you will see two trails per account.
- **Dead-letter queue**: The Lambda function includes an SQS dead-letter queue (DLQ) with a CloudWatch alarm. Failed invocations are captured in the DLQ for investigation. Monitor the `resource-tagging-automation-function-dlq-alarm` alarm for failures.
- **AWS Security Hub**: If Security Hub is enabled, it may generate findings for the trail S3 bucket (missing access logging). This is by design — enabling access logging on a CloudTrail log bucket creates a circular dependency. The bucket is protected by KMS encryption, versioning, and HTTPS-only policy instead.

## References

- [Guidance for Tagging on AWS](https://aws.amazon.com/solutions/guidance/tagging-on-aws/)
- [Best Practices and Strategies — Tagging AWS Resources](https://docs.aws.amazon.com/tag-editor/latest/userguide/best-practices-and-strats.html)
- [Implement AWS Resource Tagging Strategy Using Tag Policies and SCPs](https://aws.amazon.com/blogs/mt/implement-aws-resource-tagging-strategy-using-aws-tag-policies-and-service-control-policies-scps/)
- [Implementing Automated and Centralized Tagging Controls with AWS Config and AWS Organizations](https://aws.amazon.com/blogs/mt/implementing-automated-and-centralized-tagging-controls-with-aws-config-and-aws-organizations/)
- [Tagging Governance Using AWS Organizations in the Public Sector](https://aws.amazon.com/blogs/publicsector/tagging-governance-using-aws-organizations-in-the-public-sector/)
- [Validate and Enforce Required Tags in CloudFormation, Terraform, and Pulumi with Tag Policies](https://aws.amazon.com/about-aws/whats-new/2025/11/validate-enforce-required-tags-cloudformation-terraform-pulumi/)
- [aws-samples/resource-tagging-automation](https://github.com/aws-samples/resource-tagging-automation) — Upstream project; this repo adapts its CloudTrail → EventBridge → Lambda pipeline for LZA
- [Landing Zone Accelerator on AWS — Implementation Guide](https://docs.aws.amazon.com/solutions/latest/landing-zone-accelerator-on-aws/solution-overview.html)
- [LZA on AWS — GitHub Repository](https://github.com/awslabs/landing-zone-accelerator-on-aws)

## Contributing

See [CONTRIBUTING](CONTRIBUTING.md) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
