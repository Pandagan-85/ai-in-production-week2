# 📝 Day 4: Infrastructure as Code with Terraform

Setting up automated infrastructure deployment with Terraform. Eliminates manual AWS Console clicks, implements Least Privilege security, and enables dev/prod environment management from the same codebase.

---

## Key Accomplishments

- ✅ Terraform configuration for all AWS resources (Lambda, S3, API Gateway, CloudFront, IAM)
- ✅ Multi-environment setup (dev/test/prod) with separate `.tfvars` files
- ✅ Environment Configuration Management (same code, different values)
- ✅ Least Privilege security implementation
- ✅ Automated deployment script (`scripts/deploy.sh`)
- ✅ Automated destroy script (`scripts/destroy.sh`)
- ✅ Workspace isolation for environment state separation
- ✅ Complete Terraform documentation in `docs/terraform-intro.md`

---

## What We Built

### Terraform Configuration Files

```
terraform/
├── versions.tf          # Provider config (AWS region: eu-west-1)
├── variables.tf         # Variable definitions
├── main.tf              # All AWS resources
├── outputs.tf           # Output values (API URL, CloudFront URL, bucket names)
├── terraform.tfvars     # Dev environment values
└── prod.tfvars          # Prod environment values (template)
```

### Automation Scripts

```
scripts/
├── deploy.sh            # Automated deploy (Lambda build + Terraform + frontend sync)
└── destroy.sh           # Automated destroy (S3 cleanup + Terraform destroy)
```

---

## Key Changes Made

### 1. Fixed Region Configuration

**Problem**: Resources were being created in eu-north-1 instead of eu-west-1.

**Solution**: Added explicit region to `terraform/versions.tf`:

```hcl
provider "aws" {
  region = "eu-west-1"  # ← Explicit region
}
```

### 2. S3 Bucket Cleanup

**Problem**: `terraform destroy` failed with "BucketNotEmpty" error.

**Solution**: Added `force_destroy = true` to S3 buckets in `main.tf`:

```hcl
resource "aws_s3_bucket" "frontend" {
  bucket         = "${local.name_prefix}-frontend-${data.aws_caller_identity.current.account_id}"
  force_destroy  = true  # ← Allows destroy of non-empty buckets
  tags           = local.common_tags
}
```

### 3. Environment-Aware Scripts

**Problem**: Scripts always used default project name from hardcoded value.

**Solution**: Updated both `deploy.sh` and `destroy.sh` to read from `.tfvars`:

```bash
# Determine which tfvars file to use based on environment
if [ "$ENVIRONMENT" = "prod" ] && [ -f "terraform/prod.tfvars" ]; then
  TFVARS_FILE="terraform/prod.tfvars"
else
  TFVARS_FILE="terraform/terraform.tfvars"
fi

# Read PROJECT_NAME from correct file
if [ -f "$TFVARS_FILE" ]; then
  DEFAULT_PROJECT=$(grep '^project_name' "$TFVARS_FILE" | cut -d'=' -f2 | tr -d ' "')
else
  DEFAULT_PROJECT="twin"
fi
PROJECT_NAME=${2:-$DEFAULT_PROJECT}
```

### 4. Bedrock Model Configuration

**Problem**: Different environments need different model tiers.

**Solution**: Fixed `variables.tf` to use `eu.amazon.nova-lite-v1:0` (was defaulting to Micro):

```hcl
variable "bedrock_model_id" {
  description = "Bedrock model ID"
  type        = string
  default     = "eu.amazon.nova-lite-v1:0"  # ← Lite for better capability
}
```

**Model tiers**:
- Nova Micro: Fastest, cheapest (dev)
- Nova Lite: Balanced (prod default)
- Nova Pro: Best quality (highest cost)

---

## Terraform Concepts Implemented

### 1. Provider Configuration

Tells Terraform how to connect to AWS:

```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "eu-west-1"
}
```

### 2. Variables for Configuration

```hcl
variable "project_name" { ... }      # e.g., "twin2"
variable "environment" { ... }       # "dev", "test", "prod"
variable "bedrock_model_id" { ... }  # Model selection
variable "lambda_timeout" { ... }    # Execution time
```

### 3. Environment Configuration Management

**Concept**: Same Terraform code, different values for each environment.

```
terraform/
├── terraform.tfvars     # Dev: Nova Micro, 30s timeout, permissive IAM
├── prod.tfvars          # Prod: Nova Lite, 60s timeout, restrictive IAM
└── main.tf              # Same code used for both
```

**Deploy command**:
```bash
./scripts/deploy.sh dev     # Uses terraform.tfvars
./scripts/deploy.sh prod    # Uses prod.tfvars
```

### 4. Workspaces for State Separation

Each environment gets its own `terraform.tfstate` file:

```bash
terraform workspace new dev
terraform workspace new prod

terraform workspace select dev  # Create dev resources
terraform workspace select prod # Create prod resources separately
```

### 5. Outputs for Automation

```hcl
output "api_gateway_url" {
  value = aws_apigatewayv2_api.main.api_endpoint
}

output "s3_frontend_bucket" {
  value = aws_s3_bucket.frontend.id
}
```

These are read by `deploy.sh` to configure the frontend.

---

## Security: Least Privilege Principle

**Definition**: Give every resource only the minimum permissions it needs.

### Dev Environment (Permissive)

```hcl
resource "aws_iam_role_policy_attachment" "lambda_dev_s3" {
  role       = aws_iam_role.lambda_dev.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}
```

**Permissions**: Full access to S3 (for fast development iteration)

### Prod Environment (Restrictive)

```hcl
resource "aws_iam_role_policy" "lambda_prod_s3" {
  name = "twin-prod-s3-minimal"
  role = aws_iam_role.lambda_prod.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:PutObject"]
        Resource = "arn:aws:s3:::twin-prod-memory/*"  # ← Specific bucket only!
      },
      {
        Effect   = "Allow"
        Action   = "bedrock:InvokeModel"
        Resource = "arn:aws:bedrock:eu-west-1:*:foundation-model/amazon.nova-lite*"
      }
    ]
  })
}
```

**Permissions**: Only read/write conversation memory + invoke Bedrock (nothing else)

**Why**: If prod Lambda gets hacked, attacker can only read conversations, not delete infrastructure.

---

## Deployment Automation

### `scripts/deploy.sh` Workflow

1. **Build Lambda package**
   ```bash
   cd backend && uv run deploy.py
   # Creates: backend/lambda-deployment.zip
   ```

2. **Initialize Terraform**
   ```bash
   cd terraform && terraform init
   ```

3. **Create/select workspace**
   ```bash
   terraform workspace new $ENVIRONMENT  # First time
   terraform workspace select $ENVIRONMENT
   ```

4. **Apply Terraform**
   ```bash
   # Uses correct tfvars file based on environment
   terraform apply -var-file="$TFVARS_FILE" -auto-approve
   ```

5. **Build and deploy frontend**
   ```bash
   cd frontend
   npm install && npm run build
   aws s3 sync ./out s3://$FRONTEND_BUCKET/ --delete
   ```

**Usage**:
```bash
./scripts/deploy.sh dev              # All steps above for dev
./scripts/deploy.sh prod             # All steps above for prod
```

### `scripts/destroy.sh` Workflow

1. **Empty S3 buckets** (critical step!)
   ```bash
   aws s3 rm s3://bucket-name --recursive
   ```

2. **Destroy with Terraform**
   ```bash
   terraform destroy -var-file="$TFVARS_FILE" -auto-approve
   ```

**Usage**:
```bash
./scripts/destroy.sh dev    # Destroy dev resources
./scripts/destroy.sh prod   # Destroy prod resources
```

---

## AWS Resources Created

All these are now defined in Terraform instead of manual AWS Console clicks:

| Resource | File | Purpose |
|----------|------|---------|
| **S3 Bucket** | main.tf | Frontend static hosting |
| **S3 Bucket** | main.tf | Conversation memory storage |
| **Lambda** | main.tf | Backend API (Bedrock integration) |
| **API Gateway** | main.tf | REST endpoint for frontend |
| **CloudFront** | main.tf | CDN for global distribution |
| **IAM Role** | main.tf | Lambda permissions (least privilege) |
| **CloudWatch** | main.tf | Logging integration |

**All created in**: eu-west-1 region

---

## Documentation Created

### `docs/terraform-intro.md`

Complete Terraform reference covering:
- What is Terraform (IaC fundamentals)
- Core concepts (provider, resource, variable, output, state, workspace)
- File structure and descriptions
- Environment Configuration Management (`.tfvars` pattern)
- Least Privilege Principle (security best practices)
- IAM roles per environment (dev vs prod)
- Secrets management
- Monitoring & logging differences
- Security checklist for production

### `docs/day4.md` (this file)

Day 4 journey and implementation details.

---

## Architecture After Day 4

```
Infrastructure (Terraform)
├── Lambda function
│   ├── Connected to Bedrock
│   ├── Has minimal IAM permissions (prod)
│   └── Reads/writes conversation memory
├── S3 Buckets
│   ├── Frontend hosting (static files)
│   └── Memory storage (JSON conversations)
├── API Gateway
│   └── Routes requests to Lambda
├── CloudFront
│   └── Distributes frontend globally
└── CloudWatch
    └── Monitors Lambda execution

Automation
├── deploy.sh
│   ├── Builds Lambda
│   ├── Applies Terraform
│   └── Deploys frontend to S3
└── destroy.sh
    ├── Empties S3 buckets
    └── Destroys all Terraform resources

Environment Management
├── terraform.tfvars (dev)
├── prod.tfvars (prod - optional)
└── Workspaces (separate state per environment)
```

---

## Problems Solved

| Problem | Root Cause | Solution |
|---------|-----------|----------|
| Resources in wrong region | No explicit region in provider | Added `region = "eu-west-1"` to versions.tf |
| `terraform destroy` fails | S3 buckets not empty | Added `force_destroy = true` to buckets |
| Scripts use wrong `.tfvars` | Hardcoded project name | Made scripts read from correct tfvars file |
| Bot has no memory | Wrong Bedrock model | Changed to Nova Lite (more capable) |
| Manual deployment error-prone | No automation | Created deploy.sh and destroy.sh |

---

## Key Learnings

1. **Infrastructure as Code**
   - Define AWS resources in code files (not AWS Console)
   - Version control everything
   - Easy to recreate infrastructure

2. **Terraform State**
   - Tracks what's been created
   - Prevents accidental recreations
   - Contains secrets (don't commit!)

3. **Environment Management**
   - Same code for dev/prod
   - Different values in `.tfvars`
   - Workspaces separate state

4. **Least Privilege Security**
   - Dev: permissive (fast iteration)
   - Prod: restrictive (security first)
   - Industry standard practice

5. **Automation**
   - Scripts prevent human errors
   - Reproducible deployments
   - Documentation in code

---

## Testing

Successfully tested:

- ✅ `./scripts/deploy.sh dev` → Creates all AWS resources
- ✅ Backend connects to Bedrock correctly
- ✅ Frontend uploads to S3
- ✅ CloudFront distribution serves frontend
- ✅ API Gateway routes requests to Lambda
- ✅ Conversation memory persists to S3
- ✅ `./scripts/destroy.sh dev` → Cleans up resources
- ✅ Multiple deployments don't error out

---

## Next: Day 5

Day 5 automates deployment even further:

```
GitHub Push
  ↓
GitHub Actions triggered
  ↓
Runs ./scripts/deploy.sh dev
  ↓
Automatic deployment (no manual steps!)
```

Day 5 will add:
- GitHub repository
- GitHub Actions workflows
- S3 backend for Terraform state (team sharing)
- OIDC authentication (secure, no hardcoded AWS keys)
- Automatic deployment on push to main

---

**Status**: ✅ Complete
**Time Spent**: ~3 hours
**Key Achievement**: Infrastructure fully automated, repeatable, and version controlled
