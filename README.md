# 🤖 AI Digital Twin - Week 2 Project

A production-ready AI Digital Twin that learns about you, deployed on AWS with serverless backend, Bedrock AI, Infrastructure as Code, and CI/CD automation.

**Course**: [Generative and Agentic AI in Production](https://www.udemy.com/course/generative-and-agentic-ai-in-production/) by [Ed Donner](https://edwarddonner.com/)

---

## 📊 Week 2 Progress

| Day   | Topic                      | Status      | Docs                     |
| ----- | -------------------------- | ----------- | ------------------------ |
| Day 1 | Local Development & Memory | ✅ Complete | [📖 Day 1](docs/day1.md) |
| Day 2 | AWS Deployment             | ✅ Complete | [📖 Day 2](docs/day2.md) |
| Day 3 | Bedrock Integration        | ✅ Complete | [📖 Day 3](docs/day3.md) |
| Day 4 | Infrastructure as Code     | ✅ Complete | [📖 Day 4](docs/day4.md) |
| Day 5 | CI/CD with GitHub Actions  | ✅ Complete | [📖 Day 5](docs/day5.md) |

---

## 🚀 Quick Start (Day 1)

### Prerequisites

```
Python 3.13+  •  Node.js 20+  •  OpenAI API key
```

### Backend Setup

```bash
cd backend
uv venv && source .venv/bin/activate
uv add -r requirements.txt
echo "OPENAI_API_KEY=your_key_here" > .env
uv run uvicorn server:app --reload
```

Backend: `http://localhost:8000`

### Frontend Setup

```bash
cd frontend
npm install && npm install lucide-react
npm run dev
```

Frontend: `http://localhost:3000`

### Test Memory

1. Say: "My name is Alex and I love Python"
2. Say: "What's my name and what do I love?"
3. Expected: Twin remembers! ✨

---

## 🏗️ Architecture Overview

### Day 1 (Local)

```
Frontend (Next.js)  ←→  Backend (FastAPI)  ←→  OpenAI
                              ↓
                        Memory (JSON files)
```

### Day 2-3 (AWS + Bedrock)

```
CloudFront → S3 (Frontend)
           ↓
        API Gateway → Lambda → Bedrock (Nova)
                         ↓
                    S3 (Memory)
```

### Day 4-5 (Automated)

```
GitHub → GitHub Actions → Terraform → AWS (auto-deployed)
```

---

## 📚 Documentation

My journey each day `docs/` folder:

- **[docs/day1.md](docs/day1.md)** - Build local chat with memory
- **[docs/day2.md](docs/day2.md)** - Deploy to AWS Lambda + S3 + CloudFront
- **[docs/day3.md](docs/day3.md)** - Replace OpenAI with Bedrock Nova
- **[docs/day4.md](docs/day4.md)** - Infrastructure as Code with Terraform
- **[docs/day5.md](docs/day5.md)** - CI/CD with GitHub Actions, S3 backend, automated deployments

### Reference Guides

- **[docs/terraform-intro.md](docs/terraform-intro.md)** - Complete Terraform guide (concepts, best practices, security)
- **[docs/aws-services.md](docs/aws-services.md)** - AWS services reference (Lambda, S3, API Gateway, Bedrock, CloudFront)

---

## 🛠️ Tech Stack

| Component       | Technology                               |
| --------------- | ---------------------------------------- |
| **Frontend**    | Next.js 15 + React 19 + Tailwind CSS     |
| **Backend**     | FastAPI + Python 3.13                    |
| **Package Mgr** | uv (Python) + npm (Node)                 |
| **AI**          | OpenAI (Day 1-2) → Bedrock Nova (Day 3+) |
| **Compute**     | Local (Day 1) → AWS Lambda (Day 2+)      |
| **Storage**     | Local files (Day 1) → S3 (Day 2+)        |
| **CDN**         | CloudFront (Day 2+)                      |
| **IaC**         | Terraform (Day 4+)                       |
| **CI/CD**       | GitHub Actions (Day 5)                   |

---

## 💡 Key Concepts

- Session-based conversation memory
- Context windowing for efficiency
- Personality injection via system prompts
- CORS and API security
- Async patterns for responsive UX

---

## 🔗 Resources

### Documentation

- [Next.js Docs](https://nextjs.org/docs)
- [FastAPI Docs](https://fastapi.tiangelo.com/)
- [AWS Lambda](https://docs.aws.amazon.com/lambda/)
- [Terraform AWS](https://registry.terraform.io/providers/hashicorp/aws/latest)

### Tools

- [VS code](https://code.visualstudio.com/)
- [uv](https://docs.astral.sh/uv/) - Fast Python package manager
- [AWS CLI](https://aws.amazon.com/cli/) - AWS management

---

## 📝 Project Structure

```
twin/
├── docs/
│   ├── day1.md          # Local development guide
│   ├── day2.md          # AWS deployment guide
│   ├── day3.md          # Bedrock migration guide
│   └── ...
├── backend/
│   ├── server.py        # FastAPI app
│   ├── me.txt           # Twin personality
│   ├── requirements.txt  # Python deps
│   └── .env             # Config (not in git)
├── frontend/
│   ├── app/             # Next.js app
│   ├── components/      # React components
│   └── package.json     # Node deps
├── memory/              # Conversation storage
├── CLAUDE.md            # AI assistant guide
└── README.md            # This file
```

---
