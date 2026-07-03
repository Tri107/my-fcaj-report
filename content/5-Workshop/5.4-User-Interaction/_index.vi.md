---
title : "Tương tác người dùng"
date : 2026-07-02 
weight : 4 
chapter : false
pre : " <b> 5.4. </b> "
---

#### Tổng quan

Trong phần này, chúng ta sẽ tìm hiểu cấu trúc và luồng xử lý chi tiết cho các thành phần trực tiếp tương tác với người dùng trong nền tảng Job-Matching.

Các mô-đun chính bao gồm:
1. **Xác thực người dùng (Cognito Auth):** Thiết lập đăng ký, đăng nhập và lấy mã Token bảo mật.
2. **Cơ sở dữ liệu & Lưu trữ (Databases & Storage):** Lưu trữ danh sách việc làm yêu thích trên DynamoDB và CV trên Amazon S3.
3. **Các API nghiệp vụ chính (Core API Lambdas):** Điều phối dữ liệu công việc và hồ sơ CV thông qua AWS Lambda.
4. **Cổng API và Tường lửa (API Gateway & WAF):** Quản lý định tuyến và bảo vệ ứng dụng khỏi các mối đe dọa.
5. **Triển khai ứng dụng (Frontend Hosting Amplify):** Hosting Next.js và tự động CI/CD.
