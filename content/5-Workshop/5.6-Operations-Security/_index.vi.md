---
title : "Vận hành & Bảo mật"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

## Vận hành, Bảo mật & Tối ưu Chi phí

Phần này đề cập đến các mối quan tâm xuyên suốt áp dụng cho toàn bộ nền tảng Jobs Matching Platform. Trong khi các phần trước tập trung vào việc xây dựng từng phân hệ riêng lẻ (Jobs Ingestion, User Interaction, CV Matching Flow), phần này đảm bảo nền tảng vận hành **an toàn**, **có khả năng giám sát**, và **tối ưu chi phí**.

![Kiến trúc Vận hành & Bảo mật](/images/2-Proposal/Jobs-Matching-Platform -v3.drawio.png)

Trong sơ đồ kiến trúc, tầng dưới cùng — **Cost, Security, Operation** — trải rộng trên cả ba phân hệ chức năng. Nó bao gồm bốn trụ cột chính:

| Trụ cột | Dịch vụ AWS | Mục đích |
|---------|-------------|----------|
| **IAM** | AWS IAM | Quản lý định danh và quyền truy cập giữa các dịch vụ nội bộ |
| **API Keys** | AWS Systems Manager Parameter Store | Lưu trữ và quản lý bí mật, khóa API an toàn |
| **Logs/Alarm** | Amazon CloudWatch | Thu thập log hệ thống, metrics hiệu năng và cảnh báo tự động |
| **Budget** | AWS Budgets | Theo dõi chi phí, thiết lập ngân sách và cảnh báo chi tiêu |

### Tại sao điều này quan trọng

Trong kiến trúc serverless như dự án này, không có máy chủ đơn lẻ nào để quản lý, nhưng **phạm vi ảnh hưởng của cấu hình sai** có thể rất lớn:

- **IAM roles quá mở** có thể làm lộ bảng DynamoDB, S3 bucket, hoặc cho phép gọi Lambda trái phép.
- **Hardcode API keys** (SerpAPI, Gemini) trong mã nguồn tạo ra lỗ hổng bảo mật khi code được đẩy lên Git.
- **Không có giám sát** đồng nghĩa bạn sẽ không biết khi Lambda function lỗi, message SQS rơi vào DLQ, hoặc cuộc gọi Textract tăng đột biến.
- **Không có cảnh báo ngân sách** có thể dẫn đến hóa đơn AWS bất ngờ, đặc biệt với các dịch vụ như Gemini API, Textract, hoặc DynamoDB đọc với volume cao.

### Bạn sẽ học được gì

Trong phần này, bạn sẽ:

1. **[5.6.1 - IAM Policies](5.6.1-iam-policies/)**: Xem xét và hiểu nguyên tắc Principle of Least Privilege được áp dụng cho mọi Lambda function trong SAM template.
2. **[5.6.2 - Quản lý API Keys](5.6.2-api-keys-management/)**: Lưu trữ và truy xuất an toàn các khóa nhạy cảm (SerpAPI, Gemini API Key) bằng AWS Systems Manager Parameter Store.
3. **[5.6.3 - Giám sát & Cảnh báo](5.6.3-monitoring-alarms/)**: Thiết lập CloudWatch Logs, Metrics và Alarms để giám sát lỗi Lambda, SQS Dead Letter Queue và độ trễ API Gateway.
4. **[5.6.4 - Chi phí & Ngân sách](5.6.4-cost-budget/)**: Cấu hình AWS Budgets để thiết lập ngưỡng chi tiêu và nhận thông báo email trước khi chi phí vượt quá giới hạn.

### Nội dung

1. [IAM Policies](5.6.1-iam-policies/)
2. [Quản lý API Keys](5.6.2-api-keys-management/)
3. [Giám sát & Cảnh báo](5.6.3-monitoring-alarms/)
4. [Chi phí & Ngân sách](5.6.4-cost-budget/)
