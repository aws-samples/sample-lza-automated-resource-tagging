# Automated Resource Tagging for Landing Zone Accelerator on AWS

> This sample adapts the [aws-samples/resource-tagging-automation](https://github.com/aws-samples/resource-tagging-automation) solution for deployment in [AWS Control Tower](https://aws.amazon.com/controltower/) environments using [Landing Zone Accelerator on AWS (LZA)](https://aws.amazon.com/solutions/implementations/landing-zone-accelerator-on-aws/).

## Two Options

| | [Dynamic Account Tagging](#option-1-dynamic-account-tagging) | [Static Per-Account Tagging](#option-2-static-per-account-tagging) |
|---|---|---|
| Tag source | Control Tower account tags (read at runtime) | JSON parameter in YAML |
| Per-account YAML config | None — one StackSet for all OUs | One block per account |
| Tag changes | Edit in Control Tower | Edit YAML → push → pipeline |
| New account onboarding | Add to OU + set tags | Add YAML block → push → pipeline |
| Cross-account IAM role | Yes (read-only, org-scoped) | No |
| Best for | Many accounts, central tag management | Few accounts, tags not on the account |

### Option 1: Dynamic Account Tagging

Tags come from the account itself (set in Control Tower). The Lambda reads them at runtime. Zero tag values in YAML. One StackSet block covers all accounts.

Three steps to deploy:
1. Run one CLI command in the management account (one-time)
2. Copy one template into `customizations/`
3. Add one StackSet block to `customizations-config.yaml`, push

**→ [`dynamic-account-tagging/`](dynamic-account-tagging/)** — templates, sample config, and [deployment guide](dynamic-account-tagging/DEPLOYMENT-GUIDE.md)

### Option 2: Static Per-Account Tagging

Tags are passed as a JSON parameter (`AutomationTags`) in `customizations-config.yaml`. Each account gets its own config block with unique tag values. Supports per-account and per-OU deployment.

**→ [`resource-tagging-automation.yaml`](resource-tagging-automation.yaml)** — template  
**→ [`sample-config/`](sample-config/)** — per-account and per-OU sample configs  
**→ [`DEPLOYMENT-GUIDE.md`](DEPLOYMENT-GUIDE.md)** — step-by-step walkthrough

---

## Common to Both Options

### Supported Resource Types (30+)

| Category | Services |
|----------|----------|
| Compute | EC2 (instances, volumes, EIPs, NAT/Internet gateways, VPC endpoints, transit gateways, security groups), Lambda, ECS, SageMaker |
| Storage | S3, EBS, EFS |
| Database | RDS, DynamoDB, ElastiCache, Redshift, Redshift Serverless, Neptune |
| Networking | ELB/ALB/NLB, API Gateway |
| Analytics | OpenSearch, MSK, Kinesis, Glue |
| Integration | SNS, SQS, Step Functions, EventBridge |
| Security | KMS, Secrets Manager |
| Monitoring | CloudWatch Alarms, CloudWatch Log Groups |
| Developer Tools | CodeBuild, CodePipeline |
| Containers | ECR |
| Messaging | Amazon MQ |

### Auto-Tagged Metadata

Every resource automatically receives:
- **CreatedBy** — IAM principal that created the resource (from CloudTrail)
- **CreatedDate** — ISO 8601 UTC timestamp

### Retroactive Tagging

Both options support a one-time backfill of existing resources via the `EnableRetroactiveTagging` parameter.

### Cost

~$1–2 per account per region per month. No per-account CloudTrail trail — relies on the existing Control Tower organization trail.

### Prerequisites

- AWS Control Tower with LZA v1.5.0+
- Active Control Tower organization trail
- Access to the LZA config repo

### Security

- KMS encryption for Lambda logs, environment variables, and DLQ
- IAM least-privilege with `aws:PrincipalAccount` scoping
- Partition-aware ARNs (AWS, GovCloud, China)
- Dead-letter queue with CloudWatch alarm
- Dynamic option: cross-account role is read-only (`organizations:ListTagsForResource` only), scoped by Organization ID and role name

## References

- [Guidance for Tagging on AWS](https://aws.amazon.com/solutions/guidance/tagging-on-aws/)
- [Best Practices — Tagging AWS Resources](https://docs.aws.amazon.com/tag-editor/latest/userguide/best-practices-and-strats.html)
- [aws-samples/resource-tagging-automation](https://github.com/aws-samples/resource-tagging-automation) — Upstream solution
- [Landing Zone Accelerator on AWS](https://aws.amazon.com/solutions/implementations/landing-zone-accelerator-on-aws/)

## Contributing

See [CONTRIBUTING](CONTRIBUTING.md) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
