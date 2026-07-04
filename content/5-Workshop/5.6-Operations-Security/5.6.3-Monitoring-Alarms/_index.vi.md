---
title : "Giám sát & Cảnh báo"
date : 2024-01-01
weight : 3
chapter : false
pre : " <b> 5.6.3 </b> "
---

## Giám sát, Nhật ký & Cảnh báo với CloudWatch

Amazon CloudWatch cung cấp **tầng quan sát (observability)** cho toàn bộ Jobs Matching Platform. Trong kiến trúc serverless, nơi không có máy chủ để SSH vào, CloudWatch trở thành công cụ chính để hiểu những gì đang xảy ra bên trong hệ thống.

{{% notice info %}}
Trong dự án này, CloudWatch Logs được tự động bật cho tất cả Lambda functions thông qua cấu hình `Globals.Function.LoggingConfig` trong SAM template. CloudWatch Alarms có thể được cấu hình thêm để chủ động giám sát và cảnh báo lỗi.
{{% /notice %}}



CloudWatch thu thập dữ liệu từ mọi dịch vụ trong nền tảng:

| Dịch vụ | CloudWatch thu thập gì |
|---------|----------------------|
| **Lambda** | Số lần gọi, lỗi, thời gian thực thi, throttles, concurrent executions |
| **API Gateway** | Số request, độ trễ, lỗi 4xx/5xx |
| **DynamoDB** | Dung lượng đọc/ghi, throttled requests, lỗi hệ thống |
| **SQS** | Messages gửi/nhận, messages trong DLQ, tuổi message cũ nhất |
| **S3** | Số request, kích thước bucket, số objects |
| **Cognito** | Lần đăng nhập, lỗi token |

### Bước 1: Hiểu Cấu hình Lambda Log

Trong `template.yaml`, tất cả Lambda functions được cấu hình với logs định dạng JSON:

```yaml
Globals:
  Function:
    Runtime: nodejs24.x
    LoggingConfig:
      LogFormat: JSON
```

Điều này có nghĩa mọi `console.log()`, `console.error()`, và lỗi runtime trong code Lambda đều tự động được gửi đến CloudWatch Logs ở định dạng JSON có cấu trúc.

#### Xem Lambda Logs trên CloudWatch Console

1. Truy cập **AWS Console → CloudWatch → Log groups**

![Log Groups](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/log-groups.png)

2. Mỗi Lambda function có log group riêng được đặt tên `/aws/lambda/<tên-function>`:

| Log Group | Lambda Function |
|-----------|----------------|
| `/aws/lambda/jobs-matching-fetch-jobs-dev` | FetchJobs (Thu thập từ SerpAPI) |
| `/aws/lambda/jobs-matching-get-jobs-dev` | GetJobs (API endpoint) |
| `/aws/lambda/jobs-matching-evaluate-match-dev` | EvaluateMatch (Đánh giá AI) |
| `/aws/lambda/jobs-matching-manage-cv-dev` | ManageCv (Upload/xóa CV) |

3. Click vào log group, sau đó click vào **log stream** để xem logs từng lần gọi

![Log Stream](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/log-stream.png)

4. Mỗi log entry hiển thị JSON có cấu trúc với timestamp, request ID và log level

![Log Entry](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/log-entry.png)

#### Truy vấn Logs với CloudWatch Logs Insights

CloudWatch Logs Insights cho phép chạy các truy vấn dạng SQL trên logs:

1. Truy cập **CloudWatch → Logs Insights**
2. Chọn log group(s) bạn muốn truy vấn
3. Chạy truy vấn:

```sql
# Tìm tất cả lỗi trong 24 giờ qua
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

![Logs Insights](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/logs-insights.png)

```sql
# Tìm Lambda cold starts và thời gian khởi tạo
fields @timestamp, @initDuration, @duration, @memorySize, @maxMemoryUsed
| filter ispresent(@initDuration)
| sort @initDuration desc
| limit 20
```

```sql
# Phân tích hiệu năng hàm EvaluateMatch
fields @timestamp, @duration, @billedDuration, @memorySize, @maxMemoryUsed
| filter @logGroup = "/aws/lambda/jobs-matching-evaluate-match-dev"
| stats avg(@duration), max(@duration), min(@duration), count(*) by bin(1h)
```

### Bước 2: Các CloudWatch Metrics quan trọng cần theo dõi

#### Lambda Function Metrics

Truy cập **CloudWatch → Metrics → Lambda**:

| Metric | Mô tả | Ngưỡng khuyến nghị |
|--------|--------|-------------------|
| `Errors` | Số lần gọi bị lỗi | > 0 trong 5 phút |
| `Duration` | Thời gian thực thi function | > 80% timeout |
| `Throttles` | Số lần gọi bị throttle | > 0 |
| `ConcurrentExecutions` | Số instances function đang chạy | > 80% giới hạn tài khoản |

![Lambda Metrics](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/lambda-metrics.png)

#### SQS Dead Letter Queue Metrics

DLQ rất quan trọng — messages ở đây đại diện cho **xử lý job thất bại**:

| Metric | Mô tả | Ngưỡng khuyến nghị |
|--------|--------|-------------------|
| `ApproximateNumberOfMessagesVisible` | Số messages trong DLQ | > 0 |
| `ApproximateAgeOfOldestMessage` | Message cũ nhất đã ở DLQ bao lâu | > 3600 giây |

#### API Gateway Metrics

| Metric | Mô tả | Ngưỡng khuyến nghị |
|--------|--------|-------------------|
| `5XXError` | Lỗi phía server | > 0 trong 5 phút |
| `4XXError` | Lỗi phía client (có thể là dấu hiệu tấn công) | > 100 trong 5 phút |
| `Latency` | Thời gian từ request đến response | > 3000ms (p99) |

### Bước 3: Tạo CloudWatch Alarms

Alarms tự động thông báo cho bạn khi metrics vượt ngưỡng đã định.

#### Alarm 1: Lambda Error Alarm

1. Truy cập **CloudWatch → Alarms → Create alarm**
2. Click **Select metric**
3. Chọn **Lambda → By Function Name → Errors**
4. Chọn function (ví dụ: `jobs-matching-evaluate-match-dev`)

![Chọn Metric](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/select-metric.png)
![Chọn Metric 2](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/select-metric2.png)
5. Cấu hình alarm:

| Cài đặt | Giá trị |
|---------|---------|
| **Statistic** | Sum |
| **Period** | 5 phút |
| **Threshold type** | Static |
| **Condition** | Greater than 0 |
| **Datapoints to alarm** | 1 out of 1 |

![Cấu hình Alarm](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/alarm-config.png)

6. Cấu hình hành động — Tạo hoặc chọn **SNS topic** để gửi email thông báo:

![Cấu hình SNS](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/sns-config.png)

7. Đặt tên alarm: `jobs-matching-evaluate-match-errors`

8. Click **Create alarm**

![Alarm đã tạo](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/alarm-created.png)

#### Alarm 2: SQS Dead Letter Queue Alarm

Tạo alarm kích hoạt khi có messages xuất hiện trong DLQ:

| Cài đặt | Giá trị |
|---------|---------|
| **Metric** | SQS → Queue Metrics → ApproximateNumberOfMessagesVisible |
| **Queue** | `jobs-matching-raw-jobs-dlq-dev` |
| **Threshold** | Greater than 0 |
| **Period** | 5 phút |
| **Action** | Gửi thông báo đến SNS topic |

Alarm này đảm bảo bạn được thông báo ngay khi xử lý job thất bại và messages bị chuyển vào Dead Letter Queue.

#### Alarm 3: API Gateway 5XX Error Alarm

| Cài đặt | Giá trị |
|---------|---------|
| **Metric** | ApiGateway → 5XXError |
| **API** | BackendHttpApi |
| **Threshold** | Greater than 5 trong 5 phút |
| **Action** | Gửi thông báo đến SNS topic |

### Bước 4: Tạo CloudWatch Dashboard (Tùy chọn)

Dashboard cung cấp cái nhìn tập trung về tất cả metrics quan trọng:

1. Truy cập **CloudWatch → Dashboards → Create dashboard**
2. Đặt tên `jobs-matching-platform-dev`
3. Thêm các widgets cho:

- **Lambda Errors** trên tất cả functions (Number widget)
- **Lambda Duration** cho hàm EvaluateMatch (Line graph)
- **SQS DLQ Message Count** (Number widget)
- **API Gateway Request Count** (Line graph)
- **API Gateway Latency p99** (Line graph)

![Dashboard](/images/5-Workshop/5.6-Operations-Security/5.6.3-Monitoring-Alarms/dashboard.png)

### Xác minh với AWS CLI

```bash
# Liệt kê tất cả CloudWatch alarms
aws cloudwatch describe-alarms \
  --query "MetricAlarms[].{Name:AlarmName,State:StateValue,Metric:MetricName}" \
  --output table

# Lấy metrics lỗi Lambda trong giờ qua
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=jobs-matching-evaluate-match-dev \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum

# Kiểm tra số message trong DLQ
aws sqs get-queue-attributes \
  --queue-url <dlq-queue-url> \
  --attribute-names ApproximateNumberOfMessages
```

### Tóm tắt Alarms khuyến nghị

| Alarm | Dịch vụ | Metric | Ngưỡng | Ưu tiên |
|-------|---------|--------|--------|---------|
| Lambda Errors | Lambda | Errors | > 0 | 🔴 Cao |
| DLQ Messages | SQS | MessagesVisible | > 0 | 🔴 Cao |
| API 5XX Errors | API Gateway | 5XXError | > 5/5 phút | 🔴 Cao |
| Lambda Duration | Lambda | Duration | > 80% timeout | 🟡 Trung bình |
| API Latency | API Gateway | Latency (p99) | > 3000ms | 🟡 Trung bình |
| Lambda Throttles | Lambda | Throttles | > 0 | 🟡 Trung bình |

{{% notice tip %}}
Bắt đầu với các alarm **Ưu tiên Cao** trước. Khi nền tảng trưởng thành và xử lý traffic production, dần dần thêm các alarm Ưu tiên Trung bình và tinh chỉnh ngưỡng dựa trên mẫu sử dụng thực tế.
{{% /notice %}}
