---
title : "Thiết lập xác thực người dùng với Cognito"
date : 2026-07-02
weight : 1
chapter : false
pre : " <b> 5.4.1. </b> "
---

Mô-đun này đảm nhận vai trò quản lý vòng đời tài khoản (Đăng ký, xác nhận OTP, đăng nhập, cấp token, đăng xuất). Frontend giao tiếp trực tiếp với **AWS Cognito User Pool** thông qua thư viện `aws-amplify/auth` để tối ưu bảo mật.

---

## 1. Luồng xử lý chi tiết (Data Flow)

### A. Đăng ký tài khoản (`registerWithEmail`)
* **Mô tả:** Đăng ký tài khoản mới bằng Email và Mật khẩu. Trạng thái tài khoản ban đầu là `UNCONFIRMED`.
* **Input (Client gọi qua SDK):**
  ```json
  {
    "email": "candidate@example.com",
    "password": "SecurePassword@123"
  }
  ```
* **Output (Xử lý từ Cognito SDK):**
  ```json
  {
    "isSignUpComplete": false,
    "userId": "d74b8c9d-d81a-4b92-91ef-f6d3a82741d4",
    "nextStep": {
      "signUpStep": "CONFIRM_SIGN_UP",
      "codeDeliveryDetails": {
        "deliveryMedium": "EMAIL",
        "destination": "c***@e***.com",
        "attributeName": "email"
      }
    }
  }
  ```

### B. Xác nhận đăng ký (`confirmRegister`)
* **Mô tả:** Xác thực mã OTP gửi về email để kích hoạt tài khoản sang trạng thái `CONFIRMED`.
* **Input (Client gọi qua SDK):**
  ```json
  {
    "email": "candidate@example.com",
    "code": "482015"
  }
  ```
* **Output (Xử lý từ Cognito SDK):**
  ```json
  {
    "isSignUpComplete": true,
    "nextStep": {
      "signUpStep": "DONE"
    }
  }
  ```

### C. Đăng nhập hệ thống (`loginWithEmail`)
* **Mô tả:** Xác thực thông tin đăng nhập và lưu session token vào bộ nhớ cache của trình duyệt.
* **Input (Client gọi qua SDK):**
  ```json
  {
    "email": "candidate@example.com",
    "password": "SecurePassword@123"
  }
  ```
* **Output (Xử lý từ Cognito SDK):**
  ```json
  {
    "isSignedIn": true,
    "nextStep": {
      "signInStep": "DONE"
    }
  }
  ```

### D. Lấy ID Token để gửi API (`getIdToken`)
* **Mô tả:** Đọc từ session cache của Amplify trong trình duyệt để lấy JWT ID Token nhằm truyền vào header `Authorization: Bearer <ID_TOKEN>`.
* **Input:** Không có (đọc local cache).
* **Output:**
  ```json
  {
    "idToken": "eyJraWQiOiJ...jwt_payload_signature..."
  }
  ```
  *(Payload chứa claim `sub` tương ứng với ID duy nhất của người dùng).*

Bạn có thể kiểm tra danh sách người dùng đã được tạo và kích hoạt trong Cognito User Pool:
![Cognito Users](/images/5-Workshop/5.4-User-Interaction/cognito-users.png)

---

## 2. Cấu hình Cognito (Configuration)
* **Email làm định danh đăng nhập:** Người dùng đăng ký và đăng nhập bằng trường Email thay vì Username thô.
* **Quy tắc mật khẩu (Password Policy):** 
  * Độ dài tối thiểu: 8 ký tự.
  * Yêu cầu chứa chữ thường (lowercase), chữ hoa (uppercase) và chữ số (numbers). Không bắt buộc ký tự đặc biệt.
* **Hỗ trợ Đăng nhập Google (Google Identity Provider):** Liên kết OAuth2 để người dùng đăng nhập trực tiếp bằng Google Account.
* **URL chuyển hướng (Callbacks):**
  * Đăng nhập thành công: `http://localhost:3000/auth/callback` (hoặc domain Amplify sản xuất).
  * Đăng xuất thành công: `http://localhost:3000/login`.

![Cognito Managed Login](/images/5-Workshop/5.4-User-Interaction/cognito-managed-login.png)

---

## 3. AWS SAM Template Config (`template.yaml`)

```yaml
  AuthUserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: jobs-matching-user-pool-dev
      UsernameAttributes:
        - email
      AutoVerifiedAttributes:
        - email
      MfaConfiguration: "OFF"
      Policies:
        PasswordPolicy:
          MinimumLength: 8
          RequireLowercase: true
          RequireUppercase: true
          RequireNumbers: true
          RequireSymbols: false
      AccountRecoverySetting:
        RecoveryMechanisms:
          - Name: verified_email
            Priority: 1
      Schema:
        - Name: email
          Required: true
          Mutable: true

  AuthUserPoolDomain:
    Type: AWS::Cognito::UserPoolDomain
    Properties:
      Domain: jobs-matching-auth-dev
      UserPoolId: !Ref AuthUserPool

  GoogleIdentityProvider:
    Type: AWS::Cognito::UserPoolIdentityProvider
    Properties:
      UserPoolId: !Ref AuthUserPool
      ProviderName: Google
      ProviderType: Google
      ProviderDetails:
        client_id: !Ref GoogleClientId
        client_secret: !Ref GoogleClientSecret
        authorize_scopes: "openid email profile"
      AttributeMapping:
        email: email
        given_name: given_name
        family_name: family_name
        name: name

  AuthUserPoolClient:
    Type: AWS::Cognito::UserPoolClient
    DependsOn:
      - GoogleIdentityProvider
    Properties:
      ClientName: jobs-matching-web-client-dev
      UserPoolId: !Ref AuthUserPool
      GenerateSecret: false
      PreventUserExistenceErrors: ENABLED
      ExplicitAuthFlows:
        - ALLOW_USER_SRP_AUTH
        - ALLOW_USER_PASSWORD_AUTH
        - ALLOW_REFRESH_TOKEN_AUTH
      AccessTokenValidity: 1
      IdTokenValidity: 1
      RefreshTokenValidity: 30
      TokenValidityUnits:
        AccessToken: hours
        IdToken: hours
        RefreshToken: days
      AllowedOAuthFlowsUserPoolClient: true
      AllowedOAuthFlows:
        - code
      AllowedOAuthScopes:
        - openid
        - email
        - profile
        - aws.cognito.signin.user.admin
      SupportedIdentityProviders:
        - COGNITO
        - Google
      CallbackURLs:
        - http://localhost:3000/auth/callback
      LogoutURLs:
        - http://localhost:3000/login
```

---

## 4. Triển khai Cognito (Deployment)
1. Cấu hình tham số Google Client ID và Secret trong AWS Systems Manager Parameter Store hoặc truyền trực tiếp qua CLI.
2. Di chuyển vào thư mục `backend/` và chạy lệnh build/deploy:
   ```bash
   sam build
   sam deploy --parameter-overrides GoogleClientId="<GOOGLE_CLIENT_ID>" GoogleClientSecret="<GOOGLE_CLIENT_SECRET>"
   ```
3. Sau khi stack được tạo xong, ghi nhận giá trị Output `CognitoUserPoolId` và `CognitoUserPoolClientId` để cấu hình `.env.local` cho Frontend.

Kiểm tra cấu hình Cognito User Pool trên AWS Console:
![Cognito User Pool](/images/5-Workshop/5.4-User-Interaction/cognito-user-pool.png)

Kiểm tra thông tin App Client Settings:
![Cognito App Client](/images/5-Workshop/5.4-User-Interaction/cognito-app-client.png)
