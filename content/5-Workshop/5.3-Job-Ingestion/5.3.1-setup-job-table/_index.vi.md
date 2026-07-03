---
title : "Tạo bảng DynamoDB Jobs"
date : 2026-07-02 
weight : 1
chapter : false
pre : " <b> 5.3.1 </b> "
---

#### 1. Khởi tạo DynamoDB Table

Tạo bảng **Jobs** dùng để lưu trữ thông tin việc làm sau khi được chuẩn hóa.

Các thuộc tính chính gồm:

* Partition Key
* Sort Key (nếu có)
* Các thuộc tính lưu thông tin công việc

Cấu hình bảng Jobs như sau:

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
Thực hiện lần lượt các lệnh sau để tiến hành triển khai cấu hình lên cloud:
```bash
cd backend # Di chuyển đến thư mục chứa file template.yaml
sam build  # Build
sam validate  # Validate template.yaml
sam deploy --guided # Deploy
```
Deploy thành công:
![Deploy table successfuly](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/deploy-cloudformation.png)
![Deploy table successfuly](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/deploy-table-success.png)

---

#### 3. Cấu hình TTL

Kích hoạt TTL (Time To Live - cơ chế tự động xóa dữ liệu hết hạn) để tự động loại bỏ các bản ghi cũ sau một khoảng thời gian xác định.

Cấu hình trường TTL:

![Cấu hình TTL](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/turn-on-ttl.png)
![Cấu hình TTL](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/turn-on-ttl-2.png)


---

#### 4. Bật Point-in-Time Recovery

Kích hoạt PITR (Point-in-Time Recovery - khả năng khôi phục dữ liệu tại bất kỳ thời điểm nào trong 35 ngày gần nhất) nhằm tăng khả năng phục hồi khi xảy ra lỗi.

![Cấu hình PITR](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/pitr-config-1.png)
![Cấu hình PITR](/images/5-Workshop/5.3-Job-ingestion/5.3.1-setup-job-table/pitr-config-2.png)
