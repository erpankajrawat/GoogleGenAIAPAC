# Implementation Decisions & Setup Guide

**Date:** March 31, 2026
**Status:** Confirmed

---

## ✅ Final Technology Choices

### 1. **Gemini Model**
```
Choice: gemini-2.0-flash
Reason: Fast inference, large context (128K), tool-using capability, cost-effective
```

### 2. **Web Search API**
```
Choice: Google Custom Search API
Cost: ~$5-10/month for 1000 queries/month
Setup Time: 15 minutes
```

**Setup Steps Summary:**
1. Create Google Cloud Project
2. Enable Custom Search API
3. Create API Key
4. Create Custom Search Engine (cx)
5. Store in `.env`: GOOGLE_SEARCH_API_KEY, GOOGLE_SEARCH_ENGINE_ID

### 3. **Code Execution**
```
Choice: E2B (Sandboxed Code Interpreter)
Cost: $0.05 per execution (~$50 for 1000 executions)
Latency: ~100ms
Languages: 50+ (Python, JavaScript, Go, Java, etc.)
Security: Production-grade sandbox
```

**Why E2B:**
- Built specifically for AI/agent use cases
- Lower latency than Replit (~100ms vs 200-500ms)
- Native output capture (stdout, errors)
- Better resource control (CPU, memory, timeout)
- Scales to 1000s/second

**Setup Steps Summary:**
1. Sign up at https://e2b.dev
2. Create API key
3. Install: `pip install e2b-code-interpreter`
4. Store in `.env`: E2B_API_KEY

### 4. **Authentication**
```
Choice: Simple JWT (no third-party auth)
Setup Time: 30 minutes
Token Expiry: 24 hours
Secret Key: Stored in .env file
```

**Why Simple JWT:**
- Fast to implement (matches hackathon timeline)
- No backend/frontend separation issues
- Easy to migrate to OAuth2 later
- Industry-standard format

**Setup Steps Summary:**
1. Install: `pip install pyjwt`
2. Create JWT utils functions (create_token, verify_token)
3. Add auth middleware to FastAPI
4. Store in `.env`: JWT_SECRET_KEY

---

## 🔧 Environment Variables Required

Create `.env` file in project root:

```bash
# Application
ENVIRONMENT=development
DEBUG=true
LOG_LEVEL=INFO

# Google Cloud
GOOGLE_CLOUD_PROJECT=interview-prep-ai
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account-key.json

# Gemini API
GEMINI_API_KEY=your-gemini-api-key
GEMINI_MODEL=gemini-2.0-flash

# Web Search
GOOGLE_SEARCH_API_KEY=your-custom-search-api-key
GOOGLE_SEARCH_ENGINE_ID=your-cx-id

# Code Execution (E2B)
E2B_API_KEY=sk_your-e2b-api-key

# Database (Firestore - will setup later)
FIRESTORE_PROJECT_ID=interview-prep-ai
FIRESTORE_DATABASE=production

# JWT Authentication
JWT_SECRET_KEY=your-secret-key-min-32-chars-unique-string
JWT_ALGORITHM=HS256
JWT_EXPIRY_HOURS=24

# Server
HOST=0.0.0.0
PORT=8000
```

---

## 🚀 Setup Checklist

### **Before Implementation Starts**

- [ ] Create Google Cloud Project
- [ ] Get Gemini API key
- [ ] Get Google Custom Search API key & cx ID
- [ ] Create E2B account & API key
- [ ] Create `.env` file with all credentials
- [ ] Test credentials work

### **Setup Commands**

```bash
# 1. Create project directory
mkdir -p interview-prep-agent
cd interview-prep-agent

# 2. Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# 3. Create .env file
cp .env.example .env
# Edit .env with your credentials

# 4. Install initial dependencies
pip install fastapi uvicorn pyjwt google-generativeai e2b-code-interpreter firebase-admin

# 5. Create project structure
mkdir -p src/{main,agents,auth,middleware,tools} tests
```

---

## 📋 Next Phase: Implementation Planning

Once credentials are ready, we will:

1. **Phase 1:** FastAPI scaffolding + JWT auth
2. **Phase 2:** Agent framework + base agent class
3. **Phase 3:** Implement each agent
4. **Phase 4:** Integrate tools (Gemini, Web Search, E2B)
5. **Phase 5:** Database (Firestore setup)
6. **Phase 6:** Testing & deployment

---

## 🎯 Ready to Start?

✅ **Confirmed decisions:**
- Model: gemini-2.0-flash
- Web Search: Google Custom Search API
- Code Execution: E2B
- Auth: Simple JWT

**Next Steps:**
1. Gather these API keys (15-30 minutes)
2. Create `.env` file
3. I'll start Phase 1: Project scaffolding
