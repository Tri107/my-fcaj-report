---
title : "Triển khai ứng dụng Frontend (Frontend Hosting Amplify)"
date : 2026-07-02
weight : 5
chapter : false
pre : " <b> 5.4.5. </b> "
---

Triển khai và phân phối ứng dụng Next.js giao diện người dùng bằng **AWS Amplify Hosting** với khả năng tự động CI/CD khi đẩy mã nguồn lên GitHub.

---

## 1. Luồng xử lý chi tiết (Data Flow)
* **Khởi tạo ứng dụng:** Client tải trang web -> Đọc tệp cấu hình môi trường được nạp vào lúc build -> Khởi tạo cấu hình Amplify Auth SDK.
* **Bảo vệ route tĩnh (Routing Protection):**
  * Khi người dùng truy cập các đường dẫn như `/favorites`, `/upload-cv`, `/matching`.
  * Giao diện gọi hàm `fetchAuthSession()` để kiểm tra session hiện tại.
  * Nếu session rỗng: Chuyển hướng người dùng về trang `/login`.
  * Nếu session tồn tại: Hiển thị giao diện người dùng và nạp token tương ứng để sẵn sàng gọi API.

---

## 2. Cấu hình Biến môi trường (Environment Variables)
Khi ứng dụng được lưu trữ trên AWS Amplify Console, các biến môi trường sau phải được định nghĩa để Client gọi đúng cụm Backend:

| Tên biến môi trường | Giá trị thực tế mẫu | Vai trò |
| :--- | :--- | :--- |
| `NEXT_PUBLIC_API_BASE_URL` | `https://38ethw76uh.execute-api.ap-southeast-1.amazonaws.com/dev` | URL gốc của API Gateway để fetch dữ liệu. |
| `NEXT_PUBLIC_AWS_REGION` | `ap-southeast-1` | Vùng tài nguyên AWS chứa User Pool. |
| `NEXT_PUBLIC_COGNITO_USER_POOL_ID` | `ap-southeast-1_xxxxx` | ID của Cognito User Pool. |
| `NEXT_PUBLIC_COGNITO_USER_POOL_CLIENT_ID` | `xxxxxxxxxxxxxx` | App Client ID của Cognito. |

---

## 3. Hướng dẫn Triển khai & Cấu hình trên Amplify Console (Deployment Steps)

1. **Liên kết Repo Git:**
   * Truy cập AWS Console -> AWS Amplify -> Chọn **Create new app**.
   * Chọn nhà cung cấp mã nguồn (ví dụ **GitHub**) và liên kết với repository chứa dự án `Jobs-Matching-Platform`.
   * Chọn nhánh triển khai (ví dụ `main` hoặc `dev`).

2. **Cấu hình Trình biên dịch (Build Settings):**
   Amplify sẽ tự động nhận diện cấu hình Next.js. Đảm bảo cấu hình Amplify build spec (`amplify.yml`) khớp với cấu hình Next.js của dự án:
   ```yaml
   version: 1
   frontend:
     phases:
       preBuild:
         commands:
           - cd frontend
           - npm ci
       build:
         commands:
           - npm run build
     artifacts:
       baseDirectory: frontend/.next
       files:
         - '**/*'
     cache:
       paths:
         - frontend/node_modules/**/*
         - frontend/.next/cache/**/*
   ```

3. **Cấu hình biến môi trường trong Amplify Console:**
   * Vào mục **Hosting** -> **Environment variables**.
   * Thêm toàn bộ 4 biến môi trường trên (`NEXT_PUBLIC_API_BASE_URL`, v.v.) với giá trị output tương ứng thu được sau khi triển khai SAM Backend.

![Cấu hình biến môi trường trên Amplify](/images/5-Workshop/5.4-User-Interaction/amplify-environment-variables.png)

4. **Kích hoạt triển khai tự động (Continuous Deployment):**
   * Bấm **Save and deploy**.
   * Mỗi khi có commit mới đẩy lên nhánh Git đã chọn, AWS Amplify sẽ tự động kích hoạt luồng build và deploy lại Frontend mà không làm gián đoạn hệ thống.

![Trạng thái triển khai ứng dụng trên Amplify](/images/5-Workshop/5.4-User-Interaction/amplify-deploy-success.png)
