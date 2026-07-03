---
title : "Create DynamoDB Jobs Table"
date : 2026-07-02 
weight : 1
chapter : false
pre : " <b> 5.3.1 </b> "
---

#### 1. Initialize DynamoDB Table

Create the **Jobs** table to store job information after normalization.

Key attributes include:

* Partition Key
* Sort Key (if applicable)
* Job details properties

Configure the Jobs table as follows:

```yaml
  JobsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub "${ProjectName}-jobs-${StageName}"
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: jobId
          AttributeType: S
        - AttributeName: hash
          AttributeType: S
      KeySchema:
        - AttributeName: jobId
          KeyType: HASH
      GlobalSecondaryIndexes:
        - IndexName: HashIndex
          KeySchema:
            - AttributeName: hash
              KeyType: HASH
          Projection:
            ProjectionType: ALL
```

---

#### 2. Deploy 
Run the following commands in sequence to deploy the configuration to the cloud:
```bash
cd backend # Navigate to the directory containing template.yaml
sam build  # Build
sam validate  # Validate template.yaml
sam deploy --guided # Deploy
```
Deploy successfuly:
![Deploy table successfuly](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/deploy-cloudformation.png)
![Deploy table successfuly](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/deploy-table-success.png)

---

#### 3. Configure TTL

Enable TTL (Time To Live - mechanism to automatically delete expired data) to automatically remove old records after a specified period.

Configure the TTL field:

![TTL Configuration](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/turn-on-ttl.png)
![TTL Configuration](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/turn-on-ttl-2.png)


---

#### 4. Enable Point-in-Time Recovery

Enable PITR (Point-in-Time Recovery - ability to restore data to any point in time within the last 35 days) to improve recovery capabilities in case of failures.

![PITR Configuration](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/pitr-config-1.png)
![PITR Configuration](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/pitr-config-2.png)
