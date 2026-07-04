---
title : "IAM Policies"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.6.1 </b> "
---

## Principle of Least Privilege for Lambda Functions

AWS Identity and Access Management (IAM) is the backbone of security in AWS. Every service-to-service interaction within the Jobs Matching Platform is governed by IAM policies that follow the **Principle of Least Privilege** — each Lambda function is granted only the minimum permissions required to perform its specific task.


In this platform, IAM controls the following critical interactions:

- **Lambda → DynamoDB**: Read/Write access scoped to specific tables
- **Lambda → S3**: Object-level operations scoped to the CV bucket
- **Lambda → SQS**: Send message permissions scoped to the raw jobs queue
- **Lambda → Textract**: Document text detection
- **Lambda → Gemini API**: AI evaluation via external HTTP API (using API key from SSM)
- **Lambda → SSM Parameter Store**: Read-only access to specific parameter paths
- **Lambda → Lambda**: Invoke permissions between ProcessJobs and NormalizeAndMatch



### Understanding SAM Policy Templates

AWS SAM (Serverless Application Model) provides **policy templates** that simplify IAM policy creation. Instead of writing raw JSON IAM policies, SAM offers shorthand references that automatically generate properly scoped policies.

#### SAM Policy Templates Used in This Project

| SAM Policy Template | Description | Used By |
|---------------------|-------------|---------|
| `DynamoDBReadPolicy` | Read-only access to a specific DynamoDB table | GetJobs, GetJobById, SearchJobs, EvaluateMatch, GetMatchResults |
| `DynamoDBCrudPolicy` | Full CRUD access to a specific DynamoDB table | NormalizeAndMatch, SavedJobs, EvaluateMatch |
| `SQSSendMessagePolicy` | Permission to send messages to a specific SQS queue | FetchJobs |
| `LambdaInvokePolicy` | Permission to invoke a specific Lambda function | ProcessJobs |

#### Example: GetJobsFunction — Read-Only Access

The `GetJobsFunction` only needs to read from the Jobs Table. Its SAM policy:

```yaml
# In template.yaml - GetJobsFunction
Policies:
  - DynamoDBReadPolicy:
      TableName: !Ref JobsTable
```

This generates an IAM policy equivalent to:

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:GetItem",
    "dynamodb:Scan",
    "dynamodb:Query",
    "dynamodb:BatchGetItem",
    "dynamodb:DescribeTable"
  ],
  "Resource": [
    "arn:aws:dynamodb:ap-southeast-1:123456789012:table/jobs-matching-jobs-dev",
    "arn:aws:dynamodb:ap-southeast-1:123456789012:table/jobs-matching-jobs-dev/index/*"
  ]
}
```

{{% notice info %}}
The `DynamoDBReadPolicy` template automatically includes access to the table's indexes (GSI/LSI), which is why the `SearchJobsFunction` can also use it to query the `HashIndex`.
{{% /notice %}}

#### Example: FetchJobsFunction — SQS Send Only

The `FetchJobsFunction` fetches jobs from SerpAPI and pushes them to SQS. It only needs `sqs:SendMessage` permission:

```yaml
# In template.yaml - FetchJobsFunction
Policies:
  - SQSSendMessagePolicy:
      QueueName: !GetAtt RawJobsQueue.QueueName
```

This function does **NOT** have any DynamoDB access — it only pushes raw data to the queue. The downstream `NormalizeAndMatchFunction` is responsible for writing to DynamoDB.

#### Example: EvaluateMatchFunction — Custom Policy Statements

Some functions require permissions that SAM templates don't cover. For these, we write custom IAM policy statements:

```yaml
# In template.yaml - EvaluateMatchFunction
Policies:
  - DynamoDBReadPolicy:
      TableName: !Ref JobsTable
  - DynamoDBCrudPolicy:
      TableName: !Ref MatchResultsTable
  - Statement:
      - Effect: Allow
        Action:
          - s3:GetObject
          - s3:HeadObject
          - s3:ListBucket
        Resource:
          - !GetAtt CvBucket.Arn
          - !Sub "${CvBucket.Arn}/*"
      - Effect: Allow
        Action:
          - textract:DetectDocumentText
        Resource: "*"
      - Effect: Allow
        Action:
          - ssm:GetParameter
        Resource: !Sub "arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:parameter${GeminiApiKeyParameterName}"
```

**Key observations:**

1. **S3 access is read-only** (`GetObject`, `HeadObject`, `ListBucket`) — this function reads CVs but cannot modify or delete them.
2. **SSM Parameter access is scoped** to the exact parameter path `/jobs-matching/dev/gemini-api-key` — it cannot read other parameters.
3. **Textract** uses `Resource: "*"` because this service doesn't support resource-level restrictions. **Gemini API** is called externally via HTTP (not an AWS service), so no IAM permission is needed — only the API key stored in SSM.

### Review: Complete IAM Permission Matrix

| Lambda Function | DynamoDB | S3 | SQS | Textract | Gemini (via SSM) | SSM | Lambda |
|----------------|----------|-----|-----|----------|-----------------|-----|--------|
| HealthCheck | — | — | — | — | — | — | — |
| GetJobs | Read (Jobs) | — | — | — | — | — | — |
| GetJobById | Read (Jobs) | — | — | — | — | — | — |
| SearchJobs | Read (Jobs) | — | — | — | — | — | — |
| FetchJobs | — | — | Send | — | — | — | — |
| ProcessJobs | — | — | — | — | — | — | Invoke |
| NormalizeAndMatch | CRUD (Jobs) | — | — | — | — | — | — |
| SavedJobs | CRUD (SavedJobs) | — | — | — | — | — | — |
| EvaluateMatch | Read (Jobs) + CRUD (MatchResults) | Read | — | ✓ | ✓ (API Key) | Read | — |
| GetMatchResults | Read (MatchResults) | — | — | — | — | — | — |
| ExtractCv | — | Read | — | ✓ | — | — | — |
| ManageCv | — | CRUD | — | — | — | — | — |

### Verify IAM Policies in AWS Console

1. Navigate to **AWS Console → IAM → Roles**

2. Search for your Lambda function role (e.g., search `jobs-matching-backend` to see all roles in the stack)

![IAM Roles Search](/images/5-Workshop/5.6-Operations-Security/5.6.1-IAM-Policies/iam-roles-search.png)

3. Click on the role to inspect the attached policies

![IAM Role Detail](/images/5-Workshop/5.6-Operations-Security/5.6.1-IAM-Policies/iam-role-detail.png)

4. Expand each policy to verify the specific permissions and resources

![IAM Policy Detail](/images/5-Workshop/5.6-Operations-Security/5.6.1-IAM-Policies/iam-policy-detail.png)

{{% notice tip %}}
You can also use the AWS CLI to inspect IAM policies attached to a Lambda execution role:
```bash
# Get the execution role ARN for a Lambda function
aws lambda get-function-configuration \
  --function-name jobs-matching-get-jobs-dev \
  --query "Role" --output text

# List attached policies for the role
aws iam list-attached-role-policies \
  --role-name <role-name-from-above>

# Get detailed policy document
aws iam get-role-policy \
  --role-name <role-name> \
  --policy-name <policy-name>
```
{{% /notice %}}

### Best Practices Applied

| Practice | How It's Applied |
|----------|-----------------|
| **Least Privilege** | Each function only gets permissions it needs |
| **Resource Scoping** | Policies target specific tables, buckets, and parameters — not `*` |
| **SAM Templates** | Use SAM policy templates for common patterns instead of raw JSON |
| **No Shared Roles** | Each Lambda function has its own execution role (SAM default behavior) |
| **No Inline Secrets** | API keys are retrieved from SSM at runtime, not embedded in IAM policies |

{{% notice warning %}}
Avoid using `Resource: "*"` whenever possible. In this project, `Textract` requires it because it doesn't support resource-level restrictions. `Gemini API` is called externally via HTTP and doesn't need IAM permissions — only the API key retrieved from SSM Parameter Store. For all other services (DynamoDB, S3, SQS, SSM), always specify exact resource ARNs.
{{% /notice %}}
