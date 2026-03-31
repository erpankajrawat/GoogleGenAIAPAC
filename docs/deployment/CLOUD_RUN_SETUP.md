# Cloud Run Deployment & Configuration

**Document:** `docs/deployment/CLOUD_RUN_SETUP.md`

---

## 1. Deployment Overview

### **Containerization**
- **Base Image**: `python:3.11-slim`
- **Container Registry**: Google Artifact Registry
- **Orchestration**: Cloud Run (serverless)
- **Environment**: Production (auto-scaling)

---

## 2. Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/
COPY config/ ./config/

# Create non-root user for security
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8080/health')"

# Start application
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

---

## 3. Cloud Run Configuration

### **Deployment Command**
```bash
gcloud run deploy interview-prep-agent \
  --source . \
  --platform managed \
  --region us-central1 \
  --memory 2Gi \
  --cpu 2 \
  --timeout 900 \
  --max-instances 100 \
  --min-instances 1 \
  --allow-unauthenticated \
  --set-env-vars ENVIRONMENT=production \
  --service-account interview-prep-agent@project.iam.gserviceaccount.com
```

### **Configuration Details**

| Setting | Value | Reason |
|---------|-------|--------|
| Region | us-central1 | Low latency for US users |
| Memory | 2 GB | Sufficient for Gemini calls + agents |
| CPU | 2 | Parallel agent execution |
| Timeout | 900s (15m) | Long-running AI operations |
| Max Instances | 100 | Cost limit to prevent runaway |

---

## 4. Environment Variables

### **.env.production**
```bash
# Application
ENVIRONMENT=production
DEBUG=false
LOG_LEVEL=INFO

# Google Cloud
GOOGLE_CLOUD_PROJECT=interview-prep-ai
GOOGLE_APPLICATION_CREDENTIALS=/var/secrets/google/key.json

# Gemini API
GEMINI_API_KEY=<secret>
GEMINI_MODEL=gemini-2.0-flash

# Firebase
FIREBASE_PROJECT_ID=interview-prep-ai
FIREBASE_DATABASE_URL=https://interview-prep-ai.firebaseapp.com

# Database
FIRESTORE_DATABASE=production
FIRESTORE_COLLECTION_PREFIX=prod_

# Authentication
OAUTH_CLIENT_ID=<secret>
OAUTH_CLIENT_SECRET=<secret>
JWT_SECRET=<secret>

# External APIs
WEB_SEARCH_API_KEY=<secret>
CODE_EXECUTION_API_KEY=<secret>

# Monitoring
ENABLE_CLOUD_LOGGING=true
ENABLE_CLOUD_TRACE=true
```

---

## 5. Service Account & IAM Roles

### **Service Account Creation**
```bash
# Create service account
gcloud iam service-accounts create interview-prep-agent \
  --display-name "Interview Prep Agent"

# Grant necessary roles
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member "serviceAccount:interview-prep-agent@PROJECT_ID.iam.gserviceaccount.com" \
  --role "roles/datastore.user"  # Firestore access

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member "serviceAccount:interview-prep-agent@PROJECT_ID.iam.gserviceaccount.com" \
  --role "roles/logging.logWriter"  # Cloud Logging

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member "serviceAccount:interview-prep-agent@PROJECT_ID.iam.gserviceaccount.com" \
  --role "roles/cloudtrace.agent"  # Cloud Trace
```

---

## 6. CI/CD Pipeline

### **GitHub Actions Workflow** (.github/workflows/deploy.yml)
```yaml
name: Deploy to Cloud Run

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v1

      - name: Configure Docker to use gcloud as credential helper
        run: gcloud auth configure-docker us-central1-docker.pkg.dev

      - name: Build Docker image
        run: |
          docker build -t us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT }}/interview-prep/api:${{ github.sha }} .

      - name: Push to Artifact Registry
        run: |
          docker push us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT }}/interview-prep/api:${{ github.sha }}

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy interview-prep-agent \
            --image us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT }}/interview-prep/api:${{ github.sha }} \
            --region us-central1 \
            --platform managed
```

---

## 7. Monitoring & Logging

### **Cloud Logging**
```python
from google.cloud import logging as cloud_logging

logging_client = cloud_logging.Client()
logging_client.setup_logging()

logger = logging.getLogger(__name__)
logger.info("Interview prep agent started")
```

### **Cloud Monitoring (Metrics)**
- **Request latency**: p50, p95, p99
- **Error rate**: % of failed requests
- **Gemini API calls**: Count and latency
- **Database operations**: Read/write latency
- **Agent execution time**: per agent

### **Cloud Trace**
- Enable automatic trace collection
- Trace spans for each agent execution
- Identify performance bottlenecks

---

## 8. Scaling Strategy

### **Auto-scaling Configuration**
```bash
# Already configured in Cloud Run deployment
# - Min instances: 1 (cost optimization)
# - Max instances: 100 (prevents runaway)
# - Concurrency per instance: 80 (default)
```

### **Load Distribution**
- Load balancer distributes traffic across instances
- Each instance handles up to 80 concurrent requests
- At 100 instances: 8000 concurrent users possible

### **Cost Optimization**
- Min instances = 1 (idle cost: $0.00288/hour)
- Scale up on demand
- Monthly cost (100 active users): ~$50-100

---

## 9. Health Checks

### **Health Check Endpoint**
```python
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat()
    }
```

### **Readiness Probe**
```python
@app.get("/ready")
async def readiness_check():
    try:
        # Check Firestore connectivity
        db.collection('users').limit(1).get()
        return {"ready": True}
    except Exception as e:
        return {"ready": False, "error": str(e)}, 503
```

---

## 10. Rollback & Updates

### **Deployment Versioning**
```bash
# List all revisions
gcloud run revisions list --service=interview-prep-agent --region=us-central1

# Deploy new version
gcloud run deploy interview-prep-agent --image={new_image}

# Rollback to previous version
gcloud run services update-traffic interview-prep-agent \
  --to-revisions {revision_id}=100 \
  --region us-central1
```

### **Smoke Testing Post-Deploy**
```bash
# Test basic functionality
curl https://interview-prep-agent.run.app/health

# Test authentication
curl -H "Authorization: Bearer {test_token}" \
  https://interview-prep-agent.run.app/execute-goal \
  -d '{"goal": "get-user-progress", "user_id": "test-user"}'
```

