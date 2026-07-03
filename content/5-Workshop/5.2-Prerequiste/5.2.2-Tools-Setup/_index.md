---
title : "Tools Setup"
date : 2026-07-02 
weight : 2
chapter : false
pre : " <b> 5.2.2. </b> "
---

In this section, we will install and configure the necessary tools on your local machine to build, test, and deploy the project.

---

### 1. Install AWS CLI

#### 1.1. Download and Install (Windows):
Open your command prompt or terminal and execute the following command to download and install the AWS CLI:

```powershell
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```

#### 1.2. Verify Installation:
Verify that the AWS CLI was successfully installed by running:

```bash
aws --version
```
![aws version](/images/5-Workshop/5.2-Prerequisite/5.2.2-Tools-Setup/aws-version.png)
---

### 2. Install AWS SAM CLI

#### 2.1. Download:
Download the installer from the official page: [AWS SAM CLI Installation Guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).

#### 2.2. Verify Installation:
Verify that the SAM CLI was successfully installed by running:

```bash
sam --version
```
![sam version](/images/5-Workshop/5.2-Prerequisite/5.2.2-Tools-Setup/sam-version.png)
---

### 3. Configure AWS Profile

To allow local tools to authenticate and interact with your AWS account, configure your profile with the credentials retrieved from Section 5.2.1:

#### 3.1. Configure Credentials:
Run the configuration command:

```bash
aws configure
```
Enter the credentials and default settings as prompted:

```text
AWS Access Key ID [None]: <Your-IAM-User-Access-Key-ID>
AWS Secret Access Key [None]: <Your-IAM-User-Secret-Access-Key>
Default region name [None]: ap-southeast-1
Default output format [None]: json
```

---

### 4. Initialize Project

To bootstrap the serverless application, run the initialization command:

```bash
sam init
```

Follow the on-screen prompts to select your template and preferences.

![sam init](/images/5-Workshop/5.2-Prerequisite/5.2.2-Tools-Setup/sam-init.png)
---

### 5. Install IDE (Visual Studio Code)

We will use Visual Studio Code (VS Code) as our primary development environment.
* Download and install the version matching your OS: [Download Visual Studio Code](https://code.visualstudio.com/download).

---

### 6. Install Environments (Node.js & Python)

This project requires both Node.js and Python environments for specific development features and functions.

* **Node.js**: Download and install from [Node.js Downloads](https://nodejs.org/en/download/).
* **Python**: Download and install from [Python Downloads](https://www.python.org/downloads/).

---

### 7. Install VS Code Extensions

For an optimized development experience, open VS Code and install the following extensions:

* **AWS Toolkit for Visual Studio Code**: Offers AWS CLI integration, local debugging, and resource browsing.
* **Python**: Provides rich language support for Python scripts and Lambda functions.
