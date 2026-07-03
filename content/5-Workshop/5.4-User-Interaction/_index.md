---
title : "User Interaction"
date : 2026-07-02
weight : 4
chapter : false
pre : " <b> 5.4. </b> "
---

#### Overview

In this section, we will explore the architecture and detailed data flows for the components that directly handle user interactions within the Job-Matching platform.

The core modules include:
1. **User Authentication (Cognito Auth):** Setting up sign-up, sign-in, and securing API requests with JWT Tokens.
2. **Databases & Storage (DynamoDB & S3):** Saving favorite jobs in DynamoDB and managing resume PDFs in S3.
3. **Core API Lambdas:** Business logic coordinating job queries and resume uploads via AWS Lambda.
4. **API Gateway & WAF:** Managing request routing, CORS, and protecting endpoints from web application vulnerabilities.
5. **Frontend Hosting (Amplify):** Hosting the Next.js application with continuous deployment.
