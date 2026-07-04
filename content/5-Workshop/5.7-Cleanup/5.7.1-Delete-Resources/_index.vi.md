   ---
title : "Xóa Tài nguyên"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.7.1 </b> "
---

## Hướng dẫn Xóa Tài nguyên từng bước

Phần này hướng dẫn bạn quy trình dọn dẹp hoàn chỉnh cho Jobs Matching Platform. Thực hiện các bước **theo thứ tự**, vì một số tài nguyên phụ thuộc vào việc tài nguyên khác bị xóa trước.

### Thứ tự Xóa

Thứ tự xóa được khuyến nghị để đảm bảo giải quyết đúng phụ thuộc:

```
1. Amplify (Frontend Hosting)
2. WAF Web ACL
3. CloudWatch Alarms & Dashboard
4. AWS Budgets
5. SSM Parameters
6. S3 Bucket (làm rỗng trước, sau đó xóa)
7. SAM Stack (xóa Lambda, API Gateway, DynamoDB, SQS, Cognito, EventBridge, IAM)
8. CloudWatch Log Groups (dọn dẹp phần còn lại)
9. SAM Deployment Bucket
```

---

### Bước 1: Xóa Amplify Frontend Application

1. Truy cập **AWS Console → AWS Amplify**
2. Chọn ứng dụng Jobs Matching Platform
3. Click **Actions → Delete app**

![Xóa Amplify](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-amplify.png)

4. Nhập **delete** để xác nhận

![Xác nhận Xóa Amplify](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/confirm-delete-amplify.png)

---

### Bước 2: Xóa WAF Web ACL

1. Truy cập **AWS Console → WAF & Shield → Web ACLs**
2. Chọn Web ACL được gắn với API Gateway
3. Đầu tiên, **hủy liên kết** (disassociate) khỏi tài nguyên API Gateway
4. Sau đó click **Delete**

![Xóa WAF](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-waf.png)

{{% notice info %}}
Bạn phải hủy liên kết Web ACL khỏi tất cả tài nguyên trước khi xóa nó. Nếu xóa thất bại, kiểm tra các liên kết còn lại.
{{% /notice %}}

---

### Bước 3: Xóa CloudWatch Alarms & Dashboard

#### Xóa Alarms

1. Truy cập **AWS Console → CloudWatch → Alarms**
2. Chọn tất cả alarms được tạo trong workshop (ví dụ: `jobs-matching-*`)
3. Click **Actions → Delete**

![Xóa Alarms](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-alarms.png)

Hoặc dùng AWS CLI:

```bash
# Xóa tất cả alarms theo tên prefix
aws cloudwatch delete-alarms \
  --alarm-names \
    "jobs-matching-evaluate-match-errors" \
    "jobs-matching-dlq-messages" \
    "jobs-matching-api-5xx-errors"
```

#### Xóa Dashboard (nếu đã tạo)

```bash
aws cloudwatch delete-dashboards \
  --dashboard-names "jobs-matching-platform-dev"
```

---

### Bước 4: Xóa AWS Budgets

1. Truy cập **AWS Console → Billing → Budgets**
2. Chọn `jobs-matching-platform-monthly`
3. Click **Actions → Delete**

![Xóa Budget](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-budget.png)

Hoặc dùng CLI:

```bash
aws budgets delete-budget \
  --account-id <your-account-id> \
  --budget-name "jobs-matching-platform-monthly"
```

---

### Bước 5: Xóa SSM Parameters

Xóa các API keys được lưu trong Parameter Store:

```bash
# Xóa SerpAPI key
aws ssm delete-parameter \
  --name "/jobs-matching/dev/serpapi-key"

# Xóa Gemini API key
aws ssm delete-parameter \
  --name "/jobs-matching/dev/gemini-api-key"
```

Hoặc qua console:

1. Truy cập **AWS Console → Systems Manager → Parameter Store**
2. Chọn các parameters khớp `/jobs-matching/dev/*`
3. Click **Delete**

![Xóa SSM](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-ssm.png)

---

### Bước 6: Làm rỗng và Xóa S3 Bucket

{{% notice warning %}}
S3 buckets phải được **làm rỗng trước khi xóa**. CloudFormation/SAM không thể xóa bucket còn chứa dữ liệu.
{{% /notice %}}

#### Cách A: Làm rỗng qua AWS Console

1. Truy cập **AWS Console → S3**
2. Chọn CV bucket (`jobs-matching-cvs-dev-<account-id>`)
3. Click **Empty**
4. Nhập **permanently delete** để xác nhận

![Làm rỗng S3](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/empty-s3.png)

#### Cách B: Làm rỗng qua AWS CLI

```bash
# Làm rỗng CV bucket
aws s3 rm s3://jobs-matching-cvs-dev-<account-id> --recursive

# Xác minh bucket đã rỗng
aws s3 ls s3://jobs-matching-cvs-dev-<account-id>
```

![CLI Làm rỗng S3](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/cli-empty-s3.png)

{{% notice info %}}
KHÔNG xóa thủ công S3 bucket sau khi làm rỗng — lệnh SAM delete ở bước tiếp theo sẽ xử lý việc xóa bucket như một phần của CloudFormation stack cleanup.
{{% /notice %}}

---

### Bước 7: Xóa SAM Stack (Dọn dẹp chính)

Đây là bước quan trọng nhất. Lệnh `sam delete` xóa **toàn bộ CloudFormation stack**, bao gồm:

- ✅ Tất cả Lambda functions
- ✅ API Gateway (HTTP API)
- ✅ DynamoDB tables (Jobs, SavedJobs, MatchResults)
- ✅ SQS queues (RawJobsQueue, RawJobsDLQ)
- ✅ EventBridge Scheduler
- ✅ Cognito User Pool, Client, Identity Provider
- ✅ S3 bucket (nếu đã rỗng)
- ✅ IAM execution roles cho tất cả Lambda functions

#### Chạy SAM Delete

```bash
cd backend

# Xóa SAM stack
sam delete --stack-name jobs-matching --no-prompts
```

![SAM Delete](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/sam-delete.png)

Lệnh sẽ:
1. Xóa CloudFormation stack và tất cả tài nguyên của nó
2. Xóa SAM deployment S3 bucket (nếu dùng `--no-prompts`)

#### Kết quả Mong đợi

```
	Deleting the local artifacts and cache...
	Are you sure you want to delete the stack jobs-matching in the region ap-southeast-1 ? [y/N]: 
	Are you sure you want to delete the folder jobs-matching in S3 which contains the artifacts? [y/N]: 
	- Deleting S3 object with key jobs-matching/abc123
	- Deleting S3 object with key jobs-matching/def456
	- Deleting Cloudformation stack jobs-matching

Successfully deleted stack: jobs-matching
```

![SAM Delete Thành công](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/sam-delete-success.png)

{{% notice warning %}}
Nếu `sam delete` thất bại với lỗi về S3 bucket chưa rỗng, quay lại **Bước 6** và đảm bảo CV bucket hoàn toàn rỗng. Sau đó thử lại.
{{% /notice %}}

#### Xác minh Stack đã Xóa

```bash
# Xác minh stack không còn tồn tại
aws cloudformation describe-stacks \
  --stack-name jobs-matching 2>&1

# Kết quả mong đợi: "Stack with id jobs-matching does not exist"
```

---

### Bước 8: Xóa CloudWatch Log Groups (Dọn dẹp phần còn lại)

CloudWatch Log groups **không bị xóa** khi Lambda functions bị remove. Chúng phải được xóa riêng:

```bash
# Liệt kê tất cả log groups của dự án
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/lambda/jobs-matching" \
  --query "logGroups[].logGroupName" \
  --output text

# Xóa từng log group
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

Hoặc xóa tất cả cùng lúc bằng script:

```bash
# Xóa tất cả log groups theo prefix
for log_group in $(aws logs describe-log-groups \
  --log-group-name-prefix "/aws/lambda/jobs-matching" \
  --query "logGroups[].logGroupName" \
  --output text); do
  echo "Đang xóa $log_group"
  aws logs delete-log-group --log-group-name "$log_group"
done
```

![Xóa Log Groups](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/delete-log-groups.png)

---

### Bước 9: Xác minh Dọn dẹp Hoàn tất

Chạy các lệnh kiểm tra sau để đảm bảo tất cả tài nguyên đã bị xóa:

```bash
# 1. Kiểm tra CloudFormation stacks
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[?contains(StackName, 'jobs-matching')]" \
  --output table

# 2. Kiểm tra Lambda functions
aws lambda list-functions \
  --query "Functions[?contains(FunctionName, 'jobs-matching')]" \
  --output table

# 3. Kiểm tra DynamoDB tables
aws dynamodb list-tables \
  --query "TableNames[?contains(@, 'jobs-matching')]" \
  --output text

# 4. Kiểm tra S3 buckets
aws s3api list-buckets \
  --query "Buckets[?contains(Name, 'jobs-matching')]" \
  --output table

# 5. Kiểm tra SQS queues
aws sqs list-queues \
  --queue-name-prefix "jobs-matching" \
  --output text

# 6. Kiểm tra SSM parameters
aws ssm get-parameters-by-path \
  --path "/jobs-matching" \
  --recursive \
  --output text
```

Tất cả lệnh đều phải trả về kết quả rỗng, xác nhận dọn dẹp hoàn tất.

![Xác minh Dọn dẹp](/images/5-Workshop/5.7-Cleanup/5.7.1-Delete-Resources/verify-cleanup.png)

---

### Checklist Tóm tắt Dọn dẹp

| # | Tài nguyên | Phương pháp | Trạng thái |
|---|-----------|------------|-----------|
| 1 | Amplify App | Console Delete | ☐ |
| 2 | WAF Web ACL | Console Delete | ☐ |
| 3 | CloudWatch Alarms | Console / CLI | ☐ |
| 4 | CloudWatch Dashboard | Console / CLI | ☐ |
| 5 | AWS Budgets | Console / CLI | ☐ |
| 6 | SSM Parameters | CLI: `ssm delete-parameter` | ☐ |
| 7 | S3 Bucket (làm rỗng) | CLI: `s3 rm --recursive` | ☐ |
| 8 | SAM Stack | CLI: `sam delete` | ☐ |
| 9 | CloudWatch Log Groups | CLI: `logs delete-log-group` | ☐ |
| 10 | Xác minh | CLI: nhiều lệnh kiểm tra | ☐ |

{{% notice tip %}}
In checklist này hoặc mở sẵn trong khi thực hiện dọn dẹp để đảm bảo không bỏ sót tài nguyên nào. Chỉ cần bỏ lỡ một tài nguyên (như DynamoDB table hoặc CloudWatch log group) có thể dẫn đến chi phí phát sinh liên tục.
{{% /notice %}}
