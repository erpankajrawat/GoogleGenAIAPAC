# System Architecture — Detailed Design

**Document:** `docs/architecture/SYSTEM_DESIGN.md`

---

## 1. High-Level Architecture Layers

```
┌────────────────────────────────────────────────────────────┐
│                    Presentation Layer                       │
│              (Web, Mobile, CLI, API Clients)               │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│                    API Layer                              │
│  FastAPI: /execute-goal endpoint                          │
│  - Request validation & parsing                           │
│  - Authentication & authorization                         │
│  - Session management                                     │
│  - Response formatting                                    │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│              Multi-Agent Orchestration                     │
│  Planner Agent (Central Coordinator)                      │
│  ├─ Decomposes goals into tasks                           │
│  ├─ Routes to specialized agents                          │
│  ├─ Aggregates results                                    │
│  └─ Error recovery                                        │
└────────┬──────────────┬──────────────┬─────────────────────┘
         │              │              │
    ┌────▼────┐    ┌───▼────┐    ┌──▼──────┐
    │Study    │    │ Mock   │    │Evaluation│
    │Plan     │    │Interview│   │Agent     │
    │Agent    │    │Agent   │    └──────────┘
    └────┬────┘    └───┬────┘
         │             │
    ┌────▼──────────────▼──────┐
    │ Progress Tracker Agent   │
    │ (Database Persistence)   │
    └────┬──────────────┬──────┘
         │              │
    ┌────▼────┐    ┌───▼────────┐
    │Gemini   │    │Firestore   │
    │API      │    │Database    │
    └─────────┘    └────────────┘
```

---

## 2. Component Responsibilities

### **API Layer**
- **Purpose**: Gateway between users and the agent system
- **Framework**: FastAPI (async, auto-validation, OpenAPI docs)
- **Responsibilities**:
  - Parse and validate incoming goals
  - Authenticate users (OAuth2 / Firebase)
  - Manage user sessions
  - Call Planner Agent with structured goal
  - Return formatted responses

**Example Flow:**
```python
POST /execute-goal
{
  "user_id": "user123",
  "goal": "create-study-plan",
  "params": {
    "role": "Python Backend Developer",
    "days": 5,
    "experience_level": "intermediate"
  }
}
```

---

### **Planner Agent**
- **Purpose**: Central orchestrator and decision maker
- **Responsibility**:
  - Receive user goal + context
  - Decompose into logical sequence of tasks
  - Route tasks to appropriate agents
  - Collect results and aggregate
  - Handle errors and fallbacks
  - Return final response

**Planner Decision Logic:**
```
Goal: "create-study-plan"
├─ Task 1: Call Study Plan Agent → generate_plan()
├─ Task 2: Call Web Search (via Study Plan Agent) → find_resources()
├─ Task 3: Call Progress Tracker → save_plan()
└─ Task 4: Aggregate & return results

Goal: "start-mock-interview"
├─ Task 1: Call Mock Interview Agent → generate_question()
├─ Task 2: Collect user answer
├─ Task 3: Call Evaluation Agent → score_answer()
├─ Task 4: Call Progress Tracker → save_attempt()
└─ Task 5: Return feedback + next question
```

---

### **Study Plan Agent**
- **Purpose**: Generate structured learning schedules
- **Tools Used**:
  - `generate_plan(role, days, experience_level)` → Calls Gemini
  - `find_resources(topic)` → Calls Web Search API
- **Output**: Day-wise study plan with resources
- **Data Stored**: Via Progress Tracker Agent

**Example Output:**
```json
{
  "plan_id": "plan-abc123",
  "role": "Python Backend Developer",
  "duration_days": 5,
  "days": [
    {
      "day": 1,
      "topic": "System Design Fundamentals",
      "resources": ["URL1", "URL2"],
      "practice": ["Design Question 1", "Design Question 2"]
    }
  ]
}
```

---

### **Mock Interview Agent**
- **Purpose**: Manage interactive mock interview sessions
- **Tools Used**:
  - `generate_question(topic, difficulty)` → Calls Gemini
  - `execute_code(code, language)` → Calls Code Execution API
- **Flow**:
  1. Generate question based on topic/difficulty
  2. Present to user
  3. Collect answer (text/code)
  4. For code: execute & test
  5. Send to Evaluation Agent
  6. Get feedback
  7. Generate next question or conclude

**State Management:** Maintains interview context (questions asked, answers given, score)

---

### **Evaluation Agent**
- **Purpose**: Score and provide feedback on answers
- **Tools Used**:
  - `score_answer(question, answer, rubric)` → Calls Gemini
- **Inputs**: Question, user answer, predefined rubric
- **Output**: Score (0-100), feedback, improvement areas

**Rubric Categories:**
- Correctness (40%)
- Completeness (30%)
- Clarity (20%)
- Edge case handling (10%)

---

### **Progress Tracker Agent**
- **Purpose**: Persist user data and retrieve historical context
- **Tools Used**:
  - `save_progress(user_id, session_data)` → Writes to Firestore
  - `get_progress(user_id)` → Reads from Firestore
- **Data Tracked**:
  - Study plans completed
  - Mock interview attempts
  - Scores over time
  - Topics mastered
  - Current session state

---

## 3. Data Flow Diagrams

### **Flow: Create Study Plan**
```
User Request
    ↓
API Validation
    ↓
Planner Agent
    ├─ Calls Study Plan Agent
    │   └─ generate_plan(role, days)
    │       └─ Calls Gemini w/ structured prompt
    │           └─ Returns day-wise plan
    │
    ├─ For each day topic:
    │   └─ Calls Web Search (optional)
    │       └─ find_resources(topic)
    │           └─ Returns relevant articles/tutorials
    │
    ├─ Calls Progress Tracker Agent
    │   └─ save_plan(user_id, plan_data)
    │       └─ Writes to Firestore
    │
    └─ Aggregates & formats response
        ↓
    API Response (JSON)
```

### **Flow: Mock Interview Q&A**
```
Setup Interview Session
    ↓
Loop (until user ends):
    ├─ Planner calls Mock Interview Agent
    │   └─ generate_question(topic, difficulty)
    │       └─ Calls Gemini
    │
    ├─ Present question to user
    │
    ├─ Collect answer (text/code)
    │
    ├─ If coding question:
    │   └─ Mock Interview Agent
    │       └─ execute_code(code, language)
    │           └─ Calls Code Execution API
    │               └─ Returns execution result
    │
    ├─ Planner calls Evaluation Agent
    │   └─ score_answer(question, answer, rubric)
    │       └─ Calls Gemini w/ rubric
    │           └─ Returns score + feedback
    │
    ├─ Planner calls Progress Tracker
    │   └─ save_attempt(user_id, attempt_data)
    │       └─ Writes to Firestore
    │
    └─ Return feedback to user
        ↓
    User decides to continue or end
```

---

## 4. Error Handling & Recovery

### **Failure Points**

| Component | Failure | Impact | Recovery |
|-----------|---------|--------|----------|
| Gemini API | Rate limited | Can't generate questions | Retry with exponential backoff |
| Gemini API | Timeout | Long execution time | Serve cached response + alert |
| Web Search API | Down | Can't find resources | Use fallback curated list |
| Code Execution API | Crash | Can't test code | Inform user, skip auto-test |
| Firestore | Connection error | Can't save progress | Queue locally, retry async |
| Database | Quota exceeded | Persistent storage fails | Alert ops, fallback to cache |

### **Recovery Strategies**
- **Circuit Breaker**: Disable failing service temporarily
- **Fallbacks**: Serve cached/default responses
- **Retry Logic**: Exponential backoff for transient failures
- **Graceful Degradation**: Continue with limited functionality
- **Async Recovery**: Queue failed operations for later retry

---

## 5. Security & Isolation

### **API Security**
- OAuth2 authentication
- Rate limiting per user (100 req/min)
- Input validation & sanitization
- HTTPS only

### **Agent Isolation**
- Agents run in isolated processes/containers
- No direct file system access
- Limited external network calls
- Timeouts to prevent infinite loops

### **Data Security**
- User data encrypted at rest (Firestore encryption)
- User data encrypted in transit (TLS)
- No sensitive data in logs
- Role-based access control for database

### **Code Execution Safety**
- Sandboxed via Code Execution API (E2B / Replit)
- Resource limits (CPU, memory, time)
- Network isolation
- File system isolation

---

## 6. Performance Considerations

### **Optimization Strategies**
1. **Caching**
   - Cache Gemini prompts & responses (24h TTL)
   - Cache web search results (1h TTL)
   - Cache study plans by role/duration

2. **Async Operations**
   - Non-blocking API responses
   - Background jobs for heavy computations
   - Streaming responses where possible

3. **Batching**
   - Batch database writes when possible
   - Combine multiple API calls into single request

4. **Lazy Loading**
   - Load user history only when needed
   - Progressive data loading in interviews

### **Scalability**
- Stateless agents → horizontal scaling
- Firestore auto-scales reads/writes
- Cloud Run scales to zero
- Distributed tracing for monitoring

---

## 7. Deployment Architecture

```
┌──────────────────────────────────────────────┐
│         Cloud Load Balancer (HTTPS)          │
└──────────────────┬───────────────────────────┘
                   │
┌──────────────────▼───────────────────────────┐
│         Cloud Run (Auto-scaling)             │
│  ├─ Container 1 (FastAPI + Agents)          │
│  ├─ Container 2 (FastAPI + Agents)          │
│  └─ Container N (FastAPI + Agents)          │
└──────────────────┬───────────────────────────┘
                   │
       ┌───────────┼───────────┐
       │           │           │
   ┌───▼────┐  ┌──▼────┐  ┌──▼────┐
   │Firestore│  │Cloud  │  │Cloud  │
   │Database  │  │Logging│  │Trace  │
   └─────────┘  └────────┘  └───────┘
```

---

## 8. Database Connection

- **Type**: Firestore (real-time NoSQL)
- **Collections**:
  - users
  - study_plans
  - mock_interviews
  - user_progress
  - evaluation_rubrics
- **Read/Write**: Through Progress Tracker Agent
- **Replication**: Multi-region for high availability

---

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Latency** | < 2s | Study plan generation |
| **Throughput** | 1000+ req/min | Peak interview sessions |
| **Availability** | 99.9% | Production service |
| **Response Time** | < 5s | All endpoints (including AI inference) |
| **Storage** | Unlimited scalability | User history grows over time |
| **Concurrent Users** | 10,000+ | Horizontal Cloud Run scaling |

