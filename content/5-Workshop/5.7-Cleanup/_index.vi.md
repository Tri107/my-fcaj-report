---
title : "Dọn dẹp tài nguyên"
date : 2024-01-01
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---

## Dọn dẹp Tài nguyên

Sau khi hoàn thành workshop, điều cần thiết là **xóa tất cả tài nguyên AWS** để tránh chi phí phát sinh. Mặc dù nhiều dịch vụ nằm trong AWS Free Tier, một số tài nguyên (như DynamoDB tables, S3 buckets, và CloudWatch alarms) sẽ tiếp tục phát sinh chi phí nếu để chạy.

Phần này cung cấp hướng dẫn toàn diện để xóa tất cả tài nguyên được tạo trong workshop.

{{% notice warning %}}
**Hành động này không thể hoàn tác.** Tất cả dữ liệu trong DynamoDB tables, S3 buckets, và Cognito user pools sẽ bị xóa vĩnh viễn. Hãy chắc chắn export bất kỳ dữ liệu nào bạn cần trước khi tiến hành.
{{% /notice %}}

### Tài nguyên cần Xóa

Các tài nguyên sau đã được tạo trong workshop:

| Danh mục | Tài nguyên |
|----------|-----------|
| **SAM Stack** | Tất cả Lambda functions, API Gateway, DynamoDB tables, SQS queues, EventBridge rules, Cognito, IAM roles |
| **S3 Buckets** | CV Storage bucket, SAM deployment bucket |
| **SSM Parameters** | SerpAPI key, Gemini API key |
| **CloudWatch** | Log groups, Alarms, Dashboards |
| **AWS Budgets** | Cấu hình budget và cảnh báo |
| **Amplify** | Ứng dụng hosting Frontend |
| **WAF** | Web ACL rules |

### Nội dung

1. [Xóa Tài nguyên](5.7.1-delete-resources/)
