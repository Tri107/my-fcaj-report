---
title : "Kích hoạt Lambda Extract"
date : 2026-07-03
weight : 1
chapter : false
pre : " <b> 5.5.1. </b> "
---

## Mục tiêu

Phần này giải thích cách các endpoint trích xuất CV và đánh giá matching được kích hoạt từ frontend thông qua API Gateway.

Trong implementation hiện tại, S3 chỉ lưu file CV. S3 không tự trigger Lambda cho CV Matching Flow.

Region: `ap-southeast-1`  
API Gateway base URL: `https://38icub30ig.execute-api.ap-southeast-1.amazonaws.com/dev`

## Luồng kiến trúc

```text
Người dùng chọn job và CV
  -> Frontend gửi request
  -> API Gateway HTTP API
  -> Lambda function
  -> Đọc CV từ S3
  -> Tiếp tục luồng extract/evaluate
```

## Endpoints

- `POST /cv/extract`: gọi `ExtractCvFunction`
- `POST /cv/evaluate`: gọi `EvaluateMatchFunction`

## Input của /cv/extract

`userId` được lấy từ Cognito JWT claim `sub`. Frontend không gửi trực tiếp `userId` trong request body.

```json
{
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf"
}
```

## Input của /cv/evaluate

`jobId` xác định job cần so sánh với CV. `cvKey` trỏ đến file CV trong S3. `userId` vẫn được lấy từ Cognito JWT claim `sub`.

```json
{
  "jobId": "cbe79dc8-9aaf-4d6c-afcf-36db6533c304",
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf"
}
```

## Lambda Console event sample cho /cv/evaluate

Dùng cấu trúc này khi test `EvaluateMatchFunction` trong Lambda Console. Claim `sub` là bắt buộc vì function dùng giá trị này làm `userId`.

```json
{
  "requestContext": {
    "authorizer": {
      "jwt": {
        "claims": {
          "sub": "997ab59c-20b1-707d-4083-cb651fcdfb6f",
          "email": "user@example.com"
        }
      }
    }
  },
  "body": "{\"jobId\":\"cbe79dc8-9aaf-4d6c-afcf-36db6533c304\",\"cvKey\":\"997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf\"}"
}
```

## Kết quả mong đợi

- API Gateway nhận HTTP request từ user đã đăng nhập trên frontend.
- Lambda đọc `userId` từ `requestContext.authorizer.jwt.claims.sub`.
- Lambda dùng `cvKey` để đọc file CV từ S3.
- `/cv/extract` tiếp tục bước trích xuất văn bản.
- `/cv/evaluate` tiếp tục bước đánh giá CV với job.

![API Gateway Route](/images/5-Workshop/5.5-CV-Matching-Flow/5-1-api-gateway-cv-evaluate-route.png)

## Lỗi phổ biến

- Gửi `userId` trong request body thay vì lấy từ Cognito JWT.
- Test Lambda thiếu `requestContext.authorizer.jwt.claims.sub`.
- Dùng `cvKey` không thuộc user đang đăng nhập.

> Lưu ý chi phí: Khi trigger flow này, hệ thống có thể gọi Lambda, Textract, Gemini, CloudWatch Logs, S3 và DynamoDB tùy endpoint.
