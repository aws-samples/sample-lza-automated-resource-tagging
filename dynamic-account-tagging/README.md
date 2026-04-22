# Dynamic Account Tagging

Automatically tags resources using the account's Control Tower tags. No tag values in YAML. No per-account configuration. One StackSet covers all accounts.

## Files

| File | What It Does |
|------|-------------|
| `dynamic-account-tagging.yaml` | CloudFormation template — deploy via LZA StackSet to workload OUs |
| `mgmt-account-tag-reader-role.yaml` | IAM role — deploy once to management account via CLI |
| `sample-customizations-config.yaml` | Sample LZA config |
| `DEPLOYMENT-GUIDE.md` | Step-by-step walkthrough |

## Three Steps

1. Deploy the management account role (one CLI command, one-time)
2. Copy `dynamic-account-tagging.yaml` into `aws-accelerator-config/customizations/`
3. Add the StackSet block to `customizations-config.yaml`, push

See [DEPLOYMENT-GUIDE.md](DEPLOYMENT-GUIDE.md) for the full walkthrough.

## How It Works

```
Resource created → EventBridge → Lambda → reads account tags from Organizations → applies to resource
```

Tags are cached in Lambda memory (5-min TTL). No per-account CloudTrail trail — uses the existing Control Tower organization trail.
