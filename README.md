# 🤖 AI Digital Twin - Week 2 Project

Building and deploying a production-ready AI Digital Twin that represents a person, integrated with OpenAI, deployed on AWS, automated with Terraform, and orchestrated via GitHub Actions CI/CD.

**Course**: [Generative and Agentic AI in Production](https://www.udemy.com/course/generative-and-agentic-ai-in-production/) by [Ed Donner](https://edwarddonner.com/)

---

## 📊 Week 2 Progress

| Day | Topic | Status | Commits |
|-----|-------|--------|---------|
| Day 1 | Local Development & Memory | ✅ Complete | [See commits](#day-1) |
| Day 2 | AWS Deployment | 🔜 Next | - |
| Day 3 | Bedrock Integration | ⏳ Pending | - |
| Day 4 | Infrastructure as Code | ⏳ Pending | - |
| Day 5 | CI/CD with GitHub Actions | ⏳ Pending | - |

---

## ✅ Day 1: Introducing The Twin

**Duration**: ~3-4 hours
**Focus**: Building a conversational AI Digital Twin with persistent memory

### What Was Built

A **full-stack AI chat application** running locally that demonstrates the importance of conversation memory:

#### Frontend (Next.js)
- Modern React chat interface with Tailwind CSS styling
- Real-time message display with user/assistant differentiation
- Auto-scroll to latest message
- Loading animation while waiting for responses
- Keyboard shortcuts (Enter to send, Shift+Enter for multiline)
- Responsive design for different screen sizes

#### Backend (FastAPI)
- RESTful API with session-based memory management
- OpenAI GPT-4o-mini integration
- CORS configuration for development
- Health check endpoints
- Three main endpoints:
  - `GET /` - Root endpoint
  - `GET /health` - Health check
  - `POST /chat` - Chat endpoint with message and session_id
  - `GET /sessions` - List all conversation sessions

#### Memory System
- File-based JSON storage for conversation history
- Each session gets a unique UUID identifier
- Messages persist across server restarts
- System personality injected from `backend/me.txt`

### Key Learning: Why Memory Matters

**The Problem**: Without memory, each AI interaction feels disconnected and frustrating. The twin doesn't remember what you just told it.

**The Solution**: Store conversation history and include it as context for each new request. This creates natural, continuous conversations.

### Project Structure

```
twin/
├── backend/
│   ├── server.py              # FastAPI application with endpoints
│   ├── me.txt                 # Personality/context for the twin
│   ├── requirements.txt        # Python dependencies
│   ├── .env                   # OpenAI API key (local only)
│   └── .venv/                 # Python virtual environment
├── frontend/
│   ├── app/
│   │   ├── page.tsx           # Main landing page
│   │   ├── layout.tsx         # Root layout
│   │   └── globals.css        # Global styles
│   ├── components/
│   │   └── twin.tsx           # Chat component (the heart of the UI)
│   ├── public/                # Static assets
│   ├── package.json           # Node dependencies
│   └── tsconfig.json          # TypeScript configuration
├── memory/                     # Conversation history storage
└── README.md                  # This file
```

### Tech Stack (Day 1)

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Frontend** | Next.js 15 + React 19 | Modern web framework with SSR |
| **Styling** | Tailwind CSS v4 | Utility-first CSS framework |
| **Icons** | lucide-react | Beautiful icon library |
| **Backend** | FastAPI | High-performance Python API framework |
| **AI** | OpenAI GPT-4o-mini | Language model for responses |
| **Runtime** | Python 3.13 | Backend runtime |
| **Package Mgr** | uv | Fast Python package manager |
| **Storage** | Local Files | JSON-based memory persistence |

### Running Locally

#### Prerequisites
- Python 3.13+
- Node.js 20+
- OpenAI API key

#### Backend Setup
```bash
cd backend

# Create and activate virtual environment
uv venv
source .venv/bin/activate  # Mac/Linux
# or
.venv\Scripts\activate     # Windows

# Install dependencies
uv add -r requirements.txt

# Create .env file with your OpenAI API key
echo "OPENAI_API_KEY=your_key_here" > .env
echo "CORS_ORIGINS=http://localhost:3000" >> .env

# Start the server
uv run uvicorn server:app --reload
```

Server runs at: `http://localhost:8000`

#### Frontend Setup
```bash
cd frontend

# Install dependencies
npm install

# Install Tailwind CSS styling library
npm install lucide-react

# Start development server
npm run dev
```

Frontend runs at: `http://localhost:3000`

### Testing the Application

1. Open your browser to `http://localhost:3000`
2. Start a conversation:
   - **Message 1**: "Hi! My name is Alex and I love Python"
   - **Twin Response**: Greets you with enthusiasm
   - **Message 2**: "What's my name and what do I love?"
   - **Expected Result**: Twin remembers your name is Alex and you love Python! ✨

3. Refresh the page and send a new message:
   - **Message 3**: "Do you remember my name?"
   - **Expected Result**: Twin still remembers from previous session!

### Key Takeaways from Day 1

✅ **Memory is Essential**
- Without it, conversations feel disconnected
- Context window management is important
- Persistent storage enables true conversations

✅ **Architecture Matters**
- Separation of concerns (frontend/backend)
- RESTful API design
- Session-based state management

✅ **Modern Stack Works Well**
- Next.js provides great developer experience
- FastAPI is fast and productive
- TypeScript catches errors early

### API Endpoints Reference

```bash
# Health check
curl http://localhost:8000/health

# Get all sessions
curl http://localhost:8000/sessions

# Send a chat message
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Hello!",
    "session_id": "optional-uuid"
  }'
```

### Memory Storage Format

Each conversation is stored as `memory/{session_id}.json`:

```json
[
  {
    "role": "user",
    "content": "Hi! My name is Alex",
    "timestamp": "2024-10-21T10:30:00"
  },
  {
    "role": "assistant",
    "content": "Hello Alex! Nice to meet you...",
    "timestamp": "2024-10-21T10:30:05"
  }
]
```

### Commits for Day 1

- [x] Initial project setup with Next.js and FastAPI
- [x] Frontend chat interface component
- [x] Backend FastAPI server with OpenAI integration
- [x] Memory persistence system
- [x] CORS configuration for development
- [x] Personality configuration from me.txt
- [x] Documentation and setup instructions

---

## 🔜 Coming Next: Day 2 - AWS Deployment

Next, we'll take this local application and deploy it to AWS:
- **Lambda** for the backend
- **S3** for static hosting and memory storage
- **API Gateway** for the REST API
- **CloudFront** for global HTTPS delivery

This will transform our local app into a **production-ready, globally distributed application**!

---

## 📚 Resources

### Course Materials
- [Week 2 Daily Guides](./week2/) - Detailed instructions for each day
- [CLAUDE.md](./CLAUDE.md) - AI assistant guidance for this project

### External Links
- [Next.js Documentation](https://nextjs.org/docs)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [Tailwind CSS](https://tailwindcss.com/)
- [Course - Generative AI in Production](https://www.udemy.com/course/generative-and-agentic-ai-in-production/)

### Tools Used
- [Cursor IDE](https://cursor.sh/) - AI-powered code editor
- [uv](https://docs.astral.sh/uv/) - Fast Python package manager
- [Docker](https://www.docker.com/) - For Lambda packaging

---

## 💡 Key Concepts Demonstrated

1. **Session Management** - Unique IDs for each conversation
2. **Context Windowing** - Limiting historical messages for efficiency
3. **Personality Injection** - System prompts that define AI behavior
4. **CORS** - Cross-origin resource sharing for local development
5. **Async Patterns** - Responsive UI while waiting for AI responses

---

## 🎯 Learning Outcomes (Day 1)

After completing Day 1, you'll understand:
- How to build a Next.js frontend with real-time interactions
- FastAPI basics and RESTful API design
- Session-based conversation management
- Why memory is critical for AI applications
- Local full-stack development workflow

---

## 📝 Notes

- **Environment**: Tested on macOS, Linux, and Windows
- **Python Version**: Requires Python 3.13+
- **Node Version**: Requires Node.js 20+
- **OpenAI**: Uses gpt-4o-mini model (lower cost than gpt-4)

---

**Last Updated**: Day 1 Complete ✅
**Next Milestone**: Day 2 AWS Deployment
**Course Status**: Week 2 - 1/5 days complete (20%)
