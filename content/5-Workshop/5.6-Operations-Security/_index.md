---
title : "Operations & Security"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

## Operations, Security & Cost Optimization

This section covers the cross-cutting concerns that apply to the entire Jobs Matching Platform. While the previous sections focused on building individual subsystems (Jobs Ingestion, User Interaction, CV Matching Flow), this section ensures the platform runs **securely**, is **observable**, and remains **cost-effective**.

![Operations & Security Architecture](/images/2-Proposal/Jobs-Matching-Platform-v3.drawio.png)

In the architecture diagram, the bottom layer — **Cost, Security, Operation** — spans across all three functional subsystems. It consists of four key pillars:

| Pillar | AWS Service | Purpose |
|--------|-------------|---------|
| **IAM** | AWS IAM | Identity and access management between internal services |
| **API Keys** | AWS Systems Manager Parameter Store | Secure storage and management of secrets and API keys |
| **Logs/Alarm** | Amazon CloudWatch | System logging, metrics collection, and automated alerting |
| **Budget** | AWS Budgets | Cost tracking, budget limits, and spending alerts |

### Why This Matters

In a serverless architecture like this one, there is no single server to manage, but the **blast radius of misconfiguration** can be significant:

- **Over-permissive IAM roles** can expose DynamoDB tables, S3 buckets, or allow unauthorized Lambda invocations.
- **Hardcoded API keys** (SerpAPI, Gemini) in source code create security vulnerabilities when the code is pushed to Git.
- **No monitoring** means you won't know when a Lambda function fails, an SQS message goes to the DLQ, or Textract calls spike unexpectedly.
- **No budget alerts** can lead to unexpected AWS bills, especially with services like Gemini API, Textract, or high-volume DynamoDB reads.

### What You Will Learn

In this section, you will:

1. **[5.6.1 - IAM Policies](5.6.1-iam-policies/)**: Review and understand the Principle of Least Privilege applied to every Lambda function in the SAM template.
2. **[5.6.2 - API Keys Management](5.6.2-api-keys-management/)**: Securely store and retrieve sensitive keys (SerpAPI, Gemini API Key) using AWS Systems Manager Parameter Store.
3. **[5.6.3 - Monitoring & Alarms](5.6.3-monitoring-alarms/)**: Set up CloudWatch Logs, Metrics, and Alarms to monitor Lambda errors, SQS Dead Letter Queues, and API Gateway latency.
4. **[5.6.4 - Cost & Budget](5.6.4-cost-budget/)**: Configure AWS Budgets to set spending thresholds and receive email notifications before costs exceed your limits.

### Content

1. [IAM Policies](5.6.1-iam-policies/)
2. [API Keys Management](5.6.2-api-keys-management/)
3. [Monitoring & Alarms](5.6.3-monitoring-alarms/)
4. [Cost & Budget](5.6.4-cost-budget/)
