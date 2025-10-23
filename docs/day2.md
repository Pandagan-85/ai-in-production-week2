# 📝 Day 2: Deploy Your Digital Twin to AWS

This is where the local app becomes a real production system on AWS - Lambda for the backend, S3 for storage, CloudFront for global distribution.

## Key Steps

1. Enhance twin with personal data context
2. Create deployment packages for Lambda
3. Set up S3 for memory persistence and frontend hosting
4. Deploy Lambda function with environment configuration
5. Create API Gateway for routing
6. Set up CloudFront distribution
7. Deploy frontend to S3
8. Test production setup

## Files You'll Create

- `backend/data/facts.json` - Personal information
- `backend/data/summary.txt` - Professional summary
- `backend/data/style.txt` - Communication style
- `backend/resources.py` - Resource loading module
- `backend/context.py` - Dynamic prompt generation
- `backend/lambda_handler.py` - Lambda entry point
- `backend/deploy.py` - Lambda packaging script

## Architecture (Day 2)

```
AWS Deployment
├── CloudFront (CDN)
├── S3 (Frontend static hosting)
├── API Gateway (REST API)
├── Lambda (Backend serverless)
└── S3 (Memory persistence)
```

## Deployment Guide

### 1. Set Up S3 Buckets

Create two buckets:

- One for frontend hosting
- One for memory storage

```bash
# Frontend bucket (replace with unique name)
aws s3 mb s3://twin-frontend-YOUR-ACCOUNT-ID

# Memory bucket
aws s3 mb s3://twin-memory-YOUR-ACCOUNT-ID

# Enable versioning on memory bucket
aws s3api put-bucket-versioning \
  --bucket twin-memory-YOUR-ACCOUNT-ID \
  --versioning-configuration Status=Enabled
```

### 2. Prepare Backend for Lambda

Create `backend/lambda_handler.py`:

```python
import os
from server import app
from mangum import Mangum

handler = Mangum(app)
```

Update `backend/requirements.txt` to include Mangum:

```
fastapi
uvicorn
openai
boto3
mangum
```

### 3. Package and Deploy Lambda

Create `backend/deploy.py`:

```python
import subprocess
import os
import json
from pathlib import Path

def package_lambda():
    """Package the backend for AWS Lambda"""

    # Install dependencies
    subprocess.run(
        ["pip", "install", "-r", "requirements.txt", "-t", "package/"],
        check=True
    )

    # Copy application code
    subprocess.run(["cp", "*.py", "package/"], shell=True, check=True)

    # Create zip file
    subprocess.run(["zip", "-r", "lambda.zip", "."],
                   cwd="package/", check=True)

    print("✅ Lambda package created: lambda.zip")

if __name__ == "__main__":
    package_lambda()
```

Run deployment:

```bash
cd backend
uv run deploy.py
```

### 4. Deploy to AWS Lambda

```bash
# Create IAM role for Lambda (if not existing)
aws iam create-role \
  --role-name twin-lambda-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {"Service": "lambda.amazonaws.com"},
        "Action": "sts:AssumeRole"
      }
    ]
  }'

# Attach policy for S3 access
aws iam attach-role-policy \
  --role-name twin-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Create Lambda function
aws lambda create-function \
  --function-name twin-backend \
  --runtime python3.12 \
  --role arn:aws:iam::ACCOUNT-ID:role/twin-lambda-role \
  --handler lambda_handler.handler \
  --zip-file fileb://backend/lambda.zip \
  --timeout 30 \
  --memory-size 512 \
  --environment Variables="{
    OPENAI_API_KEY=your_key,
    MEMORY_BUCKET=twin-memory-YOUR-ACCOUNT-ID
  }"
```

### 5. Set Up API Gateway

```bash
# Create REST API
aws apigateway create-rest-api --name twin-api

# Configure routes and methods
# See AWS documentation for detailed steps
```

### 6. Deploy Frontend to S3

```bash
# Build Next.js app
cd frontend
npm run build

# Sync to S3
aws s3 sync out s3://twin-frontend-YOUR-ACCOUNT-ID --delete
```

### 7. Set Up CloudFront

```bash
# Create CloudFront distribution
aws cloudfront create-distribution \
  --origin-domain-name twin-frontend-YOUR-ACCOUNT-ID.s3.amazonaws.com \
  --default-root-object index.html
```

## Testing Production

```bash
# Test API endpoint
curl https://YOUR-API-ID.execute-api.REGION.amazonaws.com/health

# Test chat endpoint
curl -X POST https://YOUR-API-ID.execute-api.REGION.amazonaws.com/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello!", "session_id": "test"}'
```

## Notes

- Lambda timeout: Increase to 30+ seconds if needed
- CORS errors: Check API Gateway CORS settings
- S3 bucket names must be globally unique - add account ID suffix
- IAM role must have S3 access permissions

## Next: Day 3

Day 3 replaces OpenAI with Bedrock Nova models - cheaper and faster. The shift from external API to AWS-managed service changes the cost profile significantly.
