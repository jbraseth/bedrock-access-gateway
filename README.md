# Bedrock Access Gateway - Custom Fixes

Fork of [aws-samples/bedrock-access-gateway](https://github.com/aws-samples/bedrock-access-gateway) with fixes for OpenAI API compatibility.

## Problem

OpenAI-compatible clients (Codex, Continue, Cline, etc.) break when used with Bedrock Access Gateway due to bugs in how BAG translates between Bedrock's Converse API and the OpenAI streaming format. These bugs are [reported upstream](#fixes) but remain unpatched.

**Empty content blocks** — Longer conversations and agent mode produce empty content arrays that Bedrock rejects:
```
HTTP 400: Bedrock validation error
```

**Tool calls streaming index** — BAG returns `"index": -1` in streaming tool call responses instead of a valid 0-based integer, which violates the OpenAI spec and crashes clients:
```
[error] Item not found in turn state
```

## Fixes

All fixes are based on upstream commit [`a150f7b`](https://github.com/aws-samples/bedrock-access-gateway/commit/a150f7b) (2026-02-13). We reference a commit hash because upstream does not publish releases or tags.

| Fix | Tag | Upstream Issue | Description |
|-----|-----|----------------|-------------|
| Empty content blocks | `fix-empty-content` | [#167](https://github.com/aws-samples/bedrock-access-gateway/issues/167), [#214](https://github.com/aws-samples/bedrock-access-gateway/pull/214) | Bedrock rejects empty content arrays that some OpenAI clients send. This skips them. Also falls back to the tool name when a client sends an empty tool description. |
| Tool calls streaming index | `fix-tool-index` | [#134](https://github.com/aws-samples/bedrock-access-gateway/issues/134), [#203](https://github.com/aws-samples/bedrock-access-gateway/issues/203) | BAG returns `"index": -1` in streaming `tool_calls` instead of `0`. The OpenAI spec requires a non-negative integer. This is what causes the "Item not found in turn state" error in the Codex extension. |
| **All fixes combined** | **`all-fixes`** | | Both of the above. |

## Deploy

```bash
git clone https://github.com/jbraseth/bedrock-access-gateway.git
cd bedrock-access-gateway
git checkout all-fixes    # or a specific fix tag
```

Then follow the deployment steps below.

---

# Upstream Documentation

Everything below is from the original [aws-samples/bedrock-access-gateway](https://github.com/aws-samples/bedrock-access-gateway) README.

## Overview

Amazon Bedrock offers a wide range of foundation models (such as Claude 3 Opus/Sonnet/Haiku, Llama 2/3, Mistral/Mixtral,
etc.) and a broad set of capabilities for you to build generative AI applications. Check the [Amazon Bedrock](https://aws.amazon.com/bedrock) landing page for additional information.

Sometimes, you might have applications developed using OpenAI APIs or SDKs, and you want to experiment with Amazon Bedrock without modifying your codebase. Or you may simply wish to evaluate the capabilities of these foundation models in tools like AutoGen etc. Well, this repository allows you to access Amazon Bedrock models seamlessly through OpenAI APIs and SDKs, enabling you to test these models without code changes.

**Features:**

- [x] Support streaming response via server-sent events (SSE)
- [x] Support Model APIs
- [x] Support Chat Completion APIs
- [x] Support Tool Call
- [x] Support Embedding API
- [x] Support Multimodal API
- [x] Support Cross-Region Inference
- [x] Support Application Inference Profiles
- [x] Support Reasoning
- [x] Support Interleaved thinking
- [x] Support Prompt Caching

Please check [Usage Guide](./docs/Usage.md) for more details about how to use the new APIs.

## Get Started

### Prerequisites

- Access to Amazon Bedrock foundation models.

> For more information on how to request model access, please refer to the [Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html) (Set Up > Model access)

### Deployment Options

| Option | Pros | Cons | Best For |
|--------|------|------|----------|
| **API Gateway + Lambda** | No VPC required, pay-per-request, native streaming support, lower operational overhead | Potential cold starts | Most use cases, cost-sensitive deployments |
| **ALB + Fargate** | Lowest streaming latency, no cold starts | Higher cost, requires VPC | High-throughput, latency-sensitive workloads |

### Deployment

**Step 1: Create your own API key in Secrets Manager**

1. Open the AWS Management Console and navigate to the AWS Secrets Manager service.
2. Click on "Store a new secret" button.
3. In the "Choose secret type" page, select:
   - Secret type: Other type of secret
   - Key/value pairs: Key: `api_key`, Value: Enter your API key value
4. Secret name: Enter a name (e.g., "BedrockProxyAPIKey")
5. Click "Next", review, and click "Store"

After creation, make note of the secret ARN.

**Step 2: Build and push container images to ECR**

```bash
cd scripts
bash ./push-to-ecr.sh
```

Follow the prompts to configure ECR repository names, image tag, and AWS region. Copy the image URIs displayed at the end.

**Step 3: Deploy the CloudFormation stack**

1. Download the CloudFormation template:
   - For API Gateway + Lambda: [`deployment/BedrockProxy.template`](deployment/BedrockProxy.template)
   - For ALB + Fargate: [`deployment/BedrockProxyFargate.template`](deployment/BedrockProxyFargate.template)
2. Navigate to CloudFormation in your target region.
3. Click "Create stack" > "With new resources (standard)".
4. Upload the template and provide:
   - **ApiKeySecretArn**: The secret ARN from Step 1
   - **ContainerImageUri**: The ECR image URI from Step 2
5. Acknowledge IAM resource creation and submit.

Once deployed, find the API Base URL in the CloudFormation stack **Outputs** tab (`APIBaseUrl`).

### SDK/API Usage

```bash
export OPENAI_API_KEY=<API key>
export OPENAI_BASE_URL=<API base url>
```

```bash
curl $OPENAI_BASE_URL/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "anthropic.claude-3-sonnet-20240229-v1:0",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

```python
from openai import OpenAI

client = OpenAI()
completion = client.chat.completions.create(
    model="anthropic.claude-3-sonnet-20240229-v1:0",
    messages=[{"role": "user", "content": "Hello!"}],
)
print(completion.choices[0].message.content)
```

### Running Locally

```bash
cd src
uvicorn api.app:app --host 0.0.0.0 --port 8000
```

The API base url will be `http://localhost:8000/api/v1`.

### Troubleshooting

See the [Troubleshooting Guide](./docs/Troubleshooting.md).

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
