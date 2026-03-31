# Database Schema — Firestore

**Document:** `docs/data/DATABASE_SCHEMA.md`

---

## 1. Firestore Collections

### **Collection: `users`**
```
Collection: users
├─ Document: {user_id}
│  ├─ email: string
│  ├─ name: string
│  ├─ created_at: timestamp
│  ├─ updated_at: timestamp
│  ├─ experience_level: string
│  ├─ roles_interested: array[string]
│  ├─ total_study_hours: number
│  ├─ average_interview_score: number
│  └─ preferences: map
│     ├─ language: string
│     ├─ difficulty_preference: string
│     └─ notification_enabled: boolean
```

---

### **Collection: `study_plans`**
```
Collection: study_plans
├─ Document: {plan_id}
│  ├─ user_id: string (index)
│  ├─ role: string
│  ├─ days: number (1-30)
│  ├─ experience_level: string
│  ├─ status: string (active|completed|archived)
│  ├─ created_at: timestamp
│  ├─ started_at: timestamp (nullable)
│  ├─ completed_at: timestamp (nullable)
│  ├─ days_data: array
│  │  ├─ day_number: number
│  │  ├─ topic: string
│  │  ├─ description: string
│  │  ├─ learning_objectives: array[string]
│  │  ├─ resources: array
│  │  │  ├─ title: string
│  │  │  ├─ url: string
│  │  │  └─ type: string
│  │  ├─ practice_questions: array[string]
│  │  └─ estimated_hours: number
│  ├─ total_duration_hours: number
│  └─ progress_percentage: number
```

---

### **Collection: `mock_interviews`**
```
Collection: mock_interviews
├─ Document: {session_id}
│  ├─ user_id: string (index)
│  ├─ topic: string (index)
│  ├─ difficulty: string
│  ├─ status: string (in-progress|completed|abandoned)
│  ├─ created_at: timestamp
│  ├─ started_at: timestamp
│  ├─ ended_at: timestamp (nullable)
│  ├─ estimated_duration_minutes: number
│  ├─ questions: array
│  │  ├─ question_id: string
│  │  ├─ text: string
│  │  ├─ type: string (text|coding|design)
│  │  ├─ sequence_number: number
│  │  └─ time_limit_seconds: number
│  ├─ answers: array
│  │  ├─ question_id: string
│  │  ├─ answer_text: string
│  │  ├─ submitted_at: timestamp
│  │  └─ time_taken_seconds: number
│  ├─ scores: array
│  │  ├─ question_id: string
│  │  ├─ score: number (0-100)
│  │  ├─ feedback: string
│  │  └─ rubric_breakdown: map
│  ├─ overall_score: number (0-100)
│  └─ metadata: map
│     ├─ interview_round: number
│     └─ previous_topic_performance: number
```

---

### **Collection: `user_progress`**
```
Collection: user_progress
├─ Document: {user_id}
│  ├─ total_interviews_completed: number
│  ├─ average_score: number
│  ├─ total_study_hours: number
│  ├─ study_plans_completed: number
│  ├─ topics_attempted: map
│  │  ├─ system-design: object
│  │  │  ├─ attempts: number
│  │  │  ├─ average_score: number
│  │  │  └─ last_attempt: timestamp
│  │  ├─ coding: object
│  │  ├─ behavioral: object
│  │  └─ data-structures: object
│  ├─ score_timeline: array
│  │  ├─ date: timestamp
│  │  ├─ score: number
│  │  └─ topic: string
│  ├─ performance_trend: array
│  │  ├─ week: number
│  │  └─ average_score: number
│  ├─ goals_set: array
│  │  ├─ goal_id: string
│  │  ├─ description: string
│  │  ├─ target_date: timestamp
│  │  └─ achieved: boolean
│  └─ last_updated: timestamp
```

---

### **Collection: `evaluation_rubrics`**
```
Collection: evaluation_rubrics
├─ Document: {rubric_id}
│  ├─ question_type: string (system-design|coding|behavioral)
│  ├─ difficulty: string (easy|medium|hard)
│  ├─ rubric: map
│  │  ├─ correctness: object
│  │  │  ├─ weight: number (0-1)
│  │  │  ├─ description: string
│  │  │  └─ levels: array[string]
│  │  ├─ completeness: object
│  │  ├─ clarity: object
│  │  └─ edge_cases: object
│  ├─ criteria: array[string]
│  ├─ example_answers: array
│  │  ├─ answer_text: string
│  │  ├─ expected_score: number
│  │  └─ explanation: string
│  ├─ updated_at: timestamp
│  └─ version: number
```

---

### **Collection: `feedback_history`**
```
Collection: feedback_history
├─ Document: {feedback_id}
│  ├─ user_id: string (index)
│  ├─ session_id: string
│  ├─ question_id: string
│  ├─ feedback_text: string
│  ├─ score: number (0-100)
│  ├─ rubric_scores: map
│  │  ├─ correctness: number
│  │  ├─ completeness: number
│  │  ├─ clarity: number
│  │  └─ edge_cases: number
│  ├─ improvement_areas: array[string]
│  ├─ strengths: array[string]
│  ├─ follow_up_suggestions: array[string]
│  ├─ created_at: timestamp
│  └─ model_version: string (Gemini version used)
```

---

### **Collection: `conversation_logs`** (Optional - for analysis)
```
Collection: conversation_logs
├─ Document: {log_id}
│  ├─ user_id: string (index)
│  ├─ session_id: string
│  ├─ agent_name: string
│  ├─ message_type: string (task|result|error)
│  ├─ payload: map
│  ├─ timestamp: timestamp
│  ├─ processing_time_ms: number
│  └─ status: string (success|failed)
```

---

## 2. Indexes

### **Required Indexes**

```firestore
// Index on users collection
- Field: created_at (Descending)

// Index on study_plans collection
- Field: user_id (Ascending)
- Field: created_at (Descending)

// Index on study_plans collection
- Field: user_id (Ascending)
- Field: status (Ascending)

// Index on mock_interviews collection
- Field: user_id (Ascending)
- Field: created_at (Descending)

// Index on mock_interviews collection
- Field: user_id (Ascending)
- Field: topic (Ascending)
- Field: created_at (Descending)

// Index on user_progress collection
- Field: user_id (Ascending)

// Index on feedback_history collection
- Field: user_id (Ascending)
- Field: created_at (Descending)

// Index on conversation_logs collection
- Field: user_id (Ascending)
- Field: timestamp (Descending)
```

---

## 3. Data Access Patterns

### **Pattern 1: Get User's Recent Interviews**
```python
# Query
db.collection('mock_interviews') \
    .where('user_id', '==', user_id) \
    .order_by('created_at', direction=firestore.Query.DESCENDING) \
    .limit(10) \
    .stream()
```

### **Pattern 2: Get Study Plans by Status**
```python
# Query
db.collection('study_plans') \
    .where('user_id', '==', user_id) \
    .where('status', '==', 'active') \
    .order_by('created_at', direction=firestore.Query.DESCENDING) \
    .stream()
```

### **Pattern 3: Get Progress Timeline**
```python
# Query (date range)
db.collection('user_progress') \
    .document(user_id) \
    .get()

# Then extract score_timeline and filter by date
```

### **Pattern 4: Update Interview Session**
```python
# Batch write
batch = db.batch()
batch.update(
    db.collection('mock_interviews').document(session_id),
    {
        'answers': firestore.ArrayUnion([new_answer]),
        'scores': firestore.ArrayUnion([new_score])
    }
)
batch.commit()
```

---

## 4. Backup & Recovery

### **Backup Strategy**
- Automated daily backups to Cloud Storage
- 30-day retention
- Point-in-time recovery available

### **Recovery Procedure**
1. Request recovery from Firebase Console
2. Restore to specific timestamp
3. Verify data integrity
4. Notify users if data rollback occurred

---

## 5. Data Retention & Cleanup

### **Retention Policy**
| Collection | Retention | Policy |
|-----------|-----------|--------|
| users | Indefinite | Keep forever |
| study_plans | 2 years | Auto-archive |
| mock_interviews | 2 years | Auto-archive |
| user_progress | 2 years | Aggregate annually |
| feedback_history | 1 year | Delete after 1 year |
| conversation_logs | 90 days | Delete after 90 days |

---

## 6. Cost Optimization

### **Read/Write Optimization**
- Batch operations where possible (reduce operations)
- Use subcollections for nested data (better organization)
- Denormalize carefully for frequently accessed queries
- Archive old data to reduce read costs

### **Estimated Costs** (per month, 1000 active users)
- Reads: ~50,000 (estimated: $0.18)
- Writes: ~30,000 (estimated: $0.30)
- Deletes: ~5,000 (estimated: $0.05)
- **Total**: ~$0.50/month

