# Agent Framework & Design

**Document:** `docs/agents/AGENT_FRAMEWORK.md`

---

## 1. Agent Architecture

### **Base Agent Structure**
```python
class Agent:
    """Base class for all specialized agents"""

    def __init__(self, name: str, tools: List[Tool], model: str = "gemini"):
        self.name = name
        self.tools = tools
        self.model = model
        self.logger = setup_logger(self.name)

    async def execute(self, task: Task) -> Result:
        """Execute a task using available tools"""
        pass

    async def think(self, context: Context) -> Decision:
        """Reason about what to do next"""
        pass
```

---

## 2. Agent Types & Roles

### **Type A: Reasoning Agents** (Planner, Evaluation)
- Heavy use of Gemini for reasoning
- Minimal external tool calls
- Return structured decisions/feedback
- Examples: Planner Agent, Evaluation Agent

### **Type B: Action Agents** (Study Plan, Mock Interview)
- Balance reasoning with tool calls
- Execute multiple tools in sequence
- Combine results into coherent output
- Examples: Study Plan Agent, Mock Interview Agent

### **Type C: Data Agents** (Progress Tracker)
- Minimal reasoning
- Heavy I/O with external systems
- CRUD operations on data
- Examples: Progress Tracker Agent

---

## 3. Tool Abstraction

### **Tool Interface**
```python
class Tool:
    """Base class for all tools"""

    def __init__(self, name: str, description: str):
        self.name = name
        self.description = description

    async def execute(self, **kwargs) -> ToolResult:
        """Execute the tool with given parameters"""
        pass

    def get_schema(self) -> dict:
        """Return JSON schema for tool parameters"""
        pass
```

### **Built-in Tools**

| Agent | Tool | Purpose |
|-------|------|---------|
| Study Plan | `generate_plan` | Call Gemini to generate study plan |
| Study Plan | `find_resources` | Search web for learning materials |
| Mock Interview | `generate_question` | Call Gemini to generate question |
| Mock Interview | `execute_code` | Run code in sandboxed environment |
| Evaluation | `score_answer` | Call Gemini to evaluate answer |
| Progress Tracker | `save_progress` | Write to Firestore |
| Progress Tracker | `get_progress` | Read from Firestore |

---

## 4. Agent Communication Protocol

### **Message Format**
```json
{
  "sender": "planner",
  "receiver": "study_plan",
  "type": "task|result|error|ack",
  "timestamp": "ISO 8601",
  "payload": {
    "task_id": "string",
    "data": "object"
  },
  "metadata": {
    "retry_count": 0,
    "timeout_seconds": 30
  }
}
```

### **Communication Flow**

```
Planner Agent (Sender)
    │
    ├─ Create Task Message
    ├─ Send to Study Plan Agent
    │
Study Plan Agent (Receiver)
    │
    ├─ Acknowledge receipt
    ├─ Execute task (call tools)
    ├─ Generate Result Message
    │
Planner Agent (Receiver)
    │
    ├─ Receive Result
    ├─ Aggregate with other results
    ├─ Return to API
```

---

## 5. Gemini Integration

### **Prompt Structure**

Each agent uses a structured prompt with:

```markdown
# System Role
[Agent's role and responsibilities]

# Context
[Current state and relevant information]

# Instructions
[Specific task and constraints]

# Output Format
[Expected JSON structure]

# Tools Available
[List of tools the agent can use]
```

### **Example: Study Plan Agent Prompt**
```markdown
# System Role
You are the Study Plan Agent. Your job is to create personalized study plans for interview preparation.

# Context
User Role: Python Backend Developer
Days Available: 5
Experience Level: Intermediate
Focus Areas: System Design, Databases

# Instructions
Create a 5-day study plan that:
1. Focuses on system design and databases
2. Is appropriate for intermediate developers
3. Includes realistic 2-3 hour daily commitments
4. Provides actionable learning objective for each day

# Output Format
Return a JSON object with structure:
{
  "days": [
    {
      "day_number": 1,
      "topic": "...",
      "objectives": [...],
      "estimated_hours": 2.5
    }
  ]
}

# Tools Available
- generate_plan: Call Gemini to reason about study plan
- find_resources: Find learning resources via web search
```

### **Token Management**
- System prompt: ~500 tokens
- Context: ~1000 tokens
- User input: ~500 tokens
- **Per request budget**: 4096 tokens (conservative)
- **Caching**: Cache prompts to reduce token usage (24h TTL)

---

## 6. Concurrency & Async

### **Async Agent Execution**
```python
async def execute_parallel_tasks(tasks: List[Task]) -> List[Result]:
    """Execute multiple tasks concurrently"""
    results = await asyncio.gather(
        study_plan_agent.execute(tasks[0]),
        mock_interview_agent.execute(tasks[1]),
        evaluation_agent.execute(tasks[2]),
        return_exceptions=True
    )
    return results
```

### **Timeout Handling**
- Per-agent timeout: 30 seconds
- Per-tool timeout: 20 seconds
- Fallback logic if timeout occurs

---

## 7. Error Handling in Agents

### **Error Categories**

| Category | Cause | Recovery |
|----------|-------|----------|
| Tool Error | External API failure | Retry with backoff |
| Validation Error | Invalid input | Return error to planner |
| Timeout | Long processing | Return partial result |
| Gemini Error | Model unavailable | Fallback to cached response |

### **Agent-Level Error Handling**
```python
async def execute_with_fallback(self, task: Task) -> Result:
    try:
        return await self.execute(task)
    except ToolTimeoutError:
        return self.get_cached_result(task)
    except GeminiRateLimitError:
        await asyncio.sleep(60)  # Backoff
        return await self.execute(task)
    except Exception as e:
        self.logger.error(f"Unhandled error: {e}")
        return self.create_error_result(e)
```

---

## 8. State Management

### **Session State** (Progress Tracker maintains)
```json
{
  "session_id": "string",
  "user_id": "string",
  "type": "study-plan|mock-interview",
  "created_at": "ISO 8601",
  "current_task": "string",
  "context": {
    "study_plan_id": "string",
    "interview_topic": "string",
    "questions_asked": ["q1", "q2"],
    "answers_submitted": ["a1", "a2"],
    "scores": [85, 78]
  }
}
```

### **Context Passing**
- Each agent receives session context
- Agents update their portions of context
- Planner aggregates updates
- Progress Tracker persists context

---

## 9. Validation & Safety

### **Input Validation**
- All task inputs validated before sending to agent
- Type checking for parameters
- Range validation for numeric values
- Sanitization of text inputs

### **Output Validation**
- All agent outputs validated against schema
- Type checking of results
- Rejection of invalid results
- Logging of validation failures

### **Safety Boundaries**
- Agents cannot call external URLs (except through designated tools)
- Agents cannot access file system
- Agents cannot fork processes
- All dangerous operations go through controlled tools

---

## 10. Logging & Observability

### **Agent Logging**
```python
self.logger.info(f"Agent {self.name} started task {task_id}")
self.logger.debug(f"Using tools: {tool_names}")
self.logger.info(f"Task {task_id} completed in {elapsed_ms}ms")
self.logger.error(f"Task {task_id} failed: {error}")
```

### **Trace Events**
- Agent started
- Tool called
- Tool completed
- Task completed
- Error occurred

