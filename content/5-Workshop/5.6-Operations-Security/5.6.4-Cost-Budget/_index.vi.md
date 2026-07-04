---
title : "Chi phí & Ngân sách"
date : 2024-01-01
weight : 4
chapter : false
pre : " <b> 5.6.4 </b> "
---

## AWS Budgets — Theo dõi Chi phí & Cảnh báo Chi tiêu

Trong kiến trúc serverless, chi phí gắn trực tiếp với mức sử dụng. Nếu không có giám sát ngân sách phù hợp, các đợt tăng đột biến Lambda invocations, DynamoDB reads, Textract calls, hoặc Gemini API calls có thể dẫn đến hóa đơn AWS cao bất ngờ.

AWS Budgets cho phép bạn thiết lập ngưỡng chi phí tùy chỉnh và nhận **cảnh báo email** trước khi chi tiêu vượt quá giới hạn đã định.

### Phân tích Chi phí của Nền tảng

Trước khi thiết lập budgets, điều quan trọng là hiểu dịch vụ nào phát sinh chi phí và mô hình định giá:

| Dịch vụ | Mô hình Định giá | Yếu tố Chi phí chính | Free Tier |
|---------|-----------------|----------------------|-----------|
| **Lambda** | Mỗi request + thời gian thực thi | Số lần gọi, bộ nhớ × thời gian | 1M requests/tháng, 400K GB-seconds |
| **DynamoDB** | Pay-per-request (on-demand) | Read/Write request units | 25 RCU/WCU, 25 GB lưu trữ |
| **S3** | Mỗi GB lưu trữ + requests | Lưu trữ CV, GET/PUT requests | 5 GB, 20K GET, 2K PUT |
| **API Gateway (HTTP)** | Mỗi request | Lượng API calls | 1M requests/tháng (12 tháng) |
| **SQS** | Mỗi request | Messages gửi/nhận | 1M requests/tháng |
| **Cognito** | Mỗi MAU (Monthly Active User) | Người dùng hoạt động | 50K MAU |
| **Textract** | Mỗi trang xử lý | Số lần trích xuất CV | 1K trang/tháng (3 tháng) |
| **Gemini API** | Mỗi input/output token (bên ngoài) | Số lần đánh giá AI | Free tier có sẵn (giới hạn) |
| **CloudWatch** | Mỗi metric, alarm, dữ liệu log | Lưu trữ logs, custom metrics | 10 metrics, 10 alarms, 5GB logs |
| **EventBridge** | Mỗi event published | Scheduler invocations | Miễn phí (bao gồm) |

{{% notice info %}}
Hầu hết dịch vụ trong dự án này nằm trong **AWS Free Tier** cho phát triển và traffic thấp. Tuy nhiên, Textract và Gemini API có thể nhanh chóng phát sinh chi phí với mức sử dụng cao, đặc biệt trong quá trình testing.
{{% /notice %}}

### Bước 1: Tạo AWS Budget

1. Truy cập **AWS Console → Billing and Cost Management → Budgets**

![Budgets Console](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/budget-console.png)

2. Click **Create budget**

3. Chọn **Customize (advanced)**

![Loại Budget](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/budget-type.png)

4. Chọn **Cost budget - Recommended**

#### Cấu hình Chi tiết Budget

| Cài đặt | Giá trị |
|---------|---------|
| **Budget name** | `jobs-matching-platform-monthly` |
| **Period** | Monthly |
| **Budget renewal type** | Recurring budget |
| **Start month** | Tháng hiện tại |
| **Budgeting method** | Fixed |
| **Budget amount** | $10.00 (điều chỉnh theo nhu cầu) |

![Cấu hình Budget](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/budget-config.png)

{{% notice tip %}}
Với môi trường workshop/development, **$10/tháng** là ngân sách khởi đầu hợp lý. Với production, điều chỉnh dựa trên lượng traffic dự kiến và mẫu sử dụng dịch vụ.
{{% /notice %}}

5. (Tùy chọn) Áp dụng **bộ lọc** để giới hạn budget cho các dịch vụ cụ thể:

Bạn có thể lọc theo:
- **Service**: Chỉ theo dõi Lambda, DynamoDB, S3, v.v.
- **Tag**: Dùng tags để theo dõi chi phí cho dự án cụ thể
- **Region**: Lọc chỉ `ap-southeast-1`

![Bộ lọc Budget](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/budget-filter.png)

### Bước 2: Cấu hình Ngưỡng Cảnh báo

Thiết lập nhiều ngưỡng cảnh báo để nhận cảnh báo sớm:

#### Cảnh báo 1: Ngưỡng 50% (Cảnh báo sớm)

| Cài đặt | Giá trị |
|---------|---------|
| **Threshold** | 50% ngân sách ($5.00) |
| **Trigger** | Chi phí thực tế |
| **Notification** | Email |
| **Email recipients** | your-email@example.com |

#### Cảnh báo 2: Ngưỡng 80% (Cảnh báo)

| Cài đặt | Giá trị |
|---------|---------|
| **Threshold** | 80% ngân sách ($8.00) |
| **Trigger** | Chi phí thực tế |
| **Notification** | Email |
| **Email recipients** | your-email@example.com |

#### Cảnh báo 3: Ngưỡng 100% (Nghiêm trọng)

| Cài đặt | Giá trị |
|---------|---------|
| **Threshold** | 100% ngân sách ($10.00) |
| **Trigger** | Chi phí thực tế |
| **Notification** | Email |
| **Email recipients** | your-email@example.com |

#### Cảnh báo 4: Dự báo Vượt ngưỡng

| Cài đặt | Giá trị |
|---------|---------|
| **Threshold** | 100% ngân sách |
| **Trigger** | Chi phí dự báo |
| **Notification** | Email |
| **Email recipients** | your-email@example.com |

![Cấu hình Cảnh báo](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/alert-config.png)

{{% notice warning %}}
AWS Budgets alerts chỉ là **thông báo** — chúng không tự động dừng hoặc throttle dịch vụ. Nếu bạn nhận cảnh báo 100%, bạn phải chủ động hành động (ví dụ: tắt EventBridge scheduler, xóa tài nguyên tốn kém, hoặc giới hạn Lambda concurrency).
{{% /notice %}}

6. Click **Create budget**

![Budget đã tạo](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/create-budget.png)

### Bước 3: Theo dõi Cost Explorer

AWS Cost Explorer cung cấp phân tích chi phí chi tiết:

1. Truy cập **AWS Console → Billing → Cost Explorer**

2. Xem chi phí **nhóm theo dịch vụ** để xác định thành phần tốn kém nhất:

![Cost Explorer](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/cost-explorer.png)

3. Xem **xu hướng chi phí hàng ngày** để phát hiện đợt tăng bất thường:

![Xu hướng Hàng ngày](/images/5-Workshop/5.6-Operations-Security/5.6.4-Cost-Budget/daily-trend.png)

### Bước 4: Mẹo Tối ưu Chi phí

| Tối ưu | Hành động | Ảnh hưởng |
|--------|---------|-----------|
| **Lambda Memory** | Điều chỉnh bộ nhớ phù hợp (256MB mặc định có thể quá cao cho functions đơn giản) | Giảm chi phí Lambda |
| **DynamoDB TTL** | Đã bật trên MatchResultsTable — tự xóa kết quả cũ sau 90 ngày | Giảm chi phí lưu trữ |
| **S3 Lifecycle** | Thêm lifecycle rules chuyển CV cũ sang S3 Infrequent Access | Giảm chi phí lưu trữ |
| **EventBridge Schedule** | Giảm tần suất fetch hoặc tắt khi không cần | Giảm Lambda + SerpAPI calls |
| **Textract Calls** | Cache văn bản đã trích xuất để tránh xử lý lại cùng CV | Giảm chi phí Textract |
| **Gemini Model** | Dùng model nhẹ hơn (ví dụ: `gemini-3.1-flash-lite`) | Giảm chi phí đánh giá AI |

#### Cấu hình DynamoDB TTL (Đã áp dụng)

```yaml
# Trong template.yaml - MatchResultsTable
TimeToLiveSpecification:
  AttributeName: ttl
  Enabled: true
```

Cấu hình này tự động xóa kết quả matching sau khi TTL timestamp hết hạn (đặt 90 ngày trong code), giữ chi phí lưu trữ thấp mà không cần dọn dẹp thủ công.

### Xác minh với AWS CLI

```bash
# Liệt kê tất cả budgets
aws budgets describe-budgets \
  --account-id <your-account-id> \
  --query "Budgets[].{Name:BudgetName,Limit:BudgetLimit.Amount,Spent:CalculatedSpend.ActualSpend.Amount}" \
  --output table

# Lấy chi phí tháng hiện tại
aws ce get-cost-and-usage \
  --time-period Start=$(date -u +%Y-%m-01),End=$(date -u +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query "ResultsByTime[0].Groups[?Metrics.BlendedCost.Amount > '0'].{Service:Keys[0],Cost:Metrics.BlendedCost.Amount}" \
  --output table
```

### Tóm tắt

| Mục | Cấu hình |
|-----|---------|
| **Ngân sách Hàng tháng** | $10.00 (có thể điều chỉnh) |
| **Cảnh báo 50%** | Email thông báo khi chi $5.00 thực tế |
| **Cảnh báo 80%** | Email thông báo khi chi $8.00 thực tế |
| **Cảnh báo 100%** | Email thông báo khi chi $10.00 thực tế |
| **Cảnh báo Dự báo** | Email khi dự báo vượt ngân sách |
| **Tối ưu Chi phí** | DynamoDB TTL, Lambda đúng kích cỡ, AI models hiệu quả |

{{% notice tip %}}
Sau khi hoàn thành workshop, nhớ **dọn dẹp tất cả tài nguyên** (xem Phần 7) để tránh chi phí phát sinh. Ngay cả với Free Tier, một số dịch vụ (Textract, Gemini API) vẫn có thể phát sinh chi phí.
{{% /notice %}}
