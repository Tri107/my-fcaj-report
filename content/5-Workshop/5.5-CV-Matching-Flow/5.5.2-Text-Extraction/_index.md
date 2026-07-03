---
title : "Text Extraction"
date : 2026-07-03
weight : 2
chapter : false
pre : " <b> 5.5.2. </b> "
---

# Text Extraction

## Goal

Text extraction converts the uploaded CV file into plain text so the AI evaluation step can compare it with the job description. The API receives only `cvKey`; it does not receive the PDF file directly.

`ExtractCvFunction` handles `POST /cv/extract`. Lambda reads `userId` from the Cognito JWT claim `sub`, validates that the `cvKey` belongs to the authenticated user, loads the CV from S3, and calls Amazon Textract.

Region: `ap-southeast-1`  
CV bucket: `jobs-matching-cvs-dev-973370772729`

## Flow

```text
POST /cv/extract
  -> ExtractCvFunction
  -> Validate Cognito user
  -> Validate cvKey ownership
  -> Read CV file from S3
  -> Call Amazon Textract
  -> Return extracted text
```

## Request body

```json
{
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf"
}
```

`userId` is not sent in the body. It is read from `requestContext.authorizer.jwt.claims.sub`.

## Expected response

```json
{
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf",
  "pageCount": 1,
  "wordCount": 428,
  "text": "DUONG BINH CUONG\nInformation Technology Student\nEmail: example@email.com\nSkills: JavaScript, React, Node.js, SQL, Postman..."
}
```

## Response fields

- `cvKey`: S3 object key of the uploaded CV.
- `pageCount`: number of pages detected from the CV.
- `wordCount`: estimated number of words extracted from the CV.
- `text`: plain text extracted from the PDF. This field can be long, so workshop samples should show a shortened version only.

## Common mistakes

- Sending the PDF file directly to `/cv/extract` instead of sending `cvKey`.
- Using a `cvKey` that does not belong to the authenticated user.
- Testing without Cognito JWT claims.
- Showing the full CV text in screenshots or logs.
- Confusing Textract extraction with AI evaluation.

> Cost note: Amazon Textract can generate cost per processed page. This step may also generate cost from Lambda, S3, and CloudWatch Logs.
