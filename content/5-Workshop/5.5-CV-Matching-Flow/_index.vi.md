---
title : "Luồng đánh giá CV"
date : 2026-07-03
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

## Tổng quan

Phần này mô tả luồng đánh giá CV đang được dùng trong AI Job Matching Platform. Luồng bắt đầu khi người dùng yêu cầu đánh giá mức độ phù hợp giữa CV và công việc từ frontend. Request đi qua API Gateway đến Lambda, sau đó Lambda đọc CV từ S3, trích xuất văn bản bằng Amazon Textract, lấy mô tả công việc từ DynamoDB, đánh giá bằng Gemini 3.1 Flash Lite, rồi lưu kết quả vào DynamoDB.

Region hiện tại: `ap-southeast-1`.

## Luồng kiến trúc

```text
User action trên frontend
  -> API Gateway HTTP API
  -> Evaluate Match Lambda
  -> Đọc file CV từ S3
  -> Trích xuất văn bản CV bằng Amazon Textract
  -> Đọc job description từ DynamoDB Jobs table
  -> Đánh giá mức độ phù hợp bằng Gemini 3.1 Flash Lite
  -> Lưu kết quả vào DynamoDB Match Results table
```

![Luồng kiến trúc CV Matching](/images/5-Workshop/5.5-CV-Matching-Flow/Jobs-Matching-Platform-v3.drawio.png)

## Thông tin implementation

- API Gateway base URL: `https://38icub30ig.execute-api.ap-southeast-1.amazonaws.com/dev`
- CV bucket: `jobs-matching-cvs-dev-973370772729`
- Match results table: `jobs-matching-match-results-dev`
- Evaluate Lambda ARN: `arn:aws:lambda:ap-southeast-1:973370772729:function:jobs-matching-evaluate-match-dev`

## Nội dung phần này

- Cách frontend bắt đầu luồng matching thông qua API Gateway
- Cách Lambda đọc CV từ S3 và trích xuất văn bản bằng Amazon Textract
- Cách lấy dữ liệu job từ DynamoDB
- Cách Gemini 3.1 Flash Lite đánh giá mức độ phù hợp
- Cách lưu kết quả matching để sử dụng sau

> Lưu ý chi phí: Luồng này có thể phát sinh chi phí từ Amazon Textract, AWS Lambda, CloudWatch Logs, Amazon S3, Amazon DynamoDB và Gemini API tùy theo mức sử dụng và billing tier.
