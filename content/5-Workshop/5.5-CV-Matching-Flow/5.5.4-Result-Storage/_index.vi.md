---
title : "Lưu kết quả"
date : 2026-07-03
weight : 4
chapter : false
pre : " <b> 5.5.4. </b> "
---

# Lưu kết quả

## Mục tiêu

Result Storage lưu kết quả đánh giá AI sau khi `/cv/evaluate` hoàn tất. DynamoDB giúp ứng dụng hiển thị lịch sử matching và lấy lại một kết quả cụ thể sau này.

Item được gắn với user đang đăng nhập thông qua `userId` lấy từ Cognito JWT claim `sub`. Hệ thống lưu điểm, tóm tắt và gợi ý đánh giá; không lưu full raw CV text trong match result.

## Flow

```text
EvaluateMatchFunction hoàn tất AI evaluation
  -> Tạo matchId
  -> Build match result item
  -> Lưu item vào DynamoDB Match Results table
  -> Trả kết quả đã lưu về frontend
  -> Frontend hiển thị kết quả phân tích
```

## Thông tin table

- Table name: `jobs-matching-match-results-dev`
- Region: `ap-southeast-1`
- Stack output: `MatchResultsTableName`

Match results có thể được lấy lại qua các API route được hỗ trợ như `GET /cv/match-results` và, nếu route/code hiện tại có hỗ trợ, `GET /cv/match-results/{matchId}`.

## Sample DynamoDB item

```json
{
  "userId": "997ab59c-20b1-707d-4083-cb651fcdfb6f",
  "matchId": "d06123e2-93dd-4977-bf02-c8ed770a5fd9",
  "jobId": "cbe79dc8-9aaf-4d6c-afcf-36db6533c304",
  "jobTitle": "tester",
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf",
  "score": {
    "overallScore": 75,
    "skillsScore": 70,
    "experienceScore": 70,
    "educationScore": 85,
    "summary": "The candidate is a strong Information Technology student with solid fullstack development experience and familiarity with testing tools like Postman and jMeter.",
    "strengths": [
      "Strong technical background in web development and API testing"
    ],
    "weaknesses": [
      "No formal professional QA or software testing experience"
    ],
    "matchedSkills": [
      "JavaScript",
      "Postman"
    ],
    "missingSkills": [
      "ISTQB certification"
    ],
    "suggestions": [
      "Highlight testing experience in previous projects"
    ]
  },
  "evaluatedAt": "2026-06-29T06:37:15.186Z",
  "expiresAt": 1793236635
}
```

`expiresAt` là field tùy theo cấu hình TTL. Nếu TTL được bật, DynamoDB có thể tự động xóa các kết quả matching cũ sau thời gian đã cấu hình.

## Expected response khi lấy lại result

```json
{
  "matchId": "d06123e2-93dd-4977-bf02-c8ed770a5fd9",
  "jobId": "cbe79dc8-9aaf-4d6c-afcf-36db6533c304",
  "jobTitle": "tester",
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf",
  "score": {
    "overallScore": 75,
    "skillsScore": 70,
    "experienceScore": 70,
    "educationScore": 85,
    "summary": "The candidate is a strong Information Technology student with solid fullstack development experience and familiarity with testing tools like Postman and jMeter.",
    "strengths": [
      "Strong technical background in web development and API testing"
    ],
    "weaknesses": [
      "No formal professional QA or software testing experience"
    ],
    "matchedSkills": [
      "JavaScript",
      "Postman"
    ],
    "missingSkills": [
      "ISTQB certification"
    ],
    "suggestions": [
      "Highlight testing experience in previous projects"
    ]
  },
  "evaluatedAt": "2026-06-29T06:37:15.186Z"
}
```

## Giải thích field

- `userId`: ID user đã đăng nhập, lấy từ Cognito JWT claim `sub`.
- `matchId`: ID duy nhất của kết quả matching.
- `jobId`: ID công việc được chọn.
- `jobTitle`: tiêu đề công việc được chọn.
- `cvKey`: S3 key của CV đã được đánh giá.
- `score`: kết quả đánh giá từ AI.
- `evaluatedAt`: thời điểm tạo kết quả đánh giá.
- `expiresAt`: TTL timestamp tùy chọn để tự động cleanup nếu được bật.

![Match Result Item](/images/5-Workshop/5.5-CV-Matching-Flow/5-4-dynamodb-match-result-item.png)

## Lỗi phổ biến

- Lưu full raw CV text vào match result item.
- Tin `userId` từ request body thay vì Cognito JWT.
- Trả match result của user khác mà không kiểm tra ownership.
- Giả định kết quả được sắp xếp mới nhất trước khi chưa có key hoặc index theo timestamp.
- Quên hành vi TTL của DynamoDB khi item cũ biến mất.

> Lưu ý chi phí: DynamoDB có thể phát sinh chi phí từ write/read request. CloudWatch Logs và Lambda execution cũng có thể phát sinh chi phí khi lưu và lấy kết quả.
