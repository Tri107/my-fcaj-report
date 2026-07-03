---
title : "AI Evaluation"
date : 2026-07-03
weight : 3
chapter : false
pre : " <b> 5.5.3. </b> "
---

# AI Evaluation

## Goal

AI Evaluation compares extracted CV text with the selected job description. The Lambda does not send the raw PDF directly to Gemini. It first validates the user, validates `cvKey` ownership, loads the job from DynamoDB, extracts CV text with Amazon Textract, then sends structured text input to Gemini.

The current model is `gemini-1.5-flash`. The Gemini API key is read from AWS Systems Manager Parameter Store parameter `/jobs-matching/dev/gemini-api-key`; do not hard-code secrets in source code or workshop content.

Region: `ap-southeast-1`

## Flow

```text
POST /cv/evaluate
  -> EvaluateMatchFunction
  -> Validate Cognito user
  -> Validate cvKey ownership
  -> Load job from DynamoDB by jobId
  -> Extract CV text from S3 with Amazon Textract
  -> Build AI evaluation prompt
  -> Call Gemini 3.1 Flash Lite
  -> Validate JSON response
  -> Return and store matching result
```

## Request body

`userId` is not sent in the body. Lambda reads it from the Cognito JWT claim `sub`.

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
  "summary": "Short evaluation summary",
  "strengths": [
    "Strength 1",
    "Strength 2"
  ],
  "weaknesses": [
    "Weakness 1",
    "Weakness 2"
  ],
  "matchedSkills": [
    "JavaScript",
    "Postman"
  ],
  "missingSkills": [
    "ISTQB certification"
  ],
  "suggestions": [
    "Suggestion 1",
    "Suggestion 2"
  ]
}
```

## Expected response from /cv/evaluate

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

## Score explanation

- `overallScore`: overall fit between the CV and the job.
- `skillsScore`: how well the candidate skills match the job requirements.
- `experienceScore`: how well the candidate experience matches the job.
- `educationScore`: how well the candidate education background matches the job.
- `summary`: short AI-generated explanation of the match.
- `matchedSkills`: skills found in both CV and job requirements.
- `missingSkills`: important job requirements missing or weak in the CV.
- `suggestions`: recommendations for improving the CV or candidate profile.

## Common mistakes

- Sending the PDF file directly to Gemini instead of extracted text.
- Skipping job description validation.
- Not validating Gemini JSON response before returning it.
- Logging CV text, token, API key, or full sensitive response.
- Treating Bedrock as the current main provider even though this implementation uses Gemini.
- Assuming the score is deterministic. AI output can vary slightly between requests.

> Cost note: This step may generate cost from Gemini API, Lambda duration, CloudWatch Logs, Amazon Textract, S3, and DynamoDB. Gemini cost depends on input and output tokens if the project uses paid tier.
