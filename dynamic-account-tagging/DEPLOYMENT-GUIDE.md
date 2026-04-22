# Deployment Guide — Dynamic Account Tagging

A step-by-step guide to deploy the dynamic account tagging solution. Tags are managed in Control Tower and automatically applied to every resource created in your accounts.

## What This Does

When someone creates a resource (EC2 instance, S3 bucket, database, etc.) in any of your AWS accounts, this solution automatically applies the tags you've set on that account in Control Tower. You manage tags in one place, and they flow to every resource.

## Before You Start

You'll need:

- AWS Control Tower with Landing Zone Accelerator (LZA) v1.5.0+
- Access to the LZA configuration repository (usually in CodeCommit)
- Your AWS Organization ID (looks like `o-xxxxxxxxxx`)
- Your management account ID (12-digit number)
- The names of the Organizational Units (OUs) you want to enable tagging for

## Step 1: Set Tags on Your Accounts

1. Sign in to the **AWS Management Console** with your management account
2. Go to **AWS Organizations**
3. Click **Accounts** in the left sidebar
4. Click on the account you want to tag → **Tags** tab → **Manage tags**
5. Add your tags:

   | Key | Value |
   |-----|-------|
   | BillingCode | AGENCY-001 |
   | CostCenter | CC-FINANCE |
   | Department | Finance |

6. Click **Save changes**
7. Repeat for each account

## Step 2: Deploy the Tag Reader Role (One-Time)

This creates a read-only role in your management account. Run once, never again.

1. Open **CloudShell** in your management account (terminal icon in the top nav bar)
2. Upload `mgmt-account-tag-reader-role.yaml` (Actions → Upload file)
3. Run:

```bash
aws cloudformation deploy \
  --template-file mgmt-account-tag-reader-role.yaml \
  --stack-name AccountTagReaderRole \
  --parameter-overrides OrganizationId=o-xxxxxxxxxx \
  --capabilities CAPABILITY_NAMED_IAM \
  --region <your-home-region>
```

Replace `o-xxxxxxxxxx` with your Organization ID and `<your-home-region>` with your LZA home region (e.g., `us-east-1`).

Wait for: `Successfully created/updated stack - AccountTagReaderRole`

### Finding your Organization ID

Go to **AWS Organizations** in the console — the ID is shown at the top of the page.

## Step 3: Add to LZA

### 3a. Copy the template

Copy `dynamic-account-tagging.yaml` into your LZA config repo:

```
aws-accelerator-config/
├── customizations/
│   └── dynamic-account-tagging.yaml    ← put it here
└── customizations-config.yaml          ← update this
```

### 3b. Update customizations-config.yaml

Add this to your `customizations-config.yaml`:

```yaml
customizations:
  cloudFormationStackSets:
    - capabilities:
        - CAPABILITY_NAMED_IAM
      deploymentTargets:
        organizationalUnits:
          - <your-ou-name>              # List all OUs to enable tagging for
      name: DynamicAccountTagging
      description: Auto-tags resources using Control Tower account tags
      regions:
        - <your-home-region>            # List all governed regions
      template: customizations/dynamic-account-tagging.yaml
      parameters:
        - name: ManagementAccountId
          value: '<your-management-account-id>'
```

See [`sample-customizations-config.yaml`](sample-customizations-config.yaml) for a complete example.

### 3c. Push and run the pipeline

```bash
cd aws-accelerator-config
git add -A
git commit -m "Add dynamic account tagging"
git push
```

The pipeline takes about 15–30 minutes.

## Step 4: Verify

1. Sign in to a workload account
2. Create any resource (e.g., an SQS queue)
3. Wait ~60 seconds
4. Check the resource's tags — you should see your Control Tower account tags plus `CreatedBy` and `CreatedDate`

## Day-to-Day Operations

### Changing tags

Update the account's tags in AWS Organizations. New resources pick up changes within 5 minutes. No LZA changes needed.

### Adding new accounts

Create the account in Control Tower, place it in a covered OU, set its tags. The tagging automation deploys automatically.

### Adding new OUs

Add the OU name to the `organizationalUnits` list in `customizations-config.yaml`, push, and run the pipeline.

### Tagging existing resources

Add these parameters to your StackSet config:

```yaml
        - name: EnableRetroactiveTagging
          value: 'true'
        - name: MarkerTagKey
          value: BillingCode
```

This runs a one-time backfill on stack creation. Resources that already have the `MarkerTagKey` are skipped.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Tags don't appear | Check that the account has tags in AWS Organizations |
| Pipeline fails | Verify the template is in `customizations/` and the YAML is valid |
| "No account tags found" in logs | Add tags to the account in AWS Organizations |
| Stale tags after a change | Tags are cached for 5 minutes — wait and retry |

## What Gets Deployed

**Each workload account:** Lambda, EventBridge rule, KMS key, SQS dead-letter queue, CloudWatch alarm, IAM role

**Management account (one-time):** Read-only IAM role (`account-tag-reader-role`)

## Cost

~$1–2 per account per region per month.
