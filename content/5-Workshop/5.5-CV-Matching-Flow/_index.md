---
title : "CV Matching Flow"
date : 2026-07-03
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

## Overview

This section explains the CV Matching Flow used by the AI Job Matching Platform. The flow starts when a user requests a CV-to-job evaluation from the frontend. The request goes through API Gateway to Lambda, then Lambda reads the CV from S3, extracts text with Amazon Textract, loads the job description from DynamoDB, evaluates the match with Gemini 3.1 Flash Lite, and stores the result in DynamoDB.

Current region: `ap-southeast-1`.

## Architecture flow

```text
User action on frontend
  -> API Gateway HTTP API
  -> Evaluate Match Lambda
  -> Read CV file from S3
  -> Extract CV text with Amazon Textract
  -> Read job description from DynamoDB Jobs table
  -> Evaluate match with Gemini 3.1 Flash Lite
  -> Store result in DynamoDB Match Results table
```

![CV Matching Flow Architecture](/images/5-Workshop/5.5-CV-Matching-Flow/Jobs-Matching-Platform -v3.drawio.png)

## Implementation references

- API Gateway base URL: `https://38icub30ig.execute-api.ap-southeast-1.amazonaws.com/dev`
- CV bucket: `jobs-matching-cvs-dev-973370772729`
- Match results table: `jobs-matching-match-results-dev`
- Evaluate Lambda ARN: `arn:aws:lambda:ap-southeast-1:973370772729:function:jobs-matching-evaluate-match-dev`

## What this section covers

- How the frontend starts the matching flow through API Gateway
- How Lambda reads CV files from S3 and extracts text with Amazon Textract
- How job data is loaded from DynamoDB
- How Gemini 3.1 Flash Lite evaluates the match
- How match results are stored for later use

> Cost note: This flow may generate cost from Amazon Textract, AWS Lambda, CloudWatch Logs, Amazon S3, Amazon DynamoDB, and Gemini API depending on usage and billing tier.
