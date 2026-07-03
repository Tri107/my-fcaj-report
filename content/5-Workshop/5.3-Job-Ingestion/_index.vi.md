---
title : "Jobs Ingestion Pipeline"
date : 2026-07-02
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

#### Mục tiêu

Xây dựng pipeline thu thập dữ liệu việc làm tự động từ Google Jobs thông qua SerpAPI, xử lý dữ liệu theo nhiều giai đoạn và lưu trữ vào DynamoDB.

---

#### Kiến trúc 

![Kiến trúc](/images/5-Workshop/5.3-Job-ingestion/Job-ingestion.drawio.png)

---

#### Cấu trúc thư mục backend serverless
```text
backend
├── config
│   └── database.example.ts
├── events
│   ├── fetchJobs.json
│   └── searchJobs.json
├── src
│   ├── functions
│   │   ├── fetchJobs
│   │   │   └── handler.ts
│   │   ├── normalizeAndMatch
│   │   │   └── handler.py
│   │   └── processJobs
│   │       └── handler.ts
│   ├── repositories
│   │   └── jobRepository.py
│   ├── services
│   │   ├── jobService.ts
│   │   ├── matchAndNormalizeService.py
│   │   └── processJobsService.ts
│   ├── types
│   │   └── job.ts
│   └── utils
│       └── errorHandler.ts
├── .gitignore
├── package-lock.json
├── package.json
├── samconfig.toml
├── template.yaml
└── tsconfig.json
```
Các thành phần trong luồng xử lý:

* Tạo bảng DynamoDB lưu thông tin việc làm.
* Thiết lập hàng đợi Amazon SQS để tách biệt quá trình thu thập và xử lý dữ liệu.
* Xây dựng Lambda Fetch Jobs để lấy dữ liệu từ SerpAPI và đưa vào SQS theo lịch tự động.
* Xây dựng Lambda Process Job để chuyển đổi dữ liệu thô sang định dạng JSON chuẩn.
* Xây dựng Lambda Normalize để chuẩn hóa dữ liệu và lưu vào DynamoDB.

#### Kết quả
Sau khi hoàn thành, pipeline hoạt động theo trình tự:

1. EventBridge kích hoạt Fetch Jobs Lambda lúc **06:00 (UTC+7)** mỗi ngày.
2. Fetch Jobs Lambda gọi SerpAPI để lấy danh sách việc làm.
3. Kết quả được gửi vào Amazon SQS.
4. Process Job Lambda được kích hoạt từ SQS để chuyển đổi dữ liệu sang JSON chuẩn.
5. Normalize Lambda xử lý, chuẩn hóa dữ liệu và ghi vào bảng DynamoDB.
6. DynamoDB lưu trữ dữ liệu với TTL và PITR đã được cấu hình nhằm hỗ trợ quản lý vòng đời dữ liệu và khả năng khôi phục khi cần thiết.

#### Nội dung

- [Tạo bảng DynamoDB Jobs](5.3.1-setup-job-table/)
- [Thiết lập Amazon SQS](5.3.2-setup-sqs/)
- [Xây dựng Fetch Jobs Lambda](5.3.3-setup-lambda-fetchjob/)
- [Xây dựng Process Job Lambda](5.3.4-setup-lambda-processjob/)
- [Xây dựng Normalize Lambda](5.3.5-setup-lambda-normalizeandmatchjob/)
