---
title: "Blog 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---
# Building an AI Gateway to Amazon Bedrock with Amazon API Gateway

When building generative AI applications, enterprises face a major challenge: governing foundation model usage through authorization, quota management, tenant isolation, and cost control. To meet these needs, a robust AI gateway architecture has emerged as a reference pattern for organizations looking to control access to **Amazon Bedrock** services at scale.

This pattern utilizes **Amazon API Gateway** as the access layer in front of Amazon Bedrock, providing capabilities such as request authorization, usage quotas, lifecycle management, and real-time response streaming. 

---

## Reference Architecture

The architecture provides granular control over Large Language Model (LLM) access using fully managed AWS services. It is transparent to client applications (like AWS SDKs/Boto3) and seamlessly integrates into existing enterprise environments.
![Design Architecture](/images/3-BlogTranslated/3.1-blog1/Architecture-Blog1.png)
The solution consists of five core components:
1. **Amazon Route 53 (Optional):** Manages custom domain routing, allowing clients to access the gateway through a company-specific endpoint.
2. **Amazon API Gateway:** Serves as the main entry point, providing capabilities like request throttling, API keys, and lifecycle management.
3. **AWS Lambda Authorizer:** Handles request authorization by validating JWT tokens against existing authentication systems (e.g., Amazon Cognito).
4. **Lambda Integration:** Acts as a dynamic request forwarder that signs incoming requests with AWS credentials (SigV4) and routes them to the appropriate Amazon Bedrock endpoints while preserving original request details.
5. **Amazon Bedrock:** Provides the actual access to foundation models and AI capabilities.

> **Why this design?** The benefit of this architecture is its future-proof design. Clients can interact with Amazon Bedrock exactly as they would when calling the API directly. The AI gateway handles the complex governance behind the scenes, without requiring code updates every time a new Bedrock feature is released.

---

## Deployment & Testing

The quickest way to deploy the core infrastructure (API Gateway, Lambda functions, and VPC endpoints) is via **AWS CloudFormation**. 

Once deployed privately inside a VPC, you can test it using an **AWS CloudShell** environment:
1. Create a reusable client factory in Python that routes requests through your API Gateway while maintaining the standard `boto3` interface.
2. Test model inference (e.g., using `ConverseStream` API) or knowledge base retrievals.
3. Once basic functionality is verified, enable the **Lambda Authorizer** by updating the CloudFormation stack to implement custom JWT validation logic, ensuring that all API calls are securely authenticated.

---

## Enhancement Options for the AI Gateway

To implement AI governance at scale, this reference architecture can be enhanced using additional API Gateway capabilities:

- **Rate Limiting & Throttling:** Control request rates using usage plans and API keys to prevent noisy neighbor problems in multi-tenant SaaS applications.
- **Private or Edge-Optimized Endpoints:** Optimize for internal network access or global performance depending on client locations.
- **Canary Releases:** Manage multiple API versions and implement gradual rollouts with stage variables and canary deployments.
- **WAF Integration:** Protect the AI endpoints from common web exploits by adding AWS WAF rules.
- **Prompt and Response Caching:** Reduce costs and improve response times for frequently requested prompts using API Gateway caching.
- **Content Filtering:** Add custom filtering in the Lambda integration layer to screen for sensitive data (like PII), complementing the safeguards already offered by **Amazon Bedrock Guardrails**.

---

*Original article:* [Building an AI gateway to Amazon Bedrock with Amazon API Gateway](https://aws.amazon.com/blogs/architecture/building-an-ai-gateway-to-amazon-bedrock-with-amazon-api-gateway/)
