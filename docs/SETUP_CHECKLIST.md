# Setup Checklist & Quick Start

**Document:** `docs/SETUP_CHECKLIST.md`

---

## ✅ Phase 1: Get API Keys & Credentials (15-30 minutes)

### **1. Gemini API Key**
- [ ] Go to https://console.cloud.google.com
- [ ] Create new project: "interview-prep-ai"
- [ ] Search "Generative AI API" → Enable
- [ ] Go to Credentials → Create API Key
- [ ] Copy key (format: `AIzaSy...`)
- [ ] Save to `.env`: `GEMINI_API_KEY=AIzaSy...`

**✓ Docs:** `docs/integration/GEMINI_API_SETUP.md`

---

### **2. Google Custom Search API Key & Engine ID**
- [ ] In Google Cloud Console: Search "Custom Search API" → Enable
- [ ] Go to Credentials → Create API Key
- [ ] Copy key
- [ ] Go to https://programmablesearchengine.google.com/
- [ ] Click "Create" → Name: "interview-prep-search"
- [ ] Leave "Websites to search" empty (search all web)
- [ ] Copy Search Engine ID (cx parameter)
- [ ] Save to `.env`:
  - `GOOGLE_SEARCH_API_KEY=AIzaSy...`
  - `GOOGLE_SEARCH_ENGINE_ID=1234567890abc:xyz`

**✓ Docs:** `docs/integration/WEB_SEARCH_TOOL.md`

---

### **3. E2B Code Execution API Key**
- [ ] Go to https://e2b.dev
- [ ] Sign up (free account, 10 executions/month free)
- [ ] Dashboard → API Keys
- [ ] Create new key
- [ ] Copy key (format: `sk_...`)
- [ ] Save to `.env`: `E2B_API_KEY=sk_...`

**✓ Docs:** `docs/integration/CODE_EXECUTION_TOOL.md`

---

### **4. JWT Secret Key**
- [ ] Run: `python -c "import secrets; print(secrets.token_urlsafe(32))"`
- [ ] Copy output (random 32+ char string)
- [ ] Save to `.env`: `JWT_SECRET_KEY=<output>`

**✓ Docs:** `docs/integration/JWT_AUTHENTICATION.md`

---

## 📝 Phase 2: Create Environment File (5 minutes)

### **Create `.env` File**

Create file in project root: `/d/GoogleAPAC/GoogleGenAIAPAC/.env`

```bash
# ============ Application ============
ENVIRONMENT=development
DEBUG=true
LOG_LEVEL=INFO

# ============ Google Cloud ============
GOOGLE_CLOUD_PROJECT=interview-prep-ai
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account-key.json

# ============ Gemini API ============
GEMINI_API_KEY=AIzaSyD...
GEMINI_MODEL=gemini-2.0-flash

# ============ Web Search ============
GOOGLE_SEARCH_API_KEY=AIzaSy...
GOOGLE_SEARCH_ENGINE_ID=1234567890abc:xyz

# ============ Code Execution (E2B) ============
E2B_API_KEY=sk_...

# ============ JWT Authentication ============
JWT_SECRET_KEY=your-random-32-char-string-here
JWT_ALGORITHM=HS256
JWT_EXPIRY_HOURS=24

# ============ Database (Firestore - setup later) ============
FIRESTORE_PROJECT_ID=interview-prep-ai
FIRESTORE_DATABASE=production

# ============ Server ============
HOST=0.0.0.0
PORT=8000
```

---

## 📦 Phase 3: Install Dependencies (5 minutes)

### **Setup Virtual Environment**

```bash
# Navigate to project
cd /d/GoogleAPAC/GoogleGenAIAPAC

# Create virtual environment
python -m venv venv

# Activate (Windows)
venv\Scripts\activate

# Activate (Linux/Mac)
source venv/bin/activate
```

### **Install Packages**

Create `requirements.txt`:

```
# Web Framework
fastapi==0.104.1
uvicorn==0.24.0

# Authentication
pyjwt==2.8.1

# Google APIs
google-generativeai==0.3.0
google-api-python-client==2.104.0

# Code Execution
e2b-code-interpreter==0.0.13

# Database (Firestore - setup later)
firebase-admin==6.2.0

# Environment
python-dotenv==1.0.0

# Utilities
pydantic==2.5.0
pydantic-settings==2.1.0
requests==2.31.0

# Testing
pytest==7.4.3
pytest-asyncio==0.21.1
httpx==0.25.1

# Logging
python-json-logger==2.0.7
```

**Install:**

```bash
pip install -r requirements.txt
```

### **Verify Installations**

```bash
# Test each package
python -c "from fastapi import FastAPI; print('✓ FastAPI')"
python -c "from google import generativeai; print('✓ Gemini SDK')"
python -c "from e2b_code_interpreter import CodeInterpreter; print('✓ E2B')"
python -c "import jwt; print('✓ PyJWT')"
python -c "from googleapiclient.discovery import build; print('✓ Google API Client')"
```

---

## 🏗️ Phase 4: Test Credentials (10 minutes)

### **Test Gemini API**

```python
# file: test_gemini.py
import os
os.environ['GEMINI_API_KEY'] = 'YOUR_KEY_HERE'

from google import generativeai as genai

genai.configure(api_key=os.env['GEMINI_API_KEY'])
model = genai.GenerativeModel('gemini-2.0-flash')
response = model.generate_content("Test message")
print("✓ Gemini API works:", response.text[:50])
```

**Run:**
```bash
python test_gemini.py
```

### **Test Google Search API**

```python
# file: test_search.py
import os
from googleapiclient.discovery import build

api_key = os.getenv('GOOGLE_SEARCH_API_KEY')
cx_id = os.getenv('GOOGLE_SEARCH_ENGINE_ID')

service = build("customsearch", "v1", developerKey=api_key)
results = service.cse().list(q="test", cx=cx_id, num=1).execute()

print("✓ Search API works. Results:", len(results.get('items', [])))
```

**Run:**
```bash
python test_search.py
```

### **Test E2B Code Execution**

```python
# file: test_e2b.py
import os
from e2b_code_interpreter import CodeInterpreter

api_key = os.getenv('E2B_API_KEY')

with CodeInterpreter(api_key=api_key) as interpreter:
    result = interpreter.notebook.exec_cell(
        "print('Hello from E2B')"
    )
    print("✓ E2B works. Output:", result.text)
```

**Run:**
```bash
python test_e2b.py
```

### **Test JWT**

```python
# file: test_jwt.py
import os
import sys
sys.path.insert(0, '/d/GoogleAPAC/GoogleGenAIAPAC')

from src.auth.jwt_utils import JWTHandler

token = JWTHandler.create_token("test-user", email="test@example.com")
result = JWTHandler.verify_token(token)

print("✓ JWT works")
print(f"  Token: {token[:30]}...")
print(f"  Verified: {result['valid']}")
print(f"  User: {result['user_id']}")
```

**Run:**
```bash
python test_jwt.py
```

---

## 🚀 Phase 5: Project Structure Setup (5 minutes)

### **Create Directory Structure**

```bash
cd /d/GoogleAPAC/GoogleGenAIAPAC

# Create source directories
mkdir -p src/{auth,config,middleware,models,agents,tools,database}
mkdir -p tests

# Create initial files
touch src/__init__.py
touch src/config/__init__.py
touch src/auth/__init__.py
touch src/agents/__init__.py
touch src/tools/__init__.py
touch src/models/__init__.py
touch tests/__init__.py
```

### **Resulting Structure**

```
GoogleGenAIAPAC/
├── .env                          # Your credentials ⚠️
├── .gitignore
├── requirements.txt              # Dependencies
├── README.md
├── venv/                         # Virtual environment
│
├── docs/                         # 📚 All documentation
│   ├── DESIGN.md
│   ├── DOCUMENTATION_INDEX.md
│   ├── IMPLEMENTATION_DECISIONS.md
│   ├── SETUP_CHECKLIST.md (this file)
│   ├── architecture/
│   ├── api/
│   ├── agents/
│   ├── data/
│   ├── deployment/
│   ├── integration/
│   └── error-handling/
│
├── src/                          # 🔧 Source code
│   ├── __init__.py
│   ├── main.py                  # FastAPI app
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── jwt_utils.py         # JWT token functions
│   │   └── dependencies.py      # FastAPI dependency
│   ├── config/
│   │   ├── __init__.py
│   │   ├── auth_config.py
│   │   ├── gemini_config.py
│   │   └── web_search_config.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── auth.py              # Auth models
│   │   └── goals.py             # Goal models
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── base_agent.py        # Base agent class
│   │   ├── planner_agent.py
│   │   ├── study_plan_agent.py
│   │   ├── mock_interview_agent.py
│   │   ├── evaluation_agent.py
│   │   └── progress_tracker_agent.py
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── gemini_tool.py
│   │   ├── web_search_tool.py
│   │   └── code_execution_tool.py
│   ├── database/
│   │   ├── __init__.py
│   │   └── firestore_client.py  # Database utilities
│   └── middleware/
│       ├── __init__.py
│       └── logging.py           # Logging setup
│
└── tests/                        # ✅ Tests
    ├── __init__.py
    ├── test_jwt_auth.py
    ├── test_gemini_integration.py
    ├── test_web_search_integration.py
    └── test_e2b_integration.py
```

---

## ✨ Phase 6: Quick Start - Create Hello World API (10 minutes)

### **Create `src/main.py`**

```python
from fastapi import FastAPI, Depends
from fastapi.responses import JSONResponse
from src.auth.dependencies import get_current_user
from src.models.auth import LoginRequest, LoginResponse
from src.auth.jwt_utils import JWTHandler

app = FastAPI(
    title="Interview Prep Agent API",
    version="0.1.0",
    description="AI-powered interview preparation system"
)

# ============ Health Check ============

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {"status": "healthy"}

# ============ Auth Endpoints ============

@app.post("/login", response_model=LoginResponse)
async def login(request: LoginRequest):
    """User login"""
    token = JWTHandler.create_token(
        user_id=request.user_id,
        email=request.email
    )
    return LoginResponse(access_token=token)

@app.get("/me")
async def get_user_info(user = Depends(get_current_user)):
    """Get current user"""
    return {
        "user_id": user["user_id"],
        "email": user["email"],
        "role": user["role"]
    }

# ============ Main Endpoint ============

@app.post("/execute-goal")
async def execute_goal(
    goal: str,
    user = Depends(get_current_user)
):
    """Execute a goal (protected)"""
    return {
        "status": "success",
        "user_id": user["user_id"],
        "goal": goal,
        "message": "Goal processing started (more logic coming)"
    }


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### **Test the API**

```bash
# Start server
python -m uvicorn src.main:app --reload

# In another terminal:

# 1. Health check
curl http://localhost:8000/health

# 2. Login
curl -X POST http://localhost:8000/login \
  -H "Content-Type: application/json" \
  -d '{"user_id": "user-123", "email": "user@example.com"}'

# Copy the token from response and use:

# 3. Protected endpoint
curl -X POST http://localhost:8000/execute-goal \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"goal": "create-study-plan"}'
```

---

## 📚 Documentation Files Created

**Total:** 12 comprehensive documents (5,207+ lines)

### **Core Design Documents**
- ✅ `DESIGN.md` (Main entry point)
- ✅ `DOCUMENTATION_INDEX.md` (Complete index)
- ✅ `IMPLEMENTATION_DECISIONS.md` (Tech choices)
- ✅ `SETUP_CHECKLIST.md` (This file!)

### **Architecture**
- ✅ `architecture/SYSTEM_DESIGN.md` (Layers, flows, security)

### **API**
- ✅ `api/API_SPECIFICATION.md` (Endpoints, all goal types)

### **Agents**
- ✅ `agents/AGENT_FRAMEWORK.md` (Base architecture)

### **Data**
- ✅ `data/DATA_MODELS.md` (Pydantic models)
- ✅ `data/DATABASE_SCHEMA.md` (Firestore collections)

### **Deployment**
- ✅ `deployment/CLOUD_RUN_SETUP.md` (Container, CI/CD)

### **Integrations**
- ✅ `integration/GEMINI_API_SETUP.md` (Gemini with prompts)
- ✅ `integration/WEB_SEARCH_TOOL.md` (Google Search API)
- ✅ `integration/CODE_EXECUTION_TOOL.md` (E2B integration)
- ✅ `integration/JWT_AUTHENTICATION.md` (JWT tokens)

---

## 🎯 Quick Navigation

**Want to understand:**
- ❓ What is the system? → `DESIGN.md`
- ❓ How do components interact? → `architecture/SYSTEM_DESIGN.md`
- ❓ What does the API look like? → `api/API_SPECIFICATION.md`
- ❓ How are agents built? → `agents/AGENT_FRAMEWORK.md`
- ❓ How to set up Gemini? → `integration/GEMINI_API_SETUP.md`
- ❓ How to set up search? → `integration/WEB_SEARCH_TOOL.md`
- ❓ How to execute code? → `integration/CODE_EXECUTION_TOOL.md`
- ❓ How to implement auth? → `integration/JWT_AUTHENTICATION.md`
- ❓ Database schema? → `data/DATABASE_SCHEMA.md`
- ❓ Deploy to Cloud Run? → `deployment/CLOUD_RUN_SETUP.md`

---

## 📋 Next Steps After Setup

### **Phase 7: Implement Core Backend**
1. ✅ Setup complete
2. → Build base Agent class
3. → Implement Planner Agent
4. → Implement Study Plan Agent
5. → Implement Mock Interview Agent
6. → Implement Evaluation Agent
7. → Implement Progress Tracker Agent

### **Phase 8: Integrate Tools**
- Connect Gemini API
- Connect Web Search API
- Connect E2B Code Execution
- Connect Firestore Database

### **Phase 9: Testing**
- Unit tests for each agent
- Integration tests for agent interactions
- End-to-end API tests

### **Phase 10: Deploy**
- Build Docker container
- Deploy to Cloud Run
- Monitor & optimize

---

## ✅ Verification Checklist

Before moving to Phase 7, verify:

```
Setup Checklist:
☐ Gemini API key working
☐ Google Search API key working
☐ E2B API key working
☐ JWT authentication tested
☐ All dependencies installed
☐ Project structure created
☐ Hello World API running
☐ All documentation reviewed
```

**If all checked ✅ → Ready for Phase 7: Core Backend Implementation**

---

## 🆘 Troubleshooting

**Issue:** "API key not found"
- Solution: Check that `.env` file exists in project root and has correct values

**Issue:** "ModuleNotFoundError: No module named 'src'"
- Solution: Run from project root directory: `/d/GoogleAPAC/GoogleGenAIAPAC`

**Issue:** "JWT token expired"
- Solution: Get a new token via `/login` endpoint

**Issue:** "Search returns no results"
- Solution: Custom Search Engine might need to be configured to search all websites

---

## 📞 Questions?

Refer to the respective documentation files for:
- API details → `api/API_SPECIFICATION.md`
- Agent architecture → `agents/AGENT_FRAMEWORK.md`
- Setup issues → Each integration guide
- Deployment → `deployment/CLOUD_RUN_SETUP.md`
