# CloudWatch Alarms Multi-Account Inventory

## Overview

This project provides an AWS Lambda function that inventories CloudWatch alarms across multiple AWS accounts and regions, exporting the results to a formatted Excel report stored in Amazon S3.

It is designed for Cloud Operations and Platform teams that need centralized visibility, auditing, and documentation of monitoring configurations in multi-account AWS environments.

---

## Why This Exists

Managing CloudWatch alarms at scale quickly becomes opaque in multi-account setups. This solution enables:

- **Centralized monitoring visibility** across an AWS Organization
- **Audit-ready alarm inventory** for security and compliance teams
- **Operational insight** into alarm coverage, actions, and configuration consistency

---

## Key Features

✅ Cross-account alarm collection via STS AssumeRole  
✅ Multi-region support  
✅ Automatic discovery and inclusion of resource tags  
✅ Excel export with one worksheet per AWS account  
✅ Intelligent resource ID extraction from alarm dimensions  
✅ Human-readable parsing of alarm actions  
✅ Resilient execution with granular error handling

---

## Architecture

```
Lambda Function (Central Account)
        ↓
Assume Role → Target Accounts (CrossAccountCloudWatchReadRole)
        ↓
Query CloudWatch Alarms (us-east-1, us-west-2)
        ↓
Extract: Metrics, Thresholds, Actions, Tags
        ↓
Generate Excel Workbook
        ↓
Upload to S3 (cloudwatch-inventory-reports)
```

---

## Prerequisites

### 1. Cross-Account IAM Role (Target Accounts)

Create the following role in each target account.

**Role name:** `CrossAccountCloudWatchReadRole`

#### Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR-LAMBDA-ACCOUNT-ID:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Permissions

Attach the included `iam-policy.json` with read-only permissions for CloudWatch, EC2, RDS, Logs, and SNS.

### 2. Lambda Configuration

- **Runtime:** Python 3.11+
- **Memory:** 512 MB
- **Timeout:** 5 minutes
- **Dependencies:** `openpyxl` (Lambda Layer or packaged with deployment)

### 3. Lambda Execution Role Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::*:role/CrossAccountCloudWatchReadRole"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::cloudwatch-inventory-reports/*"
    }
  ]
}
```

---

## Deployment

### 1. Package Dependencies

```bash
pip install openpyxl -t package/
cp lambda_function.py package/
cd package
zip -r ../lambda_deployment.zip .
```

### 2. Deploy the Lambda Function

```bash
aws lambda create-function \
  --function-name cloudwatch-alarms-inventory \
  --runtime python3.11 \
  --role arn:aws:iam::YOUR-ACCOUNT:role/LambdaExecutionRole \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://lambda_deployment.zip \
  --timeout 300 \
  --memory-size 512
```

### 3. (Optional) Scheduled Execution

**Example:** Weekly execution using EventBridge

```bash
aws events put-rule \
  --name cloudwatch-inventory-weekly \
  --schedule-expression "cron(0 9 ? * MON *)"

aws events put-targets \
  --rule cloudwatch-inventory-weekly \
  --targets "Id"="1","Arn"="arn:aws:lambda:REGION:ACCOUNT:function:cloudwatch-alarms-inventory"
```

---

## Configuration

Update the following variables in `lambda_function.py`:

```python
ROLE_NAME = "CrossAccountCloudWatchReadRole"

REGIONS = ['us-east-1', 'us-west-2']

account_ids = {
    '111111111111': 'production-account',
    '222222222222': 'staging-account',
    # Add additional accounts as needed
}

S3_BUCKET = 'cloudwatch-inventory-reports'
S3_KEY = 'cloudwatch-alarms-inventory.xlsx'
```

---

## Output

The generated Excel workbook includes:

- **One worksheet per AWS account**
- **Standardized columns:**
  - Account, Region, Service, AlarmName, MetricName, Statistic
  - Period, Threshold, Datapoints, Actions
  - ResourceId, Description, State
- **Dynamic columns** for all discovered resource tags
- **Styled headers**, borders, and auto-sized columns

---

## Example Use Cases

- **Compliance Audits** – Validate alarm coverage across environments
- **Incident Response** – Quickly identify alarm behavior during outages
- **Cost Optimization** – Detect unused or redundant alarms
- **Operational Documentation** – Maintain up-to-date monitoring inventories

---

## Error Handling Behavior

- ✅ Continues execution if individual alarms fail
- ✅ Logs account-level and region-level errors
- ✅ Gracefully handles missing tags and optional fields
- ✅ Automatically skips AWS-managed alarms (DO NOT EDIT OR DELETE)

---

## Cost Estimate

Assuming ~500 alarms across 5 accounts and 2 regions:

- **Lambda execution:** ~$0.01 per run
- **S3 storage:** ~$0.01/month
- **CloudWatch API calls:** Minimal (typically within free tier)

**Estimated total:** Less than $1/month for weekly execution.

---

## Security Considerations

- ✅ Enforces least-privilege IAM policies
- ✅ Supports CloudTrail auditing
- ✅ S3 bucket should be encrypted (SSE-S3 or SSE-KMS)
- ✅ Cross-account role trust limited to the Lambda execution role

---

## Note

> **Important:** Replace placeholder account IDs, role names, and bucket names before deployment.