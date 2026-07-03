---
title : "Setup Amazon SQS"
date : 2026-07-02 
weight : 2
chapter : false
pre : " <b> 5.3.2 </b> "
---

#### 1. Create Queue

Initialize a Standard Queue to store the job list before processing.

```yaml
  RawJobsDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "${ProjectName}-raw-jobs-dlq-${StageName}"
      MessageRetentionPeriod: 1209600 # 14 days

  RawJobsQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "${ProjectName}-raw-jobs-queue-${StageName}"
      VisibilityTimeout: 60 # Should be >= Lambda timeout processing it
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt RawJobsDLQ.Arn
        maxReceiveCount: 3

```

---

#### 2. Deploy
Run the following commands in sequence:
```bash
sam build
sam validate
sam deploy
```

Deploy successfully:
![SQS Deploy successfully](/images/5-Workshop/5.3-Job-ingestion/5.3.2-setup-sqs/deploy-sqs.png)
