---
title : "User Authentication Setup with Cognito"
date : 2026-07-02
weight : 1
chapter : false
pre : " <b> 5.4.1. </b> "
---

This module handles user account lifecycle management (Registration, OTP verification, login, token issuance, logout). The frontend communicates directly with the **AWS Cognito User Pool** via the `aws-amplify/auth` library to optimize security.

---

## 1. Detailed Data Flow

### A. Account Registration (`registerWithEmail`)
* **Description:** Register a new account using Email and Password. The initial status of the account is `UNCONFIRMED`.
* **Input (Client calls via SDK):**
  ```json
  {
    "email": "candidate@example.com",
    "password": "SecurePassword@123"
  }
  ```
* **Output (Processed by Cognito SDK):**
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

### B. Confirm Registration (`confirmRegister`)
* **Description:** Verify the OTP code sent to the email to activate the account status to `CONFIRMED`.
* **Input (Client calls via SDK):**
  ```json
  {
    "email": "candidate@example.com",
    "code": "482015"
  }
  ```
* **Output (Processed by Cognito SDK):**
  ```json
  {
    "isSignUpComplete": true,
    "nextStep": {
      "signUpStep": "DONE"
    }
  }
  ```

### C. System Login (`loginWithEmail`)
* **Description:** Authenticate login credentials and store the session token in the browser's local cache.
* **Input (Client calls via SDK):**
  ```json
  {
    "email": "candidate@example.com",
    "password": "SecurePassword@123"
  }
  ```
* **Output (Processed by Cognito SDK):**
  ```json
  {
    "isSignedIn": true,
    "nextStep": {
      "signInStep": "DONE"
    }
  }
  ```

### D. Get ID Token for API Calls (`getIdToken`)
* **Description:** Read from the Amplify session cache in the browser to retrieve the JWT ID Token to pass in the `Authorization: Bearer <ID_TOKEN>` header.
* **Input:** None (reads from local cache).
* **Output:**
  ```json
  {
    "idToken": "eyJraWQiOiJ...jwt_payload_signature..."
  }
  ```
  *(The payload contains the `sub` claim corresponding to the unique user ID).*

You can view the list of registered users in the Cognito Console:
![Cognito Users](/images/5-Workshop/5.4-User-Interaction/cognito-users.png)

---

## 2. Cognito Configuration
* **Email as Login Identifier:** Users sign up and sign in using their Email address instead of a raw Username.
* **Password Policy:** 
  * Minimum length: 8 characters.
  * Must contain lowercase letters, uppercase letters, and numbers. Special characters are optional.
* **Google Sign-In Support (Google Identity Provider):** OAuth2 integration allows users to log in directly with their Google Account.
* **Redirect URLs (Callbacks):**
  * Login Callback: `http://localhost:3000/auth/callback` (or production Amplify domain).
  * Logout Callback: `http://localhost:3000/login`.

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

## 4. Cognito Deployment
1. Configure Google Client ID and Secret parameters in AWS Systems Manager Parameter Store or pass them directly via CLI.
2. Navigate to the `backend/` directory and run the build/deploy commands:
   ```bash
   sam build
   sam deploy --parameter-overrides GoogleClientId="<GOOGLE_CLIENT_ID>" GoogleClientSecret="<GOOGLE_CLIENT_SECRET>"
   ```
3. Once the stack is deployed, record the `CognitoUserPoolId` and `CognitoUserPoolClientId` output values to configure `.env.local` for the Frontend.

Verify the created Cognito User Pool in the AWS Console:
![Cognito User Pool](/images/5-Workshop/5.4-User-Interaction/cognito-user-pool.png)

Verify the App Client Settings:
![Cognito App Client](/images/5-Workshop/5.4-User-Interaction/cognito-app-client.png)
