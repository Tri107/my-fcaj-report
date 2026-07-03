---
title : "Các bước chuẩn bị"
date : 2026-07-02 
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

Trước khi bắt đầu khởi tạo và xây dựng dự án, chúng ta cần chuẩn bị một số tài nguyên và cấu hình môi trường cần thiết. Các bước chuẩn bị này được chia làm ba phần chính sau đây:

#### 1. Tài khoản AWS & Quyền IAM
Bạn cần một tài khoản AWS đang hoạt động với các quyền IAM phù hợp để triển khai hạ tầng đám mây. Đảm bảo rằng user hoặc role của bạn có đủ quyền đối với:
* **IAM Roles & Policies**: Để tạo/chỉnh sửa instance profile và execution role.
* **Mạng & Tính toán (Network & Compute)**: Các tài nguyên EC2, VPC, Route 53, và Lambda.
* **Lưu trữ & Bảo mật (Storage & Security)**: S3 bucket và Secrets Manager.

Để xem hướng dẫn chi tiết các bước thiết lập tài khoản, vui lòng tham khảo [AWS IAM Account](5.2.1-AWS-IAM-Account).

#### 2. Cài đặt Môi trường & Công cụ Local
Để tương tác với AWS và xây dựng/triển khai dự án, bạn cần cài đặt và cấu hình các công cụ cốt lõi sau trên máy cá nhân:
* **AWS CLI**: Đã được cấu hình thông tin xác thực (credentials) AWS của bạn.
* **Git**: Dùng cho quản lý mã nguồn và phiên bản.
* **IDE (ví dụ: VS Code)**: Được trang bị các extension cần thiết cho việc phát triển.

Để xem hướng dẫn cài đặt và cấu hình chi tiết, vui lòng tham khảo [Tools Setup](5.2.2-Tools-Setup).

#### 3. API bên thứ ba & Tài nguyên ngoài
Một số tính năng của dự án yêu cầu tích hợp với các dịch vụ API từ bên thứ ba. Trước khi khởi tạo dự án, bạn nên:
* Đăng ký tài khoản nhà cung cấp dịch vụ API được yêu cầu.
* Lấy các mã token hoặc API key cần thiết để cấu hình dưới dạng secret.

Để xem chi tiết thông tin tích hợp và thiết lập thông tin xác thực API, vui lòng tham khảo [Third-party APIs](5.2.3-Third-party-APIs).

#### Nội dung

- [Tài khoản AWS & Quyền IAM](5.2.1-AWS-IAM-Account/)
- [Cài đặt Môi trường & Công cụ Local](5.2.2-Tools-Setup/)
- [API bên thứ ba & Tài nguyên ngoài](5.2.3-Third-party-APIs/)
