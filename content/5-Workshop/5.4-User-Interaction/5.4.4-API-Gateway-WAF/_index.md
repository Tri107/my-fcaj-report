---
title : "API Gateway & WAF (API Gateway and Web Application Firewall)"
date : 2026-07-02
weight : 4
chapter : false
pre : " <b> 5.4.4. </b> "
---

This component controls incoming HTTP traffic, provides secure request routing, and prevents malicious web application attacks.

---

## 1. Detailed Data Flow

* **Pre-processing Authentication (Authentication Guard):**
  * API Gateway receives the HTTP Request -> Checks the Route configuration.
  * If the Route is protected by `CognitoJwtAuthorizer`: Extracts the `Authorization` header and validates the signature of the JWT Token using public keys from the Cognito User Pool.
  * If the token is invalid or expired: Returns an immediate **`HTTP 401 Unauthorized`** response without triggering the backend Lambda.
  * If the token is valid: Decodes the payload and injects claims (including `sub`) into `event.requestContext.authorizer.jwt.claims`.

* **AWS WAF Filtering Flow:**
  * Requests first pass through the AWS WAF WebACL associated with the API Gateway.
  * WAF inspects requests for SQL Injection (SQLi) patterns in the URI and body, and Cross-Site Scripting (XSS) attacks in the headers.
  * Tracks request frequency from the source IP (Rate Limiting). If the request count exceeds the limit within a 5-minute window, returns **`HTTP 429 Too Many Requests`** and logs the event.

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
> To protect an API with AWS WAFv2, we associate a WebACL resource via the `AWS::WAFv2::WebACL` resource and perform a WebACL association (`WebACLAssociation`) with the API Gateway's ARN (only applies to REST APIs or Application Load Balancers). For HTTP APIs, WAF is usually associated directly through the AWS Console or the API Gateway Stage settings.

---

## 3. WAF Deployment & Association
1. Deploy the API Gateway using SAM:
   ```bash
   sam build
   sam deploy
   ```
2. Create a WebACL in the AWS WAF Console in the same AWS Region (e.g., `ap-southeast-1`).
3. Associate the created WebACL with the API Gateway Stage (e.g., stage `dev`).

---

## 4. Gateway & Firewall Configuration
* **CORS Settings:**
  * `AllowOrigins`: Supports `http://localhost:3000` for local development and the official production hosting domain.
  * `AllowHeaders`: Restricts headers to `Content-Type`, `Authorization`, and `x-original-filename`.
  * `AllowMethods`: Accepts only safe HTTP methods: `GET`, `POST`, `DELETE`, and `OPTIONS`.

![API Gateway Routes List](/images/5-Workshop/5.4-User-Interaction/api-gateway-routes-1.png)
![API Gateway Routes Details](/images/5-Workshop/5.4-User-Interaction/api-gateway-routes-2.png)
* **Cognito Authorizer JWT Config:**
  * `IdentitySource`: `$request.header.Authorization`
  * `Issuer`: `https://cognito-idp.<region>.amazonaws.com/<UserPoolId>`
  * `Audience`: The authorized App Client ID.

![API Gateway JWT Authorizer Configuration](/images/5-Workshop/5.4-User-Interaction/api-gateway-jwt-authorizer.png)
* **AWS WAF Rules:**
  * AWS Managed Rule Groups: `AWSManagedRulesCommonRuleSet` (protects against common OWASP vulnerabilities) and `AWSManagedRulesSQLiRuleSet`.
  * Rate-limit Rule: Restricts traffic to a maximum of 100 requests per IP address per 5 minutes.

Once deployed successfully, you can verify the API Gateway routes, authorizers, and WAF Web ACL configurations in the AWS Console.
