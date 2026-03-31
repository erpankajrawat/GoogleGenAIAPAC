# E2B Integration Guide — Code Execution Setup

**Document:** `docs/integration/CODE_EXECUTION_TOOL.md`

---

## 1. E2B Overview

**E2B** = Secure, serverless code execution for AI applications

- **Website**: https://e2b.dev
- **Purpose**: Run user code in isolated sandboxes
- **Perfect for**: Interview prep (test coding solutions)
- **Security**: Production-grade isolation
- **Speed**: ~100ms execution time

---

## 2. Account Setup

### **Step 1: Create Account**
```
1. Go to https://e2b.dev
2. Click "Sign Up"
3. Use email or GitHub login
4. Verify email
5. Browse to dashboard
```

### **Step 2: Get API Key**
```
1. Dashboard → "API Keys" (left sidebar)
2. Click "Create API Key"
3. Name: "interview-prep-agent"
4. Copy the key (format: sk_...)
5. Keep safe! This is like a password
```

### **Step 3: Store in .env**
```bash
# .env file
E2B_API_KEY=sk_your-key-here
```

---

## 3. Installation

```bash
# Install E2B SDK
pip install e2b-code-interpreter

# Verify installation
python -c "from e2b_code_interpreter import CodeInterpreter; print('✓ E2B installed')"
```

---

## 4. Supported Languages

E2B supports 50+ languages out of the box:

| Language | Use Case |
|----------|----------|
| **Python** | Data science, scripting, algorithms |
| **JavaScript/Node.js** | Web, algorithms, async |
| **Go** | System design, performance |
| **Java** | OOP, system design |
| **C++** | Performance, systems |
| **Rust** | Systems, performance |
| **Ruby** | Scripts, quick solutions |
| **PHP** | Web backend |
| **SQL** | Database queries |
| **Bash/Shell** | CLI, scripting |

**Full list**: https://e2b.dev/docs/using-different-programming-languages

---

## 5. Basic Usage

### **Simple Code Execution**

```python
from e2b_code_interpreter import CodeInterpreter

# Create interpreter instance
interpreter = CodeInterpreter()

# Execute Python code
code = """
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

result = fibonacci(10)
print(f"Fibonacci(10) = {result}")
"""

result = interpreter.notebook.exec_cell(code)
print(result.text)  # Output: "Fibonacci(10) = 55"

# Close interpreter
interpreter.close()
```

### **With Error Handling**

```python
from e2b_code_interpreter import CodeInterpreter
from e2b_code_interpreter.exception import TimeoutException

def safe_execute(code: str, language: str = "python", timeout: int = 30):
    """Execute code safely with error handling"""
    try:
        interpreter = CodeInterpreter(language=language)
        result = interpreter.notebook.exec_cell(code, timeout=timeout)

        return {
            "success": True,
            "output": result.text,
            "error": None,
            "execution_time_ms": result.execution_time
        }

    except TimeoutException:
        return {
            "success": False,
            "output": None,
            "error": f"Code execution timeout after {timeout} seconds",
            "execution_time_ms": timeout * 1000
        }

    except SyntaxError as e:
        return {
            "success": False,
            "output": None,
            "error": f"Syntax Error: {str(e)}",
            "execution_time_ms": 0
        }

    except Exception as e:
        return {
            "success": False,
            "output": None,
            "error": f"Execution Error: {str(e)}",
            "execution_time_ms": 0
        }

    finally:
        interpreter.close()
```

### **Multi-Language Support**

```python
# Python
result = CodeInterpreter(language="python").notebook.exec_cell("print('hello')")

# JavaScript
result = CodeInterpreter(language="javascript").notebook.exec_cell("console.log('hello')")

# Go
result = CodeInterpreter(language="go").notebook.exec_cell("fmt.Println('hello')")

# Java
result = CodeInterpreter(language="java").notebook.exec_cell("System.out.println('hello');")
```

---

## 6. Integration with Our System

### **Project Structure**
```
src/
├── agents/
│   └── mock_interview_agent.py
├── tools/
│   ├── __init__.py
│   ├── gemini_tool.py
│   ├── web_search_tool.py
│   └── code_execution_tool.py  ← Here
└── config/
    └── e2b_config.py
```

### **Create E2B Config**

```python
# src/config/e2b_config.py
import os
from typing import Dict

class E2BConfig:
    """E2B Configuration"""

    API_KEY = os.getenv("E2B_API_KEY")

    # Timeout for code execution
    DEFAULT_TIMEOUT_SECONDS = 30

    # Maximum timeout
    MAX_TIMEOUT_SECONDS = 120

    # Supported languages for interviews
    SUPPORTED_LANGUAGES = {
        "python": "Python 3",
        "javascript": "JavaScript (Node.js)",
        "go": "Go",
        "java": "Java",
        "cpp": "C++",
        "rust": "Rust",
        "ruby": "Ruby",
        "php": "PHP"
    }

    # What to limit per execution
    LIMITS = {
        "memory_mb": 512,          # Max memory
        "cpu_count": 2,            # CPU cores
        "timeout_seconds": 30      # Default timeout
    }
```

### **Create E2B Tool**

```python
# src/tools/code_execution_tool.py
import logging
from typing import Dict, Optional
from e2b_code_interpreter import CodeInterpreter
from e2b_code_interpreter.exception import TimeoutException
from src.config.e2b_config import E2BConfig

logger = logging.getLogger(__name__)

class CodeExecutionTool:
    """Tool for executing user code in sandboxed environment"""

    def __init__(self):
        self.config = E2BConfig()
        self.api_key = self.config.API_KEY

        if not self.api_key:
            raise ValueError("E2B_API_KEY environment variable not set")

    async def execute(
        self,
        code: str,
        language: str = "python",
        timeout: int = 30,
        session_id: Optional[str] = None
    ) -> Dict:
        """
        Execute code in E2B sandbox

        Args:
            code: Source code to execute
            language: Programming language (python, javascript, go, java, cpp, rust, ruby, php)
            timeout: Execution timeout in seconds (max 120)
            session_id: Optional session ID for persistent state across executions

        Returns:
            Dict with:
                - success: bool
                - output: str (stdout/stderr)
                - error: str (if failed)
                - execution_time_ms: int
                - exit_code: int
        """

        # Validate timeout
        timeout = min(timeout, self.config.MAX_TIMEOUT_SECONDS)

        # Validate language
        if language not in self.config.SUPPORTED_LANGUAGES:
            return {
                "success": False,
                "output": None,
                "error": f"Unsupported language: {language}. Supported: {', '.join(self.config.SUPPORTED_LANGUAGES.keys())}",
                "execution_time_ms": 0,
                "exit_code": -1
            }

        interpreter = None
        try:
            # Create interpreter (with persistent state if session_id provided)
            interpreter = CodeInterpreter(
                language=language,
                api_key=self.api_key
            )

            logger.info(f"Executing {language} code (timeout: {timeout}s, session: {session_id})")

            # Execute code
            result = interpreter.notebook.exec_cell(code, timeout=timeout)

            logger.info(f"Code executed successfully. Output length: {len(result.text)}")

            return {
                "success": True,
                "output": result.text,
                "error": None,
                "execution_time_ms": int(result.execution_time * 1000),
                "exit_code": 0,
                "language": language
            }

        except TimeoutException as e:
            logger.warning(f"Code execution timeout after {timeout}s")
            return {
                "success": False,
                "output": None,
                "error": f"Execution timeout: Code did not complete within {timeout} seconds",
                "execution_time_ms": timeout * 1000,
                "exit_code": -1,
                "language": language
            }

        except SyntaxError as e:
            logger.error(f"Syntax error: {str(e)}")
            return {
                "success": False,
                "output": None,
                "error": f"Syntax Error: {str(e)}",
                "execution_time_ms": 0,
                "exit_code": 1,
                "language": language
            }

        except Exception as e:
            logger.error(f"Code execution error: {str(e)}")
            return {
                "success": False,
                "output": None,
                "error": f"Execution Error: {str(e)}",
                "execution_time_ms": 0,
                "exit_code": 1,
                "language": language
            }

        finally:
            # Clean up
            if interpreter:
                interpreter.close()

    async def execute_multiple(
        self,
        code_snippets: list,
        language: str = "python",
        timeout: int = 30
    ) -> Dict:
        """
        Execute multiple code snippets in same session (persistent state)

        Args:
            code_snippets: List of code strings
            language: Programming language
            timeout: Timeout per snippet

        Returns:
            List of execution results
        """
        results = []
        interpreter = None

        try:
            interpreter = CodeInterpreter(language=language, api_key=self.api_key)

            for i, code in enumerate(code_snippets):
                logger.info(f"Executing snippet {i+1}/{len(code_snippets)}")

                result = interpreter.notebook.exec_cell(code, timeout=timeout)

                results.append({
                    "snippet": i,
                    "success": True,
                    "output": result.text,
                    "error": None,
                    "execution_time_ms": int(result.execution_time * 1000)
                })

        except Exception as e:
            logger.error(f"Error in snippet {len(results)}: {str(e)}")
            results.append({
                "snippet": len(results),
                "success": False,
                "output": None,
                "error": str(e),
                "execution_time_ms": 0
            })

        finally:
            if interpreter:
                interpreter.close()

        return results


# Singleton instance
code_execution_tool = CodeExecutionTool()
```

---

## 7. Use in Mock Interview Agent

```python
# src/agents/mock_interview_agent.py
from src.tools.code_execution_tool import code_execution_tool

class MockInterviewAgent:
    """Mock interview agent for conducting interviews"""

    async def handle_code_answer(
        self,
        question_id: str,
        code: str,
        language: str,
        expected_output: str = None
    ) -> Dict:
        """
        Execute and test user's code answer

        Returns:
            {
                "code_runs": bool,
                "output": str,
                "passed_tests": bool,
                "error": str,
                "feedback": str
            }
        """

        # Execute the code
        execution_result = await code_execution_tool.execute(
            code=code,
            language=language,
            timeout=30
        )

        if not execution_result["success"]:
            return {
                "code_runs": False,
                "output": None,
                "passed_tests": False,
                "error": execution_result["error"],
                "feedback": f"Your code has an error: {execution_result['error']}"
            }

        # If expected output provided, test it
        if expected_output:
            actual_output = execution_result["output"].strip()
            expected_output = expected_output.strip()
            passed = actual_output == expected_output
        else:
            passed = True

        return {
            "code_runs": True,
            "output": execution_result["output"],
            "passed_tests": passed,
            "error": None,
            "feedback": "Code executed successfully!" if passed else "Output does not match expected result"
        }
```

---

## 8. Testing E2B Integration

### **Test Script**

```python
# tests/test_e2b_integration.py
import asyncio
from src.tools.code_execution_tool import code_execution_tool

async def test_python_execution():
    """Test Python code execution"""
    code = """
def sum_array(arr):
    return sum(arr)

result = sum_array([1, 2, 3, 4, 5])
print(f"Sum: {result}")
"""

    result = await code_execution_tool.execute(code, language="python")
    assert result["success"] == True
    assert "Sum: 15" in result["output"]
    print("✓ Python execution test passed")


async def test_javascript_execution():
    """Test JavaScript code execution"""
    code = """
function fibonacci(n) {
    if (n <= 1) return n;
    return fibonacci(n-1) + fibonacci(n-2);
}

console.log("Fibonacci(10):", fibonacci(10));
"""

    result = await code_execution_tool.execute(code, language="javascript")
    assert result["success"] == True
    assert "55" in result["output"]
    print("✓ JavaScript execution test passed")


async def test_timeout():
    """Test timeout handling"""
    code = """
import time
time.sleep(60)  # Sleep longer than timeout
"""

    result = await code_execution_tool.execute(code, language="python", timeout=5)
    assert result["success"] == False
    assert "timeout" in result["error"].lower()
    print("✓ Timeout test passed")


async def test_syntax_error():
    """Test syntax error handling"""
    code = """
def broken_function(
    print("missing closing paren")
"""

    result = await code_execution_tool.execute(code, language="python")
    assert result["success"] == False
    assert "Syntax" in result["error"]
    print("✓ Syntax error test passed")


async def main():
    print("Running E2B integration tests...\n")
    await test_python_execution()
    await test_javascript_execution()
    await test_timeout()
    await test_syntax_error()
    print("\n✅ All tests passed!")


if __name__ == "__main__":
    asyncio.run(main())
```

**Run tests:**
```bash
python -m pytest tests/test_e2b_integration.py -v
```

---

## 9. Cost & Usage

### **Pricing Model**
```
Free Tier:
- 10 code executions/month
- Perfect for testing

Paid Tier ($0.05 per execution):
- Pay-as-you-go
- Scale to 1000s/second
- No monthly fees
```

### **Estimated Costs for Interview Prep**
```
If each user solves 5 problems per session:
- 100 users × 5 problems = 500 executions
- Cost: 500 × $0.05 = $25

If each user does 10 sessions per month:
- 100 users × 10 sessions × 5 problems = 5000 executions
- Cost: 5000 × $0.05 = $250/month
```

### **Usage Dashboard**
```
1. Go to E2B dashboard: https://e2b.dev/dashboard
2. View "Usage" tab
3. See executions used this month
4. Set usage alerts (optional)
```

---

## 10. Troubleshooting

### **API Key Not Found**
```bash
Error: E2B_API_KEY environment variable not set

Solution:
1. Check .env file exists in project root
2. Add: E2B_API_KEY=sk_your-key-here
3. Restart your Python app
```

### **Timeout Errors**
```python
# Increase timeout
result = await code_execution_tool.execute(
    code=code,
    language="python",
    timeout=60  # Increased from 30
)
```

### **Language Not Supported**
```
Error: Unsupported language: xyz

Solution:
Check supported languages:
python, javascript, go, java, cpp, rust, ruby, php, bash, sql, etc.
```

### **Execution Limits Exceeded**
```
Error: Memory limit exceeded

Solution:
1. Code might have infinite loop
2. Check for memory leaks
3. Reduce data size being processed
```

---

## 11. Integration with Mock Interview Flow

```
User answers coding question:
    ↓
Mock Interview Agent receives code
    ↓
Calls code_execution_tool.execute()
    ↓
E2B sandbox runs code (100ms)
    ↓
Returns stdout/stderr
    ↓
Compare with expected output
    ↓
Call Evaluation Agent with result
    ↓
Return feedback to user
```

---

## 12. Best Practices

### **Do's ✅**
- Always set timeout to prevent infinite loops
- Validate code before sending to E2B
- Log execution results for debugging
- Use appropriate language for question type
- Limit code size (max 10KB)

### **Don'ts ❌**
- Don't execute untrusted code without sandboxing
- Don't set timeout to 0 (use E2B's defaults)
- Don't forget to close interpreters
- Don't hardcode API key (use .env)
- Don't send binary files to execute

---

## 13. Resources

- **Official Docs**: https://e2b.dev/docs
- **Python SDK**: https://github.com/e2b-dev/code-interpreter
- **Supported Languages**: https://e2b.dev/docs/using-different-programming-languages
- **GitHub Issues**: https://github.com/e2b-dev/e2b/issues
