# API Specification — Complete Endpoint Design

**Document:** `docs/api/API_SPECIFICATION.md`

---

## 1. Overview

### **Base URL**
```
https://interview-prep-agent.run.app
```

### **Authentication**
- **Type**: OAuth2 with Bearer Token
- **Flow**: Authorization Code Grant (Google OAuth)
- **Headers**: `Authorization: Bearer {token}`

### **Rate Limiting**
- **Per User**: 100 requests/minute
- **Per API Key**: 1000 requests/minute
- **Headers**:
  - `X-RateLimit-Limit`: Total allowed
  - `X-RateLimit-Remaining`: Remaining requests
  - `X-RateLimit-Reset`: Unix timestamp when limit resets

---

## 2. Core Endpoint: `/execute-goal`

### **Endpoint Details**
```http
POST /execute-goal
Authorization: Bearer {token}
Content-Type: application/json
```

### **Request Structure**
```json
{
  "user_id": "string (required)",
  "goal": "string (required)",
  "params": "object (required)",
  "context": "object (optional)"
}
```

### **Response Structure (Success)**
```json
{
  "status": "success",
  "data": {
    "result": "object",
    "execution_id": "string",
    "timestamp": "ISO 8601"
  },
  "metadata": {
    "processing_time_ms": 1234,
    "agents_used": ["string"]
  }
}
```

### **Response Structure (Error)**
```json
{
  "status": "error",
  "error": {
    "code": "string",
    "message": "string",
    "details": "object"
  }
}
```

---

## 3. Supported Goal Types

### **Goal 1: Create Study Plan**
```json
{
  "goal": "create-study-plan",
  "params": {
    "role": "string (required)",
    "days": "integer (required, 1-30)",
    "experience_level": "string (required: 'beginner', 'intermediate', 'advanced')",
    "focus_areas": ["string (optional)"],
    "include_resources": "boolean (default: true)",
    "language": "string (default: 'en')"
  }
}
```

**Response:**
```json
{
  "plan_id": "string",
  "role": "string",
  "created_at": "ISO 8601",
  "days": [
    {
      "day_number": 1,
      "topic": "string",
      "description": "string",
      "learning_objectives": ["string"],
      "resources": [
        {
          "title": "string",
          "url": "string",
          "type": "article|video|interactive"
        }
      ],
      "practice_questions": ["string"],
      "estimated_hours": 2.5
    }
  ],
  "total_duration_hours": 12.5
}
```

---

### **Goal 2: Start Mock Interview**
```json
{
  "goal": "start-mock-interview",
  "params": {
    "topic": "string (required: 'system-design', 'coding', 'behavioral', 'data-structures')",
    "difficulty": "string (required: 'easy', 'medium', 'hard')",
    "duration_minutes": "integer (default: 45, range: 15-120)",
    "language": "string (default: 'en')",
    "code_languages": ["string (optional: 'python', 'java', 'go', 'javascript')"]
  }
}
```

**Response:**
```json
{
  "session_id": "string",
  "topic": "string",
  "difficulty": "string",
  "started_at": "ISO 8601",
  "estimated_end_time": "ISO 8601",
  "first_question": {
    "question_id": "string",
    "text": "string",
    "type": "text|coding|design",
    "time_limit_seconds": 300,
    "hints_available": 2
  }
}
```

---

### **Goal 3: Submit Interview Answer**
```json
{
  "goal": "submit-interview-answer",
  "params": {
    "session_id": "string (required)",
    "question_id": "string (required)",
    "answer": "string (required)",
    "answer_type": "string (required: 'text', 'code', 'design')",
    "language": "string (for code answers)",
    "time_taken_seconds": "integer"
  }
}
```

**Response:**
```json
{
  "evaluation": {
    "score": 85,
    "max_score": 100,
    "feedback": "string",
    "improvements": ["string"],
    "rubric_scores": {
      "correctness": 90,
      "completeness": 80,
      "clarity": 85,
      "edge_cases": 75
    }
  },
  "next_question": {
    "question_id": "string",
    "text": "string",
    "type": "text|coding|design"
  },
  "session_progress": {
    "questions_answered": 3,
    "average_score": 82,
    "time_remaining_minutes": 35
  }
}
```

---

### **Goal 4: End Mock Interview**
```json
{
  "goal": "end-mock-interview",
  "params": {
    "session_id": "string (required)"
  }
}
```

**Response:**
```json
{
  "session_summary": {
    "session_id": "string",
    "topic": "string",
    "total_questions": 5,
    "duration_minutes": 45,
    "overall_score": 83,
    "performance_by_topic": {
      "system_design": 85,
      "edge_cases": 80,
      "communication": 82
    }
  },
  "improvement_areas": ["string"],
  "strengths": ["string"],
  "recommended_next_steps": ["string"]
}
```

---

### **Goal 5: Get User Progress**
```json
{
  "goal": "get-user-progress",
  "params": {
    "from_date": "ISO 8601 (optional)",
    "to_date": "ISO 8601 (optional)"
  }
}
```

**Response:**
```json
{
  "user_id": "string",
  "summary": {
    "total_interviews": 15,
    "average_score": 78,
    "topics_covered": ["system-design", "coding"],
    "hours_studied": 42
  },
  "study_plans": [
    {
      "plan_id": "string",
      "role": "string",
      "status": "in-progress|completed",
      "progress_percentage": 60
    }
  ],
  "recent_interviews": [
    {
      "session_id": "string",
      "date": "ISO 8601",
      "topic": "string",
      "score": 85
    }
  ],
  "scored_timeline": [
    {
      "date": "ISO 8601",
      "average_score": 82,
      "interviews_count": 2
    }
  ]
}
```

---

### **Goal 6: Get Evaluation Rubric**
```json
{
  "goal": "get-evaluation-rubric",
  "params": {
    "question_type": "string (required: 'system-design', 'coding', 'behavioral')"
  }
}
```

**Response:**
```json
{
  "question_type": "string",
  "rubric": {
    "correctness": {
      "weight": 0.4,
      "description": "string",
      "levels": ["Poor", "Fair", "Good", "Excellent"]
    },
    "completeness": {
      "weight": 0.3,
      "description": "string",
      "levels": ["Incomplete", "Partial", "Complete", "Thorough"]
    }
  },
  "system_prompt": "string (for evaluators)"
}
```

---

## 4. Error Codes & Handling

### **HTTP Status Codes**

| Status | Scenario | Response |
|--------|----------|----------|
| 200 | Success | Standard success response |
| 400 | Bad Request | Missing/invalid parameters |
| 401 | Unauthorized | Invalid/expired token |
| 403 | Forbidden | User not authorized for resource |
| 404 | Not Found | Session/resource doesn't exist |
| 429 | Rate Limited | Too many requests |
| 500 | Server Error | Internal server error |
| 503 | Service Unavailable | Gemini API/database down |

### **Error Response Format**
```json
{
  "status": "error",
  "error": {
    "code": "INVALID_ROLE",
    "message": "The specified role is not supported",
    "details": {
      "provided_role": "quantum-engineer",
      "supported_roles": ["python-backend", "frontend", "devops"]
    },
    "timestamp": "ISO 8601",
    "request_id": "string"
  }
}
```

### **Common Error Codes**

| Code | HTTP | Description |
|------|------|-------------|
| `INVALID_GOAL` | 400 | Goal type not recognized |
| `MISSING_PARAM` | 400 | Required parameter missing |
| `INVALID_PARAM` | 400 | Parameter value invalid |
| `SESSION_NOT_FOUND` | 404 | Interview session doesn't exist |
| `USER_NOT_FOUND` | 404 | User not found in database |
| `AUTH_FAILED` | 401 | Authentication token invalid |
| `RATE_LIMIT_EXCEEDED` | 429 | User exceeded rate limit |
| `GEMINI_ERROR` | 503 | Gemini API error |
| `DATABASE_ERROR` | 503 | Database connection error |
| `TIMEOUT` | 504 | Request exceeded timeout |

---

## 5. Request Validation Rules

### **Study Plan Request**
- `role`: 1-100 chars, alphanumeric + spaces
- `days`: 1-30 (inclusive)
- `experience_level`: Must be one of ['beginner', 'intermediate', 'advanced']
- `focus_areas`: Max 5 items, each 1-50 chars
- `language`: Valid ISO 639-1 code (e.g., 'en', 'es')

### **Mock Interview Request**
- `topic`: Must be one of ['system-design', 'coding', 'behavioral', 'data-structures']
- `difficulty`: Must be one of ['easy', 'medium', 'hard']
- `duration_minutes`: 15-120 (inclusive)
- `code_languages`: Each must be valid (Python, Java, Go, JavaScript)

### **Answer Submission Request**
- `session_id`: Non-empty UUID string
- `question_id`: Non-empty string
- `answer`: Non-empty, up to 10KB
- `answer_type`: Must be one of ['text', 'code', 'design']
- `time_taken_seconds`: 0-3600

---

## 6. Authentication Flow

### **OAuth2 Authorization Code Flow**

```
1. User clicks "Sign in with Google"
   → Redirects to https://accounts.google.com/o/oauth2/v2/auth

2. User authenticates with Google

3. Redirected back to app with auth code

4. Backend exchanges code for access token
   https://oauth2.googleapis.com/token

5. Backend stores token + refresh token

6. User makes API requests with token
   Authorization: Bearer {access_token}

7. API validates token with Firebase Auth

8. Request proceeds if valid
```

---

## 7. Request/Response Examples

### **Example 1: Create Study Plan**

**Request:**
```bash
curl -X POST https://interview-prep-agent.run.app/execute-goal \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user-123",
    "goal": "create-study-plan",
    "params": {
      "role": "Python Backend Developer",
      "days": 5,
      "experience_level": "intermediate",
      "focus_areas": ["system design", "databases"]
    }
  }'
```

**Response:**
```json
{
  "status": "success",
  "data": {
    "result": {
      "plan_id": "plan-abc123",
      "role": "Python Backend Developer",
      "days": [
        {
          "day_number": 1,
          "topic": "System Design Fundamentals",
          "resources": [
            {
              "title": "System Design Primer",
              "url": "https://example.com/...",
              "type": "article"
            }
          ]
        }
      ]
    },
    "execution_id": "exec-xyz789"
  },
  "metadata": {
    "processing_time_ms": 2341,
    "agents_used": ["planner", "study_plan", "web_search", "progress_tracker"]
  }
}
```

---

## 8. Versioning & Deprecation

- **Current Version**: v1
- **Versioning Strategy**: URI versioning (`/v1/execute-goal`, `/v2/execute-goal`)
- **Deprecation**: 6-month notice before removing endpoints
- **Sunset Header**: `Sunset: date` included in responses for deprecated endpoints

