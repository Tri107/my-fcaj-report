---
title : "Jobs Ingestion Pipeline"
date : 2026-07-02
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

#### Objectives

Build an automated job collection pipeline from Google Jobs via SerpAPI, process data in multiple stages, and store it in DynamoDB.

---

#### Architecture 

![Architecture](/images/5-Workshop/5.3-Job-ingestion/Job-ingestion.drawio.png)

---

#### Backend Serverless Folder Structure
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
Components in the processing flow:

* Create a DynamoDB table to store job information.
* Set up Amazon SQS queue to decouple the data collection and processing.
* Build Fetch Jobs Lambda to get data from SerpAPI and push it to SQS on an automated schedule.
* Build Process Job Lambda to convert raw data into standard JSON format.
* Build Normalize Lambda to normalize the data and save it to DynamoDB.

#### Execution Sequence
After completion, the pipeline operates in the following order:

1. EventBridge triggers Fetch Jobs Lambda at **06:00 (UTC+7)** every day.
2. Fetch Jobs Lambda calls SerpAPI to retrieve the job list.
3. The results are sent to Amazon SQS.
4. Process Job Lambda is triggered by SQS to transform the data into standard JSON.
5. Normalize Lambda processes, normalizes the data, and writes it to the DynamoDB table.
6. DynamoDB stores the data with TTL and PITR configured to support data lifecycle management and recovery capabilities.

#### Content

- [Create DynamoDB Jobs Table](5.3.1-setup-job-table/)
- [Setup Amazon SQS](5.3.2-setup-sqs/)
- [Build Fetch Jobs Lambda](5.3.3-setup-lambda-fetchjob/)
- [Build Process Job Lambda](5.3.4-setup-lambda-processjob/)
- [Build Normalize Lambda](5.3.5-setup-lambda-normalizeandmatchjob/)
