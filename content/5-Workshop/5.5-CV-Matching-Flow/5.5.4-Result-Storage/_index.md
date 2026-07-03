---
title : "Result Storage"
date : 2026-07-03
weight : 4
chapter : false
pre : " <b> 5.5.4. </b> "
---

# Result Storage

## Goal

Result storage persists the AI evaluation result after `/cv/evaluate` completes. DynamoDB allows the application to show match history and retrieve a specific match result later.

The saved item belongs to the authenticated user through `userId` from the Cognito JWT claim `sub`. The system stores the evaluation summary and scores, not the full raw CV text.

## Flow

```text
EvaluateMatchFunction completes AI evaluation
  -> Generate matchId
  -> Build match result item
  -> Save item to DynamoDB Match Results table
  -> Return saved result to frontend
  -> Frontend renders the analysis result
```

## Table information

- Table name: `jobs-matching-match-results-dev`
- Region: `ap-southeast-1`
- Stack output: `MatchResultsTableName`

Match results can be retrieved through supported API routes such as `GET /cv/match-results` and, when available in the current route/code, `GET /cv/match-results/{matchId}`.

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

`expiresAt` is optional depending on the TTL configuration. If TTL is enabled, DynamoDB can automatically remove old match results after the configured period.

## Expected response when retrieving a result

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

## Field explanation

- `userId`: authenticated user ID from Cognito JWT claim `sub`.
- `matchId`: unique ID of the match result.
- `jobId`: selected job ID.
- `jobTitle`: title of the selected job.
- `cvKey`: S3 key of the evaluated CV.
- `score`: AI evaluation result.
- `evaluatedAt`: timestamp when the evaluation was created.
- `expiresAt`: optional TTL timestamp for automatic cleanup if enabled.

![Match Result Item](/images/5-Workshop/5.5-CV-Matching-Flow/5-4-dynamodb-match-result-item.png)

## Common mistakes

- Storing full raw CV text in the match result item.
- Trusting `userId` from request body instead of Cognito JWT.
- Returning match results across users without checking ownership.
- Assuming results are sorted by newest first without a timestamp-based key or index.
- Forgetting DynamoDB TTL behavior when old items disappear.

> Cost note: DynamoDB can generate cost from write and read requests. CloudWatch Logs and Lambda execution can also generate cost during result storage and retrieval.
