---
title : "Các API nghiệp vụ chính (Core API Lambdas)"
date : 2026-07-02
weight : 3
chapter : false
pre : " <b> 5.4.3. </b> "
---

Phần này chịu trách nhiệm hiện thực hóa các Lambda logic nghiệp vụ để kết nối API Gateway với Database và S3.

---

## 1. Luồng xử lý chi tiết (Data Flow)

### A. `GetJobsFunction` (`GET /jobs`)
* **Mô tả:** Lấy danh sách việc làm từ cơ sở dữ liệu.
* **Input (Query Parameters):**
  * `limit` (tùy chọn): số lượng bản ghi (mặc định 20, tối đa 50).
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
        "description": "Yêu cầu kỹ năng...",
        "scheduleType": "Full-time",
        "createdAt": "2026-07-01T10:00:00Z"
      }
    ]
  }
  ```

### B. `GetJobByIdFunction` (`GET /jobs/{jobId}`)
* **Mô tả:** Lấy thông tin chi tiết một công việc.
* **Input (Path Parameters):**
  * `jobId`: ID của công việc.
* **Output:**
  * **Thành công (HTTP 200):** Trả về đối tượng Job chi tiết.
  * **Không tìm thấy (HTTP 404):** `{ "message": "Job not found" }`

### C. `SearchJobsFunction` (`GET /jobs/search`)
* **Mô tả:** Tìm kiếm nâng cao kết hợp phân trang cursor và sắp xếp theo Relevance, Latest hoặc Date.
* **Input (Query Parameters):**
  * `keyword` (tùy chọn): từ khóa tìm kiếm (lọc theo title, description, company).
  * `location` (tùy chọn): lọc địa điểm.
  * `scheduleType` (tùy chọn): lọc hình thức (Full-time, Part-time, Remote, Hybrid).
  * `postedAt` (tùy chọn): lọc theo ngày.
  * `sort` (tùy chọn): sắp xếp (`relevance`, `latest`, `posted_at_asc`).
  * `limit` (tùy chọn): số lượng (mặc định 10).
  * `nextToken` (tùy chọn): token phân trang từ lượt query trước.
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
* **Mô tả:** Điều phối các thao tác yêu thích việc làm của người dùng đã xác thực.
* **Đầu vào bắt buộc:** Header `Authorization: Bearer <JWT Token>`.

| Phương thức & Endpoint | Input bổ sung | Output trả về |
| :--- | :--- | :--- |
| **`POST /saved-jobs/{jobId}`** | `jobId` (Path Parameter) | `{ "message": "Job saved successfully", "item": { "userId": "...", "jobId": "...", "savedAt": "..." } }` |
| **`DELETE /saved-jobs/{jobId}`** | `jobId` (Path Parameter) | `{ "message": "Job removed from saved list" }` |
| **`GET /saved-jobs`** | Không | `{ "count": 1, "items": [ { "userId": "...", "jobId": "...", "savedAt": "..." } ] }` |

### E. `ManageCvFunction` (`/cv`)
* **Mô tả:** Điều phối các thao tác quản lý CV trên S3 cho người dùng đã xác thực.
* **Đầu vào bắt buộc:** Header `Authorization: Bearer <JWT Token>`.

| Phương thức & Endpoint | Input bổ sung | Output trả về |
| :--- | :--- | :--- |
| **`POST /cv/upload`** | Header `x-original-filename`<br>Body: binary data | `{ "message": "CV uploaded successfully", "data": { "key": "...", "url": "...", "originalFilename": "...", "sizeBytes": 1024, "uploadedAt": "..." } }` |
| **`GET /cv`** | Không | `{ "count": 1, "items": [ { "key": "...", "filename": "...", "sizeBytes": 1024, "url": "...", "contentHash": "..." } ] }` |
| **`DELETE /cv/{filename}`** | `filename` (Path Parameter) | `{ "message": "CV deleted successfully", "deletedKey": "..." }` |

---

## 2. Cấu hình Runtime & Trình biên dịch (Configuration)
* **Runtime:** Node.js 24.x (`nodejs24.x`).
* **Trình biên dịch:** `esbuild` cấu hình thu gọn (minify) code, tạo sourcemap để hỗ trợ giám sát lỗi.
* **Phân quyền tối thiểu (Least Privilege):** 
  * Lambda đọc Jobs chỉ được gán Policy `DynamoDBReadPolicy` trên `JobsTable`.
  * Lambda quản lý CV chỉ được gán Policy thao tác trên tiền tố S3 nhất định và không có quyền xóa bucket.

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

## 4. Triển khai (Deployment)
Biên dịch mã nguồn TypeScript sang mã máy tối ưu bằng `sam build`, và đẩy lên AWS:
```bash
sam build
sam deploy
```

Sau khi triển khai thành công, bạn có thể kiểm tra danh sách các Lambda Function trên AWS Console:

![Lambda Functions](/images/5-Workshop/5.4-User-Interaction/lambdas_funtion.png)
