---
title : "Build Process Job Lambda"
date : 2026-07-02 
weight : 4
chapter : false
pre : " <b> 5.3.4 </b> "
---

#### 1. Initialize Lambda

Create the Lambda Function using TypeScript.

This Lambda is triggered by Amazon SQS.

Configure template:

```yaml
ProcessJobsFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: .
      Handler: src/functions/processJobs/handler.handler
      Description: Receive SQS messages and process/forward them
      Policies:
        - LambdaInvokePolicy:
            FunctionName: !Ref NormalizeAndMatchFunction
      Events:
        SQSEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt RawJobsQueue.Arn
            BatchSize: 10
      Environment:
        Variables:
          NORMALIZE_MATCH_FUNCTION_NAME: !Ref NormalizeAndMatchFunction
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        Minify: true
        Target: "es2020"
        Sourcemap: true
        EntryPoints: 
        - src/functions/processJobs/handler.ts
```

---

#### 2. Process Message

The Lambda performs:

* Read message from SQS.
* Convert data to a unified JSON structure.
* Prepare data for the Normalize stage.

Source code of the feature (paste the following snippets into their corresponding files):

* `src/functions/processJobs/handler.ts`:
```typescript
import type { SQSEvent } from "aws-lambda";
import { ProcessJobsService } from "../../services/processJobsService.js";

const processJobsService = new ProcessJobsService();
const NORMALIZE_MATCH_FUNCTION_NAME = process.env.NORMALIZE_MATCH_FUNCTION_NAME;

export const handler = async (event: SQSEvent): Promise<void> => {
  if (!NORMALIZE_MATCH_FUNCTION_NAME) {
    throw new Error("Missing NORMALIZE_MATCH_FUNCTION_NAME environment variable");
  }

  await processJobsService.processSQSEvent(event, NORMALIZE_MATCH_FUNCTION_NAME);
};

```

* `src/services/processJobsService.ts`:
```typescript
import type { SQSEvent } from "aws-lambda";
import { LambdaClient, InvokeCommand } from "@aws-sdk/client-lambda";
import type { RawJob, NormalizeMatchResult } from "../types/job.js";

export class ProcessJobsService {
  private lambdaClient: LambdaClient;

  constructor() {
    this.lambdaClient = new LambdaClient({});
  }

  async processSQSEvent(event: SQSEvent, normalizeMatchFunctionName: string): Promise<void> {
    console.log("Processing SQS records:", event.Records.length);

    for (const record of event.Records) {
      try {
        console.log(`Processing message: ${record.messageId}`);
        
        // The body is a JSON string of the raw job
        const rawJob = JSON.parse(record.body) as RawJob;

        // Invoke Python Lambda function
        const invokeCommand = new InvokeCommand({
          FunctionName: normalizeMatchFunctionName,
          Payload: Buffer.from(JSON.stringify(rawJob)),
        });

        const response = await this.lambdaClient.send(invokeCommand);
        
        if (response.FunctionError) {
          const payloadStr = response.Payload ? Buffer.from(response.Payload).toString() : "";
          console.error(`Python lambda execution failed for message ${record.messageId}: ${response.FunctionError}. Payload: ${payloadStr}`);
          throw new Error(`Process failed: ${response.FunctionError}`);
        }

        const responsePayload = response.Payload 
          ? (JSON.parse(Buffer.from(response.Payload).toString()) as NormalizeMatchResult) 
          : null;
        console.log(`Processed message ${record.messageId}. Result:`, JSON.stringify(responsePayload));
      } catch (error) {
        console.error(`Failed to process message ${record.messageId}:`, error);
        throw error; // Re-throw to make SQS retry or send to DLQ
      }
    }
  }
}

```

* `src/types/job.ts`:
```typescript
export interface RawJob {
  title?: string;
  job_title?: string;
  company_name: string;
  location?: string;
  detected_extensions?: {
    posted_at?: string;
    schedule_type?: string;
  };
  extensions?: string[];
  source_link?: string;
  description?: string;
  thumbnail?: string;
}

export interface Job {
  jobId: string;
  hash: string;
  title: string;
  originalTitle: string;
  companyName: string;
  location: string;
  postedAt: string;
  originalPostedAt: string;
  scheduleType: string;
  sourceLink: string;
  description: string;
  createdAt: string;
  thumbnail?: string;
}

export interface NormalizeMatchResult {
  status: "inserted" | "duplicate" | "rejected";
  jobId?: string;
  hash?: string;
  reason?: string;
  similarity?: number;
  matched_job_id?: string;
  received_fields?: string[];
}

```

---

#### 3. Deploy
Run the following commands in sequence:
```bash
sam build
sam validate
sam deploy
```

Deploy successfully:

![Deploy process job](/images/5-Workshop/5.3-Job-ingestion/5.3.4-setup-lambda-processjob/deploy-processjob.png)

---
