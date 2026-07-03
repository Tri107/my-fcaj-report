---
title : "Thiết lập Amazon SQS"
date : 2026-07-02 
weight : 2
chapter : false
pre : " <b> 5.3.2 </b> "
---

#### 1. Tạo Queue

Khởi tạo một Standard Queue để lưu trữ danh sách việc làm trước khi xử lý.

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
Chạy lần lượt các lệnh sau:
```bash
sam build
sam validate
sam deploy
```

Deploy thành công:
![SQS Deploy successfully](/images/5-Workshop/5.3-Job-ingestion/5.3.2-setup-sqs/deploy-sqs.png)
