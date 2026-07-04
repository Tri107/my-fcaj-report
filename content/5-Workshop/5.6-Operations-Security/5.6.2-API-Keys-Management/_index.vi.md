---
title : "Quản lý API Keys"
date : 2024-01-01
weight : 2
chapter : false
pre : " <b> 5.6.2 </b> "
---

## Quản lý API Keys an toàn với AWS Systems Manager Parameter Store

Trong Jobs Matching Platform, một số Lambda function cần truy cập đến API keys bên ngoài:

- **SerpAPI Key**: Được sử dụng bởi `FetchJobsFunction` để truy vấn kết quả tìm kiếm Google Jobs
- **Gemini API Key**: Được sử dụng bởi `EvaluateMatchFunction` để gọi Google Gemini AI đánh giá mức độ phù hợp CV-Job

Các khóa này **không bao giờ** được hardcode trong mã nguồn, biến môi trường dạng plaintext, hoặc commit lên Git repositories. Thay vào đó, chúng ta sử dụng **AWS Systems Manager Parameter Store** để lưu trữ, quản lý và truy xuất an toàn các bí mật này lúc runtime.





Luồng hoạt động như sau:

1. **Admin** lưu trữ API key vào SSM Parameter Store (mã hóa bằng AWS KMS)
2. **SAM template** truyền tên parameter (không phải giá trị) dưới dạng biến môi trường cho Lambda
3. **Lambda function** truy xuất giá trị thực của key từ SSM lúc runtime bằng AWS SDK
4. **IAM policy** giới hạn Lambda nào được truy cập parameter nào

### Bước 1: Lưu trữ API Keys vào Parameter Store

#### Lưu trữ SerpAPI Key

1. Truy cập **AWS Console → Systems Manager → Parameter Store**
2. Click **Create parameter**

![Tạo Parameter](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/create-parameter.png)

3. Cấu hình parameter:

| Trường | Giá trị |
|--------|---------|
| **Name** | `/jobs-matching/dev/serpapi-key` |
| **Description** | SerpAPI key cho Google Jobs search |
| **Tier** | Standard |
| **Type** | SecureString |
| **KMS Key Source** | My current account |
| **KMS Key ID** | alias/aws/ssm (mặc định) |
| **Value** | `<your-serpapi-api-key>` |

![Cấu hình SerpAPI Parameter](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/serpapi-param-config.png)
![Cấu hình SerpAPI Parameter 2](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/serpapi-param-config2.png)
4. Click **Create parameter**

{{% notice info %}}
Sử dụng kiểu **SecureString** tự động mã hóa giá trị bằng AWS KMS. Giá trị được mã hóa khi lưu trữ (at rest) và chỉ được giải mã khi truy xuất bởi IAM principal được ủy quyền.
{{% /notice %}}

#### Lưu trữ Gemini API Key

Lặp lại quy trình tương tự cho Gemini API key:

| Trường | Giá trị |
|--------|---------|
| **Name** | `/jobs-matching/dev/gemini-api-key` |
| **Description** | Google Gemini API key cho đánh giá CV-Job matching |
| **Tier** | Standard |
| **Type** | SecureString |
| **Value** | `<your-gemini-api-key>` |

![Cấu hình Gemini Parameter](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/gemini-param-config.png)

Sau khi tạo cả hai parameter, bạn sẽ thấy chúng trong danh sách:

![Danh sách Parameter](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/parameter-list.png)

### Bước 2: Tham chiếu Parameters trong SAM Template

Trong `template.yaml`, API keys được tham chiếu **bằng tên parameter**, không phải bằng giá trị:

#### Tham chiếu SerpAPI Key

```yaml
Parameters:
  SerpApiKey:
    Type: String
    Description: API Key cho SerpAPI - lưu trong SSM Parameter Store
    Default: /jobs-matching/dev/serpapi-key

Resources:
  FetchJobsFunction:
    Type: AWS::Serverless::Function
    Properties:
      Environment:
        Variables:
          SERPAPI_KEY: !Ref SerpApiKey  # Truyền TÊN PARAMETER, không phải giá trị
```

#### Tham chiếu Gemini API Key

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
Lưu ý điểm quan trọng: `SERPAPI_KEY` lưu **đường dẫn tên parameter** (không phải khóa thực tế). Lambda function phải gọi SSM để lấy giá trị thực lúc runtime.
{{% /notice %}}

### Bước 3: Truy xuất Keys trong Lambda Code lúc Runtime

Trong mã Lambda function, sử dụng AWS SDK để lấy giá trị parameter:

```typescript
// src/functions/evaluateMatch/handler.ts
import { SSMClient, GetParameterCommand } from "@aws-sdk/client-ssm";

const ssmClient = new SSMClient({});

async function getGeminiApiKey(): Promise<string> {
  const parameterName = process.env.GEMINI_API_KEY_PARAMETER_NAME;
  
  const command = new GetParameterCommand({
    Name: parameterName,
    WithDecryption: true,  // Bắt buộc cho SecureString parameters
  });
  
  const response = await ssmClient.send(command);
  return response.Parameter?.Value || "";
}
```

**Những điểm quan trọng:**

- `WithDecryption: true` yêu cầu SSM giải mã giá trị `SecureString` trước khi trả về
- Quá trình giải mã được KMS xử lý tự động — không cần thêm code
- IAM policy gắn với Lambda function phải bao gồm quyền `ssm:GetParameter`

### Bước 4: Xác minh bằng AWS CLI

Bạn có thể xác minh các parameter được lưu trữ đúng bằng CLI:

```bash
# Liệt kê tất cả parameter dưới đường dẫn jobs-matching
aws ssm get-parameters-by-path \
  --path "/jobs-matching/dev" \
  --recursive \
  --query "Parameters[].{Name:Name,Type:Type,Version:Version}" \
  --output table
```

![Danh sách Parameter CLI](/images/5-Workshop/5.6-Operations-Security/5.6.2-API-Keys-Management/cli-param-list.png)

```bash
# Truy xuất parameter cụ thể (đã giải mã)
aws ssm get-parameter \
  --name "/jobs-matching/dev/serpapi-key" \
  --with-decryption \
  --query "Parameter.Value" \
  --output text
```

{{% notice warning %}}
Cẩn thận khi sử dụng `--with-decryption` — nó sẽ hiển thị giá trị bí mật thực tế trên terminal của bạn. Chỉ sử dụng cho mục đích xác minh và không bao giờ chuyển output vào file log.
{{% /notice %}}

### So sánh Mức độ Bảo mật

| Phương pháp | Mức Bảo mật | Rủi ro |
|-------------|-------------|--------|
| ❌ Hardcode trong mã nguồn | Rất Thấp | Key bị lộ trong lịch sử Git mãi mãi |
| ❌ Biến môi trường plaintext | Thấp | Hiển thị trên Lambda console, CloudFormation outputs |
| ⚠️ SSM Parameter (kiểu String) | Trung bình | Giá trị lưu plaintext trong SSM |
| ✅ **SSM Parameter (SecureString)** | **Cao** | **Giá trị mã hóa bằng KMS, chỉ giải mã lúc runtime** |
| ✅ AWS Secrets Manager | Cao nhất | Tự động xoay khóa, chi phí cao hơn ($0.40/secret/tháng) |

Với dự án này, **SSM Parameter Store với SecureString** cung cấp sự cân bằng phù hợp giữa bảo mật và chi phí — miễn phí cho standard parameters và cung cấp mã hóa KMS.

### Tóm tắt Nguyên tắc Tốt nhất

| Nguyên tắc | Cách triển khai |
|-----------|----------------|
| **Không commit secrets** | API keys lưu trong SSM, chỉ tên parameter có trong code |
| **Mã hóa khi lưu trữ** | Kiểu SecureString sử dụng AWS KMS để mã hóa |
| **Giới hạn quyền truy cập** | IAM policies giới hạn Lambda nào được đọc parameter nào |
| **Đặt tên phân cấp** | Parameters tuân theo quy ước `/project/stage/key-name` |
| **Tách biệt theo môi trường** | Đường dẫn parameter khác nhau cho dev/staging/prod |
