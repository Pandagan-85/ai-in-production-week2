# 📝 Day 3: Transition to AWS Bedrock

Swapping OpenAI for AWS Bedrock's Nova models. This cuts costs significantly while keeping quality high, and keeps everything within the AWS ecosystem.

## Key Changes

- Replace OpenAI with Bedrock
- Switch from `openai` package to `boto3`
- Update Lambda environment variables
- Implement Bedrock API integration
- Set up cost monitoring

## Why Bedrock?

| Aspect             | OpenAI         | Bedrock Nova           |
| ------------------ | -------------- | ---------------------- |
| **Cost**           | ~$1-2/day      | ~$0.50-1/day           |
| **Latency**        | 2-4s           | 1-3s                   |
| **Models**         | GPT-4o-mini    | Nova Micro/Lite/Pro    |
| **Management**     | External API   | AWS-native             |
| **Monitoring**     | Basic          | CloudWatch integration |
| **Data Residency** | OpenAI servers | AWS region (yours)     |

## Changes in the code

### 1. Update `backend/requirements.txt`

```diff
- openai
+ boto3
  fastapi
  uvicorn
  mangum
```

### 2. Create `backend/bedrock_client.py`

```python
import boto3
import json

bedrock_client = boto3.client('bedrock-runtime')

def get_bedrock_response(messages: list, system_prompt: str) -> str:
    """Call AWS Bedrock Nova model"""

    body = {
        "messages": messages,
        "system": system_prompt,
        "max_tokens": 1024,
    }

    response = bedrock_client.invoke_model(
        modelId="amazon.nova-pro-v1:0",  # or nova-lite, nova-micro
        body=json.dumps(body),
        contentType="application/json",
    )

    result = json.loads(response['body'].read().decode())
    return result['content'][0]['text']
```

### 3. Update `backend/server.py`

```python
# Replace OpenAI import
from bedrock_client import get_bedrock_response

# In your chat endpoint, replace:
# response = client.chat.completions.create(...)

# With:
# response_text = get_bedrock_response(messages, system_prompt)
```

### 4. Update Lambda Environment

Set these environment variables in AWS Lambda:

```
BEDROCK_REGION=us-east-1  # Change to your region
BEDROCK_MODEL_ID=amazon.nova-pro-v1:0
```

## Step-by-Step Migration

### 1. Request Bedrock Access

Go to AWS Console → Bedrock → Model Access:

- Request access to Nova models
- Wait for approval (usually instant)

### 2. Update Backend Code

Replace OpenAI calls with Bedrock:

```python
# Old (OpenAI)
from openai import OpenAI
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=messages,
)

# New (Bedrock)
import boto3
bedrock = boto3.client('bedrock-runtime')
response = bedrock.invoke_model(
    modelId="amazon.nova-pro-v1:0",
    body=json.dumps({"messages": messages}),
)
```

### 3. Update `requirements.txt`

```bash
cd backend
# Remove openai
uv pip uninstall openai
# Add boto3
uv add boto3
```

### 4. Test Locally

```bash
# Set AWS credentials
export AWS_ACCESS_KEY_ID=your_key
export AWS_SECRET_ACCESS_KEY=your_secret
export AWS_DEFAULT_REGION=us-east-1

# Run locally
uv run uvicorn server:app --reload
```

### 5. Rebuild Lambda Package

```bash
cd backend
uv run deploy.py
```

### 6. Update Lambda Function

```bash
# Update function code
aws lambda update-function-code \
  --function-name twin-backend \
  --zip-file fileb://lambda.zip

# Update environment variables
aws lambda update-function-configuration \
  --function-name twin-backend \
  --environment Variables="{
    BEDROCK_REGION=us-east-1,
    BEDROCK_MODEL_ID=amazon.nova-pro-v1:0
  }"
```

## Nova Model Selection

| Model     | Speed     | Quality | Cost     | Use Case                          |
| --------- | --------- | ------- | -------- | --------------------------------- |
| **Micro** | Very Fast | Good    | Cheapest | Development, simple conversations |
| **Lite**  | Fast      | Better  | Low      | Production, general use           |
| **Pro**   | Medium    | Best    | Medium   | Complex reasoning, high quality   |

For this project: Start with **Nova Lite** (best balance)

## Monitoring in CloudWatch

View Bedrock API calls:

```bash
# Check CloudWatch logs
aws logs tail /aws/lambda/twin-backend --follow
```

Set up cost alerts:

```bash
# Create SNS topic
aws sns create-topic --name bedrock-costs

# Set up budget alert in AWS Console
```

## Notes

- Bedrock must be available in your region (us-east-1, us-west-2, eu-west-1)
- Lambda IAM role needs `bedrock:InvokeModel` permission
- Model access requests in Bedrock console are usually instant
- CloudWatch logs help debug if calls fail

## Next: Day 4

Day 4 moves away from manual AWS clicks and automates everything with Terraform - infrastructure as code, dev/test/prod environments, deployment scripts. This is the DevOps phase.
