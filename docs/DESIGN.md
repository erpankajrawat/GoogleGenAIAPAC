# 🎯 Interview Preparation Agent — Design & Architecture Document

**Version:** 1.0
**Last Updated:** March 31, 2026
**Status:** Design Phase

---

## 📑 Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Overview](#system-overview)
3. [Documentation Structure](#documentation-structure)
4. [Key Design Principles](#key-design-principles)
5. [Technology Stack](#technology-stack)
6. [Quick Links to Detailed Docs](#quick-links)

---

## Executive Summary

The **Interview Preparation Agent** is a goal-oriented, multi-agent AI system powered by Google Gemini. It provides end-to-end interview preparation through:

- **Personalized study plan generation**
- **Interactive mock interviews** (behavioral & technical)
- **Intelligent evaluation** with actionable feedback
- **Progress tracking** with performance analytics

**Core Design Philosophy:**
- Single `/execute-goal` endpoint for flexibility
- Modular, tool-using agents with clear responsibilities
- Centralized orchestration via Planner Agent
- Scalable for future enhancements

---

## System Overview

### Architecture Flow

```
┌─────────────┐
│   User      │
│   (Web/API) │
└──────┬──────┘
       │
       ↓
┌──────────────────────────────┐
│  FastAPI /execute-goal       │
│  - Input validation          │
│  - Auth & session mgmt       │
└──────────────┬───────────────┘
               │
               ↓
┌──────────────────────────────────────────┐
│        Planner Agent (Orchestrator)      │
│  - Decomposes goals into tasks           │
│  - Routes to appropriate agents          │
└──────┬───────────┬────────────┬──────────┘
       │           │            │
   ┌───↓───┐   ┌───↓────┐   ┌──↓──────┐
   │Study  │   │ Mock   │   │Evaluation│
   │Plan   │   │Interview│   │Agent    │
   │Agent  │   │Agent   │   └─────────┘
   └───┬───┘   └───┬────┘
       │           │
       ↓           ↓
   ┌──────────────────────────┐
   │ External Tools           │
   │ - Web Search API         │
   │ - Code Execution API     │
   └──────────────────────────┘
       │
       ↓
   ┌──────────────────────────┐
   │ Progress Tracker Agent   │
   │ (Database I/O)           │
   └──────────────────────────┘
       │
       ↓
   ┌──────────────────────────┐
   │ Firestore/AlloyDB        │
   │ (Persistent Storage)     │
   └──────────────────────────┘
```

---

## Documentation Structure

```
docs/
├── DESIGN.md                          ← You are here
├── architecture/
│   ├── SYSTEM_DESIGN.md              ← Detailed system architecture
│   ├── COMPONENT_INTERACTION.md      ← Agent interactions & flow
│   └── SEQUENCE_DIAGRAMS.md          ← Key workflows (visual)
├── api/
│   ├── API_SPECIFICATION.md          ← /execute-goal endpoint details
│   ├── GOAL_TYPES.md                 ← All supported goal types
│   └── ERROR_HANDLING.md             ← Error codes & strategies
├── agents/
│   ├── AGENT_FRAMEWORK.md            ← Base agent architecture
│   ├── PLANNER_AGENT.md              ← Orchestration logic
│   ├── STUDY_PLAN_AGENT.md           ← Study plan generation
│   ├── MOCK_INTERVIEW_AGENT.md       ← Interview management
│   ├── EVALUATION_AGENT.md           ← Answer evaluation
│   └── PROGRESS_TRACKER_AGENT.md     ← Data persistence
├── data/
│   ├── DATA_MODELS.md                ← Request/response schemas
│   ├── DATABASE_SCHEMA.md            ← Firestore/AlloyDB structure
│   └── USER_SESSION.md               ← Session & context management
├── deployment/
│   ├── CLOUD_RUN_SETUP.md            ← Container & deployment
│   ├── ENVIRONMENT_CONFIG.md         ← Environment variables
│   └── SCALING_STRATEGY.md           ← Performance & scaling
├── integration/
│   ├── GEMINI_API_SETUP.md           ← Google Gemini integration
│   ├── WEB_SEARCH_TOOL.md            ← Web search API
│   └── CODE_EXECUTION_TOOL.md        ← Sandboxed code execution
└── error-handling/
    ├── FAILURE_SCENARIOS.md          ← Common failures & recovery
    ├── LOGGING_STRATEGY.md           ← Observability & debugging
    └── RESILIENCE.md                 ← Rate limiting, retries, etc.
```

---

## Key Design Principles

### 1. **Goal-Oriented Design**
- Single `/execute-goal` endpoint supports any goal type
- Extensible without API contract changes
- Client sends natural language abstracted as structured goals

### 2. **Modular Agent Architecture**
- Each agent has one clear responsibility
- Agents communicate through well-defined interfaces
- Tools are isolated from business logic

### 3. **Centralized Orchestration**
- Planner Agent acts as the "brain"
- Prevents chaotic, uncoordinated agent interactions
- Creates auditable, traceable execution plans

### 4. **Tool-Based Extensibility**
- Agents use tools to accomplish tasks
- New tools can be added without changing agent logic
- Standardized tool interface

### 5. **Separation of Concerns**
- **API Layer**: Request handling, validation, auth
- **Agent Layer**: Business logic & reasoning
- **Tool Layer**: External integrations
- **Data Layer**: Persistence

### 6. **Scalability First**
- Stateless agents for horizontal scaling
- Async operations where possible
- Firestore for distributed, serverless database

---

## Technology Stack

| Component | Technology | Justification |
|-----------|-----------|---------------|
| **Language** | Python 3.10+ | Google Gemini SDK, FastAPI support |
| **API Framework** | FastAPI | Async, built-in validation, OpenAPI docs |
| **AI Model** | Google Gemini | Multi-modal, tool-using, reasoning |
| **Database** | Firestore (primary) | Serverless, auto-scaling, real-time |
| **Deployment** | Google Cloud Run | Container-based, auto-scaling, pay-per-use |
| **Auth** | OAuth2 + Firebase Auth | Google Cloud native, secure |
| **Web Search** | Google Custom Search API | Reliable, company-specific prep |
| **Code Execution** | Replit API / E2B | Sandboxed, secure, multi-language |
| **Logging** | Cloud Logging | Centralized, integrated with Cloud Run |
| **Monitoring** | Cloud Monitoring | Performance tracking, alerting |

---

## Quick Links

### 📐 Architecture & Design
→ [**System Architecture**](./architecture/SYSTEM_DESIGN.md)
→ [**Component Interactions**](./architecture/COMPONENT_INTERACTION.md)
→ [**Sequence Diagrams**](./architecture/SEQUENCE_DIAGRAMS.md)

### 🔌 API Specification
→ [**API Specification**](./api/API_SPECIFICATION.md)
→ [**Goal Types**](./api/GOAL_TYPES.md)
→ [**Error Handling**](./api/ERROR_HANDLING.md)

### 🧠 Agent Design
→ [**Agent Framework**](./agents/AGENT_FRAMEWORK.md)
→ [**All Agents**](./agents/) (subdirectory)

### 💾 Data Models
→ [**Request/Response Schemas**](./data/DATA_MODELS.md)
→ [**Database Schema**](./data/DATABASE_SCHEMA.md)
→ [**User Sessions**](./data/USER_SESSION.md)

### 🚀 Deployment
→ [**Cloud Run Setup**](./deployment/CLOUD_RUN_SETUP.md)
→ [**Environment Config**](./deployment/ENVIRONMENT_CONFIG.md)
→ [**Scaling Strategy**](./deployment/SCALING_STRATEGY.md)

### 🔗 Integrations
→ [**Gemini API**](./integration/GEMINI_API_SETUP.md)
→ [**Web Search**](./integration/WEB_SEARCH_TOOL.md)
→ [**Code Execution**](./integration/CODE_EXECUTION_TOOL.md)

### ⚠️ Error Handling & Resilience
→ [**Failure Scenarios**](./error-handling/FAILURE_SCENARIOS.md)
→ [**Logging Strategy**](./error-handling/LOGGING_STRATEGY.md)
→ [**Resilience Patterns**](./error-handling/RESILIENCE.md)

---

## Next Steps

1. **Review System Architecture** → Understand component interactions
2. **Validate Agent Design** → Confirm agent responsibilities
3. **Finalize Data Models** → Define database schema
4. **Approve API Specification** → Lock in endpoint contracts
5. **Plan Implementation** → Create sprint backlog

---

**Questions or clarifications needed?** Each linked document provides detailed specifications for implementation.
