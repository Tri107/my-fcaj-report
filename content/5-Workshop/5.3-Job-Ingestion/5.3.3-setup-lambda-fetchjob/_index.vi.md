---
title : "Xây dựng Fetch Jobs Lambda"
date : 2026-07-02 
weight : 3
chapter : false
pre : " <b> 5.3.3 </b> "
---

#### 1. Khởi tạo Lambda

Tạo Lambda Function sử dụng TypeScript.

Chức năng:
* Được trigger bởi event schedule
* Đọc danh sách từ khóa tìm kiếm.
* Gọi SerpAPI.
* Nhận kết quả Google Jobs.
* Đẩy từng bản ghi vào Amazon SQS.

> Định nghĩa function trong template:
```yaml
FetchJobsFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub "${ProjectName}-fetch-jobs-${StageName}"
      CodeUri: .
      Handler: src/functions/fetchJobs/handler.handler
      Description: Fetch jobs from SerpAPI and push to SQS
      Timeout: 120
      Policies:
        - SQSSendMessagePolicy:
            QueueName: !GetAtt RawJobsQueue.QueueName
      Environment:
        Variables:
          QUEUE_URL: !Ref RawJobsQueue
          SERPAPI_KEY: !Ref SerpApiKey
      Events:
        DailyFetchJobsSchedule:
          Type: ScheduleV2
          Properties:
            Name: !Sub "${ProjectName}-fetch-jobs-6am-${StageName}"
            Description: "Run FetchJobsFunction every day at 06:00 Vietnam time"
            ScheduleExpression: "cron(0 6 * * ? *)"
            ScheduleExpressionTimezone: "Asia/Ho_Chi_Minh"
            FlexibleTimeWindow:
              Mode: "OFF"
            State: ENABLED
            Input: |
              {
                "source": "eventbridge-scheduler",
                "location": "Vietnam",
               "queries": [
                  "Backend Developer",
                  "Frontend Developer",
                  "Cloud Engineer",
                  "Tester",
                  "Business Analyst",
                  "IT Helpdesk",
                  "Game Developer",
                  "DevOps Engineer"
                ]
              }
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        Minify: true
        Target: "es2020"
        Sourcemap: true
        EntryPoints: 
        - src/functions/fetchJobs/handler.ts
```

Code thực thi (paste các đoạn mã sau vào các file tương ứng):

* `src/functions/fetchJobs/handler.ts`:
```typescript
import { JobService } from "../../services/jobService.js";

const jobService = new JobService();

export const handler = async (event: any): Promise<any> => {
  console.log("Event:", JSON.stringify(event));

  const queueUrl = process.env.QUEUE_URL;
  const serpApiKey = process.env.SERPAPI_KEY;

  if (!queueUrl || !serpApiKey) {
    throw new Error("Missing QUEUE_URL or SERPAPI_KEY environment variables");
  }

  const location = event.location || "Vietnam";

  // EventBridge gửi mảng queries, test thủ công gửi 1 query string
  const queries: string[] = event.queries
    ? event.queries
    : [event.query || "Backend Developer"];

  let totalFound = 0;
  let totalPushed = 0;
  const results: Array<{ query: string; found: number; pushed: number }> = [];

  for (const query of queries) {
    try {
      console.log(`--- Fetching: "${query}" in "${location}" ---`);
      const jobs = await jobService.fetchJobsFromSerpApi(query, location, serpApiKey);

      let pushedCount = 0;
      if (jobs.length > 0) {
        pushedCount = await jobService.pushJobsToSQS(jobs, queueUrl);
      }

      totalFound += jobs.length;
      totalPushed += pushedCount;
      results.push({ query, found: jobs.length, pushed: pushedCount });

      // Delay 2s giữa các request để tránh rate limit SerpAPI
      if (queries.length > 1) {
        await new Promise((resolve) => setTimeout(resolve, 2000));
      }
    } catch (error) {
      console.error(`Error fetching "${query}":`, error);
      results.push({ query, found: 0, pushed: 0 });
    }
  }

  console.log(`=== SUMMARY: ${totalFound} jobs found, ${totalPushed} pushed to SQS ===`);

  return {
    message: "Fetch jobs completed",
    source: event.source || "manual",
    location,
    totalQueries: queries.length,
    totalFound,
    totalPushed,
    results,
  };
};

```

* `src/services/jobService.ts`:
```typescript
import { SQSClient, SendMessageBatchCommand, type SendMessageBatchRequestEntry } from "@aws-sdk/client-sqs";
import { randomUUID } from "crypto";

export class JobService {
  private sqsClient: SQSClient;
  
  constructor() {
    this.sqsClient = new SQSClient({});
  }

  async fetchJobsFromSerpApi(query: string, location: string, apiKey: string): Promise<any[]> {
    const params = new URLSearchParams({
      engine: "google_jobs",
      q: query,
      location: location,
      google_domain: "google.com.vn",
      hl: "vi",
      gl: "vn",
      api_key: apiKey
    });

    const url = `https://serpapi.com/search?${params.toString()}`;
    console.log(`Fetching jobs from SerpAPI for query: ${query} in ${location}`);
    
    const response = await fetch(url);
    
    if (!response.ok) {
      const errText = await response.text();
      throw new Error(`SerpAPI error: ${response.status} - ${errText}`);
    }

    const data = (await response.json()) as Record<string, any>;
    const jobs = data.jobs_results || [];
    console.log(`SerpAPI returned ${jobs.length} job results`);
    
    return jobs;
  }

  async pushJobsToSQS(jobs: any[], queueUrl: string): Promise<number> {
    if (jobs.length === 0) return 0;
    const batchSize = 10;
    let processedCount = 0;

    for (let i = 0; i < jobs.length; i += batchSize) {
      const batch = jobs.slice(i, i + batchSize);
      
      const entries: SendMessageBatchRequestEntry[] = batch.map((job: any) => ({
        Id: randomUUID(),
        MessageBody: JSON.stringify(job),
      }));

      const command = new SendMessageBatchCommand({
        QueueUrl: queueUrl,
        Entries: entries,
      });

      const batchResult = await this.sqsClient.send(command);
      console.log(`Batch sent. Successful: ${batchResult.Successful?.length || 0}, Failed: ${batchResult.Failed?.length || 0}`);
      processedCount += batchResult.Successful?.length || 0;
    }

    return processedCount;
  }
}
```

---

#### 2. Cấu hình Environment Variables

Cấu hình các biến môi trường:
Thực hiện lệnh sau để đẩy biến môi trường vào Parameter Store để sử dụng trên môi trường Lambda:
```bash
aws ssm put-parameter \
  --name "/jobs-matching/dev/serpapi-key" \
  --value <SERPAPI KEY> \
  --type SecureString
```

---

#### 3. Deploy
Chạy lần lượt các lệnh sau:
```bash
sam build
sam validate
sam deploy
```
Deploy schedule event & lambda thành công:
![Deploy schedule event](/images/5-Workshop/5.3-Job-ingestion/5.3.3-setup-lambda-fetchjob/deploy-schedule-event.png)
![Deploy lambda](/images/5-Workshop/5.3-Job-ingestion/5.3.3-setup-lambda-fetchjob/deploy-fetchjob.png)

---

#### 4. Kiểm tra

Thực hiện kiểm tra:

* Lambda thực thi thành công.
* Gọi được SerpAPI.
* Message được gửi tới SQS.

![Kiểm tra Fetch Jobs](/images/5-Workshop/5.3-Job-ingestion/5.3.3-setup-lambda-fetchjob/test-fetchjob.png)
