---
title : "Delete Resources"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.7.1 </b> "
---

## Step-by-Step Resource Deletion Guide

This section walks you through the complete cleanup process for the Jobs Matching Platform. Follow the steps **in order**, as some resources depend on others being deleted first.

### Deletion Order

The recommended deletion order ensures dependencies are resolved properly:

```
1. Amplify (Frontend Hosting)
2. WAF Web ACL
3. CloudWatch Alarms & Dashboard
4. AWS Budgets
5. SSM Parameters
6. S3 Bucket (empty first, then delete)
7. SAM Stack (deletes Lambda, API Gateway, DynamoDB, SQS, Cognito, EventBridge, IAM)
8. CloudWatch Log Groups (cleanup residuals)
9. SAM Deployment Bucket
```

---

### Step 1: Delete Amplify Frontend Application

1. Navigate to **AWS Console → AWS Amplify**
2. Select the Jobs Matching Platform application
3. Click **Actions → Delete app**

![Delete Amplify](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-amplify.png)

4. Type **delete** to confirm

![Confirm Delete Amplify](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/confirm-delete-amplify.png)

---

### Step 2: Delete WAF Web ACL

1. Navigate to **AWS Console → WAF & Shield → Web ACLs**
2. Select the Web ACL associated with the API Gateway
3. First, **disassociate** it from the API Gateway resource
4. Then click **Delete**

![Delete WAF](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-waf.png)

{{% notice info %}}
You must disassociate the Web ACL from all resources before deleting it. If the deletion fails, check for remaining associations.
{{% /notice %}}

---

### Step 3: Delete CloudWatch Alarms & Dashboard

#### Delete Alarms

1. Navigate to **AWS Console → CloudWatch → Alarms**
2. Select all alarms created during the workshop (e.g., `jobs-matching-*`)
3. Click **Actions → Delete**

![Delete Alarms](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-alarms.png)

Or use the AWS CLI:

```bash
# Delete all alarms matching the project prefix
aws cloudwatch delete-alarms \
  --alarm-names \
    "jobs-matching-evaluate-match-errors" \
    "jobs-matching-dlq-messages" \
    "jobs-matching-api-5xx-errors"
```

#### Delete Dashboard (if created)

```bash
aws cloudwatch delete-dashboards \
  --dashboard-names "jobs-matching-platform-dev"
```

---

### Step 4: Delete AWS Budgets

1. Navigate to **AWS Console → Billing → Budgets**
2. Select `jobs-matching-platform-monthly`
3. Click **Actions → Delete**

![Delete Budget](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-budget.png)

Or use the CLI:

```bash
aws budgets delete-budget \
  --account-id <your-account-id> \
  --budget-name "jobs-matching-platform-monthly"
```

---

### Step 5: Delete SSM Parameters

Delete the API keys stored in Parameter Store:

```bash
# Delete SerpAPI key
aws ssm delete-parameter \
  --name "/jobs-matching/dev/serpapi-key"

# Delete Gemini API key
aws ssm delete-parameter \
  --name "/jobs-matching/dev/gemini-api-key"
```

Or via the console:

1. Navigate to **AWS Console → Systems Manager → Parameter Store**
2. Select parameters matching `/jobs-matching/dev/*`
3. Click **Delete**

![Delete SSM](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-ssm.png)

---

### Step 6: Empty and Delete S3 Bucket

{{% notice warning %}}
S3 buckets must be **emptied before they can be deleted**. CloudFormation/SAM cannot delete a non-empty bucket.
{{% /notice %}}

#### Option A: Empty via AWS Console

1. Navigate to **AWS Console → S3**
2. Select the CV bucket (`jobs-matching-cvs-dev-<account-id>`)
3. Click **Empty**
4. Type **permanently delete** to confirm

![Empty S3](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/empty-s3.png)

#### Option B: Empty via AWS CLI

```bash
# Empty the CV bucket
aws s3 rm s3://jobs-matching-cvs-dev-<account-id> --recursive

# Verify the bucket is empty
aws s3 ls s3://jobs-matching-cvs-dev-<account-id>
```

![CLI Empty S3](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/cli-empty-s3.png)

{{% notice info %}}
Do NOT manually delete the S3 bucket after emptying it — the SAM delete command in the next step will handle the bucket deletion as part of the CloudFormation stack cleanup.
{{% /notice %}}

---

### Step 7: Delete SAM Stack (Main Cleanup)

This is the most important step. The `sam delete` command removes the **entire CloudFormation stack**, which includes:

- ✅ All Lambda functions
- ✅ API Gateway (HTTP API)
- ✅ DynamoDB tables (Jobs, SavedJobs, MatchResults)
- ✅ SQS queues (RawJobsQueue, RawJobsDLQ)
- ✅ EventBridge Scheduler
- ✅ Cognito User Pool, Client, Identity Provider
- ✅ S3 bucket (if empty)
- ✅ IAM execution roles for all Lambda functions

#### Run SAM Delete

```bash
cd backend

# Delete the SAM stack
sam delete --stack-name jobs-matching --no-prompts
```

![SAM Delete](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/sam-delete.png)

The command will:
1. Delete the CloudFormation stack and all its resources
2. Delete the SAM deployment S3 bucket (if `--no-prompts` is used)

#### Expected Output

```
	Deleting the local artifacts and cache...
	Are you sure you want to delete the stack jobs-matching in the region ap-southeast-1 ? [y/N]: 
	Are you sure you want to delete the folder jobs-matching in S3 which contains the artifacts? [y/N]: 
	- Deleting S3 object with key jobs-matching/abc123
	- Deleting S3 object with key jobs-matching/def456
	- Deleting Cloudformation stack jobs-matching

Successfully deleted stack: jobs-matching
```

![SAM Delete Success](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/sam-delete-success.png)

{{% notice warning %}}
If `sam delete` fails with an error about a non-empty S3 bucket, go back to **Step 6** and ensure the CV bucket is completely empty. Then retry.
{{% /notice %}}

#### Verify Stack Deletion

```bash
# Verify the stack no longer exists
aws cloudformation describe-stacks \
  --stack-name jobs-matching 2>&1

# Expected output: "Stack with id jobs-matching does not exist"
```

---

### Step 8: Delete CloudWatch Log Groups (Residual Cleanup)

CloudWatch Log groups are **not deleted** when Lambda functions are removed. They must be deleted separately:

```bash
# List all log groups for the project
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/lambda/jobs-matching" \
  --query "logGroups[].logGroupName" \
  --output text

# Delete each log group
aws logs delete-log-group --log-group-name "/aws/lambda/jobs-matching-fetch-jobs-dev"
aws logs delete-log-group --log-group-name "/aws/lambda/jobs-matching-get-jobs-dev"
aws logs delete-log-group --log-group-name "/aws/lambda/jobs-matching-get-job-by-id-dev"
aws logs delete-log-group --log-group-name "/aws/lambda/jobs-matching-search-jobs-dev"
aws logs delete-log-group --log-group-name "/aws/lambda/jobs-matching-saved-jobs-dev"
aws logs delete-log-group --log-group-name "/aws/lambda/jobs-matching-evaluate-match-dev"
aws logs delete-log-group --log-group-name "/aws/lambda/jobs-matching-get-match-results-dev"
aws logs delete-log-group --log-group-name "/aws/lambda/jobs-matching-extract-cv-dev"
aws logs delete-log-group --log-group-name "/aws/lambda/jobs-matching-manage-cv-dev"
aws logs delete-log-group --log-group-name "/aws/lambda/jobs-matching-health-check-dev"
```

Or delete all at once with a script:

```bash
# Delete all log groups matching the prefix
for log_group in $(aws logs describe-log-groups \
  --log-group-name-prefix "/aws/lambda/jobs-matching" \
  --query "logGroups[].logGroupName" \
  --output text); do
  echo "Deleting $log_group"
  aws logs delete-log-group --log-group-name "$log_group"
done
```

![Delete Log Groups](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-log-groups.png)

---

### Step 9: Verify Complete Cleanup

Run the following checks to ensure all resources are deleted:

```bash
# 1. Check CloudFormation stacks
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[?contains(StackName, 'jobs-matching')]" \
  --output table

# 2. Check Lambda functions
aws lambda list-functions \
  --query "Functions[?contains(FunctionName, 'jobs-matching')]" \
  --output table

# 3. Check DynamoDB tables
aws dynamodb list-tables \
  --query "TableNames[?contains(@, 'jobs-matching')]" \
  --output text

# 4. Check S3 buckets
aws s3api list-buckets \
  --query "Buckets[?contains(Name, 'jobs-matching')]" \
  --output table

# 5. Check SQS queues
aws sqs list-queues \
  --queue-name-prefix "jobs-matching" \
  --output text

# 6. Check SSM parameters
aws ssm get-parameters-by-path \
  --path "/jobs-matching" \
  --recursive \
  --output text
```

All commands should return empty results, confirming the complete cleanup.

![Verify Cleanup](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/verify-cleanup.png)

---

### Cleanup Summary Checklist

| # | Resource | Method | Status |
|---|----------|--------|--------|
| 1 | Amplify App | Console Delete | ☐ |
| 2 | WAF Web ACL | Console Delete | ☐ |
| 3 | CloudWatch Alarms | Console / CLI | ☐ |
| 4 | CloudWatch Dashboard | Console / CLI | ☐ |
| 5 | AWS Budgets | Console / CLI | ☐ |
| 6 | SSM Parameters | CLI: `ssm delete-parameter` | ☐ |
| 7 | S3 Bucket (empty) | CLI: `s3 rm --recursive` | ☐ |
| 8 | SAM Stack | CLI: `sam delete` | ☐ |
| 9 | CloudWatch Log Groups | CLI: `logs delete-log-group` | ☐ |
| 10 | Verification | CLI: multiple checks | ☐ |

{{% notice tip %}}
Print this checklist or keep it open while performing the cleanup to ensure no resources are missed. Missing even one resource (like a DynamoDB table or CloudWatch log group) can lead to ongoing charges.
{{% /notice %}}
