---
title : "Frontend Application Deployment (Frontend Hosting Amplify)"
date : 2026-07-02
weight : 5
chapter : false
pre : " <b> 5.4.5. </b> "
---

Deploy and distribute the Next.js user interface application using **AWS Amplify Hosting** with automatic CI/CD configured to trigger when source code is pushed to GitHub.

---

## 1. Detailed Data Flow
* **Application Initialization:** Client loads the webpage -> Reads environment configuration files loaded at build time -> Initializes Amplify Auth SDK settings.
* **Static Route Protection (Routing Protection):**
  * When a user accesses protected paths such as `/favorites`, `/upload-cv`, or `/matching`.
  * The frontend calls the `fetchAuthSession()` function to check the current session.
  * If the session is empty: Redirects the user to the `/login` page.
  * If the session exists: Renders the user interface and retrieves the corresponding token to prepare for API calls.

---

## 2. Environment Variables Configuration
When hosting the application on the AWS Amplify Console, the following environment variables must be defined so that the client points to the correct backend endpoint:

| Environment Variable Name | Sample Value | Role |
| :--- | :--- | :--- |
| `NEXT_PUBLIC_API_BASE_URL` | `https://38ethw76uh.execute-api.ap-southeast-1.amazonaws.com/dev` | Root URL of the API Gateway to fetch data. |
| `NEXT_PUBLIC_AWS_REGION` | `ap-southeast-1` | AWS Region hosting the Cognito User Pool. |
| `NEXT_PUBLIC_COGNITO_USER_POOL_ID` | `ap-southeast-1_xxxxx` | Cognito User Pool ID. |
| `NEXT_PUBLIC_COGNITO_USER_POOL_CLIENT_ID` | `xxxxxxxxxxxxxx` | Cognito App Client ID. |

---

## 3. Deployment & Configuration Guide on Amplify Console

1. **Link Git Repository:**
   * Go to the AWS Console -> AWS Amplify -> Select **Create new app**.
   * Choose your source code provider (e.g. **GitHub**) and connect the repository containing the `Jobs-Matching-Platform` project.
   * Select the deployment branch (e.g. `main` or `dev`).

2. **Configure Build Settings:**
   Amplify will automatically detect the Next.js configuration. Ensure the Amplify build spec (`amplify.yml`) matches your project's Next.js setup:
   ```yaml
   version: 1
   frontend:
     phases:
       preBuild:
         commands:
           - cd frontend
           - npm ci
       build:
         commands:
           - npm run build
     artifacts:
       baseDirectory: frontend/.next
       files:
         - '**/*'
     cache:
       paths:
         - frontend/node_modules/**/*
         - frontend/.next/cache/**/*
   ```

3. **Configure Environment Variables in Amplify Console:**
   * Navigate to **Hosting** -> **Environment variables**.
   * Add all 4 environment variables listed above (`NEXT_PUBLIC_API_BASE_URL`, etc.) with their respective backend outputs obtained after SAM deployment.

![Amplify Environment Variables](/images/5-Workshop/5.4-User-Interaction/amplify-environment-variables.png)

4. **Enable Continuous Deployment:**
   * Click **Save and deploy**.
   * Every time a new commit is pushed to the chosen Git branch, AWS Amplify will automatically trigger the build and deployment pipeline for the frontend without downtime.

![Amplify Deployment Status](/images/5-Workshop/5.4-User-Interaction/amplify-deploy-success.png)
