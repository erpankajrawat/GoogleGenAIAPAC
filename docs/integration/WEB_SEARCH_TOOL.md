# Web Search API Integration Guide

**Document:** `docs/integration/WEB_SEARCH_TOOL.md`

---

## 1. Overview

**Google Custom Search API** = Search the web via API

- **Purpose**: Find learning resources (articles, tutorials, videos)
- **Use Case**: Study Plan Agent fetches resources for each topic
- **Cost**: ~$5-10/month for typical usage
- **Speed**: ~500ms per search

---

## 2. Setup

### **Step 1: Get API Key**

```
1. Go to https://console.cloud.google.com
2. Go to "APIs & Services" → "Credentials"
3. Click "Create Credentials" → "API Key"
4. Copy the key (AIzaSy...)
5. Store in .env: GOOGLE_SEARCH_API_KEY
```

### **Step 2: Create Search Engine**

```
1. Go to https://programmablesearchengine.google.com/
2. Click "Create" (left sidebar)
3. Name: "interview-prep-search"
4. Websites to search: Leave EMPTY (search all web)
5. Click "Create"
6. Copy the Search Engine ID (cx parameter, something like: 1234567890abc:xyz)
7. Store in .env: GOOGLE_SEARCH_ENGINE_ID
```

### **Step 3: Store Credentials**

```bash
# .env file
GOOGLE_SEARCH_API_KEY=AIzaSyD...
GOOGLE_SEARCH_ENGINE_ID=1234567890abc:xyz
```

---

## 3. Installation

```bash
# Install Google API client
pip install google-api-python-client

# Verify
python -c "from googleapiclient.discovery import build; print('✓ Google API client installed')"
```

---

## 4. Basic Usage

### **Simple Search**

```python
from googleapiclient.discovery import build

API_KEY = "YOUR_API_KEY"
SEARCH_ENGINE_ID = "YOUR_CX_ID"

# Create service
service = build("customsearch", "v1", developerKey=API_KEY)

# Search
results = service.cse().list(
    q="system design interview preparation",
    cx=SEARCH_ENGINE_ID,
    num=10  # Number of results
).execute()

# Process results
for item in results.get('items', []):
    print(f"Title: {item['title']}")
    print(f"URL: {item['link']}")
    print(f"Snippet: {item['snippet']}\n")
```

### **With Pagination**

```python
def search_with_pagination(query: str, num_pages: int = 1):
    """Search with pagination (max 10 results per page)"""
    service = build("customsearch", "v1", developerKey=API_KEY)
    all_results = []

    for page in range(num_pages):
        start_index = (page * 10) + 1  # startIndex is 1-based

        results = service.cse().list(
            q=query,
            cx=SEARCH_ENGINE_ID,
            num=10,
            start=start_index
        ).execute()

        all_results.extend(results.get('items', []))

    return all_results
```

---

## 5. Project Integration

### **Project Structure**
```
src/
├── tools/
│   ├── __init__.py
│   ├── gemini_tool.py
│   ├── code_execution_tool.py
│   └── web_search_tool.py  ← Here
└── config/
    └── web_search_config.py
```

### **Web Search Config**

```python
# src/config/web_search_config.py
import os

class WebSearchConfig:
    """Google Custom Search API Configuration"""

    API_KEY = os.getenv("GOOGLE_SEARCH_API_KEY")
    SEARCH_ENGINE_ID = os.getenv("GOOGLE_SEARCH_ENGINE_ID")

    # Search settings
    DEFAULT_RESULTS = 10
    MAX_RESULTS_PER_REQUEST = 10
    MAX_PAGES = 5  # Max 50 results total

    # Filtering
    PREFERRED_CONTENT_TYPES = {
        "article": ["medium.com", "dev.to", "linkedin.com", "github.com"],
        "video": ["youtube.com", "coursera.org", "udemy.com"],
        "documentation": ["docs.", ".io/docs", ".dev"]
    }

    # Caching
    CACHE_TTL_SECONDS = 3600  # 1 hour
    CACHE_ENABLED = True
```

### **Web Search Tool Wrapper**

```python
# src/tools/web_search_tool.py
import logging
import time
from typing import Dict, List, Optional
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError
from src.config.web_search_config import WebSearchConfig

logger = logging.getLogger(__name__)

class WebSearchTool:
    """Tool for searching the web for resources"""

    def __init__(self):
        self.api_key = WebSearchConfig.API_KEY
        self.search_engine_id = WebSearchConfig.SEARCH_ENGINE_ID
        self.cache = {}
        self.cache_timestamps = {}

        if not self.api_key or not self.search_engine_id:
            raise ValueError("GOOGLE_SEARCH_API_KEY or GOOGLE_SEARCH_ENGINE_ID not set")

    async def search(
        self,
        query: str,
        num_results: int = 10,
        content_type: Optional[str] = None,
        use_cache: bool = True
    ) -> Dict:
        """
        Search for resources

        Args:
            query: Search query
            num_results: Number of results (1-50)
            content_type: Filter by type (article, video, documentation)
            use_cache: Use cached results if available

        Returns:
            {
                "success": bool,
                "results": [
                    {
                        "title": str,
                        "url": str,
                        "snippet": str,
                        "type": str  # article, video, documentation
                    }
                ],
                "total_results": int,
                "error": str (if failed)
            }
        """

        # Check cache
        cache_key = f"{query}_{num_results}_{content_type}"
        if use_cache and cache_key in self.cache:
            age = time.time() - self.cache_timestamps[cache_key]
            if age < WebSearchConfig.CACHE_TTL_SECONDS:
                logger.info(f"Using cached search results for: {query}")
                return self.cache[cache_key]

        try:
            logger.info(f"Searching for: {query}")

            service = build(
                "customsearch", "v1",
                developerKey=self.api_key
            )

            response = service.cse().list(
                q=query,
                cx=self.search_engine_id,
                num=min(num_results, WebSearchConfig.MAX_RESULTS_PER_REQUEST)
            ).execute()

            # Process results
            results = []
            for item in response.get('items', []):
                result = {
                    "title": item.get('title', ''),
                    "url": item.get('link', ''),
                    "snippet": item.get('snippet', '')[:200],  # Truncate snippet
                    "type": self._determine_type(item.get('link', ''))
                }

                # Filter by content type if specified
                if content_type and result["type"] != content_type:
                    continue

                results.append(result)

            response_data = {
                "success": True,
                "results": results[:num_results],
                "total_results": response.get('queries', {}).get('request', [{}])[0].get('totalResults', 0),
                "error": None
            }

            # Cache result
            self.cache[cache_key] = response_data
            self.cache_timestamps[cache_key] = time.time()

            logger.info(f"Found {len(results)} results")
            return response_data

        except HttpError as e:
            logger.error(f"Search API error: {str(e)}")
            return {
                "success": False,
                "results": [],
                "total_results": 0,
                "error": f"API Error: {str(e)}"
            }

        except Exception as e:
            logger.error(f"Search error: {str(e)}")
            return {
                "success": False,
                "results": [],
                "total_results": 0,
                "error": str(e)
            }

    def _determine_type(self, url: str) -> str:
        """Determine content type from URL"""
        url_lower = url.lower()

        if any(site in url_lower for site in ["youtube.com", "coursera.org", "udemy.com"]):
            return "video"
        elif any(site in url_lower for site in ["docs.", ".io/docs", ".dev", "github.com"]):
            return "documentation"
        else:
            return "article"

    async def search_by_topic(self, topic: str) -> Dict:
        """
        Search for resources for a specific topic

        Returns mix of article, video, and documentation
        """

        results = {
            "articles": [],
            "videos": [],
            "documentation": []
        }

        # Search for articles
        article_response = await self.search(
            query=f"{topic} tutorial article",
            num_results=5,
            content_type="article"
        )

        if article_response["success"]:
            results["articles"] = article_response["results"][:3]

        # Search for videos
        video_response = await self.search(
            query=f"{topic} tutorial video",
            num_results=5,
            content_type="video"
        )

        if video_response["success"]:
            results["videos"] = video_response["results"][:2]

        # Search for documentation
        doc_response = await self.search(
            query=f"{topic} documentation",
            num_results=5,
            content_type="documentation"
        )

        if doc_response["success"]:
            results["documentation"] = doc_response["results"][:2]

        return {
            "success": all([
                article_response.get("success"),
                video_response.get("success"),
                doc_response.get("success")
            ]),
            "resources": results,
            "error": None
        }


# Singleton instance
web_search_tool = WebSearchTool()
```

---

## 6. Integration with Study Plan Agent

```python
# src/agents/study_plan_agent.py
from src.tools.web_search_tool import web_search_tool
from src.tools.gemini_tool import gemini_tool

class StudyPlanAgent:
    """Agent for generating personalized study plans"""

    async def generate_study_plan_with_resources(
        self,
        role: str,
        days: int,
        experience_level: str
    ) -> Dict:
        """
        Generate study plan with associated resources

        Returns:
            {
                "plan_id": str,
                "days": [
                    {
                        "day_number": int,
                        "topic": str,
                        "resources": [
                            {
                                "title": str,
                                "url": str,
                                "type": str  # article, video, documentation
                            }
                        ]
                    }
                ]
            }
        """

        # Step 1: Generate study plan structure
        plan_response = await gemini_tool.generate_json(
            prompt=f"Create a {days}-day study plan for {role} ({experience_level})",
            system_instruction="You are an expert interview coach"
        )

        if not plan_response["success"]:
            return {"success": False, "error": plan_response["error"]}

        plan_data = plan_response["json"]

        # Step 2: For each day, search for resources
        for day in plan_data.get("days", []):
            topic = day.get("topic", "")

            logger.info(f"Searching resources for: {topic}")

            resources_response = await web_search_tool.search_by_topic(topic)

            if resources_response["success"]:
                # Combine all resource types
                all_resources = (
                    resources_response["resources"].get("articles", []) +
                    resources_response["resources"].get("videos", []) +
                    resources_response["resources"].get("documentation", [])
                )

                day["resources"] = all_resources[:5]  # Top 5 resources
            else:
                day["resources"] = []

        return {
            "success": True,
            "plan": plan_data,
            "error": None
        }
```

---

## 7. Testing Integration

### **Test Script**

```python
# tests/test_web_search_integration.py
import asyncio
from src.tools.web_search_tool import web_search_tool

async def test_basic_search():
    """Test basic search functionality"""
    response = await web_search_tool.search("system design interview")

    assert response["success"] == True
    assert len(response["results"]) > 0
    assert "title" in response["results"][0]
    assert "url" in response["results"][0]
    print("✓ Basic search test passed")
    print(f"  Found {len(response['results'])} results")


async def test_search_by_topic():
    """Test searching by topic (articles + videos + docs)"""
    response = await web_search_tool.search_by_topic("databases")

    assert response["success"] == True
    assert "articles" in response["resources"]
    assert "videos" in response["resources"]
    assert "documentation" in response["resources"]
    print("✓ Search by topic test passed")


async def test_caching():
    """Test that results are cached"""
    query = "python algorithms"

    # First search
    import time
    start = time.time()
    response1 = await web_search_tool.search(query)
    time1 = time.time() - start

    # Second search (should be cached)
    start = time.time()
    response2 = await web_search_tool.search(query)
    time2 = time.time() - start

    assert response1 == response2  # Same results
    assert time2 < time1  # Faster (from cache)
    print("✓ Caching test passed")
    print(f"  First search: {time1*1000:.0f}ms, Cached: {time2*1000:.0f}ms")


async def main():
    print("Running Web Search integration tests...\n")
    await test_basic_search()
    await test_search_by_topic()
    await test_caching()
    print("\n✅ All tests passed!")


if __name__ == "__main__":
    asyncio.run(main())
```

Run tests:
```bash
python -m pytest tests/test_web_search_integration.py -v
```

---

## 8. Cost Optimization

### **Pricing**
```
- Free tier: 100 searches/day
- Paid tier: $5 per 1000 searches

Estimated costs for interview prep:
- 100 users × 1 study plan/week × 5 topics = 500 searches/week
- Cost: ~$0.25/week = $1/month for 100 users
```

### **Tips**
```
1. Cache results (1 hour TTL)
2. Batch searches where possible
3. Use fallback resources if search fails
4. Limit to top 10 results per query
```

---

## 9. Fallback Resources

In case search fails, use curated resources:

```python
FALLBACK_RESOURCES = {
    "system-design": [
        {
            "title": "System Design Primer",
            "url": "https://github.com/donnemartin/system-design-primer",
            "type": "documentation"
        },
        {
            "title": "Designing Data-Intensive Applications",
            "url": "https://dataintensive.net/",
            "type": "article"
        }
    ],
    "algorithms": [
        {
            "title": "LeetCode",
            "url": "https://leetcode.com/",
            "type": "article"
        },
        {
            "title": "GeeksforGeeks",
            "url": "https://www.geeksforgeeks.org/",
            "type": "documentation"
        }
    ]
}

async def search_with_fallback(topic: str):
    """Search with fallback to curated resources"""
    response = await web_search_tool.search(f"{topic} tutorial")

    if not response["success"] or len(response["results"]) == 0:
        logger.warning(f"Search failed for {topic}, using fallback")
        return FALLBACK_RESOURCES.get(topic, [])

    return response["results"]
```

---

## 10. Resources

- **Official Docs**: https://developers.google.com/custom-search
- **API Reference**: https://developers.google.com/custom-search/v1/reference/rest
- **Python Client**: https://github.com/googleapis/google-api-python-client
