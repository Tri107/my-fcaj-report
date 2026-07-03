---
title: "Blog 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---
# Xây dựng AI Gateway cho Amazon Bedrock bằng Amazon API Gateway

Khi xây dựng các ứng dụng Generative AI, các doanh nghiệp phải đối mặt với một thách thức lớn: quản trị việc sử dụng mô hình nền tảng (foundation model) thông qua xác thực, quản lý hạn ngạch (quota), cô lập người thuê (tenant isolation) và kiểm soát chi phí. Để đáp ứng những nhu cầu này, kiến trúc **AI Gateway** đã nổi lên như một giải pháp chuẩn mực giúp các tổ chức kiểm soát truy cập vào các dịch vụ **Amazon Bedrock** ở quy mô lớn.

Mô hình này sử dụng **Amazon API Gateway** làm lớp truy cập (access layer) nằm trước Amazon Bedrock, cung cấp các tính năng như xác thực yêu cầu, hạn ngạch sử dụng, quản lý vòng đời ứng dụng, và phản hồi luồng (response streaming) theo thời gian thực.

---

## Kiến Trúc Tham Khảo (Reference Architecture)

![Kiến trúc tham khảo AI Gateway](/images/3-BlogsTranslated/Architecture-Blog1.png)

Kiến trúc này cung cấp khả năng kiểm soát chi tiết quyền truy cập vào các Mô hình Ngôn ngữ Lớn (LLM) bằng các dịch vụ được quản lý hoàn toàn của AWS. Nó hoạt động một cách vô hình (transparent) đối với các ứng dụng máy khách (như AWS SDKs/Boto3) và tích hợp liền mạch vào các môi trường doanh nghiệp hiện tại.

Giải pháp bao gồm 5 thành phần cốt lõi:
1. **Amazon Route 53 (Tùy chọn):** Quản lý định tuyến tên miền tùy chỉnh, cho phép máy khách truy cập gateway thông qua endpoint riêng của công ty.
2. **Amazon API Gateway:** Đóng vai trò là điểm vào (entry point) chính, cung cấp các khả năng như kiểm soát lưu lượng (throttling), API keys và quản lý vòng đời.
3. **AWS Lambda Authorizer:** Xử lý xác thực yêu cầu bằng cách đối chiếu token JWT với các hệ thống xác thực hiện có (ví dụ: Amazon Cognito).
4. **Lambda Integration:** Hoạt động như một bộ chuyển tiếp yêu cầu động, tiến hành ký (sign) các yêu cầu bằng thông tin xác thực AWS (SigV4) và định tuyến chúng đến đúng các endpoint của Amazon Bedrock, trong khi vẫn giữ nguyên các chi tiết của yêu cầu gốc.
5. **Amazon Bedrock:** Cung cấp quyền truy cập thực tế vào các mô hình nền tảng và các khả năng AI.

> **Tại sao chọn thiết kế này?** Lợi ích lớn nhất là tính hướng tới tương lai (future-proof). Máy khách có thể tương tác với Amazon Bedrock chính xác như khi họ gọi API trực tiếp. AI Gateway sẽ âm thầm xử lý toàn bộ các khâu quản trị phức tạp ở hậu trường mà không yêu cầu cập nhật code mỗi khi Bedrock ra mắt tính năng mới.

---

## Triển Khai & Kiểm Thử

Cách nhanh nhất để triển khai cơ sở hạ tầng cốt lõi (API Gateway, Lambda, VPC endpoints) là thông qua **AWS CloudFormation**.

Khi đã được triển khai nội bộ trong một VPC, bạn có thể kiểm thử nó bằng môi trường **AWS CloudShell**:
1. Tạo một `boto3 client factory` bằng Python để định tuyến các yêu cầu qua API Gateway của bạn trong khi vẫn duy trì interface chuẩn của `boto3`.
2. Kiểm thử khả năng suy luận của mô hình (ví dụ: sử dụng API `ConverseStream`) hoặc truy xuất dữ liệu từ cơ sở tri thức (Knowledge Base).
3. Sau khi xác minh các chức năng cơ bản, hãy kích hoạt **Lambda Authorizer** bằng cách cập nhật CloudFormation stack để triển khai logic xác thực JWT tùy chỉnh, đảm bảo mọi lệnh gọi API đều được bảo mật.

---

## Các Tùy Chọn Nâng Cấp cho AI Gateway

Để triển khai quản trị AI ở quy mô lớn, kiến trúc tham khảo này có thể được mở rộng bằng các tính năng bổ sung của API Gateway:

- **Giới Hạn Tốc Độ & Throttling (Rate Limiting):** Kiểm soát tốc độ yêu cầu bằng usage plans và API keys để ngăn chặn vấn đề "người hàng xóm ồn ào" (noisy neighbor) trong các ứng dụng multi-tenant SaaS.
- **Endpoint Riêng Tư hoặc Tối Ưu Hóa Biên (Edge-Optimized):** Tối ưu hóa cho truy cập mạng nội bộ hoặc hiệu suất toàn cầu tùy thuộc vào vị trí của máy khách.
- **Canary Releases:** Quản lý nhiều phiên bản API và triển khai cuốn chiếu (gradual rollouts) thông qua stage variables và canary deployments.
- **Tích hợp AWS WAF:** Bảo vệ các endpoint AI khỏi các lỗ hổng web phổ biến bằng cách thêm các bộ quy tắc AWS WAF.
- **Bộ Đệm Prompt và Phản Hồi (Caching):** Giảm chi phí và cải thiện thời gian phản hồi cho các câu lệnh thường xuyên được yêu cầu bằng cách sử dụng bộ đệm (caching) của API Gateway.
- **Lọc Nội Dung (Content Filtering):** Thêm bộ lọc tùy chỉnh trong lớp Lambda Integration để quét các dữ liệu nhạy cảm (như PII), nhằm bổ sung cho các cơ chế bảo vệ sẵn có từ **Amazon Bedrock Guardrails**.

---

*Bài viết gốc:* [Building an AI gateway to Amazon Bedrock with Amazon API Gateway](https://aws.amazon.com/blogs/architecture/building-an-ai-gateway-to-amazon-bedrock-with-amazon-api-gateway/)
