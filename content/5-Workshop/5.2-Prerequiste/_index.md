---
title : "Prerequiste"
date : 2026-07-02 
weight : 2 
chapter : false
pre : " <b> 5.2. </b> "
---

Before initializing and building the project, we need to prepare several essential resources and environment setups. These are categorized into the following three main areas:

#### 1. AWS Account & IAM Permissions
You will need an active AWS account with appropriate IAM permissions to provision the cloud infrastructure. Ensure your user or role has permissions for:
* **IAM Roles & Policies**: To create/modify instance profiles and execution roles.
* **Network & Compute**: EC2, VPC, Route 53, and Lambda resources.
* **Storage & Security**: S3 buckets and Secrets Manager.

For detailed steps on setting up your account, refer to [AWS IAM Account](5.2.1-AWS-IAM-Account).

#### 2. Local Environment & Tools Setup
To interact with AWS and build/deploy the project, you must install and configure the following core tools on your local machine:
* **AWS CLI**: Configured with your AWS credentials.
* **Git**: For version control and repository management.
* **IDE (e.g., VS Code)**: Equipped with the necessary extensions for development.

For installation instructions and configuration guides, refer to [Tools Setup](5.2.2-Tools-Setup).

#### 3. Third-party APIs & External Resources
Some features of the project rely on integration with external third-party API services. Before initializing the project, you should:
* Sign up for the required API provider accounts.
* Retrieve the necessary access tokens or API keys to be configured as secrets.

For information on external integrations and setting up API credentials, refer to [Third-party APIs](5.2.3-Third-party-APIs).

#### Content

- [AWS IAM Account](5.2.1-AWS-IAM-Account/)
- [Tools Setup](5.2.2-Tools-Setup/)
- [Third-party APIs](5.2.3-Third-party-APIs/)
