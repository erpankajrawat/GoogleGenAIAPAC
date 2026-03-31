# JWT Authentication Implementation Guide

**Document:** `docs/integration/JWT_AUTHENTICATION.md`

---

## 1. JWT Overview

**JWT** = JSON Web Token

- **Format**: `header.payload.signature`
- **Stateless**: No server-side session storage needed
- **Secure**: Cryptographically signed
- **Standard**: RFC 7519

**Example:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJ1c2VyX2lkIjoidXNlci0xMjMiLCJleHAiOjE2NDQ0NzE4MDB9.
ZhQlmQgQXvU5UbJxT5W-c5e7eQz9dW3mKvU7rJ8n4Ys
```

---

## 2. Installation

```bash
# Install PyJWT
pip install pyjwt

# Verify
python -c "import jwt; print('✓ PyJWT installed')"
```

---

## 3. Basic Usage

### **Create a Token**

```python
import jwt
import datetime

# Secret key (store in .env!)
SECRET_KEY = "your-secret-key-min-32-chars"

# Create payload
payload = {
    "user_id": "user-123",
    "email": "user@example.com",
    "exp": datetime.datetime.utcnow() + datetime.timedelta(hours=24),
    "iat": datetime.datetime.utcnow()
}

# Encode to token
token = jwt.encode(payload, SECRET_KEY, algorithm="HS256")
print(token)  # Output: eyJhbGc...
```

### **Verify a Token**

```python
import jwt

# Verify and decode
try:
    payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    print(payload)  # Output: {"user_id": "user-123", ...}

except jwt.ExpiredSignatureError:
    print("Token expired")

except jwt.InvalidTokenError:
    print("Invalid token")
```

---

## 4. Project Setup

### **Project Structure**
```
src/
├── auth/
│   ├── __init__.py
│   ├── jwt_utils.py        ← Core JWT functions
│   └── dependencies.py      ← FastAPI dependencies
├── config/
│   └── auth_config.py       ← Auth configuration
├── main.py                  ← FastAPI app
└── models/
    └── auth.py              ← Auth request/response models
```

### **Auth Configuration**

```python
# src/config/auth_config.py
import os
from datetime import timedelta

class AuthConfig:
    """JWT Authentication Configuration"""

    # Secret key (should be long and random)
    SECRET_KEY = os.getenv(
        "JWT_SECRET_KEY",
        "default-secret-key-change-in-production"  # NEVER use in production!
    )

    # Algorithm
    ALGORITHM = "HS256"

    # Token expiry
    ACCESS_TOKEN_EXPIRY = timedelta(hours=24)

    # Minimum password length (for future use)
    MIN_PASSWORD_LENGTH = 8

    # Allowed roles (for RBAC)
    ALLOWED_ROLES = ["user", "admin", "moderator"]
```

### **JWT Utils**

```python
# src/auth/jwt_utils.py
import jwt
import logging
from datetime import datetime, timedelta
from typing import Dict, Optional
from src.config.auth_config import AuthConfig

logger = logging.getLogger(__name__)

class JWTHandler:
    """Handle JWT token creation and verification"""

    @staticmethod
    def create_token(
        user_id: str,
        email: Optional[str] = None,
        role: str = "user"
    ) -> str:
        """
        Create JWT token

        Args:
            user_id: Unique user identifier
            email: User email (optional)
            role: User role (user, admin, etc.)

        Returns:
            JWT token string
        """

        payload = {
            "user_id": user_id,
            "email": email,
            "role": role,
            "iat": datetime.utcnow(),
            "exp": datetime.utcnow() + AuthConfig.ACCESS_TOKEN_EXPIRY
        }

        token = jwt.encode(
            payload,
            AuthConfig.SECRET_KEY,
            algorithm=AuthConfig.ALGORITHM
        )

        logger.info(f"Created token for user: {user_id}")
        return token

    @staticmethod
    def verify_token(token: str) -> Dict:
        """
        Verify and decode JWT token

        Returns:
            {
                "valid": bool,
                "user_id": str,
                "email": str,
                "role": str,
                "error": str (if invalid)
            }
        """

        try:
            payload = jwt.decode(
                token,
                AuthConfig.SECRET_KEY,
                algorithms=[AuthConfig.ALGORITHM]
            )

            return {
                "valid": True,
                "user_id": payload.get("user_id"),
                "email": payload.get("email"),
                "role": payload.get("role", "user"),
                "error": None
            }

        except jwt.ExpiredSignatureError:
            logger.warning(f"Token expired")
            return {
                "valid": False,
                "user_id": None,
                "email": None,
                "role": None,
                "error": "Token has expired"
            }

        except jwt.InvalidTokenError as e:
            logger.warning(f"Invalid token: {str(e)}")
            return {
                "valid": False,
                "user_id": None,
                "email": None,
                "role": None,
                "error": "Invalid token"
            }

        except Exception as e:
            logger.error(f"Token verification error: {str(e)}")
            return {
                "valid": False,
                "user_id": None,
                "email": None,
                "role": None,
                "error": str(e)
            }

    @staticmethod
    def refresh_token(token: str) -> Dict:
        """
        Refresh an existing token

        Returns:
            {
                "success": bool,
                "new_token": str,
                "error": str (if failed)
            }
        """

        # Verify current token
        result = JWTHandler.verify_token(token)

        if not result["valid"]:
            return {
                "success": False,
                "new_token": None,
                "error": result["error"]
            }

        # Create new token with same claims
        new_token = JWTHandler.create_token(
            user_id=result["user_id"],
            email=result["email"],
            role=result["role"]
        )

        return {
            "success": True,
            "new_token": new_token,
            "error": None
        }
```

### **FastAPI Dependencies**

```python
# src/auth/dependencies.py
from typing import Dict
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthCredentials
from src.auth.jwt_utils import JWTHandler

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthCredentials = Depends(security)
) -> Dict:
    """
    Dependency for protected endpoints

    Usage:
        @app.get("/protected")
        async def protected_route(user: Dict = Depends(get_current_user)):
            return {"user_id": user["user_id"]}
    """

    token = credentials.credentials

    # Verify token
    result = JWTHandler.verify_token(token)

    if not result["valid"]:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=result["error"],
            headers={"WWW-Authenticate": "Bearer"},
        )

    return result


async def get_current_admin(
    user: Dict = Depends(get_current_user)
) -> Dict:
    """
    Dependency for admin-only endpoints

    Usage:
        @app.delete("/admin/users/{user_id}")
        async def delete_user(admin: Dict = Depends(get_current_admin)):
            # Only admins can access
            pass
    """

    if user["role"] != "admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required"
        )

    return user
```

### **Auth Models**

```python
# src/models/auth.py
from pydantic import BaseModel, Field
from typing import Optional

class LoginRequest(BaseModel):
    """Login request model"""
    user_id: str = Field(..., min_length=1, max_length=256)
    email: Optional[str] = Field(None, max_length=256)

    class Config:
        example = {
            "user_id": "user-123",
            "email": "user@example.com"
        }


class LoginResponse(BaseModel):
    """Login response model"""
    access_token: str
    token_type: str = "bearer"
    expires_in_hours: int = 24

    class Config:
        example = {
            "access_token": "eyJhbGc...",
            "token_type": "bearer",
            "expires_in_hours": 24
        }


class RefreshTokenRequest(BaseModel):
    """Refresh token request"""
    token: str


class RefreshTokenResponse(BaseModel):
    """Refresh token response"""
    access_token: str
    token_type: str = "bearer"


class UserInfo(BaseModel):
    """Current user info"""
    user_id: str
    email: Optional[str]
    role: str
```

---

## 5. FastAPI Integration

### **Create Endpoints**

```python
# src/main.py
from fastapi import FastAPI, Depends, HTTPException, status
from src.auth.jwt_utils import JWTHandler
from src.auth.dependencies import get_current_user, get_current_admin
from src.models.auth import (
    LoginRequest, LoginResponse,
    RefreshTokenRequest, RefreshTokenResponse,
    UserInfo
)

app = FastAPI()

# ============ Auth Endpoints ============

@app.post("/login", response_model=LoginResponse)
async def login(request: LoginRequest):
    """
    User login endpoint

    Returns JWT access token
    """

    # In production, validate user credentials here
    # For now, just create token

    token = JWTHandler.create_token(
        user_id=request.user_id,
        email=request.email,
        role="user"
    )

    return LoginResponse(
        access_token=token,
        expires_in_hours=24
    )


@app.post("/refresh-token", response_model=RefreshTokenResponse)
async def refresh_token(request: RefreshTokenRequest):
    """
    Refresh JWT token
    """

    result = JWTHandler.refresh_token(request.token)

    if not result["success"]:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=result["error"]
        )

    return RefreshTokenResponse(access_token=result["new_token"])


@app.get("/me", response_model=UserInfo)
async def get_current_user_info(user = Depends(get_current_user)):
    """Get current user information"""

    return UserInfo(
        user_id=user["user_id"],
        email=user["email"],
        role=user["role"]
    )


# ============ Protected Endpoints ============

@app.post("/execute-goal")
async def execute_goal(
    request: ExecuteGoalRequest,
    user: Dict = Depends(get_current_user)
):
    """
    Execute user goal (protected by JWT)
    """

    # User ID is available from JWT
    user_id = user["user_id"]

    # Process goal...
    return {
        "status": "success",
        "user_id": user_id,
        "goal": request.goal
    }


@app.delete("/admin/users/{user_id}")
async def admin_delete_user(
    user_id: str,
    admin: Dict = Depends(get_current_admin)
):
    """
    Admin-only endpoint to delete user
    """

    # Only admins can reach here
    return {
        "deleted": user_id,
        "by_admin": admin["user_id"]
    }
```

---

## 6. Usage Examples

### **Example 1: User Login & Request**

```bash
# Step 1: Login
curl -X POST http://localhost:8000/login \
  -H "Content-Type: application/json" \
  -d '{"user_id": "user-123", "email": "user@example.com"}'

# Response:
# {
#   "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
#   "token_type": "bearer",
#   "expires_in_hours": 24
# }

# Step 2: Use token in requests
curl -X POST http://localhost:8000/execute-goal \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "goal": "create-study-plan",
    "params": {"role": "Python Developer", "days": 5}
  }'

# Response:
# {
#   "status": "success",
#   "user_id": "user-123",
#   "goal": "create-study-plan"
# }
```

### **Example 2: Token Expiry & Refresh**

```bash
# If token expired, refresh it
curl -X POST http://localhost:8000/refresh-token \
  -H "Content-Type: application/json" \
  -d '{"token": "expired_token_here"}'

# Response:
# {
#   "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
#   "token_type": "bearer"
# }
```

---

## 7. Configuration

### **.env File**

```bash
# JWT Security
JWT_SECRET_KEY=your-super-secret-key-min-32-chars-unique-string

# Token expiry (in hours)
JWT_EXPIRY_HOURS=24

# Algorithm (HS256 recommended)
JWT_ALGORITHM=HS256
```

### **Generate Secure Secret Key**

```python
import secrets
import string

# Generate 32-character random secret
secret = secrets.token_urlsafe(32)
print(f"JWT_SECRET_KEY={secret}")

# Output: JWT_SECRET_KEY=AbCdEfGhIjKlMnOpQrStUvWxYz1234567
```

---

## 8. Testing

### **Test Script**

```python
# tests/test_jwt_auth.py
import pytest
from src.auth.jwt_utils import JWTHandler
import jwt
from src.config.auth_config import AuthConfig

def test_create_token():
    """Test token creation"""
    token = JWTHandler.create_token(
        user_id="user-123",
        email="user@example.com",
        role="user"
    )

    assert isinstance(token, str)
    assert len(token.split('.')) == 3  # JWT has 3 parts
    print("✓ Token creation test passed")


def test_verify_token():
    """Test token verification"""
    token = JWTHandler.create_token("user-123")
    result = JWTHandler.verify_token(token)

    assert result["valid"] == True
    assert result["user_id"] == "user-123"
    assert result["error"] == None
    print("✓ Token verification test passed")


def test_expired_token():
    """Test expired token handling"""
    # Create token with 0 expiry
    payload = {
        "user_id": "user-123",
        "iat": 1,
        "exp": 1  # Already expired
    }

    token = jwt.encode(
        payload,
        AuthConfig.SECRET_KEY,
        algorithm=AuthConfig.ALGORITHM
    )

    result = JWTHandler.verify_token(token)

    assert result["valid"] == False
    assert "expired" in result["error"].lower()
    print("✓ Expired token test passed")


def test_invalid_token():
    """Test invalid token handling"""
    result = JWTHandler.verify_token("invalid.token.here")

    assert result["valid"] == False
    assert result["error"] is not None
    print("✓ Invalid token test passed")


def test_refresh_token():
    """Test token refresh"""
    token = JWTHandler.create_token("user-123", email="user@example.com")
    result = JWTHandler.refresh_token(token)

    assert result["success"] == True
    assert result["new_token"] is not None

    # Verify new token
    verify_result = JWTHandler.verify_token(result["new_token"])
    assert verify_result["valid"] == True
    assert verify_result["user_id"] == "user-123"
    print("✓ Token refresh test passed")


if __name__ == "__main__":
    test_create_token()
    test_verify_token()
    test_expired_token()
    test_invalid_token()
    test_refresh_token()
    print("\n✅ All JWT tests passed!")
```

Run tests:
```bash
python -m pytest tests/test_jwt_auth.py -v
```

---

## 9. Security Best Practices

### **Do's ✅**
- Store SECRET_KEY in environment variable
- Use HTTPS in production (never HTTP)
- Set reasonable token expiry (24h recommended)
- Use HS256 algorithm minimum
- Validate token on every protected endpoint

### **Don'ts ❌**
- Never hardcode SECRET_KEY in code
- Don't send tokens in URL query strings
- Don't use weak secrets (min 32 chars)
- Don't skip HTTPS in production
- Don't store sensitive data in token payload

---

## 10. Migration Path to OAuth2

In the future, you can upgrade to OAuth2 without breaking changes:

```python
# Future: just replace dependencies
from fastapi_oauth2 import OAuth2PasswordBearer, oauth2_scheme

# Your code stays the same:
@app.get("/me")
async def get_user(user = Depends(get_current_user)):
    # This works with both JWT and OAuth2!
    return user
```

---

## 11. Resources

- **PyJWT Docs**: https://pyjwt.readthedocs.io
- **JWT.io**: https://jwt.io (decode/verify tokens)
- **RFC 7519**: https://tools.ietf.org/html/rfc7519
- **FastAPI Security**: https://fastapi.tiangolo.com/tutorial/security/
