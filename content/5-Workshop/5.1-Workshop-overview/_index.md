---
title : "Introduction"
date : 2026-07-02 
weight : 1 
chapter : false
pre : " <b> 5.1. </b> "
---

#### Introduction to Serverless Architecture
+ **Serverless** is a cloud computing execution model in which the cloud provider dynamically manages the allocation and provisioning of servers. This eliminates infrastructure management tasks, allowing developers to focus entirely on writing and deploying code.
+ Utilizing Serverless services such as AWS Lambda, Amazon DynamoDB, and Amazon SQS not only minimizes costs through a "pay-as-you-go" pricing model but also inherently provides automatic scalability and high availability for the system.

#### Workshop overview
In this workshop, you will build the **Jobs Matching Platform** - an automated system for aggregating job listings and evaluating candidate profiles (CVs) entirely using AWS Serverless services.

The workshop consists of these main workflows:
+ **Data Ingestion Flow**: Using AWS EventBridge Scheduler to trigger AWS Lambda for retrieving job data via an external API (SerpApi), buffering the workload through Amazon SQS, and storing the standardized data in Amazon DynamoDB.
+ **CV Matching Flow**: When a user uploads a resume to Amazon S3, an event automatically triggers Lambda to extract the text content, communicate with an AI service (Gemini) to evaluate the CV against the job description, and generate a suitability score.
+ **User Interaction & Security**: The front-end web application is hosted on AWS Amplify. User authentication is managed by Amazon Cognito, and all RESTful APIs are securely distributed and protected by AWS WAF combined with Amazon API Gateway.


![Design Architecture](/images/2-Proposal/Jobs-Matching-Platform-v3.drawio.png)