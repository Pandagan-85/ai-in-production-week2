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

### Terraform File Descriptions

#### `main.tf` (Main Infrastructure Definition)
**Cosa contiene**: Tutte le risorse AWS che vuoi creare (Lambda, S3, API Gateway, CloudFront, IAM roles, etc).

**Perché**: È il "cuore" dell'infrastruttura. Definisce cosa viene effettivamente creato su AWS.

**Esempio**:
```hcl
resource "aws_s3_bucket" "frontend" { ... }
resource "aws_lambda_function" "api" { ... }
resource "aws_apigatewayv2_api" "main" { ... }
```

---

#### `variables.tf` (Input Variables)
**Cosa contiene**: Dichiarazioni di variabili che puoi cambiare senza modificare il codice.

**Perché**: Rende il codice riutilizzabile. Stessi `.tf` file, ma con valori diversi per dev/test/prod.

**Esempio**:
```hcl
variable "project_name" { ... }
variable "environment" { ... }
variable "lambda_timeout" { ... }
```

**Utilizzo**: Le variabili sono referenziate in `main.tf` con `var.project_name`, `var.environment`, etc.

---

#### `terraform.tfvars` (Default Variable Values)
**Cosa contiene**: I valori concreti per le variabili definite in `variables.tf`.

**Perché**: Terraform legge questo file automaticamente e popola i valori delle variabili.

**Esempio**:
```hcl
project_name = "twin"
environment  = "dev"
lambda_timeout = 30
```

**Nota**: Per ambienti diversi, crei file separati:
- `terraform.tfvars` → default (dev)
- `prod.tfvars` → per produzione
- `test.tfvars` → per testing

---

#### `versions.tf` (Provider Configuration)
**Cosa contiene**: Configurazione del provider AWS (region, versione), Terraform versioning, e provider aliases.

**Perché**: Specifica QUALE cloud usi (AWS), IN QUALE REGIONE (eu-west-1), e quali versioni sono compatibili.

**Esempio**:
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

---

#### `outputs.tf` (Output Values)
**Cosa contiene**: Valori che Terraform stampa a schermo dopo il deploy, utili per sapere cosa è stato creato.

**Perché**: Dopo `terraform apply`, vuoi sapere subito:
- URL dell'API Gateway
- URL di CloudFront
- Nomi dei bucket S3
- etc.

**Esempio**:
```hcl
output "api_gateway_url" {
  value = aws_apigatewayv2_api.main.api_endpoint
}

output "cloudfront_url" {
  value = aws_cloudfront_distribution.main.domain_name
}
```

**Utilizzo**: Lo script `deploy.sh` legge questi outputs per configurare il frontend.

---

#### `backend.tf` (State Storage - Day 5)
**Cosa contiene**: Configurazione di dove Terraform salva lo stato (per ora locale, ma Day 5 lo mette su S3).

**Perché**:
- **Dev/Local**: `terraform.tfstate` sul tuo computer
- **Team/Prod**: `terraform.tfstate` su S3 + DynamoDB lock (così il team condivide lo stesso stato)

**Esempio** (Day 5):
```hcl
terraform {
  backend "s3" {
    bucket         = "twin-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-lock"
  }
}
```

---

#### `github-oidc.tf` (GitHub Actions Authentication - Day 5)
**Cosa contiene**: Configurazione OIDC che permette a GitHub Actions di assumere un IAM role su AWS senza credenziali hardcoded.

**Perché**: Sicurezza. GitHub Actions può deployare su AWS senza memorizzare AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY.

**Flusso**:
```
GitHub Actions push
  ↓
GitHub genera token OIDC
  ↓
AWS verifica token OIDC
  ↓
GitHub Actions assume IAM role
  ↓
Deploy su AWS (senza credenziali in memoria)
```

---

### Creating `.tfvars` Files for Different Environments

Questo processo si chiama **Environment Configuration Management** o **Infrastructure Templating**. L'idea è: **stesso codice Terraform, configurazioni diverse per ogni ambiente**.

#### Come funziona:

```
1. Scrivi il codice generico in .tf files (main.tf, variables.tf)
   ↓
2. Ogni ambiente ha il suo .tfvars file con valori specifici
   ↓
3. terraform apply -var-file="ambiente.tfvars" applica quell'ambiente
```

#### File di configurazione per nostro progetto:

```
terraform/
├── main.tf              # Codice generico (uguale per tutti gli ambienti)
├── variables.tf         # Dichiarazioni variabili (definisce cosa è configurabile)
├── outputs.tf
├── versions.tf
│
├── terraform.tfvars     # ← Dev environment (default)
├── test.tfvars          # ← Test environment (opzionale)
├── prod.tfvars          # ← Prod environment (quando pronto)
└── staging.tfvars       # ← Staging environment (opzionale)
```

#### Processo passo-passo:

**Step 1: Crea il file base (terraform.tfvars)**
```hcl
# terraform.tfvars (dev)
project_name             = "twin2"
environment              = "dev"
bedrock_model_id         = "eu.amazon.nova-micro-v1:0"
lambda_timeout           = 30
lambda_memory            = 256
api_throttle_burst_limit = 5
enable_cloudtrail        = false
```

**Step 2: Crea file per prod**
```bash
cp terraform/terraform.tfvars terraform/prod.tfvars
```

**Step 3: Modifica i valori in prod.tfvars**
```hcl
# prod.tfvars (modificato per produzione)
project_name             = "twin2"
environment              = "prod"                          # ← Cambiato
bedrock_model_id         = "eu.amazon.nova-lite-v1:0"     # ← Cambiato
lambda_timeout           = 60                              # ← Cambiato
lambda_memory            = 512                             # ← Cambiato
api_throttle_burst_limit = 20                              # ← Cambiato
enable_cloudtrail        = true                            # ← Cambiato
```

**Step 4: Deploy con il file giusto**
```bash
# Dev
terraform workspace select dev
terraform apply -var-file="terraform.tfvars"

# Prod
terraform workspace select prod
terraform apply -var-file="prod.tfvars"
```

#### Vantaggi di questo approccio:

| Aspetto | Vantaggio |
|---------|-----------|
| **Riusabilità** | Stesso codice `main.tf` per tutti gli ambienti |
| **Consistenza** | Nessun rischio di dimenticare un valore quando si cambia ambiente |
| **Tracciabilità** | Ogni file `.tfvars` è in git, puoi vedere la storia dei cambiamenti |
| **Sicurezza** | Differenze di sicurezza tra ambienti sono esplicite (cloudtrail, timeout, etc) |
| **Scalabilità** | Facile aggiungere un nuovo ambiente (copia un `.tfvars` e modifica) |

#### Naming Convention Standard:

```
terraform.tfvars    ← Default (di solito dev)
dev.tfvars          ← Ambiente di sviluppo
test.tfvars         ← Ambiente di testing
staging.tfvars      ← Ambiente di pre-produzione (opzionale)
prod.tfvars         ← Ambiente di produzione
```

**Nota**: Alcuni team usano anche `terraform-dev.tfvars`, `terraform-prod.tfvars`, ma il nostro naming è più semplice e leggibile.

#### Quando usare `.tfvars` vs Workspaces:

| Situazione | Usa `.tfvars` | Usa `Workspaces` |
|-----------|---|---|
| **Diversi valori di variabili** | ✅ Sì | Opzionale |
| **Stato separato (tfstate diversi)** | ✅ Sì | ✅ Sì |
| **Team diversi con permessi diversi** | ✅ Sì | ✅ Sì |
| **Semplice (dev vs prod)** | ✅ Consigliato | Funziona ma overkill |

**In pratica**: Usiamo **`.tfvars` + Workspaces** insieme:
- `.tfvars` gestisce i valori di configurazione
- `Workspaces` gestisce lo stato separato

---

### File Organization Best Practices

Molti team dividono `main.tf` in file più piccoli per mantenere il codice organizzato:

```
terraform/
├── main.tf              # Risorse principali (Lambda, API Gateway)
├── s3.tf                # Tutti i bucket S3 e configurazioni
├── iam.tf               # IAM roles e policies
├── cloudfront.tf        # CloudFront distribution
├── variables.tf         # Variables
├── outputs.tf           # Outputs
├── versions.tf          # Provider config
├── terraform.tfvars     # Values for dev
├── prod.tfvars          # Values for prod
└── backend.tf           # State config
```

**Vantaggio**: Facile trovare una risorsa specifica. **Svantaggio**: Più file da mantenere.

Per il nostro progetto, `main.tf` va bene così com'è (tutto in un file). Se cresce, potrai dividerlo.

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

## Environment Management & Security Best Practices

### Multi-Environment Architecture

In production, you should use **separate IAM roles and configurations** for each environment:

```
AWS Account (or VPC)
├── Dev Environment
│   ├── Lambda with dev IAM role (permissive)
│   ├── Nova Micro model (cheap)
│   └── CloudWatch basic logging
├── Test Environment
│   ├── Lambda with test IAM role (restricted)
│   ├── Nova Lite model (balanced)
│   └── CloudWatch detailed logging
└── Prod Environment
    ├── Lambda with prod IAM role (minimal permissions)
    ├── Nova Lite/Pro model (reliable)
    ├── CloudWatch + CloudTrail (full audit)
    └── AWS Secrets Manager (for sensitive data)
```

### Security Principle: Least Privilege (Principio del Minimo Privilegio)

#### Cos'è Least Privilege?

**Definizione formale**:
> *"Ogni utente, processo, o sistema deve avere accesso SOLO ai permessi minimi necessari per svolgere il proprio compito, nulla di più."*

**Nella pratica**: Anziché dare accesso totale, dai solo quello che serve per fare il lavoro specifico.

**Perché importa**: Se un sistema viene compromesso (hacked, bug, malware), i danni sono limitati a quello che quel sistema può fare. Con permessi ristretti, l'attaccante non può fare danno.

#### Esempio nel nostro progetto:

```
SBAGLIATO (❌ Mai fare):
Lambda ha AmazonS3FullAccess
  → Può leggere/scrivere/eliminare TUTTI i bucket S3
  → Se hackerata: Attaccante elimina tutto

CORRETTO (✅ Least Privilege):
Lambda ha permesso SOLO per s3:GetObject, s3:PutObject su "twin-prod-memory/*"
  → Può SOLO leggere/scrivere conversazioni
  → Se hackerata: Attaccante legge conversazioni ma non elimina nulla
```

#### Principi di sicurezza correlati:

| Principio | Significato | Esempio |
|-----------|-------------|---------|
| **Least Privilege** | Dai MENO permessi possibile | Lambda solo s3:GetObject |
| **Defense in Depth** | Tanti strati di difesa | IAM + VPC + Encryption + CloudTrail |
| **Zero Trust** | Non fidarsi di niente | Verifica SEMPRE, anche risorse interne |
| **Separation of Duties** | Divide responsabilità | Dev non può deployare su Prod |

#### Come implementare Least Privilege:

**Passo 1: Identifica cosa serve**
```
Lambda deve:
- Leggere conversazioni da S3
- Scrivere conversazioni su S3
- Invocare Bedrock per AI
- Niente più
```

**Passo 2: Crea policy specifica**
```hcl
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::twin-prod-memory/*"  # ← Specifico!
    },
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:eu-west-1:*:foundation-model/amazon.nova-lite*"
    }
  ]
}
```

**Passo 3: Testa regolarmente**
- Leggi i CloudTrail logs
- Vedi se Lambda usa permessi che non ha
- Se un permesso non viene usato, rimuovilo

#### Checklist Least Privilege per il tuo progetto:

- [ ] Lambda dev ha S3 + Bedrock full access (OK per dev)
- [ ] Lambda prod ha SOLO permessi per s3:GetObject, s3:PutObject su memoria
- [ ] Lambda prod ha SOLO permesso di invocare specifico modello Bedrock
- [ ] Lambda non ha permessi per eliminare risorse
- [ ] Lambda non ha permessi per cambiare IAM policies
- [ ] IAM role non ha azioni wildcard (*) in prod
- [ ] I permessi sono documentati e revisionati mensile

---

### IAM Roles per Environment: Configurazione Pratica

**Regola d'oro**: In produzione, dai **MENO permessi possibile**. In dev, dai più permessi per facilitare sviluppo veloce.

**Perché**: Se la tua Lambda viene compromessa o hackerata, con permessi ristretti il danno è limitato. Con permessi ampi (Admin), l'attaccante ha accesso totale ad AWS.

| Ambiente | Approccio | Rischio |
|----------|-----------|--------|
| **Dev** | Permessi ampi | Basso (è il tuo computer) |
| **Prod** | Permessi minimi | Altissimo (risorse in produzione) |

#### Dev: Permissivo (Ma MAI in Produzione!)
```hcl
# ❌ SBAGLIATO - Mai fare
resource "aws_iam_role_policy_attachment" "lambda_dev_admin" {
  role       = aws_iam_role.lambda_dev.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}
```

**Permessi**: Accesso totale ad AWS (cancellare tutto, cambiare IAM, etc) - OK per dev, DISASTRO in prod.

#### Prod: Restrittivo (Security First)
```hcl
# ✅ CORRETTO
resource "aws_iam_role_policy" "lambda_prod_s3" {
  name   = "twin-prod-s3-minimal"
  role   = aws_iam_role.lambda_prod.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:PutObject"]
        Resource = "arn:aws:s3:::twin-prod-memory/*"  # ← Specifico!
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

**Permessi**: Solo lettura/scrittura su S3 memory + invocare Bedrock - Nient'altro.

**Se Lambda viene hackerata**: L'attaccante può solo leggere conversazioni e chiamare Bedrock, non cancellare l'infra.

---

Each environment should have its own IAM role with minimal permissions (principle of least privilege):

```hcl
# Dev: More permissive for development
resource "aws_iam_role" "lambda_dev" {
  name = "twin-dev-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_dev_s3" {
  role       = aws_iam_role.lambda_dev.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

# Prod: Minimal, restricted permissions
resource "aws_iam_role" "lambda_prod" {
  name = "twin-prod-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

# Custom policy: only access specific S3 bucket
resource "aws_iam_role_policy" "lambda_prod_s3" {
  name = "twin-prod-s3-policy"
  role = aws_iam_role.lambda_prod.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = ["s3:GetObject", "s3:PutObject"]
      Resource = "arn:aws:s3:::twin-prod-memory/*"
    }]
  })
}
```

### Configuration Differences per Environment

Use separate `.tfvars` files to configure each environment:

**terraform.tfvars** (dev):
```hcl
project_name             = "twin"
environment              = "dev"
bedrock_model_id         = "eu.amazon.nova-micro-v1:0"    # Micro: fastest, cheapest
lambda_timeout           = 30
lambda_memory            = 256
api_throttle_burst_limit = 5
use_cloudtrail           = false
enable_secrets_manager   = false
```

**prod.tfvars** (prod):
```hcl
project_name             = "twin"
environment              = "prod"
bedrock_model_id         = "eu.amazon.nova-lite-v1:0"     # Lite: balanced performance
lambda_timeout           = 60
lambda_memory            = 512
api_throttle_burst_limit = 20
use_cloudtrail           = true              # Enable audit logging
enable_secrets_manager   = true              # Use AWS Secrets Manager
```

### Deployment by Environment

```bash
# Dev: Fast iteration
terraform workspace select dev
terraform apply -var-file="terraform.tfvars"

# Prod: Slower, more careful
terraform workspace select prod
terraform apply -var-file="prod.tfvars"
# ^ Terraform will show detailed plan first
```

### Secrets Management

**Development**: `.env` files (local only)
```bash
BEDROCK_API_KEY=xxx
OPENAI_API_KEY=xxx
```

**Production**: AWS Secrets Manager (secure, audited)
```hcl
resource "aws_secretsmanager_secret" "bedrock_key" {
  name = "${local.name_prefix}-bedrock-key"
}

resource "aws_iam_role_policy" "lambda_secrets" {
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "secretsmanager:GetSecretValue"
      Resource = aws_secretsmanager_secret.bedrock_key.arn
    }]
  })
}
```

Then in Lambda, retrieve secrets:
```python
import boto3
secrets = boto3.client('secretsmanager')
response = secrets.get_secret_value(SecretId='twin-prod-bedrock-key')
api_key = response['SecretString']
```

### Monitoring & Logging Differences

| Aspect | Dev | Prod |
|--------|-----|------|
| **CloudWatch Logs** | Basic | Detailed + retention 30 days |
| **CloudTrail** | Optional | Required (audit trail) |
| **Alarms** | None | Critical errors only |
| **Metrics** | Basic | Full (latency, errors, throttles) |
| **X-Ray** | No | Yes (distributed tracing) |

### Security Checklist for Production

- [ ] Separate IAM roles per environment
- [ ] Use AWS Secrets Manager for API keys (not env vars)
- [ ] Enable CloudTrail for audit logging
- [ ] Enable CloudWatch alarms for errors
- [ ] Use KMS encryption for S3 buckets
- [ ] Enable versioning on S3 buckets (disaster recovery)
- [ ] Use different AWS accounts for dev/prod (optional but recommended)
- [ ] Regular backup of Terraform state
- [ ] Restrict IAM permissions to minimum needed

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
