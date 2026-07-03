---
title : "Đánh giá bằng AI"
date : 2026-07-03
weight : 3
chapter : false
pre : " <b> 5.5.3. </b> "
---

## Mục tiêu

AI Evaluation so sánh text đã trích xuất từ CV với job description của công việc được chọn. Lambda không gửi trực tiếp file PDF thô vào Gemini. Lambda kiểm tra user, kiểm tra `cvKey` thuộc user hiện tại, lấy job từ DynamoDB, trích xuất CV text bằng Amazon Textract, sau đó gửi text đã xử lý vào Gemini.

Model hiện tại là `gemini-1.5-flash`. Gemini API key được đọc từ AWS Systems Manager Parameter Store parameter `/jobs-matching/dev/gemini-api-key`; không hard-code secret trong source code hoặc nội dung workshop.

Region: `ap-southeast-1`

## Flow

```text
POST /cv/evaluate
  -> EvaluateMatchFunction
  -> Kiểm tra user từ Cognito
  -> Kiểm tra cvKey thuộc user hiện tại
  -> Lấy job từ DynamoDB bằng jobId
  -> Trích xuất CV text từ S3 bằng Amazon Textract
  -> Build prompt đánh giá AI
  -> Gọi Gemini 3.1 Flash Lite
  -> Validate JSON response
  -> Trả về và lưu kết quả matching
```

## Request body

`userId` không nằm trong body. Lambda lấy giá trị này từ Cognito JWT claim `sub`.

```json
{
  "jobId": "cbe79dc8-9aaf-4d6c-afcf-36db6533c304",
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf"
}
```

## AI output schema

```json
{
  "overallScore": 75,
  "skillsScore": 70,
  "experienceScore": 70,
  "educationScore": 85,
  "summary": "Tóm tắt đánh giá ngắn",
  "strengths": [
    "Điểm mạnh 1",
    "Điểm mạnh 2"
  ],
  "weaknesses": [
    "Điểm yếu 1",
    "Điểm yếu 2"
  ],
  "matchedSkills": [
    "JavaScript",
    "Postman"
  ],
  "missingSkills": [
    "ISTQB certification"
  ],
  "suggestions": [
    "Gợi ý 1",
    "Gợi ý 2"
  ]
}
```

## Expected response từ /cv/evaluate

```json
{
  "matchId": "d06123e2-93dd-4977-bf02-c8ed770a5fd9",
  "jobId": "cbe79dc8-9aaf-4d6c-afcf-36db6533c304",
  "jobTitle": "tester",
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf",
  "pageCount": 1,
  "score": {
    "overallScore": 75,
    "skillsScore": 70,
    "experienceScore": 70,
    "educationScore": 85,
    "summary": "The candidate is a strong Information Technology student with solid fullstack development experience and familiarity with testing tools like Postman and jMeter. While they lack formal QA experience, their project-based background and technical skill set provide a good foundation for a Junior Tester role.",
    "strengths": [
      "Strong technical background in web development and API testing",
      "Familiarity with Postman and jMeter, which are relevant to QA",
      "Proven project management and collaboration skills",
      "Relevant degree in Information Technology"
    ],
    "weaknesses": [
      "No formal professional QA or software testing experience",
      "Missing formal certification such as ISTQB",
      "Lack of experience in formal test plan and script design documentation"
    ],
    "matchedSkills": [
      "JavaScript",
      "Java",
      "SQL",
      "Postman",
      "Jira"
    ],
    "missingSkills": [
      "Formal QA methodology implementation",
      "Test plan and test procedure design",
      "ISTQB certification",
      "Bug tracking and reporting lifecycle experience"
    ],
    "suggestions": [
      "Highlight any instances of writing unit or integration tests in current projects",
      "Pursue ISTQB Foundation Level certification to align with job requirements",
      "Emphasize the analytical and debugging skills used during the development of previous projects",
      "Clearly define any testing methodologies used (e.g., black-box, white-box testing) in personal projects"
    ]
  },
  "evaluatedAt": "2026-06-29T06:37:15.186Z"
}
```

## Giải thích score

- `overallScore`: mức độ phù hợp tổng thể giữa CV và công việc.
- `skillsScore`: mức độ kỹ năng trong CV khớp với yêu cầu công việc.
- `experienceScore`: mức độ kinh nghiệm ứng viên phù hợp với công việc.
- `educationScore`: mức độ nền tảng học vấn phù hợp với công việc.
- `summary`: tóm tắt ngắn do AI sinh ra.
- `matchedSkills`: các kỹ năng xuất hiện trong cả CV và yêu cầu công việc.
- `missingSkills`: các yêu cầu quan trọng còn thiếu hoặc còn yếu trong CV.
- `suggestions`: gợi ý để cải thiện CV hoặc hồ sơ ứng viên.

## Lỗi phổ biến

- Gửi trực tiếp file PDF vào Gemini thay vì text đã trích xuất.
- Bỏ qua bước kiểm tra job description.
- Không validate JSON response từ Gemini trước khi trả về.
- Log CV text, token, API key hoặc response nhạy cảm đầy đủ.
- Ghi Bedrock là provider chính hiện tại trong khi implementation này đang dùng Gemini.
- Giả định score luôn giống nhau tuyệt đối. Output AI có thể thay đổi nhẹ giữa các lần gọi.

> Lưu ý chi phí: Bước này có thể phát sinh chi phí từ Gemini API, thời gian chạy Lambda, CloudWatch Logs, Amazon Textract, S3 và DynamoDB. Chi phí Gemini phụ thuộc vào input/output tokens nếu project dùng paid tier.
