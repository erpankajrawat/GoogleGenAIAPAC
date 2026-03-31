# Data Models & Database Schema

**Document:** `docs/data/DATA_MODELS.md`

---

## 1. Request/Response Data Models

### **ExecuteGoal Request**
```python
from pydantic import BaseModel, Field
from typing import Optional, Dict, Any

class ExecuteGoalRequest(BaseModel):
    """Main API request model"""
    user_id: str = Field(..., min_length=1, max_length=256)
    goal: str = Field(..., description="Goal type to execute")
    params: Dict[str, Any] = Field(..., description="Goal-specific parameters")
    context: Optional[Dict[str, Any]] = Field(
        None, description="Optional context to pass to agents"
    )

    class Config:
        example = {
            "user_id": "user-123",
            "goal": "create-study-plan",
            "params": {
                "role": "Python Backend Developer",
                "days": 5,
                "experience_level": "intermediate"
            }
        }
```

### **ExecuteGoal Response**
```python
class ExecuteGoalResponse(BaseModel):
    """Main API response model"""
    status: str = Field(..., pattern="^(success|error)$")
    data: Optional[Dict[str, Any]] = None
    error: Optional[Dict[str, Any]] = None
    metadata: Dict[str, Any] = Field(
        default_factory=dict,
        description="Execution metadata"
    )

    class Config:
        example = {
            "status": "success",
            "data": {
                "result": {...},
                "execution_id": "exec-123"
            },
            "metadata": {
                "processing_time_ms": 1234
            }
        }
```

---

## 2. Domain Models

### **Study Plan Model**
```python
class StudyPlanDay(BaseModel):
    day_number: int
    topic: str
    description: str
    learning_objectives: List[str]
    resources: List[Resource]
    practice_questions: List[str]
    estimated_hours: float

class Resource(BaseModel):
    title: str
    url: str
    type: str  # "article", "video", "interactive"
    difficulty: str  # "beginner", "intermediate", "advanced"

class StudyPlan(BaseModel):
    plan_id: str
    user_id: str
    role: str
    experience_level: str
    days: List[StudyPlanDay]
    total_hours: float
    created_at: datetime
    updated_at: datetime
    status: str  # "active", "completed", "archived"
