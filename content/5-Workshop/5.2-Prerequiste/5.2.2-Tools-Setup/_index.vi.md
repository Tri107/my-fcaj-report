---
title : "Cài đặt Công cụ"
date : 2026-07-02 
weight : 2
chapter : false
pre : " <b> 5.2.2. </b> "
---

Trong phần này, chúng ta sẽ cài đặt và cấu hình các công cụ cần thiết trên máy cá nhân để xây dựng, chạy thử nghiệm và triển khai dự án.

---

### 1. Cài đặt AWS CLI

#### 1.1. Tải và cài đặt (Dành cho Windows):
Mở Command Prompt hoặc Terminal của bạn và chạy lệnh dưới đây để tải và cài đặt AWS CLI:

```powershell
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```

#### 1.2. Xác thực cài đặt:
Kiểm tra xem AWS CLI đã được cài đặt thành công chưa bằng cách chạy lệnh:

```bash
aws --version
```
![aws version](/images/5-Workshop/5.2-Prerequisite/5.2.2-Tools-Setup/aws-version.png)
---

### 2. Cài đặt AWS SAM CLI

#### 2.1. Tải về:
Tải bộ cài đặt từ trang tài liệu chính thức: [Hướng dẫn cài đặt AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).

#### 2.2. Xác thực cài đặt:
Kiểm tra xem SAM CLI đã được cài đặt thành công chưa bằng cách chạy lệnh:

```bash
sam --version
```
![sam version](/images/5-Workshop/5.2-Prerequisite/5.2.2-Tools-Setup/sam-version.png)
---

### 3. Cấu hình AWS Profile

Để cho phép các công cụ local xác thực và tương tác với tài khoản AWS của bạn, hãy cấu hình profile bằng các thông tin bảo mật đã lấy ở Mục 5.2.1:

#### 3.1. Chạy lệnh cấu hình:
Thực hiện lệnh cấu hình:

```bash
aws configure
```
Nhập các thông tin Credentials và khu vực mặc định khi được yêu cầu:

```text
AWS Access Key ID [None]: <Access-Key-ID-của-IAM-User>
AWS Secret Access Key [None]: <Secret-Access-Key-của-IAM-User>
Default region name [None]: ap-southeast-1
Default output format [None]: json
```

---

### 4. Khởi tạo Dự án

Để khởi tạo nhanh cấu trúc dự án serverless, chạy lệnh:

```bash
sam init
```

Làm theo các hướng dẫn trên màn hình để chọn mẫu (template) và các tùy chọn tương ứng.

![sam init](/images/5-Workshop/5.2-Prerequisite/5.2.2-Tools-Setup/sam-init.png)
---

### 5. Cài đặt IDE (Visual Studio Code)

Chúng ta sẽ sử dụng Visual Studio Code (VS Code) làm môi trường phát triển chính.
* Tải xuống và cài đặt phiên bản phù hợp với hệ điều hành của bạn tại: [Tải về Visual Studio Code](https://code.visualstudio.com/download).

---

### 6. Cài đặt Môi trường (Node.js & Python)

Dự án này yêu cầu cả môi trường Node.js và Python cho một số tính năng phát triển cụ thể.

* **Node.js**: Tải và cài đặt tại [Tải về Node.js](https://nodejs.org/en/download/).
* **Python**: Tải và cài đặt tại [Tải về Python](https://www.python.org/downloads/).

---

### 7. Cài đặt các Extension cho VS Code

Để tối ưu hóa trải nghiệm lập trình, hãy mở VS Code và cài đặt các tiện ích mở rộng sau:

* **AWS Toolkit for Visual Studio Code**: Hỗ trợ tích hợp AWS CLI, debug local và quản lý tài nguyên trực quan.
* **Python**: Cung cấp hỗ trợ đầy đủ các tính năng lập trình Python và viết Lambda function.
