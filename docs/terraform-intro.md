# Terraform Introduction

Infrastructure as Code (IaC) tool. Define your entire AWS infrastructure in code, version control it, and deploy it reproducibly.

---

## What is Terraform?

Instead of clicking around the AWS Console to create resources, you write code that describes what you want. Terraform reads that code and creates/updates/destroys resources to match it.

### The Problem it Solves

**Without Terraform**:

- Create Lambda manually via AWS Console
- Create S3 bucket manually
- Create API Gateway manually
- Create CloudFront distribution manually
- Someone else wants to replicate? Repeat all clicks manually
- Update something? Forget what you changed last time

**With Terraform**:

- Write `.tf` files describing everything
- Run `terraform apply` → everything gets created
- Someone else clones repo, runs same command → identical setup
- Change code → run `terraform apply` → updates happen automatically
- All changes are version controlled in git

---

## Core Concepts

### Provider

**What it is**: Connection to a cloud service (AWS, Azure, Google Cloud, etc).

**In your code**:

```hcl
provider "aws" {
  region = "us-east-1"
}
```

**What it does**: Tells Terraform "I'm using AWS in us-east-1 region". All resources created will be in that region with those credentials.

**Why it matters**: Lets you manage multiple clouds or regions in same codebase.

---

### Resource

**What it is**: A specific thing you want to create (Lambda function, S3 bucket, API Gateway, etc).

**In your code**:

```hcl
resource "aws_lambda_function" "backend" {
  filename      = "lambda.zip"
  function_name = "twin-backend"
  role          = aws_iam_role.lambda_role.arn
  handler       = "lambda_handler.handler"
}
```

**What it does**: Creates a Lambda function named "twin-backend" using the zip file you specified.

**How Terraform tracks it**: Records that this resource exists, what it's named, what settings it has. If you run `terraform apply` again, it knows not to recreate it.

---

### Variable

**What it is**: A placeholder for values you might want to change without editing code.

**In your code**:

```hcl
variable "aws_region" {
  type        = string
  default     = "us-east-1"
  description = "AWS region for resources"
}

variable "environment" {
  type = string
  # No default - must be provided
}
```

**Why it matters**:

- Use same code for dev, test, and prod environments
- Different values each time (dev uses cheaper Lambda, prod uses bigger)
- Don't hardcode secrets in code

**How to use**:

```hcl
provider "aws" {
  region = var.aws_region
}
```

---

### Output

**What it is**: Values that Terraform prints after creating resources, so you know what was created.

**In your code**:

```hcl
output "lambda_function_name" {
  value       = aws_lambda_function.backend.function_name
  description = "Name of the Lambda function"
}

output "api_endpoint" {
  value       = aws_apigatewayv2_api.main.api_endpoint
}
```

**Why it matters**: After `terraform apply` completes, you see:

```
Outputs:
lambda_function_name = "twin-backend"
api_endpoint = "https://abc123.execute-api.us-east-1.amazonaws.com"
```

Now you know what was created and can test it.

---

### State

**What it is**: A file (usually `terraform.tfstate`) that tracks what Terraform created and current settings.

**Why it matters**:

- Terraform reads it to know "this Lambda already exists"
- If you run `apply` again, it sees state and doesn't recreate
- Shows what changed between runs

**Important**:

- `tfstate` contains sensitive info (passwords, keys) - DON'T commit to git
- Add `terraform.tfstate*` to `.gitignore`
- In team/production: Store state in S3 (Day 5)

**In your code**:

```
twin/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfstate        ← DO NOT COMMIT
└── .gitignore (includes tfstate files)
```

---

### Workspace

**What it is**: Multiple versions of the same infrastructure in same code.

**Why it matters**: Create dev, test, and prod environments from same `.tf` files, each with their own state.

**In your code**:

```hcl
# Default workspace
terraform workspace new dev
terraform workspace new test
terraform workspace new prod

# Switch workspaces
terraform workspace select dev
terraform apply  # Creates dev infrastructure

terraform workspace select prod
terraform apply  # Creates prod infrastructure (separate state)
```

**What happens**:

- `dev` and `prod` have separate Lambda functions, S3 buckets, etc
- Each has its own `terraform.tfstate` file
- Can test changes in dev before pushing to prod

---

## Basic Workflow

```
1. Write .tf files
   ↓
2. terraform init (initialize working directory)
   ↓
3. terraform plan (preview what will change)
   ↓
4. terraform apply (create/update resources)
   ↓
5. Review outputs
   ↓
6. Commit code to git
```

---

## File Structure (Day 4)

```
terraform/
├── main.tf              # Main resource definitions
├── variables.tf         # Input variables
├── outputs.tf           # Output values
├── versions.tf          # Provider versions
├── terraform.tfvars     # Default variable values
├── backend.tf           # State backend config (Day 5)
├── github-oidc.tf       # GitHub Actions auth (Day 5)
└── (other resource files)
```

---

## Common Terraform Commands

```bash
# Initialize Terraform (download provider plugins)
terraform init

# Preview changes (what will be created/updated/destroyed)
terraform plan

# Apply changes (actually create/update resources)
terraform apply

# Destroy everything (delete all resources)
terraform destroy

# List current state
terraform state list

# Show details of resource
terraform state show aws_lambda_function.backend

# Switch workspace
terraform workspace select dev

# Check syntax
terraform validate
```

---

## Why Terraform for This Project?

### Before (Manual AWS)

- Day 2: Click around AWS Console for 2 hours
- Day 3: Update settings manually in 5 places
- Day 4: Try to replicate prod setup for testing? Good luck remembering what you clicked

### After (Terraform)

- Day 4: Write `.tf` files describing everything
- Run `terraform apply` → everything matches code
- Want a test environment? `terraform workspace new test` + different variables
- Change anything? Update code, commit, `terraform apply` → repeatable

### Real Benefits

1. **Version control**: All infrastructure changes are in git history
2. **Reproducible**: New person runs `terraform apply`, gets identical setup
3. **Safe changes**: `terraform plan` shows exactly what will happen before you do it
4. **Multi-environment**: dev/test/prod from same code
5. **Automated**: Day 5 uses GitHub Actions + Terraform = push code → auto-deploy

---

## State Management

### Local State (Day 4)

```
terraform.tfstate lives on your computer
Problem: Other team members can't see it
Solution: Commit? NO - it has secrets
```

### Remote State (Day 5)

```
terraform.tfstate lives in S3
Everyone sees same state
DynamoDB locks it during apply (prevents conflicts)
Problem solved ✓
```

---

## Variables Example: Dev vs Prod

**variables.tf**:

```hcl
variable "lambda_memory" {
  type = number
}

variable "lambda_timeout" {
  type = number
}
```

**dev.tfvars**:

```
lambda_memory = 128  # Cheap
lambda_timeout = 10  # Fast testing
```

**prod.tfvars**:

```
lambda_memory = 512  # More powerful
lambda_timeout = 30  # Production reliability
```

**Command**:

```bash
terraform apply -var-file="dev.tfvars"   # Dev setup
terraform apply -var-file="prod.tfvars"  # Prod setup
```

Same code, different results based on variables.

---

## What Terraform Doesn't Do

**It doesn't manage code deploys** - Use GitHub Actions for that (Day 5)
**It doesn't manage configuration** - Use environment variables for that
**It doesn't manage containers** - That's Docker's job

---

## Next Steps

Day 4 will use these concepts to:

1. Define all infrastructure in `.tf` files
2. Use variables for dev/test/prod
3. Create workspaces for environment isolation
4. Automate deployment with terraform init/plan/apply
5. Store everything in git

Day 5 will add:

- Remote state storage (S3 + DynamoDB)
- GitHub Actions automation
- GitHub OIDC for secure AWS authentication

---

**Last Updated**: Day 3
