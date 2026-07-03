---
title : "Core Business APIs (Core API Lambdas)"
date : 2026-07-02
weight : 3
chapter : false
pre : " <b> 5.4.3. </b> "
---

This section is responsible for implementing business logic inside AWS Lambda functions to connect API Gateway with the DynamoDB database and S3 storage.

---

## 1. Detailed Data Flow

### A. `GetJobsFunction` (`GET /jobs`)
* **Description:** Retrieve job listings from the database.
* **Input (Query Parameters):**
  * `limit` (optional): number of records to retrieve (default 20, max 50).
* **Output:**
  ```json
  {
    "count": 1,
    "items": [
      {
        "jobId": "job-abc-123",
        "title": "React Frontend Developer",
        "companyName": "Tech Corp",
        "location": "Hồ Chí Minh",
        "description": "Skills requirement...",
        "scheduleType": "Full-time",
        "createdAt": "2026-07-01T10:00:00Z"
      }
    ]
  }
  ```

### B. `GetJobByIdFunction` (`GET /jobs/{jobId}`)
* **Description:** Retrieve details of a specific job by its ID.
* **Input (Path Parameters):**
  * `jobId`: The ID of the job.
* **Output:**
  * **Success (HTTP 200):** Detailed Job object.
  * **Not Found (HTTP 404):** `{ "message": "Job not found" }`

### C. `SearchJobsFunction` (`GET /jobs/search`)
* **Description:** Advanced search combining cursor pagination and sorting by Relevance, Latest, or Date.
* **Input (Query Parameters):**
  * `keyword` (optional): search query (filters by title, description, company).
  * `location` (optional): filters by location.
  * `scheduleType` (optional): filters by type (Full-time, Part-time, Remote, Hybrid).
  * `postedAt` (optional): filters by date.
  * `sort` (optional): sorting criteria (`relevance`, `latest`, `posted_at_asc`).
  * `limit` (optional): result limit (default 10).
  * `nextToken` (optional): pagination token from the previous query.
* **Output:**
  ```json
  {
    "count": 1,
    "items": [ ... ],
    "nextToken": "eyJsYXN0S2V5Ijp7ImpvYklkIjp7IlMiOiJqb2ItMTIzIn19fQ==",
    "filters": {
      "keyword": "react",
      "limit": 10,
      "sort": "relevance"
    }
  }
  ```

### D. `SavedJobsFunction` (`/saved-jobs`)
* **Description:** Orchestrates favoriting operations for authenticated users.
* **Required Input:** `Authorization: Bearer <JWT Token>` header.

| Method & Endpoint | Additional Input | Output Returned |
| :--- | :--- | :--- |
| **`POST /saved-jobs/{jobId}`** | `jobId` (Path Parameter) | `{ "message": "Job saved successfully", "item": { "userId": "...", "jobId": "...", "savedAt": "..." } }` |
| **`DELETE /saved-jobs/{jobId}`** | `jobId` (Path Parameter) | `{ "message": "Job removed from saved list" }` |
| **`GET /saved-jobs`** | None | `{ "count": 1, "items": [ { "userId": "...", "jobId": "...", "savedAt": "..." } ] }` |

### E. `ManageCvFunction` (`/cv`)
* **Description:** Orchestrates resume management operations on S3 for authenticated users.
* **Required Input:** `Authorization: Bearer <JWT Token>` header.

| Method & Endpoint | Additional Input | Output Returned |
| :--- | :--- | :--- |
| **`POST /cv/upload`** | `x-original-filename` Header<br>Body: binary data | `{ "message": "CV uploaded successfully", "data": { "key": "...", "url": "...", "originalFilename": "...", "sizeBytes": 1024, "uploadedAt": "..." } }` |
| **`GET /cv`** | None | `{ "count": 1, "items": [ { "key": "...", "filename": "...", "sizeBytes": 1024, "url": "...", "contentHash": "..." } ] }` |
| **`DELETE /cv/{filename}`** | `filename` (Path Parameter) | `{ "message": "CV deleted successfully", "deletedKey": "..." }` |

---

## 2. Runtime & Compiler Configuration
* **Runtime:** Node.js 24.x (`nodejs24.x`).
* **Compiler:** `esbuild` configured to minify code and generate sourcemaps for error monitoring.
* **Least Privilege Access:** 
  * Jobs-reading Lambda is granted only the `DynamoDBReadPolicy` on the `JobsTable`.
  * Resume-managing Lambda is granted access to actions under specific S3 prefixes and cannot delete the bucket.

---

## 3. AWS SAM Template Config (`template.yaml`)

```yaml
  GetJobsFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: jobs-matching-get-jobs-dev
      CodeUri: .
      Handler: src/functions/jobs/getJobs.handler
      Description: Public endpoint to list jobs from DynamoDB
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref JobsTable
      Environment:
        Variables:
          JOBS_TABLE: !Ref JobsTable
      Events:
        GetJobsApi:
          Type: HttpApi
          Properties:
            ApiId: !Ref BackendHttpApi
            Path: /jobs
            Method: GET
            PayloadFormatVersion: "2.0"
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        Minify: true
        Target: "es2020"
        Sourcemap: true
        EntryPoints:
          - src/functions/jobs/getJobs.ts

  GetJobByIdFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: jobs-matching-get-job-by-id-dev
      CodeUri: .
      Handler: src/functions/jobs/getJobById.handler
      Description: Public endpoint to get a single job by ID from DynamoDB
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref JobsTable
      Environment:
        Variables:
          JOBS_TABLE: !Ref JobsTable
      Events:
        GetJobByIdApi:
          Type: HttpApi
          Properties:
            ApiId: !Ref BackendHttpApi
            Path: /jobs/{jobId}
            Method: GET
            PayloadFormatVersion: "2.0"
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        Minify: true
        Target: "es2020"
        Sourcemap: true
        EntryPoints:
          - src/functions/jobs/getJobById.ts

  SearchJobsFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: jobs-matching-search-jobs-dev
      CodeUri: .
      Handler: src/functions/jobs/searchJobs.handler
      Description: Public endpoint to search and filter jobs from DynamoDB
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref JobsTable
      Environment:
        Variables:
          JOBS_TABLE: !Ref JobsTable
      Events:
        SearchJobsApi:
          Type: HttpApi
          Properties:
            ApiId: !Ref BackendHttpApi
            Path: /jobs/search
            Method: GET
            PayloadFormatVersion: "2.0"
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        Minify: true
        Target: "es2020"
        Sourcemap: true
        EntryPoints:
          - src/functions/jobs/searchJobs.ts

  SavedJobsFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: jobs-matching-saved-jobs-dev
      CodeUri: .
      Handler: src/functions/savedJobs/handler.handler
      Description: Manage user saved/favorite jobs (save, unsave, list)
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref SavedJobsTable
      Environment:
        Variables:
          SAVED_JOBS_TABLE: !Ref SavedJobsTable
      Events:
        SaveJobApi:
          Type: HttpApi
          Properties:
            ApiId: !Ref BackendHttpApi
            Path: /saved-jobs/{jobId}
            Method: POST
            PayloadFormatVersion: "2.0"
            Auth:
              Authorizer: CognitoJwtAuthorizer
        UnsaveJobApi:
          Type: HttpApi
          Properties:
            ApiId: !Ref BackendHttpApi
            Path: /saved-jobs/{jobId}
            Method: DELETE
            PayloadFormatVersion: "2.0"
            Auth:
              Authorizer: CognitoJwtAuthorizer
        GetSavedJobsApi:
          Type: HttpApi
          Properties:
            ApiId: !Ref BackendHttpApi
            Path: /saved-jobs
            Method: GET
            PayloadFormatVersion: "2.0"
            Auth:
              Authorizer: CognitoJwtAuthorizer
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        Minify: true
        Target: "es2020"
        Sourcemap: true
        EntryPoints:
          - src/functions/savedJobs/handler.ts

  ManageCvFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: jobs-matching-manage-cv-dev
      CodeUri: .
      Handler: src/functions/manageCv/handler.handler
      Description: Protected endpoint to manage user CVs (Upload, Get, Delete)
      Policies:
        - Statement:
            - Effect: Allow
              Action:
                - s3:ListBucket
              Resource: !GetAtt CvBucket.Arn
            - Effect: Allow
              Action:
                - s3:PutObject
                - s3:GetObject
                - s3:DeleteObject
                - s3:HeadObject
              Resource: !Sub "${CvBucket.Arn}/*"
      Environment:
        Variables:
          CV_BUCKET_NAME: !Ref CvBucket
      Events:
        UploadCvApi:
          Type: HttpApi
          Properties:
            ApiId: !Ref BackendHttpApi
            Path: /cv/upload
            Method: POST
            PayloadFormatVersion: "2.0"
            Auth:
              Authorizer: CognitoJwtAuthorizer
        GetCvsApi:
          Type: HttpApi
          Properties:
            ApiId: !Ref BackendHttpApi
            Path: /cv
            Method: GET
            PayloadFormatVersion: "2.0"
            Auth:
              Authorizer: CognitoJwtAuthorizer
        DeleteCvApi:
          Type: HttpApi
          Properties:
            ApiId: !Ref BackendHttpApi
            Path: /cv/{filename}
            Method: DELETE
            PayloadFormatVersion: "2.0"
            Auth:
              Authorizer: CognitoJwtAuthorizer
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        Minify: true
        Target: "es2020"
        Sourcemap: true
        EntryPoints:
          - src/functions/manageCv/handler.ts
```

---

## 4. Deployment
Compile the TypeScript source code into optimized production builds using `sam build`, and deploy to AWS:
```bash
sam build
sam deploy
```

Once deployed successfully, you can verify the Lambda functions list in the AWS Console:

![Lambda Functions](/images/5-Workshop/5.4-User-Interaction/lambdas_funtion.png)
