---
title: "Proposal"
date: 2026-07-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---
# Jobs Matching Platform
## Comprehensive Serverless AWS Solution for Automated Job Aggregation and Profile Analysis

### 1. Executive Summary
The Jobs Matching Platform, designed by the NOVA team, aims to fully automate the job data collection process and evaluate the suitability of candidate profiles (CVs). This solution directly serves the need to optimize the job search process, particularly targeting students and IT personnel. By applying artificial intelligence to competency matching, the platform helps users shorten search times and accurately pinpoint suitable job positions. The system fully leverages the AWS Serverless ecosystem to ensure high availability, flexible scalability, and optimal operational costs.

### 2. Problem Statement
*Current Problem*
The current process of searching for recruitment information across various decentralized platforms requires a significant amount of time and manual effort. Job seekers often encounter obstacles in objectively and accurately self-evaluating their personal competencies against the specific requirements of each job description, especially for specialized skill sets like programming or cloud infrastructure administration.

*Solution*
The system utilizes AWS Lambda services to automatically collect job information via SerpApi, coordinates and controls data flow through Amazon SQS, and then stores it structurally in Amazon DynamoDB. The user interaction subsystem is protected by AWS WAF and handles authentication via Amazon Cognito. The automated CV analysis workflow extracts text from documents uploaded to Amazon S3, followed by integrating AI services for matching, semantic analysis, and suitability scoring. The entire software development and task management process is tightly operated through Jira, combined with source code version control on GitHub.

*Benefits and Return on Investment (ROI)*
The solution moves towards standardizing the entire end-to-end job search and filtering process. Applying practical knowledge from labs and self-research on AWS Cloud architecture helps design a high-performance system with minimal operational costs. The serverless infrastructure platform ensures costs are incurred solely based on actual computational traffic, providing optimal economic efficiency and transparent system scalability in the future.

### 3. Solution Architecture
The platform is built on AWS Serverless architecture principles, divided into independent yet tightly interconnected processing workflows: data ingestion, user interaction, profile analysis, and security operations.

*AWS Services Used*
- *AWS EventBridge Scheduler*: Schedules automated triggers for job data retrieval from SerpApi.
- *AWS Lambda*: Handles business logic processing for data fetching, searching, CV text extraction, and communication with the AI model API.
- *Amazon SQS*: Manages message queues, acting as a buffer mechanism to prevent overload during the standardization of large volumes of job data.
- *Amazon Cognito*: Manages user identity, authentication, and access authorization.
- *AWS Amplify*: Provides the hosting and distribution environment for the web front-end application (Hosting).
- *AWS WAF & Amazon API Gateway*: Establishes a defense line to protect the application from web security vulnerabilities and securely distributes RESTful API traffic.
- *Amazon DynamoDB*: Stores job lists, personal wishlists, and CV evaluation results.
- *Amazon S3*: Provides highly durable, secure, and infinitely scalable storage for candidate profile files.
- *AWS IAM, Amazon CloudWatch, AWS Budgets*: Enforces core authorization management, monitors system health, and strictly controls operational budgets.
- *AWS Systems Manager (SSM) Parameter Store*: Securely stores and manages API keys and sensitive configurations.

*Component Design*
![Component Design Architecture](/images/2-Proposal/edge_architecture.jpeg)

- *Jobs Ingestion*: Collects scheduled data from third-party platforms, processes it asynchronously via SQS, performs data standardization using Lambda, and stores it in DynamoDB with a configured Time-To-Live (TTL) mechanism.
- *User Interaction*: The front-end interface communicates via API Gateway, enabling queries on DynamoDB to filter, search for jobs, and save them to an independent wishlist table.
- *CV Matching Flow*: Automatically triggered based on user CV upload events to S3. The system extracts text content, forwards data to the AI service for multi-dimensional matching against job descriptions, and records the returned score.

### 4. Technical Implementation
The project is structured into 3 main parts: Data Collection and Processing; Building the Job Aggregation System; Job-Based CV Evaluation. Each part is implemented across 4 specific phases:
1. *Research and Design*: In-depth analysis of the data structure returned by SerpApi and design of a multi-layer user authentication process. Researching AWS Serverless architecture documentation to select appropriate services.
2. *Cost Calculation and Configuration*: Initializing automated budget limits on the AWS platform and preparing the local testing environment via Docker containers.
3. *Business Flow Development and Feasibility Assessment*: Building Lambda computation functions, setting up API Gateway routing, configuring the SQS queue processing pipeline, and optimizing DynamoDB's NoSQL data structure. Continuous refinement to ensure a balance between performance and cost.
4. *Testing and Deployment*: Performing load testing, validating system logic accuracy, handling exceptions, and deploying into the actual production environment.

*Technical Requirements*
- *Data Collection and Processing*: Raw data received from SerpApi must undergo cleaning and format standardization before being written to DynamoDB.
- *Database Design*: Requires designing a DynamoDB data model with optimal partition keys and sort keys to support high-speed retrievals and minimize read/write costs.
- *AI Integration*: The integrated language service must ensure low latency and accurate domain-specific context understanding to objectively analyze complex technical profiles.

### 5. Roadmap & Implementation Milestones
The execution roadmap is planned within the 12-week framework of the First Cloud AI Journey program at AWS Vietnam:
- *Week 1 - Week 6*: Acquiring foundational knowledge and completing scheduled labs to solidify expertise in the AWS Cloud service ecosystem.
- *Week 6 - Week 10*: Brainstorming ideas, finalizing the overall architectural design for the team project, and proceeding with coding and building the core system.
- *Week 11 - Week 12*: Executing comprehensive testing procedures, assessing system completeness, making final adjustments, and deploying for actual use.

### 6. Budget Estimation
The system operates entirely on a pay-as-you-go model.

*Expected Infrastructure Costs*
- *SerpApi*: Utilizing the free tier of 250 queries/month. With the current usage frequency of 8 queries/day, the maximum total usage reaches 248 queries/month, equating to a cost of $0.
- *AWS Lambda*: Optimizing costs by leveraging the free tier (1 million requests and 400,000 GB-seconds of compute time per month). Incurred costs remain close to zero for standard processing. Once the quota is exceeded, the applicable cost is $0.20 per 1 million requests.
- *AWS Amplify*: Covered by the Free Tier with 1,000 build minutes/month, 5 GB storage, and 15 GB bandwidth served per month. Given the current project scale, the expected cost is $0. If the quota is exceeded, the cost is $0.01/build minute, $0.023/GB stored, and $0.15/GB served.
- *Amazon SQS & DynamoDB*: Covered by the Free Tier with 1 million SQS requests/month and 25 GB storage, 25 WCU/RCU for DynamoDB. For the job data volume, the expected cost is $0. If exceeded, DynamoDB charges $1.25/million WCU and $0.25/million RCU.
- *Amazon S3*: Covered by the Free Tier with 5 GB storage, 20,000 GET requests, and 2,000 PUT requests. Estimating 1,000 CVs (~1 GB), the cost is $0. If exceeded, the cost is $0.023/GB.
- *Amazon API Gateway*: Covered by the Free Tier of 1 million API calls/month. Expected cost is $0. If exceeded, the cost is $1.00/million calls.
- *AWS WAF*: No permanent free tier. Charges are $5.00/month per Web ACL, $1.00/month per rule, and $0.60/million requests. Minimum expected cost is about $6.00 - $7.00/month (if enabled).
- *Gemini Flash Lite 3.1*:
  - *Free tier limits*: 15 requests/minute; 250,000 tokens/minute; 500 requests/day.
  - *Paid tier rates*: Input text at $0.25 / 1 million tokens; Output (including reasoning tokens) at $1.50 / 1 million tokens.

*Total Cost and Sustainability*
- **Estimated Total Cost per Month**: Approximately **$0.00** (if WAF is disabled) or **~$8.00** (if WAF is maintained).
- **Sustainability**: By using a Serverless architecture, the project fully capitalizes on AWS's generous Free Tier offerings. This enables long-term system sustainability with near-zero ongoing costs. Expenses only scale linearly when user traffic grows significantly, effectively mitigating financial risks for students.

### 7. Risk Assessment
*Risk Matrix*
- *Data Dependency*: High impact, medium probability (The system relies on a third-party job data collection API - SerpApi).
- *CV Evaluation Quality*: High impact, medium probability (Results depend entirely on the capability of the utilized AI model).

*Mitigation Strategies*
- *API & Traffic*: Applying an event-driven architecture with SQS to regulate processing traffic. Implementing an exponential backoff algorithm to effectively handle rate limit errors when retrieving external data.
- *Cost*: Setting up AWS Budgets to automatically trigger alert notifications to administrators when actual consumption reaches 80% of the allowed budget.
- *AI Assessment*: Developing transparency features, allowing candidates to self-verify and confirm the list of skills extracted from their CV before the system proceeds with the final match.

*Contingency Plan*
- Activate a manual internal data collection process in case the third-party API service or AWS platform experiences downtime.
- Use Infrastructure as Code tools (AWS CloudFormation) to quickly restore and recreate related network and service configurations to control incidents.

### 8. Expected Outcomes
*Technical Improvements*: Realizing full automation of the data ingestion and competency screening workflow. The AI-driven matching method provides higher data resolution, optimizing highly personalized search results compared to traditional rigid keyword filtering methods.
*Long-term Value*: Building a continuously enriched quantitative job data warehouse, setting the stage for in-depth analysis of labor market trends. The system allows for independent scalability to integrate predictive analytics and machine learning features in subsequent development cycles.