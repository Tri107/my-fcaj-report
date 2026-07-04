---
title : "API Keys Management"
date : 2024-01-01
weight : 2
chapter : false
pre : " <b> 5.6.2 </b> "
---

## Secure API Key Management with AWS Systems Manager Parameter Store

In the Jobs Matching Platform, several Lambda functions require access to external API keys:

- **SerpAPI Key**: Used by `FetchJobsFunction` to query Google Jobs search results
- **Gemini API Key**: Used by `EvaluateMatchFunction` to call Google Gemini AI for CV-Job matching evaluation

These keys must **never** be hardcoded in source code, environment variables in plain text, or committed to Git repositories. Instead, we use **AWS Systems Manager Parameter Store** to securely store, manage, and retrieve these secrets at runtime.





The flow works as follows:

1. **Admin** stores the API key in SSM Parameter Store (encrypted with AWS KMS)
2. **SAM template** passes the parameter name (not the value) as an environment variable to Lambda
3. **Lambda function** retrieves the actual key value from SSM at runtime using the AWS SDK
4. **IAM policy** restricts which Lambda can access which parameter

### Step 1: Store API Keys in Parameter Store

#### Store the SerpAPI Key

1. Navigate to **AWS Console → Systems Manager → Parameter Store**
2. Click **Create parameter**

![Create Parameter](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/create-parameter.png)

3. Configure the parameter:

| Field | Value |
|-------|-------|
| **Name** | `/jobs-matching/dev/serpapi-key` |
| **Description** | SerpAPI key for Google Jobs search |
| **Tier** | Standard |
| **Type** | SecureString |
| **KMS Key Source** | My current account |
| **KMS Key ID** | alias/aws/ssm (default) |
| **Value** | `<your-serpapi-api-key>` |

![SerpAPI Parameter Config](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/serpapi-param-config.png)
![SerpAPI Parameter Config 2](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/serpapi-param-config2.png)

4. Click **Create parameter**

{{% notice info %}}
Using **SecureString** type automatically encrypts the value using AWS KMS. The value is encrypted at rest and decrypted only when retrieved by an authorized IAM principal.
{{% /notice %}}

#### Store the Gemini API Key

Repeat the same process for the Gemini API key:

| Field | Value |
|-------|-------|
| **Name** | `/jobs-matching/dev/gemini-api-key` |
| **Description** | Google Gemini API key for CV-Job matching evaluation |
| **Tier** | Standard |
| **Type** | SecureString |
| **Value** | `<your-gemini-api-key>` |

![Gemini Parameter Config](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/gemini-param-config.png)

After creating both parameters, you should see them listed:

![Parameter List](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/parameter-list.png)

### Step 2: Reference Parameters in SAM Template

In the `template.yaml`, API keys are referenced **by parameter name**, not by value:

#### SerpAPI Key Reference

```yaml
Parameters:
  SerpApiKey:
    Type: String
    Description: API Key for SerpAPI - stored in SSM Parameter Store
    Default: /jobs-matching/dev/serpapi-key

Resources:
  FetchJobsFunction:
    Type: AWS::Serverless::Function
    Properties:
      Environment:
        Variables:
          SERPAPI_KEY: !Ref SerpApiKey  # This passes the PARAMETER NAME, not the value
```

#### Gemini API Key Reference

```yaml
Parameters:
  GeminiApiKeyParameterName:
    Type: String
    Default: /jobs-matching/dev/gemini-api-key

Resources:
  EvaluateMatchFunction:
    Type: AWS::Serverless::Function
    Properties:
      Policies:
        - Statement:
            - Effect: Allow
              Action:
                - ssm:GetParameter
              Resource: !Sub "arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:parameter${GeminiApiKeyParameterName}"
      Environment:
        Variables:
          GEMINI_API_KEY_PARAMETER_NAME: !Ref GeminiApiKeyParameterName
```

{{% notice warning %}}
Notice the critical distinction: `SERPAPI_KEY` stores the **parameter name path** (not the actual key). The Lambda function must call SSM to retrieve the actual value at runtime.
{{% /notice %}}

### Step 3: Retrieve Keys in Lambda Code at Runtime

In the Lambda function code, use the AWS SDK to fetch the parameter value:

```typescript
// src/functions/evaluateMatch/handler.ts
import { SSMClient, GetParameterCommand } from "@aws-sdk/client-ssm";

const ssmClient = new SSMClient({});

async function getGeminiApiKey(): Promise<string> {
  const parameterName = process.env.GEMINI_API_KEY_PARAMETER_NAME;
  
  const command = new GetParameterCommand({
    Name: parameterName,
    WithDecryption: true,  // Required for SecureString parameters
  });
  
  const response = await ssmClient.send(command);
  return response.Parameter?.Value || "";
}
```

**Key points:**

- `WithDecryption: true` tells SSM to decrypt the `SecureString` value before returning it
- The decryption is handled automatically by KMS — no additional code needed
- The IAM policy attached to the Lambda function must include `ssm:GetParameter` permission

### Step 4: Verify Using AWS CLI

You can verify the parameters are correctly stored using the CLI:

```bash
# List all parameters under the jobs-matching path
aws ssm get-parameters-by-path \
  --path "/jobs-matching/dev" \
  --recursive \
  --query "Parameters[].{Name:Name,Type:Type,Version:Version}" \
  --output table
```

![CLI Parameter List](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/cli-param-list.png)

```bash
# Retrieve a specific parameter (decrypted)
aws ssm get-parameter \
  --name "/jobs-matching/dev/serpapi-key" \
  --with-decryption \
  --query "Parameter.Value" \
  --output text
```

{{% notice warning %}}
Be careful when using `--with-decryption` — it will display the actual secret value in your terminal. Only use this for verification purposes and never pipe the output to log files.
{{% /notice %}}

### Security Comparison

| Approach | Security Level | Risk |
|----------|---------------|------|
| ❌ Hardcoded in source code | Very Low | Key exposed in Git history forever |
| ❌ Plain text environment variable | Low | Visible in Lambda console, CloudFormation outputs |
| ⚠️ SSM Parameter (String type) | Medium | Value stored in plaintext in SSM |
| ✅ **SSM Parameter (SecureString)** | **High** | **Value encrypted with KMS, decrypted only at runtime** |
| ✅ AWS Secrets Manager | Highest | Automatic rotation, higher cost ($0.40/secret/month) |

For this project, **SSM Parameter Store with SecureString** provides the right balance of security and cost — it's free for standard parameters and provides KMS encryption.

### Best Practices Summary

| Practice | Implementation |
|----------|---------------|
| **Never commit secrets** | API keys are stored in SSM, only parameter names are in code |
| **Encrypt at rest** | SecureString type uses AWS KMS for encryption |
| **Scope access** | IAM policies restrict which Lambda can read which parameter |
| **Use hierarchical naming** | Parameters follow `/project/stage/key-name` convention |
| **Separate by environment** | Different parameter paths for dev/staging/prod |
