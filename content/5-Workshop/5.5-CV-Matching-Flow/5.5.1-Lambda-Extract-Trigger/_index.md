---
title : "Lambda Extract Trigger"
date : 2026-07-03
weight : 1
chapter : false
pre : " <b> 5.5.1. </b> "
---

# Lambda Extract Trigger

## Goal

This section explains how the CV extraction and matching endpoints are triggered from the frontend through API Gateway.

S3 only stores CV files in the current implementation. It does not trigger Lambda automatically for the CV Matching Flow.

Region: `ap-southeast-1`  
API Gateway base URL: `https://38icub30ig.execute-api.ap-southeast-1.amazonaws.com/dev`

## Architecture flow

```text
User selects a job and CV
  -> Frontend sends request
  -> API Gateway HTTP API
  -> Lambda function
  -> Read CV from S3
  -> Continue extraction/evaluation flow
```

## Endpoints

- `POST /cv/extract`: invokes `ExtractCvFunction`
- `POST /cv/evaluate`: invokes `EvaluateMatchFunction`

## Input for /cv/extract

`userId` is read from the Cognito JWT claim `sub`. The frontend should not send `userId` in the request body.

```json
{
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf"
}
```

## Input for /cv/evaluate

`jobId` identifies the job to compare with the CV. `cvKey` points to the CV file in S3. `userId` still comes from the Cognito JWT claim `sub`.

```json
{
  "jobId": "cbe79dc8-9aaf-4d6c-afcf-36db6533c304",
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf"
}
```

## Lambda Console event sample for /cv/evaluate

Use this shape when testing `EvaluateMatchFunction` from the Lambda Console. The `sub` claim is required because the function uses it as `userId`.

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

## Expected behavior

- API Gateway receives the HTTP request from the authenticated frontend user.
- Lambda reads `userId` from `requestContext.authorizer.jwt.claims.sub`.
- Lambda uses `cvKey` to read the CV file from S3.
- `/cv/extract` continues with text extraction.
- `/cv/evaluate` continues with CV-to-job evaluation.

![API Gateway Route](/images/5-Workshop/5.5-CV-Matching-Flow/5-1-api-gateway-cv-evaluate-route.png)

## Common mistakes

- Sending `userId` in the request body instead of reading it from Cognito JWT.
- Testing Lambda without `requestContext.authorizer.jwt.claims.sub`.
- Using a `cvKey` that does not belong to the authenticated user.

> Cost note: Triggering this flow may invoke Lambda, Textract, Gemini, CloudWatch Logs, S3, and DynamoDB depending on the endpoint.
