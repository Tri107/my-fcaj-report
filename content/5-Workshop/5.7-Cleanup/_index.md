---
title : "Cleanup"
date : 2024-01-01
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---

## Clean Up Resources

After completing the workshop, it is essential to **delete all AWS resources** to avoid ongoing charges. Even though many services fall under the AWS Free Tier, some resources (like DynamoDB tables, S3 buckets, and CloudWatch alarms) will continue to incur costs if left running.

This section provides a comprehensive guide to remove all resources created during the workshop.

{{% notice warning %}}
**This action is irreversible.** All data in DynamoDB tables, S3 buckets, and Cognito user pools will be permanently deleted. Make sure to export any data you need before proceeding.
{{% /notice %}}

### Resources to Delete

The following resources were created during this workshop:

| Category | Resources |
|----------|-----------|
| **SAM Stack** | All Lambda functions, API Gateway, DynamoDB tables, SQS queues, EventBridge rules, Cognito, IAM roles |
| **S3 Buckets** | CV Storage bucket, SAM deployment bucket |
| **SSM Parameters** | SerpAPI key, Gemini API key |
| **CloudWatch** | Log groups, Alarms, Dashboards |
| **AWS Budgets** | Budget configurations and alerts |
| **Amplify** | Frontend hosting application |
| **WAF** | Web ACL rules |

### Content

1. [Delete Resources](5.7.1-delete-resources/)
