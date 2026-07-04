---
title : "Cost & Budget"
date : 2024-01-01
weight : 4
chapter : false
pre : " <b> 5.6.4 </b> "
---

## AWS Budgets — Cost Tracking & Spending Alerts

In a serverless architecture, costs are directly tied to usage. Without proper budget monitoring, unexpected spikes in Lambda invocations, DynamoDB reads, Textract calls, or Gemini API calls can lead to surprisingly high AWS bills.

AWS Budgets allows you to set custom cost thresholds and receive **email alerts** before your spending exceeds defined limits.

### Cost Breakdown of the Platform

Before setting up budgets, it's important to understand which services incur costs and their pricing models:

| Service | Pricing Model | Key Cost Driver | Free Tier |
|---------|--------------|-----------------|-----------|
| **Lambda** | Per request + duration | Number of invocations, memory × time | 1M requests/month, 400K GB-seconds |
| **DynamoDB** | Pay-per-request (on-demand) | Read/Write request units | 25 RCU/WCU, 25 GB storage |
| **S3** | Per GB stored + requests | CV file storage, GET/PUT requests | 5 GB, 20K GET, 2K PUT |
| **API Gateway (HTTP)** | Per request | API call volume | 1M requests/month (12 months) |
| **SQS** | Per request | Messages sent/received | 1M requests/month |
| **Cognito** | Per MAU (Monthly Active User) | Active users | 50K MAU |
| **Textract** | Per page processed | CV text extraction calls | 1K pages/month (3 months) |
| **Gemini API** | Per input/output token (external) | AI evaluation calls | Free tier available (limited) |
| **CloudWatch** | Per metric, alarm, log data | Logs storage, custom metrics | 10 metrics, 10 alarms, 5GB logs |
| **EventBridge** | Per event published | Scheduler invocations | Free (included) |

{{% notice info %}}
Most services in this project fall within the **AWS Free Tier** for development and low-traffic usage. However, Textract and Gemini API can quickly incur costs with heavy usage, especially during testing.
{{% /notice %}}

### Step 1: Create an AWS Budget

1. Navigate to **AWS Console → Billing and Cost Management → Budgets**

![Budgets Console](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/budget-console.png)

2. Click **Create budget**

3. Select **Customize (advanced)**

![Budget Type](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/budget-type.png)

4. Choose **Cost budget - Recommended**

#### Configure Budget Details

| Setting | Value |
|---------|-------|
| **Budget name** | `jobs-matching-platform-monthly` |
| **Period** | Monthly |
| **Budget renewal type** | Recurring budget |
| **Start month** | Current month |
| **Budgeting method** | Fixed |
| **Budget amount** | $10.00 (adjust based on your needs) |

![Budget Config](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/budget-config.png)

{{% notice tip %}}
For a workshop/development environment, **$10/month** is a reasonable starting budget. For production, adjust based on expected traffic volume and service usage patterns.
{{% /notice %}}

5. (Optional) Apply **filters** to scope the budget to specific services:

You can filter by:
- **Service**: Only track Lambda, DynamoDB, S3, etc.
- **Tag**: Use tags to track costs for this specific project
- **Region**: Filter to `ap-southeast-1` only

![Budget Filters](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/budget-filter.png)

### Step 2: Configure Alert Thresholds

Set up multiple alert thresholds to get early warnings:

#### Alert 1: 50% Threshold (Early Warning)

| Setting | Value |
|---------|-------|
| **Threshold** | 50% of budgeted amount ($5.00) |
| **Trigger** | Actual cost |
| **Notification** | Email |
| **Email recipients** | your-email@example.com |

#### Alert 2: 80% Threshold (Warning)

| Setting | Value |
|---------|-------|
| **Threshold** | 80% of budgeted amount ($8.00) |
| **Trigger** | Actual cost |
| **Notification** | Email |
| **Email recipients** | your-email@example.com |

#### Alert 3: 100% Threshold (Critical)

| Setting | Value |
|---------|-------|
| **Threshold** | 100% of budgeted amount ($10.00) |
| **Trigger** | Actual cost |
| **Notification** | Email |
| **Email recipients** | your-email@example.com |

#### Alert 4: Forecasted Overspend

| Setting | Value |
|---------|-------|
| **Threshold** | 100% of budgeted amount |
| **Trigger** | Forecasted cost |
| **Notification** | Email |
| **Email recipients** | your-email@example.com |

![Alert Config](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/alert-config.png)

{{% notice warning %}}
AWS Budgets alerts are **notification-only** — they do not automatically stop or throttle services. If you receive a 100% alert, you must manually take action (e.g., disable the EventBridge scheduler, delete expensive resources, or set Lambda concurrency limits).
{{% /notice %}}

6. Click **Create budget**

![Budget Created](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/create-budget.png)

### Step 3: Monitor Cost Explorer

AWS Cost Explorer provides detailed cost analysis:

1. Navigate to **AWS Console → Billing → Cost Explorer**

2. View costs **grouped by service** to identify the most expensive components:

![Cost Explorer](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/cost-explorer.png)

3. View **daily cost trend** to detect unusual spikes:

![Daily Trend](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/daily-trend.png)

### Step 4: Cost Optimization Tips

| Optimization | Action | Impact |
|-------------|--------|--------|
| **Lambda Memory** | Right-size memory allocation (256MB default might be too high for simple functions) | Reduce Lambda cost |
| **DynamoDB TTL** | Already enabled on MatchResultsTable — auto-deletes old results after 90 days | Reduce storage cost |
| **S3 Lifecycle** | Add lifecycle rules to move old CVs to S3 Infrequent Access | Reduce storage cost |
| **EventBridge Schedule** | Reduce fetch frequency or disable when not needed | Reduce Lambda + SerpAPI calls |
| **Textract Calls** | Cache extracted text to avoid re-processing same CV | Reduce Textract cost |
| **Gemini Model** | Use lighter models (e.g., `gemini-3.1-flash-lite`) | Reduce AI evaluation cost |

#### DynamoDB TTL Configuration (Already Applied)

```yaml
# In template.yaml - MatchResultsTable
TimeToLiveSpecification:
  AttributeName: ttl
  Enabled: true
```

This automatically deletes match results after the TTL timestamp expires (set to 90 days in the application code), keeping storage costs low without manual cleanup.

### Verify with AWS CLI

```bash
# List all budgets
aws budgets describe-budgets \
  --account-id <your-account-id> \
  --query "Budgets[].{Name:BudgetName,Limit:BudgetLimit.Amount,Spent:CalculatedSpend.ActualSpend.Amount}" \
  --output table

# Get current month cost
aws ce get-cost-and-usage \
  --time-period Start=$(date -u +%Y-%m-01),End=$(date -u +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query "ResultsByTime[0].Groups[?Metrics.BlendedCost.Amount > '0'].{Service:Keys[0],Cost:Metrics.BlendedCost.Amount}" \
  --output table
```

### Summary

| What | Configuration |
|------|--------------|
| **Monthly Budget** | $10.00 (adjustable) |
| **Alert at 50%** | Email notification at $5.00 actual spend |
| **Alert at 80%** | Email notification at $8.00 actual spend |
| **Alert at 100%** | Email notification at $10.00 actual spend |
| **Forecast Alert** | Email when forecasted to exceed budget |
| **Cost Optimization** | DynamoDB TTL, right-sized Lambda, efficient AI models |

{{% notice tip %}}
After completing the workshop, remember to **clean up all resources** (see Section 7) to avoid ongoing charges. Even with the Free Tier, some services (Textract, Gemini API) may still incur costs.
{{% /notice %}}
