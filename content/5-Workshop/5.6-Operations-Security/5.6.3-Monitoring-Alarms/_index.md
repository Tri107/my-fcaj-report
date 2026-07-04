---
title : "Monitoring & Alarms"
date : 2024-01-01
weight : 3
chapter : false
pre : " <b> 5.6.3 </b> "
---

## CloudWatch Monitoring, Logs & Alarms

Amazon CloudWatch provides the **observability layer** for the entire Jobs Matching Platform. In a serverless architecture, where there are no servers to SSH into, CloudWatch becomes the primary tool for understanding what's happening inside your system.

{{% notice info %}}
In this project, CloudWatch Logs are automatically enabled for all Lambda functions via the SAM template's `Globals.Function.LoggingConfig` configuration. CloudWatch Alarms can be configured additionally to proactively monitor and alert on errors.
{{% /notice %}}



CloudWatch collects data from every service in the platform:

| Service | What CloudWatch Captures |
|---------|------------------------|
| **Lambda** | Invocation count, errors, duration, throttles, concurrent executions |
| **API Gateway** | Request count, latency, 4xx/5xx errors |
| **DynamoDB** | Read/Write capacity, throttled requests, system errors |
| **SQS** | Messages sent/received, messages in DLQ, age of oldest message |
| **S3** | Request count, bucket size, object count |
| **Cognito** | Sign-in attempts, token errors |

### Step 1: Understanding Lambda Log Configuration

In the `template.yaml`, all Lambda functions are configured with JSON-formatted logs:

```yaml
Globals:
  Function:
    Runtime: nodejs24.x
    LoggingConfig:
      LogFormat: JSON
```

This means every `console.log()`, `console.error()`, and runtime error in your Lambda code is automatically sent to CloudWatch Logs in structured JSON format.

#### View Lambda Logs in CloudWatch Console

1. Navigate to **AWS Console → CloudWatch → Log groups**

![Log Groups](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/log-groups.png)

2. Each Lambda function has its own log group named `/aws/lambda/<function-name>`:

| Log Group | Lambda Function |
|-----------|----------------|
| `/aws/lambda/jobs-matching-fetch-jobs-dev` | FetchJobs (SerpAPI ingestion) |
| `/aws/lambda/jobs-matching-get-jobs-dev` | GetJobs (API endpoint) |
| `/aws/lambda/jobs-matching-evaluate-match-dev` | EvaluateMatch (AI evaluation) |
| `/aws/lambda/jobs-matching-manage-cv-dev` | ManageCv (CV upload/delete) |

3. Click on a log group, then click on a **log stream** to see individual invocation logs

![Log Stream](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/log-stream.png)

4. Each log entry shows structured JSON with timestamp, request ID, and log level

![Log Entry](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/log-entry.png)

#### Query Logs with CloudWatch Logs Insights

CloudWatch Logs Insights allows you to run SQL-like queries against your logs:

1. Navigate to **CloudWatch → Logs Insights**
2. Select the log group(s) you want to query
3. Run a query:

```sql
# Find all errors in the last 24 hours
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

![Logs Insights](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/logs-insights.png)

```sql
# Find Lambda cold starts and their duration
fields @timestamp, @initDuration, @duration, @memorySize, @maxMemoryUsed
| filter ispresent(@initDuration)
| sort @initDuration desc
| limit 20
```

```sql
# Analyze EvaluateMatch function performance
fields @timestamp, @duration, @billedDuration, @memorySize, @maxMemoryUsed
| filter @logGroup = "/aws/lambda/jobs-matching-evaluate-match-dev"
| stats avg(@duration), max(@duration), min(@duration), count(*) by bin(1h)
```

### Step 2: Key CloudWatch Metrics to Monitor

#### Lambda Function Metrics

Navigate to **CloudWatch → Metrics → Lambda**:

| Metric | Description | Recommended Threshold |
|--------|-------------|----------------------|
| `Errors` | Number of invocations that result in a function error | > 0 in 5 min |
| `Duration` | Time taken to execute the function | > 80% of timeout |
| `Throttles` | Number of invocations that are throttled | > 0 |
| `ConcurrentExecutions` | Number of function instances running | > 80% of account limit |

![Lambda Metrics](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/lambda-metrics.png)

#### SQS Dead Letter Queue Metrics

The DLQ is critical — messages here represent **failed job processing**:

| Metric | Description | Recommended Threshold |
|--------|-------------|----------------------|
| `ApproximateNumberOfMessagesVisible` | Messages in the DLQ | > 0 |
| `ApproximateAgeOfOldestMessage` | How long the oldest message has been in DLQ | > 3600 seconds |

#### API Gateway Metrics

| Metric | Description | Recommended Threshold |
|--------|-------------|----------------------|
| `5XXError` | Server-side errors | > 0 in 5 min |
| `4XXError` | Client-side errors (may indicate attacks) | > 100 in 5 min |
| `Latency` | Time from request to response | > 3000ms (p99) |

### Step 3: Create CloudWatch Alarms

Alarms automatically notify you when metrics cross defined thresholds.

#### Alarm 1: Lambda Error Alarm

1. Navigate to **CloudWatch → Alarms → Create alarm**
2. Click **Select metric**
3. Choose **Lambda → By Function Name → Errors**
4. Select the function (e.g., `jobs-matching-evaluate-match-dev`)

![Select Metric](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/select-metric.png)
![Select Metric 2](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/select-metric2.png)

5. Configure the alarm:

| Setting | Value |
|---------|-------|
| **Statistic** | Sum |
| **Period** | 5 minutes |
| **Threshold type** | Static |
| **Condition** | Greater than 0 |
| **Datapoints to alarm** | 1 out of 1 |

![Alarm Config](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/alarm-config.png)

6. Configure actions — Create or select an **SNS topic** to send email notifications:

![SNS Config](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/sns-config.png)

7. Name the alarm: `jobs-matching-evaluate-match-errors`

8. Click **Create alarm**

![Alarm Created](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/alarm-created.png)

#### Alarm 2: SQS Dead Letter Queue Alarm

Create an alarm that triggers when messages appear in the DLQ:

| Setting | Value |
|---------|-------|
| **Metric** | SQS → Queue Metrics → ApproximateNumberOfMessagesVisible |
| **Queue** | `jobs-matching-raw-jobs-dlq-dev` |
| **Threshold** | Greater than 0 |
| **Period** | 5 minutes |
| **Action** | Send notification to SNS topic |

This alarm ensures you're immediately notified when job processing fails and messages are moved to the Dead Letter Queue.

#### Alarm 3: API Gateway 5XX Error Alarm

| Setting | Value |
|---------|-------|
| **Metric** | ApiGateway → 5XXError |
| **API** | BackendHttpApi |
| **Threshold** | Greater than 5 in 5 minutes |
| **Action** | Send notification to SNS topic |

### Step 4: Create a CloudWatch Dashboard (Optional)

A dashboard provides a centralized view of all key metrics:

1. Navigate to **CloudWatch → Dashboards → Create dashboard**
2. Name it `jobs-matching-platform-dev`
3. Add widgets for:

- **Lambda Errors** across all functions (Number widget)
- **Lambda Duration** for EvaluateMatch function (Line graph)
- **SQS DLQ Message Count** (Number widget)
- **API Gateway Request Count** (Line graph)
- **API Gateway Latency p99** (Line graph)

![Dashboard](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/dashboard.png)

### Verify with AWS CLI

```bash
# List all CloudWatch alarms
aws cloudwatch describe-alarms \
  --query "MetricAlarms[].{Name:AlarmName,State:StateValue,Metric:MetricName}" \
  --output table

# Get Lambda error metrics for the last hour
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=jobs-matching-evaluate-match-dev \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum

# Check DLQ message count
aws sqs get-queue-attributes \
  --queue-url <dlq-queue-url> \
  --attribute-names ApproximateNumberOfMessages
```

### Summary of Recommended Alarms

| Alarm | Service | Metric | Threshold | Priority |
|-------|---------|--------|-----------|----------|
| Lambda Errors | Lambda | Errors | > 0 | 🔴 High |
| DLQ Messages | SQS | MessagesVisible | > 0 | 🔴 High |
| API 5XX Errors | API Gateway | 5XXError | > 5/5min | 🔴 High |
| Lambda Duration | Lambda | Duration | > 80% timeout | 🟡 Medium |
| API Latency | API Gateway | Latency (p99) | > 3000ms | 🟡 Medium |
| Lambda Throttles | Lambda | Throttles | > 0 | 🟡 Medium |

{{% notice tip %}}
Start with the **High priority** alarms first. As your platform matures and handles production traffic, gradually add Medium priority alarms and fine-tune thresholds based on actual usage patterns.
{{% /notice %}}
