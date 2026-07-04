---
title : "Chính sách IAM"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.6.1 </b> "
---

## Nguyên tắc Quyền hạn Tối thiểu cho Lambda Functions

AWS Identity and Access Management (IAM) là nền tảng bảo mật trong AWS. Mọi tương tác giữa các dịch vụ trong Jobs Matching Platform đều được quản lý bởi IAM policies tuân theo **Nguyên tắc Quyền hạn Tối thiểu (Principle of Least Privilege)** — mỗi Lambda function chỉ được cấp quyền tối thiểu cần thiết để thực hiện tác vụ cụ thể của nó.



Trong nền tảng này, IAM kiểm soát các tương tác quan trọng sau:

- **Lambda → DynamoDB**: Quyền đọc/ghi được giới hạn cho từng bảng cụ thể
- **Lambda → S3**: Thao tác cấp đối tượng được giới hạn cho CV bucket
- **Lambda → SQS**: Quyền gửi message được giới hạn cho hàng đợi raw jobs
- **Lambda → Textract**: Trích xuất văn bản từ tài liệu
- **Lambda → Gemini API**: Đánh giá AI qua HTTP API bên ngoài (sử dụng API key từ SSM)
- **Lambda → SSM Parameter Store**: Quyền đọc giới hạn cho đường dẫn parameter cụ thể
- **Lambda → Lambda**: Quyền invoke giữa ProcessJobs và NormalizeAndMatch



### Hiểu về SAM Policy Templates

AWS SAM (Serverless Application Model) cung cấp **policy templates** giúp đơn giản hóa việc tạo IAM policy. Thay vì viết raw JSON IAM policies, SAM cung cấp các tham chiếu rút gọn tự động tạo ra policies được giới hạn đúng phạm vi.

#### SAM Policy Templates được sử dụng trong dự án

| SAM Policy Template | Mô tả | Sử dụng bởi |
|---------------------|-------|-------------|
| `DynamoDBReadPolicy` | Quyền đọc cho một bảng DynamoDB cụ thể | GetJobs, GetJobById, SearchJobs, EvaluateMatch, GetMatchResults |
| `DynamoDBCrudPolicy` | Quyền CRUD đầy đủ cho một bảng DynamoDB cụ thể | NormalizeAndMatch, SavedJobs, EvaluateMatch |
| `SQSSendMessagePolicy` | Quyền gửi message vào một hàng đợi SQS cụ thể | FetchJobs |
| `LambdaInvokePolicy` | Quyền invoke một Lambda function cụ thể | ProcessJobs |

#### Ví dụ: GetJobsFunction — Quyền Đọc

Hàm `GetJobsFunction` chỉ cần đọc từ bảng Jobs. SAM policy của nó:

```yaml
# Trong template.yaml - GetJobsFunction
Policies:
  - DynamoDBReadPolicy:
      TableName: !Ref JobsTable
```

Câu lệnh trên tự động tạo ra IAM policy tương đương:

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
Template `DynamoDBReadPolicy` tự động bao gồm quyền truy cập vào các indexes (GSI/LSI) của bảng, đó là lý do tại sao `SearchJobsFunction` cũng có thể dùng nó để query `HashIndex`.
{{% /notice %}}

#### Ví dụ: FetchJobsFunction — Chỉ Gửi SQS

Hàm `FetchJobsFunction` thu thập jobs từ SerpAPI và đẩy vào SQS. Nó chỉ cần quyền `sqs:SendMessage`:

```yaml
# Trong template.yaml - FetchJobsFunction
Policies:
  - SQSSendMessagePolicy:
      QueueName: !GetAtt RawJobsQueue.QueueName
```

Hàm này **KHÔNG** có quyền truy cập DynamoDB — nó chỉ đẩy dữ liệu thô vào hàng đợi. Hàm `NormalizeAndMatchFunction` ở downstream chịu trách nhiệm ghi vào DynamoDB.

#### Ví dụ: EvaluateMatchFunction — Custom Policy Statements

Một số hàm cần quyền mà SAM templates không hỗ trợ sẵn. Với những trường hợp này, chúng ta viết custom IAM policy statements:

```yaml
# Trong template.yaml - EvaluateMatchFunction
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

**Những điểm quan trọng:**

1. **Quyền S3 chỉ đọc** (`GetObject`, `HeadObject`, `ListBucket`) — hàm này đọc CV nhưng không thể sửa hoặc xóa chúng.
2. **Quyền SSM Parameter được giới hạn** chính xác đến đường dẫn `/jobs-matching/dev/gemini-api-key` — không thể đọc các parameter khác.
3. **Textract** sử dụng `Resource: "*"` vì dịch vụ này không hỗ trợ giới hạn ở cấp resource. **Gemini API** được gọi bên ngoài qua HTTP (không phải dịch vụ AWS), nên không cần quyền IAM — chỉ cần API key lưu trong SSM.

### Tổng quan: Ma trận Quyền IAM đầy đủ

| Lambda Function | DynamoDB | S3 | SQS | Textract | Gemini (qua SSM) | SSM | Lambda |
|----------------|----------|-----|-----|----------|-----------------|-----|--------|
| HealthCheck | — | — | — | — | — | — | — |
| GetJobs | Đọc (Jobs) | — | — | — | — | — | — |
| GetJobById | Đọc (Jobs) | — | — | — | — | — | — |
| SearchJobs | Đọc (Jobs) | — | — | — | — | — | — |
| FetchJobs | — | — | Gửi | — | — | — | — |
| ProcessJobs | — | — | — | — | — | — | Invoke |
| NormalizeAndMatch | CRUD (Jobs) | — | — | — | — | — | — |
| SavedJobs | CRUD (SavedJobs) | — | — | — | — | — | — |
| EvaluateMatch | Đọc (Jobs) + CRUD (MatchResults) | Đọc | — | ✓ | ✓ (API Key) | Đọc | — |
| GetMatchResults | Đọc (MatchResults) | — | — | — | — | — | — |
| ExtractCv | — | Đọc | — | ✓ | — | — | — |
| ManageCv | — | CRUD | — | — | — | — | — |

### Kiểm tra IAM Policies trên AWS Console

1. Truy cập **AWS Console → IAM → Roles**

2. Tìm kiếm role của Lambda function (ví dụ: tìm `jobs-matching-backend` để xem tất cả roles của stack)

![Tìm kiếm IAM Roles](/images/5-Workshop/5.6-Operations-Security/5.6.1-IAM-Policies/iam-roles-search.png)

3. Click vào role để xem các policies được gắn

![Chi tiết IAM Role](/images/5-Workshop/5.6-Operations-Security/5.6.1-IAM-Policies/iam-role-detail.png)

4. Mở rộng từng policy để xác minh quyền và tài nguyên cụ thể

![Chi tiết IAM Policy](/images/5-Workshop/5.6-Operations-Security/5.6.1-IAM-Policies/iam-policy-detail.png)

{{% notice tip %}}
Bạn cũng có thể dùng AWS CLI để kiểm tra IAM policies gắn với Lambda execution role:
```bash
# Lấy ARN execution role của Lambda function
aws lambda get-function-configuration \
  --function-name jobs-matching-get-jobs-dev \
  --query "Role" --output text

# Liệt kê các policies gắn với role
aws iam list-attached-role-policies \
  --role-name <tên-role-ở-trên>

# Xem chi tiết policy document
aws iam get-role-policy \
  --role-name <tên-role> \
  --policy-name <tên-policy>
```
{{% /notice %}}

### Các Nguyên tắc Tốt nhất đã áp dụng

| Nguyên tắc | Cách áp dụng |
|-----------|-------------|
| **Quyền hạn Tối thiểu** | Mỗi hàm chỉ nhận quyền mà nó cần |
| **Giới hạn Tài nguyên** | Policies nhắm đến bảng, bucket và parameter cụ thể — không dùng `*` |
| **SAM Templates** | Sử dụng SAM policy templates cho các mẫu phổ biến thay vì viết JSON thủ công |
| **Không Chia sẻ Roles** | Mỗi Lambda function có execution role riêng (hành vi mặc định của SAM) |
| **Không Nhúng Secrets** | API keys được truy xuất từ SSM lúc runtime, không nhúng trong IAM policies |

{{% notice warning %}}
Tránh sử dụng `Resource: "*"` bất cứ khi nào có thể. Trong dự án này, `Textract` yêu cầu nó vì không hỗ trợ giới hạn ở cấp tài nguyên. `Gemini API` được gọi bên ngoài qua HTTP và không cần quyền IAM — chỉ cần API key được truy xuất từ SSM Parameter Store. Với tất cả dịch vụ khác (DynamoDB, S3, SQS, SSM), luôn chỉ định chính xác ARN tài nguyên.
{{% /notice %}}
