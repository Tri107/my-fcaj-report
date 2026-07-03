---
title : "Cổng API và Tường lửa ứng dụng web (API Gateway & WAF)"
date : 2026-07-02
weight : 4
chapter : false
pre : " <b> 5.4.4. </b> "
---

Thành phần kiểm soát lưu lượng truy cập HTTP, thực hiện điều hướng an toàn và ngăn chặn các cuộc tấn công mạng độc hại.

---

## 1. Luồng xử lý chi tiết (Data Flow)

* **Xác thực trước khi xử lý (Authentication Guard):**
  * API Gateway tiếp nhận Request -> Kiểm tra cấu hình Route.
  * Nếu Route được bảo vệ bởi `CognitoJwtAuthorizer`: Trích xuất Header `Authorization` và kiểm tra chữ ký khóa công khai của JWT Token với Cognito User Pool.
  * Nếu Token không hợp lệ hoặc hết hạn: Trả về **`HTTP 401 Unauthorized`** ngay lập tức mà không kích hoạt Lambda.
  * Nếu Token hợp lệ: Giải mã payload và đưa claims (bao gồm `sub`) vào `event.requestContext.authorizer.jwt.claims`.

* **Luồng WAF filtering:**
  * Request trước tiên qua AWS WAF WebACL gắn với API Gateway.
  * Kiểm tra các mẫu SQL Injection (SQLi) trong URI và Body, Cross-Site Scripting (XSS) trong Header.
  * Đếm tần suất truy cập dựa trên IP nguồn (Rate Limiting). Nếu vượt quá số request giới hạn trên một chu kỳ 5 phút, trả về **`HTTP 429 Too Many Requests`** và ghi log.

---

## 2. AWS SAM Template Config (`template.yaml`)

```yaml
  BackendHttpApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      StageName: dev
      CorsConfiguration:
        AllowOrigins:
          - http://localhost:3000
        AllowHeaders:
          - Content-Type
          - Authorization
          - x-original-filename
        AllowMethods:
          - GET
          - POST
          - DELETE
          - OPTIONS
        MaxAge: 600
      Auth:
        Authorizers:
          CognitoJwtAuthorizer:
            IdentitySource: "$request.header.Authorization"
            JwtConfiguration:
              issuer: !Sub "https://cognito-idp.${AWS::Region}.amazonaws.com/${AuthUserPool}"
              audience:
                - !Ref AuthUserPoolClient
```

> [!NOTE]
> Để bảo vệ API bằng AWS WAFv2, ta liên kết tài nguyên WebACL thông qua tài nguyên AWS::WAFv2::WebACL và thực hiện liên kết (WebACLAssociation) với ARN của API Gateway (chỉ áp dụng đối với REST API hoặc Application Load Balancer). Đối với HTTP API, ta thường liên kết WAF trực tiếp thông qua AWS Console hoặc qua cấu hình API Gateway Stage.

---

## 3. Triển khai & Liên kết WAF (Deployment)
1. Triển khai API Gateway bằng SAM:
   ```bash
   sam build
   sam deploy
   ```
2. Thực hiện tạo WebACL trên AWS WAF Console tại cùng khu vực (ví dụ `ap-southeast-1`).
3. Chọn liên kết WebACL vừa tạo với API Gateway Stage (Ví dụ stage `dev`).

---

## 4. Cấu hình Gateway & Tường lửa (Configuration)
* **CORS Settings:**
  * `AllowOrigins`: Hỗ trợ `http://localhost:3000` cho phát triển cục bộ và domain hosting chính thức.
  * `AllowHeaders`: Chỉ cho phép `Content-Type`, `Authorization`, `x-original-filename`.
  * `AllowMethods`: Chỉ chấp nhận các phương thức an toàn: `GET`, `POST`, `DELETE`, `OPTIONS`.

![Danh sách Routes của API Gateway](/images/5-Workshop/5.4-User-Interaction/api-gateway-routes-1.png)
![Thông tin chi tiết cấu hình Route](/images/5-Workshop/5.4-User-Interaction/api-gateway-routes-2.png)
* **Cognito Authorizer JWT Config:**
  * `IdentitySource`: `$request.header.Authorization`
  * `Issuer`: `https://cognito-idp.<region>.amazonaws.com/<UserPoolId>`
  * `Audience`: ID của App Client được cấp phép.

![Cấu hình JWT Authorizer cho API Gateway](/images/5-Workshop/5.4-User-Interaction/api-gateway-jwt-authorizer.png)
* **AWS WAF Rules:**
  * Rule Group do AWS quản lý: `AWSManagedRulesCommonRuleSet` (chống lỗ hổng OWASP phổ biến) và `AWSManagedRulesSQLiRuleSet`.
  * Rate-limit Rule: Giới hạn tối đa 100 requests/IP/5 phút.

Sau khi triển khai thành công, bạn có thể cấu hình và kiểm tra API Gateway cũng như WAF trên AWS Console.
