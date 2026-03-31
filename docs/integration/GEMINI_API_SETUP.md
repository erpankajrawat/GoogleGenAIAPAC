# Gemini API Integration Guide

**Document:** `docs/integration/GEMINI_API_SETUP.md`

---

## 1. Gemini API Overview

**Gemini** = Google's latest AI model family

- **Model**: gemini-2.0-flash (recommended)
- **Speed**: Fast inference (~500ms)
- **Context Window**: 128K tokens (can handle long conversations)
- **Tool-Using**: Native support for function calling
- **Cost**: $0.05/1M input tokens, $0.15/1M output tokens

---

## 2. Account Setup

### **Step 1: Create Google Cloud Project**
```
1. Go to https://console.cloud.google.com
2. Click "Select Project" (top left)
3. Click "NEW PROJECT"
4. Name: "interview-prep-ai"
5. Click "Create"
6. Wait 1-2 minutes for project to be created
```

### **Step 2: Enable Gemini API**
```
1. In Google Cloud Console, search for "Generative AI API"
2. Click "Enable"
3. Wait for activation
```

### **Step 3: Create API Key**
```
1. Go to "APIs & Services" → "Credentials" (left menu)
2. Click "Create Credentials" → "API Key"
3. Copy the key (format: AIzaSy...)
4. Click "Close"
```

### **Step 4: Store in .env**
```bash
# .env file
GEMINI_API_KEY=AIzaSyD...
GEMINI_MODEL=gemini-2.0-flash
```

---

## 3. Installation

```bash
# Install Google Gemini SDK
pip install google-generativeai

# Verify
python -c "from google import generativeai; print('✓ Gemini SDK installed')"
```

---

## 4. Basic Usage

### **Simple Text Generation**

```python
import google.generativeai as genai

# Configure API
genai.configure(api_key="YOUR_API_KEY")

# Create model instance
model = genai.GenerativeModel("gemini-2.0-flash")

# Generate text
response = model.generate_content("What are the top 5 system design patterns?")

print(response.text)
```

### **With Configuration**

```python
import google.generativeai as genai
from google.generativeai.types import GenerationConfig

# Configure API
genai.configure(api_key="YOUR_API_KEY")

# Create model with specific settings
model = genai.GenerativeModel(
    "gemini-2.0-flash",
    generation_config=GenerationConfig(
        temperature=0.7,           # Creativity level (0-2)
        top_p=0.95,               # Diversity (0-1)
        top_k=40,                 # Token variety
        max_output_tokens=2048    # Max response length
    ),
    system_instruction="You are an expert interview coach helping candidates prepare."
)

# Generate content
response = model.generate_content("Generate a system design interview question")
print(response.text)
```

---

## 5. Prompt Structure for Agents

### **Study Plan Generation Prompt**

```python
def generate_study_plan(role: str, days: int, experience_level: str):
    """Generate personalized study plan using Gemini"""

    system_prompt = """You are an expert interview coach specializing in tech interviews.
    Your job is to create personalized, day-by-day study plans for candidates.

    Rules:
    - Each day should have realistic 2-3 hour commitment
    - Provide concrete learning objectives
    - Include practice question types
    - Suggest specific resources

    Return ONLY valid JSON matching this structure:
    {
        "days": [
            {
                "day_number": 1,
                "topic": "string",
                "description": "string",
                "learning_objectives": ["string"],
                "practice_questions": ["string"],
                "estimated_hours": 2.5
            }
        ]
    }"""

    user_prompt = f"""Create a {days}-day interview prep plan for:
Role: {role}
Experience Level: {experience_level}

Requirements:
- Focus on system design and coding interviews
- Include behavioral interview prep
- Realistic pacing for working professionals"""

    model = genai.GenerativeModel(
        "gemini-2.0-flash",
        system_instruction=system_prompt
    )

    response = model.generate_content(user_prompt)

    # Parse JSON response
    import json
    return json.loads(response.text)
```

### **Mock Interview Question Generation**

```python
def generate_interview_question(topic: str, difficulty: str):
    """Generate a mock interview question"""

    system_prompt = """You are an expert interviewer conducting mock interviews.
    Generate realistic, thoughtful interview questions that test deep understanding.

    Return ONLY valid JSON matching this structure:
    {
        "question": "string",
        "topic": "string",
        "difficulty": "string",
        "hint": "string",
        "solution_summary": "string",
        "time_limit_minutes": 15
    }"""

    user_prompt = f"""Generate a {difficulty} {topic} interview question"""

    model = genai.GenerativeModel(
        "gemini-2.0-flash",
        system_instruction=system_prompt
    )

    response = model.generate_content(user_prompt)

    import json
    return json.loads(response.text)
```

### **Answer Evaluation**

```python
def evaluate_answer(question: str, answer: str, rubric: dict):
    """Evaluate user's interview answer"""

    system_prompt = f"""You are an expert interviewer evaluating candidate responses.
    Use this rubric for scoring:
    {rubric}

    Return ONLY valid JSON matching this structure:
    {{
        "score": 0-100,
        "feedback": "string",
        "strengths": ["string"],
        "improvements": ["string"],
        "rubric_scores": {{
            "correctness": 0-100,
            "completeness": 0-100,
            "clarity": 0-100,
            "edge_cases": 0-100
        }}
    }}"""

    user_prompt = f"""Evaluate this interview answer:

Question: {question}

Answer: {answer}

Provide constructive feedback and scoring."""

    model = genai.GenerativeModel(
        "gemini-2.0-flash",
        system_instruction=system_prompt
    )

    response = model.generate_content(user_prompt)

    import json
    return json.loads(response.text)
```

---

## 6. Token Management & Optimization

### **Token Counting**

```python
import google.generativeai as genai

model = genai.GenerativeModel("gemini-2.0-flash")

# Count tokens in text
text = "This is a sample text to count tokens"
token_data = model.count_tokens(text)
print(f"Tokens: {token_data.total_tokens}")

# Count tokens in conversation
messages = [
    {"role": "user", "parts": [{"text": "Hello"}]},
    {"role": "model", "parts": [{"text": "Hi, how can I help?"}]}
]
token_count = model.count_tokens(messages)
print(f"Conversation tokens: {token_count.total_tokens}")
```

### **Cost Calculation**

```python
# Pricing (as of March 2026)
INPUT_COST_PER_1M = 0.05      # $0.05 per 1M input tokens
OUTPUT_COST_PER_1M = 0.15     # $0.15 per 1M output tokens

def estimate_cost(input_tokens: int, output_tokens: int) -> float:
    """Estimate API cost for a request"""
    input_cost = (input_tokens / 1_000_000) * INPUT_COST_PER_1M
    output_cost = (output_tokens / 1_000_000) * OUTPUT_COST_PER_1M
    return input_cost + output_cost

# Example
input_tokens = 500
output_tokens = 200
cost = estimate_cost(input_tokens, output_tokens)
print(f"Estimated cost: ${cost:.6f}")
```

### **Caching for Cost Optimization**

```python
# Cache prompts that are used repeatedly (24h TTL)
import hashlib
import time

class PromptCache:
    def __init__(self):
        self.cache = {}
        self.timestamps = {}
        self.ttl_seconds = 24 * 3600  # 24 hours

    def get(self, key: str):
        """Get cached response"""
        if key in self.cache:
            age = time.time() - self.timestamps[key]
            if age < self.ttl_seconds:
                return self.cache[key]
            else:
                del self.cache[key]
                del self.timestamps[key]
        return None

    def set(self, key: str, value: str):
        """Cache response"""
        self.cache[key] = value
        self.timestamps[key] = time.time()

    def get_key(self, *args):
        """Generate cache key from arguments"""
        content = str(args).encode()
        return hashlib.md5(content).hexdigest()

# Usage
cache = PromptCache()

def generate_cached(role: str, days: int):
    """Generate with caching"""
    cache_key = cache.get_key("study_plan", role, days)

    # Check cache
    cached = cache.get(cache_key)
    if cached:
        print("Using cached response")
        return cached

    # Generate new
    print("Generating new response")
    response = generate_study_plan(role, days, "intermediate")

    # Cache it
    cache.set(cache_key, response)
    return response
```

---

## 7. Project Integration

### **Project Structure**
```
src/
├── tools/
│   ├── __init__.py
│   ├── gemini_tool.py  ← Here
│   └── code_execution_tool.py
├── config/
│   └── gemini_config.py
└── agents/
    └── ...
```

### **Gemini Config**

```python
# src/config/gemini_config.py
import os
from google.generativeai.types import GenerationConfig

class GeminiConfig:
    """Gemini API Configuration"""

    # API Setup
    API_KEY = os.getenv("GEMINI_API_KEY")
    MODEL = os.getenv("GEMINI_MODEL", "gemini-2.0-flash")

    # Generation settings
    GENERATION_CONFIG = GenerationConfig(
        temperature=0.7,           # Moderate creativity
        top_p=0.95,               # Diversity
        top_k=40,                 # Token variety
        max_output_tokens=2048    # Max response length
    )

    # For strict outputs (study plans, evaluations)
    STRICT_CONFIG = GenerationConfig(
        temperature=0.3,          # Low creativity (deterministic)
        top_p=0.9,
        max_output_tokens=1024
    )

    # Caching settings
    CACHE_TTL_SECONDS = 24 * 3600  # 24 hours
    CACHE_ENABLED = True

    # Token limits
    MAX_INPUT_TOKENS = 4000
    MAX_OUTPUT_TOKENS = 2048
```

### **Gemini Tool Wrapper**

```python
# src/tools/gemini_tool.py
import logging
import json
import google.generativeai as genai
from typing import Dict, Optional
from src.config.gemini_config import GeminiConfig

logger = logging.getLogger(__name__)

class GeminiTool:
    """Wrapper for Gemini API calls"""

    def __init__(self):
        self.api_key = GeminiConfig.API_KEY
        self.model_name = GeminiConfig.MODEL

        if not self.api_key:
            raise ValueError("GEMINI_API_KEY environment variable not set")

        genai.configure(api_key=self.api_key)
        self.model = genai.GenerativeModel(
            self.model_name,
            generation_config=GeminiConfig.GENERATION_CONFIG
        )

    async def generate_text(
        self,
        prompt: str,
        system_instruction: str = None,
        strict_format: bool = False
    ) -> Dict:
        """
        Generate text using Gemini

        Args:
            prompt: User prompt
            system_instruction: System instruction for context
            strict_format: Use deterministic settings for structured output

        Returns:
            {
                "success": bool,
                "text": str,
                "tokens_used": int,
                "error": str (if failed)
            }
        """

        try:
            config = GeminiConfig.STRICT_CONFIG if strict_format else GeminiConfig.GENERATION_CONFIG

            model = genai.GenerativeModel(
                self.model_name,
                generation_config=config,
                system_instruction=system_instruction
            )

            logger.info(f"Calling Gemini API (strict={strict_format})")

            response = model.generate_content(prompt)

            logger.info(f"Gemini response received: {len(response.text)} chars")

            return {
                "success": True,
                "text": response.text,
                "tokens_used": response.usage_metadata.total_token_count if hasattr(response, 'usage_metadata') else None,
                "error": None
            }

        except Exception as e:
            logger.error(f"Gemini API error: {str(e)}")
            return {
                "success": False,
                "text": None,
                "tokens_used": None,
                "error": str(e)
            }

    async def generate_json(
        self,
        prompt: str,
        system_instruction: str = None,
        schema: Optional[dict] = None
    ) -> Dict:
        """
        Generate structured JSON using Gemini

        Args:
            prompt: User prompt
            system_instruction: System instruction
            schema: Expected JSON schema for validation

        Returns:
            {
                "success": bool,
                "json": dict,
                "error": str (if failed)
            }
        """

        # Append schema to prompt if provided
        if schema:
            prompt += f"\n\nExpected JSON structure:\n{json.dumps(schema, indent=2)}"

        response = await self.generate_text(
            prompt,
            system_instruction=system_instruction,
            strict_format=True
        )

        if not response["success"]:
            return {"success": False, "json": None, "error": response["error"]}

        try:
            # Extract JSON from response
            json_str = response["text"]
            # Try to find JSON block
            if "```json" in json_str:
                json_str = json_str.split("```json")[1].split("```")[0]
            elif "```" in json_str:
                json_str = json_str.split("```")[1].split("```")[0]

            parsed = json.loads(json_str)
            return {"success": True, "json": parsed, "error": None}

        except json.JSONDecodeError as e:
            logger.error(f"JSON parsing error: {str(e)}")
            return {
                "success": False,
                "json": None,
                "error": f"Invalid JSON response: {str(e)}"
            }

    def count_tokens(self, text: str) -> int:
        """Count tokens in text"""
        token_data = self.model.count_tokens(text)
        return token_data.total_tokens


# Singleton instance
gemini_tool = GeminiTool()
```

---

## 8. Testing Integration

### **Test Script**

```python
# tests/test_gemini_integration.py
import asyncio
from src.tools.gemini_tool import gemini_tool

async def test_text_generation():
    """Test basic text generation"""
    response = await gemini_tool.generate_text(
        "Explain what system design is in one sentence"
    )

    assert response["success"] == True
    assert len(response["text"]) > 0
    print("✓ Text generation test passed")
    print(f"Response: {response['text'][:100]}...")


async def test_json_generation():
    """Test JSON generation"""
    schema = {
        "questions": [
            {
                "question": "string",
                "topic": "string",
                "difficulty": "string"
            }
        ]
    }

    response = await gemini_tool.generate_json(
        prompt="Generate 2 system design interview questions",
        schema=schema,
        system_instruction="You are an expert interviewer"
    )

    assert response["success"] == True
    assert isinstance(response["json"], dict)
    assert "questions" in response["json"]
    print("✓ JSON generation test passed")


async def test_token_counting():
    """Test token counting"""
    text = "This is a test text to count tokens" * 10
    tokens = gemini_tool.count_tokens(text)

    assert tokens > 0
    print(f"✓ Token counting test passed ({tokens} tokens)")


async def main():
    print("Running Gemini integration tests...\n")
    await test_text_generation()
    await test_json_generation()
    await test_token_counting()
    print("\n✅ All tests passed!")


if __name__ == "__main__":
    asyncio.run(main())
```

Run tests:
```bash
python -m pytest tests/test_gemini_integration.py -v
```

---

## 9. Error Handling

### **Common Errors**

| Error | Cause | Solution |
|-------|-------|----------|
| InvalidAPIKeyError | Wrong/missing API key | Check GEMINI_API_KEY in .env |
| ResourceExhausted | Rate limited | Wait and retry with backoff |
| InvalidArgument | Bad prompt format | Validate prompt syntax |
| DeadlineExceeded | Request timeout | Increase timeout or shorten prompt |

---

## 10. Cost Optimization

### **Tips**

```python
# 1. Use caching for repeated prompts
# 2. Reuse GenerativeModel instances
# 3. Batch multiple requests
# 4. Use lower temperature when determinism needed
# 5. Count tokens before sending expensive prompts
```

### **Monthly Cost Estimate**

```
For 100 active users, 10 study plans/month, 20 interviews/month:

Study Plans: 100 × 10 × (500 input + 1000 output) tokens = 15M tokens
- Cost: (15M / 1M) × ($0.05 input + $0.15 output) = ~$3

Mock Interviews: 100 × 20 × 10 questions × (300 input + 500 output) = 16M tokens
- Cost: (16M / 1M) × $0.20 = ~$3.20

Evaluations: 100 × 20 × 10 × (400 input + 300 output) = 14M tokens
- Cost: (14M / 1M) × $0.20 = ~$2.80

Total Gemini Cost: ~$9/month for 100 active users
```

---

## 11. Resources

- **Official Docs**: https://ai.google.dev
- **API Reference**: https://ai.google.dev/api/rest
- **Python SDK**: https://github.com/google/generative-ai-python
- **Pricing**: https://ai.google.dev/pricing
- **Models**: https://ai.google.dev/models

