# AWS Services Reference

Quick reference for AWS services used in this project. Each service has a specific purpose and use case.

---

## Compute Services (How to run code)

### Lambda

**What it is**: Serverless compute service. Upload code, AWS manages execution infrastructure. You pay only for execution time, billed in 100ms increments.

---

### EC2

**What it is**: Virtual machine instances. Rent scalable compute capacity with full control over OS, network, and installed software.

---

### App Runner

**What it is**: Container-based compute service. Runs Docker containers automatically with built-in auto-scaling and load balancing.

---

### ECS (Elastic Container Service)

**What it is**: Container orchestration service. Manages deployment and scaling of Docker containers across clusters with service discovery and load balancing.

---

## Storage Services (Where to put files)

### S3 (Simple Storage Service)

**What it is**: Object storage service. Stores and retrieves any amount of data via REST API with automatic replication and durability guarantees.

**In this project**:

- Frontend S3 bucket: Hosts static Next.js files
- Memory S3 bucket: Persists conversation JSON files

---

## API & Networking

### API Gateway

**What it is**: HTTP REST API gateway. Creates public endpoints that route HTTP requests to backend services (Lambda, EC2, etc.) with built-in request handling (routing, CORS, throttling, SSL/TLS).

**How it works**: Frontend calls the API Gateway URL → Gateway forwards request to Lambda → Lambda processes → Response returned to frontend.

---

### CloudFront

**What it is**: Content Delivery Network (CDN). Globally distributed edge locations cache and serve content with low latency. Integrates with S3 and other origins with SSL/TLS termination and compression.

**How it works**: CloudFront edge servers cached static files geographically close to users → users download from nearest location → faster page loads globally.

---

## AI & ML Services

### Bedrock

**What it is**: Managed AI service providing access to foundation models (Claude, Llama, Nova) via API without managing infrastructure or model hosting.

**Available Models**:

- **Nova Micro**: Fastest, lowest cost inference (development/testing)
- **Nova Lite**: Balanced performance and quality (production recommended)
- **Nova Pro**: Highest quality responses (higher latency and cost)

---

## Comparison: When to use what

### For Backend Compute:

| Service    | Cost     | Complexity | Scalability | Best For      |
| ---------- | -------- | ---------- | ----------- | ------------- |
| **Lambda** | Cheapest | Simple     | Auto        | Our project ✓ |
| App Runner | Low      | Medium     | Auto        | Web apps      |
| ECS        | Medium   | High       | Manual      | Microservices |
| EC2        | Variable | High       | Manual      | Custom needs  |

### For Storage:

| Service  | Use Case                       |
| -------- | ------------------------------ |
| **S3**   | Files, backups, static content |
| DynamoDB | Real-time database             |
| RDS      | SQL database                   |

### For Frontend:

| Service             | When                      |
| ------------------- | ------------------------- |
| **S3 + CloudFront** | Static sites (our choice) |
| Vercel              | Prefer managed Next.js    |
| App Runner          | Docker container          |

### For API:

| Service         | When                        |
| --------------- | --------------------------- |
| **API Gateway** | REST API with Lambda        |
| ALB             | Load balancing many servers |
| AppSync         | GraphQL API                 |

---

## Services Not Used Here

### Vercel

A platform optimized for Next.js hosting. They handle deployments automatically when you push to GitHub.

**Why not used**: We want full control with AWS, and S3 + CloudFront is cheaper.

---

## Architecture Recap day 2

```
User Browser
    ↓
CloudFront (CDN) → Serves frontend from S3
    ↓
API Gateway (REST endpoint)
    ↓
Lambda (backend code)
    ↓
Bedrock (AI responses)
    ↓
S3 (memory storage)
```

Each layer has a specific job. Removing or replacing any one changes the entire system.

---

## Quick Decision Tree

**"How should I run my code?"**

- Need simple HTTP API → **Lambda**
- Need long-running process → **EC2** or **App Runner**
- Need complex orchestration → **ECS**

**"Where should I store files?"**

- Static website files → **S3**
- User data/database → **DynamoDB** or **RDS**
- Temporary files → **S3**

**"How should I serve my frontend?"**

- Static files (like ours) → **S3 + CloudFront**
- Next.js with dynamic pages → **Vercel** or **App Runner**

**"Which AI model should I use?"**

- Quick responses, low cost → **Nova Micro**
- Balance → **Nova Lite** (our choice)
- Best quality → **Nova Pro**

---

**Last Updated**: Day 3
