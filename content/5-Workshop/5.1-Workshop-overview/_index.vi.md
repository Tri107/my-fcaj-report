---
title : "Giới thiệu"
date : 2026-07-02 
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

#### Giới thiệu về kiến trúc Serverless

+ **Serverless** là mô hình thực thi điện toán đám mây trong đó nhà cung cấp đám mây tự động quản lý việc phân bổ tài nguyên, vận hành máy chủ và bảo trì cơ sở hạ tầng. Điều này giúp các nhà phát triển có thể tập trung hoàn toàn vào việc xây dựng ứng dụng thay vì quản lý máy chủ.
+ Việc sử dụng các dịch vụ Serverless như AWS Lambda, Amazon DynamoDB hay Amazon SQS không chỉ giảm thiểu chi phí theo mô hình "dùng bao nhiêu trả bấy nhiêu" (pay-as-you-go) mà còn đảm bảo khả năng mở rộng tự động (auto-scaling) và tính sẵn sàng cao (High Availability) cho hệ thống.

#### Tổng quan về workshop
Trong workshop này, bạn sẽ xây dựng **Jobs Matching Platform** - một hệ thống tự động tổng hợp việc làm và phân tích hồ sơ ứng viên (CV) bằng cách sử dụng hoàn toàn các dịch vụ AWS Serverless.

Workshop sẽ bao gồm các luồng xử lý chính:
+ **Luồng thu thập dữ liệu (Data Ingestion)**: Sử dụng AWS EventBridge Scheduler để kích hoạt AWS Lambda lấy dữ liệu việc làm qua API (SerpApi), điều phối luồng dữ liệu thông qua Amazon SQS và lưu trữ thông tin tại Amazon DynamoDB.
+ **Luồng phân tích CV (CV Matching Flow)**: Khi ứng viên tải hồ sơ (CV) lên Amazon S3, sự kiện sẽ tự động kích hoạt Lambda trích xuất văn bản, gọi dịch vụ AI (Gemini) để phân tích, đối chiếu với mô tả công việc và chấm điểm mức độ phù hợp.
+ **Tương tác và bảo mật (User Interaction & Security)**: Giao diện web được lưu trữ trên AWS Amplify. Định danh người dùng bằng Amazon Cognito và bảo vệ toàn bộ RESTful API thông qua AWS WAF kết hợp cùng Amazon API Gateway.


![Design Architecture](/images/2-Proposal/Jobs-Matching-Platform -v3.drawio.png)