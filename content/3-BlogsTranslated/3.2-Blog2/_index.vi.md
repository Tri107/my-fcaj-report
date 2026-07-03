---
title: "Blog 2"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---
# Tối ưu hóa hoạt động đám mây với sức mạnh Kiro cho AWS DevOps Agent

Khi có cảnh báo lúc 2 giờ sáng, việc đầu tiên các kỹ sư thường làm là tìm kiếm log, kiểm tra các lần triển khai gần đây và dò tìm đường dẫn code. Tuy nhiên, ngữ cảnh họ cần — metrics, traces, cấu trúc mạng (topology), cấu hình — lại nằm ở các tab trình duyệt và ứng dụng riêng biệt. **Sức mạnh Kiro (Kiro power) cho AWS DevOps Agent** loại bỏ việc chuyển đổi ngữ cảnh đó bằng cách kết nối IDE của bạn trực tiếp với AWS DevOps Agent, cho phép bạn điều tra sự cố, xác định nguyên nhân gốc rễ và tạo bản sửa lỗi ngay tại nơi bạn viết code.

---

## Những thách thức trong hoạt động đám mây hiện nay

Việc vận hành các ứng dụng đám mây hiện đại đồng nghĩa với việc phải xử lý một ma trận các dịch vụ liên kết với nhau. Người vận hành phải đối mặt với các thách thức liên tục:
- **Chuyển đổi ngữ cảnh**: Việc điều tra sự cố đòi hỏi phải chuyển đổi liên tục giữa IDE, AWS Management Console, trình xem log, và tài liệu.
- **Kiến thức bị cô lập**: Việc hiểu metric nào là quan trọng thường chỉ nằm trong các runbook lỗi thời hoặc trong đầu của các kỹ sư cấp cao.
- **Khoảng cách khắc phục (Remediation gap)**: Việc dịch các phát hiện thành một bản sửa lỗi hoạt động được lại yêu cầu chuyển đổi ngữ cảnh và áp dụng thay đổi theo cách thủ công.

---

## Những thách thức trong phân phối phần mềm hiện đại

Các công cụ lập trình AI đã tăng tốc quá trình viết code, nhưng quy trình đánh giá code và pipeline chưa bắt kịp:
- **Năng lực đánh giá**: Quá trình phát triển có sự hỗ trợ của AI tạo ra các thay đổi nhanh hơn khả năng con người có thể đánh giá, dẫn đến rủi ro sai lệch tiêu chuẩn.
- **Sự phụ thuộc vô hình**: Một thay đổi nhỏ ở một repository có thể âm thầm làm hỏng các hệ thống phụ thuộc phía sau.

---

## Kiro powers là gì?

Kiro power là một gói được tinh chỉnh mang lại cho Kiro các khả năng chuyên biệt trong một lĩnh vực cụ thể. Mỗi power thường bao gồm:
- **Cấu hình máy chủ MCP**: Kết nối Kiro với các công cụ và dữ liệu bên ngoài thông qua Model Context Protocol.
- **Các file điều hướng (Steering files)**: Hướng dẫn dành riêng cho lĩnh vực giúp Kiro định tuyến các ý định.
- **Kiến thức theo ngữ cảnh**: Hướng dẫn cụ thể theo lĩnh vực được lưu giữ trong các tệp markdown và lifecycle hooks.

---

## Kiro power cho AWS DevOps Agent

![Kiro Power Architecture](/images/3-blogstranslated/3.2-blog2/Architecture-Blog2.png)

Gói sức mạnh Kiro cho AWS DevOps Agent tích hợp toàn bộ khả năng của AWS DevOps Agent vào một bản cài đặt duy nhất. Các tính năng bao gồm:
- **Điều tra sự cố**: Mô tả triệu chứng bằng ngôn ngữ tự nhiên, và Kiro sẽ điều phối một cuộc điều tra chuyên sâu.
- **Tối ưu hóa chi phí**: Yêu cầu tiết kiệm chi phí và nhận được các đề xuất cụ thể, dựa trên dữ liệu.
- **Đánh giá kiến trúc**: Yêu cầu sơ đồ cấu trúc mạng hoặc kiểm toán bảo mật với các đề xuất cải thiện thực tế.
- **Trò chuyện trên nhiều không gian tác nhân (agent spaces)**: Hoạt động trên nhiều không gian AWS DevOps Agent từ một phiên Kiro duy nhất.
- **Tạo mã khắc phục**: Kiro có thể tạo bản sửa lỗi trực tiếp trong workspace của bạn.
- **Đánh giá mức độ sẵn sàng phát hành**: Yêu cầu DevOps Agent đánh giá các thay đổi về rủi ro phụ thuộc và độ lệch tiêu chuẩn trước khi đẩy code.
- **Thực hiện kiểm thử phát hành thăm dò**: Kiro có thể chạy các bài kiểm thử trên các ứng dụng đã được triển khai.

---

## Hướng dẫn từng bước: Điều tra sự cố môi trường production

1. **Mô tả vấn đề**: Báo cho Kiro về sự cố (ví dụ: "Dịch vụ ECS của tôi đang báo lỗi 503"). Kiro sẽ tự động đính kèm ngữ cảnh workspace liên quan.
2. **Kiro bắt đầu điều tra**: Kiro truy vấn CloudWatch, X-Ray và các sự kiện ECS, phân tích các metrics so với số lượng task.
3. **Xem xét các phát hiện**: DevOps Agent trả về phân tích nguyên nhân gốc rễ chi tiết và đề xuất cách giảm thiểu rủi ro.
4. **Tạo và áp dụng bản sửa lỗi**: Kiro triển khai đề xuất trực tiếp vào các tệp cấu hình của bạn (ví dụ: cập nhật kích thước pool kết nối cơ sở dữ liệu) sẵn sàng để xem xét và commit.

---

*Bài viết gốc:* [Supercharge your cloud operations with the Kiro power for AWS DevOps Agent](https://aws.amazon.com/blogs/devops/supercharge-your-cloud-operations-with-the-kiro-power-for-aws-devops-agent/)