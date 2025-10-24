# Day 5: CI/CD with GitHub Actions

## Overview

Day 5 è l'ultimo giorno della Week 2! Abbiamo implementato il ciclo DevOps completo: dalla gestione del codice su GitHub, alle pipeline di deployment automatiche con GitHub Actions, fino alla gestione multi-environment su AWS.

In sostanza: **push to main → GitHub Actions → Terraform → deployment automatico su AWS dev, con possibilità di deployment manuale su test e prod.**

---

## Parte 1: Preparazione - Pulizia dell'infrastruttura

Prima di configurare GitHub Actions, abbiamo pulito tutti gli ambienti creati nei giorni precedenti per ricominciare da zero.

### Step 1: Destroy degli ambienti

Usato gli script di destroy creati il Day 4:

```bash
./scripts/destroy.sh dev
./scripts/destroy.sh test
./scripts/destroy.sh prod
```

Ogni destroy ha preso 5-10 minuti per rimuovere le distribuzioni CloudFront.

### Step 2: Pulizia dei workspace Terraform

```bash
cd terraform
terraform workspace select default
terraform workspace delete dev
terraform workspace delete test
terraform workspace delete prod
```

### Step 3: Verifica della pulizia

Controllato su AWS Console che non rimangessero risorse `twin-*`:

- ✅ Nessun Lambda
- ✅ Nessun S3 bucket
- ✅ Nessun API Gateway
- ✅ Nessun CloudFront

---

## Parte 2: Git & GitHub Setup

### Step 1: .gitignore e .env.example

Creato `.gitignore` completo per escludere:

- Terraform state files (`*.tfstate`)
- Lambda packages
- Memory storage
- Environment variables (`.env`)
- Node e Python dependencies

Creato `.env.example` come template per chi clona il repo.

### Step 2: Inizializzazione Git

```bash
git init -b main
git config user.name "Your Name"
git config user.email "your.email@example.com"
git add .
git commit -m "Initial commit: Digital Twin infrastructure and application"
```

### Step 3: GitHub Repository

Creato il repository su GitHub e fatto push:

```bash
git remote add origin https://github.com/YOUR_USERNAME/digital-twin.git
git push -u origin main
```

✅ **Checkpoint**: Codice e infrastruttura su GitHub!

---

## Parte 3: S3 Backend per lo stato Terraform

### Step 1-2: backend-setup.tf

Creato un file temporaneo `terraform/backend-setup.tf` che contiene:

- S3 bucket per lo stato Terraform
- DynamoDB table per i lock (evita conflitti quando più persone deployano)
- Encryption e versioning abilitati

Eseguito:

```bash
terraform apply -target=aws_s3_bucket.terraform_state -target=aws_dynamodb_table.terraform_locks ...
```

### Step 3: Eliminazione del file setup

Dopo la creazione:

```bash
rm terraform/backend-setup.tf
```

**Importante**: Il file è "usa-e-getta". Lo usi una volta per creare le risorse, poi lo elimini. Chiunque cloni il repo leggendo la documentazione (week2/day5.md) saprà come ricrearlo per il PROPRIO account.

### Step 4: Aggiornamento degli script

Modificato `scripts/deploy.sh` per configurare il backend dinamicamente:

```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=${DEFAULT_AWS_REGION:-eu-west-1}
terraform init -input=false \
  -backend-config="bucket=twin-terraform-state-${AWS_ACCOUNT_ID}" \
  -backend-config="key=${ENVIRONMENT}/terraform.tfstate" \
  -backend-config="region=${AWS_REGION}" \
  -backend-config="dynamodb_table=twin-terraform-locks" \
  -backend-config="encrypt=true"
```

Questo rende il deploy portabile: chiunque lo esegue usa il PROPRIO account AWS.

---

## Parte 4: GitHub OIDC e IAM Role

### Step 1: github-oidc.tf

Creato `terraform/github-oidc.tf` che contiene:

- OIDC Provider (GitHub)
- IAM Role per GitHub Actions
- Policy attachments (Lambda, S3, API Gateway, Bedrock, DynamoDB, ecc.)

Questo ruolo è quello che GitHub Actions usa per fare deploy su AWS, **senza salvare access keys**.

### Step 2: Applicazione della configurazione

```bash
terraform apply -target=aws_iam_openid_connect_provider.github \
  -target=aws_iam_role.github_actions \
  ... [altre policy]
```

### Step 3: Eliminazione del file

Dopo la creazione:

```bash
rm github-oidc.tf
```

**Stesso concetto di backend-setup.tf**: è temporaneo. Chiunque voglia ricreare il ruolo per il PROPRIO GitHub repo legge la documentazione e sa come farlo.

### Step 4: Aggiunta dei Secrets su GitHub

Navigato in **GitHub → Settings → Secrets and variables → Actions** e creato 3 secrets:

- `AWS_ROLE_ARN`: L'ARN del ruolo appena creato
- `AWS_ACCOUNT_ID`: L'ID del tuo account AWS
- `DEFAULT_AWS_REGION`: La regione (eu-west-1)

✅ **Checkpoint**: GitHub può ora autenticarsi con AWS usando OIDC!

---

## Parte 5: GitHub Actions Workflows

### Step 1: Struttura

Creato `.github/workflows/` con due file:

```
.github/workflows/
├── deploy.yml
└── destroy.yml
```

### Step 2: deploy.yml

Workflow che:

1. Fa checkout del codice
2. Configura le credenziali AWS usando OIDC
3. Installa Python, Node.js, Terraform
4. Esegue `./scripts/deploy.sh`
5. Invalida la cache CloudFront
6. Mostra gli URL di deployment

**Trigger**:

- Push a `main` → deployment automatico a `dev`
- `workflow_dispatch` manuale → scelta tra dev/test/prod

### Step 3: destroy.yml

Workflow per distruggere gli ambienti:

- Richiede conferma (devi digitare il nome dell'environment)
- Esegue `./scripts/destroy.sh`
- Svuota gli S3 bucket prima della distruzione

**Trigger**: Solo `workflow_dispatch` (manuale)

### Step 4: Commit e Push

```bash
git add .github/workflows/
git commit -m "Add CI/CD with GitHub Actions, S3 backend, and updated scripts"
git push
```

Questo push ha automaticamente lanciato il deploy a `dev`! 🚀

---

## Parte 6: Test dei deployment

### Step 1: Dev (Automatico)

Il push a main ha lanciato GitHub Actions automaticamente:

- GitHub Actions tab → "Deploy Digital Twin" workflow in esecuzione
- Ha creato il workspace dev
- Ha eseguito deploy.sh
- Deployment completo in 5-10 minuti

### Step 2: Test (Manuale)

Da GitHub Actions UI:

- "Run workflow" → scegli environment "test"
- Deployed manualmente

### Step 3: Verifica

Ogni workflow mostra alla fine:

- 🌐 CloudFront URL (il sito live)
- 📡 API Gateway URL
- 🪣 Frontend Bucket name

---

## Parte 7: Aggiustamenti UI

Aggiornato `frontend/components/twin.tsx` per:

- Fixare il focus dell'input dopo l'invio del messaggio (risolto il bug dell'UX)
- Aggiungere support per avatar personalizzato

Fatto push → GitHub Actions ha ri-deployato automaticamente. ✅

---

## Parte 8-9: Monitoraggio e gestione ambienti

### Esplorazione AWS Console

Controllato:

- Lambda invocations e logs su CloudWatch
- Bedrock metrics (token usage, latency)
- S3 buckets con la conversation history
- API Gateway traffic
- CloudFront analytics

### Destroy dei test environment

Da GitHub → "Destroy Environment" workflow:

- Scelto "test"
- Digitato "test" per confermare
- Environment completamente rimosso in 5-10 minuti

Poi rilanciato il deploy a test → tutto di nuovo online. Perfetto per testare il ciclo completo!

---

## Parte 10: Cleanup finale e costi

### Distruzione completa

Distrutti tutti gli ambienti (dev, test, prod) via GitHub Actions.

### Cleanup opzionale

Rimangono questi resource a costo minimo:

- **IAM Role** (`github-actions-twin-deploy`): FREE
- **S3 State Bucket**: ~$0.02/mese
- **DynamoDB Table**: ~$0.00/mese (pay-per-request)

**Raccomandazione**: Lasciarli attivi! Costano quasi nulla e permettono di rifare il deploy in futuro.

---

## Key Learnings 🎓

### 1. **Infrastructure as Code + CI/CD = Magia**

Prima: Deploy manuale su AWS Console (lento, errori, non reproducibile)
Dopo: `git push` → deployment automatico (veloce, affidabile, reproducibile)

### 2. **GitHub Secrets + OIDC = Sicurezza senza compromessi**

Non abbiamo mai salvato access keys nel repo. OIDC ci ha permesso:

- GitHub Actions → OIDC token → AWS role assunzione
- Niente credenziali long-lived
- Facile rotate/revoke

### 3. **Terraform state remoto = team collaboration**

S3 backend + DynamoDB locks:

- Lo stato è centralizzato
- Più persone possono deployare senza conflitti
- Version history del stato

### 4. **GitHub Actions workflows = documentazione vivente**

I workflow sono la documentazione del come deployare. Chiunque legge `.github/workflows/deploy.yml` capisce:

- Quali step vengono eseguiti
- In quale ordine
- Con quali variabili

### 5. **Multi-environment via workspace**

Dev/test/prod sono gestiti dallo stesso codice Terraform, separati via workspace. Fatto deployment su dev automaticamente, manuale su test/prod. Perfetto per safety!

---

## Architecture finale 🏗️

```
GitHub (main branch)
    ↓ git push
GitHub Actions
    ├─ Dev: automatico su ogni push
    └─ Test/Prod: manuale da UI
    ↓
Terraform (via scripts/deploy.sh)
    ├─ Workspace dev/test/prod
    └─ State: S3 + DynamoDB locks
    ↓
AWS Infrastructure
    ├─ Lambda (backend)
    ├─ API Gateway
    ├─ S3 (frontend + memory)
    ├─ CloudFront (CDN)
    └─ Bedrock (AI)
```

---

## Cosa è stato difficile

1. **Capire il flusso OIDC**: Non è ovvio come GitHub usa il token per assumere un ruolo AWS
2. **File "usa-e-getta" (backend-setup.tf, github-oidc.tf)**: Inizialmente non era chiaro perché eliminarli se li potevamo tenere. Dopo ho capito: sono setup one-time, non fanno parte del deployment normale
3. **DynamoDB table per i lock**: Non necessario per uno sviluppatore solo, ma essenziale per un team
4. **Debugging dei workflow**: Quando fallisce un GitHub Actions workflow, i log erano inizialmente poco chiari

---

## Cosa abbiamo guadagnato

✅ Deployment completamente automatico
✅ Multi-environment management
✅ State management remoto e sicuro
✅ No hardcoded credentials
✅ Replicabile per chiunque cloni il repo
✅ Audit trail dei deployment (git history)

---

## Prossimi step - Refactoring del website-agent 🚀

Ho un altro progetto precedente: il [website-agent](https://github.com/Pandagan-85/website-agent) - un agente autonomo con LangGraph già in produzione.

Adesso voglio refactorizzarlo usando la **nuova struttura e metodologia imparata questa settimana**:

### Piano di migrazione:

1. **Terraform + AWS Infrastructure**

   - Migrare da deployment manuale a Infrastructure as Code
   - Creare dev/test/prod environments separati
   - S3 backend per lo stato Terraform

2. **GitHub Actions CI/CD**

   - Setup GitHub Actions workflows
   - Deploy automatico a dev su ogni push
   - Deployment manuale a test/prod

3. **Migliorare lo stato**

   - DynamoDB locks per team collaboration
   - Centralizzare lo stato remoto
   - Audit trail dei deployment

4. **Aggiornare il flusso di deploy**
   - Scripts bash automatici (come deploy.sh)
   - Multi-environment management
   - Distruzione facile dei test environment

---

**Week 2 completata! 🎉**

Abbiamo costruito un sistema AI production-ready con:

- Local development (Day 1)
- AWS deployment (Day 2)
- AI integration (Day 3)
- Infrastructure as Code (Day 4)
- Fully automated CI/CD (Day 5)

This is how real companies do it! 🚀
