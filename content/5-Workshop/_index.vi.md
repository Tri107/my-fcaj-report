---
title: "Workshop"
date: 2026-07-02
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Xây dựng Hệ thống Jobs Matching Platform Serverless

#### Tổng quan

Trong workshop này, chúng ta sẽ xây dựng **Jobs Matching Platform** - một hệ thống tự động tổng hợp việc làm và phân tích hồ sơ ứng viên (CV) hoàn toàn bằng các dịch vụ AWS Serverless.

Dự án sử dụng kiến trúc Serverless (như AWS Lambda, Amazon DynamoDB, Amazon SQS, EventBridge) để đảm bảo khả năng mở rộng tự động, độ sẵn sàng cao và tối ưu chi phí. Hệ thống sẽ có hai luồng xử lý chính: luồng thu thập dữ liệu việc làm (Job Ingestion) và luồng phân tích CV của người dùng (User Interaction & CV Matching).

#### Nội dung

1. [Tổng quan về workshop](5.1-Workshop-overview/)
2. [Các bước chuẩn bị](5.2-Prerequiste/)
3. [Jobs Ingestion Pipeline](5.3-Job-Ingestion/)
4. [Tương tác người dùng & Phân tích CV](5.4-User-Interaction/)
5. [VPC Endpoint Policies (Làm thêm)](5.5-Policy/)
6. [Dọn dẹp tài nguyên](5.6-Cleanup/)

Github:https://github.com/Tri107/Jobs-Matching-Platform