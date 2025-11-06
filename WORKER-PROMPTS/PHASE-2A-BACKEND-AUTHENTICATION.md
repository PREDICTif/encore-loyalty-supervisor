# Phase 2A: Backend Authentication Implementation
## JWT Authentication System for Encore Loyalty

**Assigned To**: Cursor Chat (Claude Sonnet 4.5)
**Phase**: 2A of 8 - Backend Authentication
**Duration**: 3-4 hours
**Date**: November 6, 2025 (or whenever Phase 1 + Backend Setup complete)
**Priority**: P1 - Critical (blocks all other backend development)
**Parallel**: Can work alongside Phase 2B (Frontend Auth)

---

## 🎯 MISSION

Implement JWT-based authentication system for the FastAPI backend, replacing the need for API keys with secure user authentication.

**This Phase**: Create authentication endpoints, JWT token management, user authentication against MySQL database.

---

## 📋 CONTEXT

### Current State

**Backend Status**:
- ✅ FastAPI running on port 8000
- ✅ MySQL connection via SSH tunnel
- ✅ Basic endpoints working (health, clients)
- ❌ No authentication system
- ❌ Currently using API_KEY or no auth (REQUIRE_AUTH=false)

**Database Available**:
- MySQL database: `encoree3_mobile`
- User table exists with credentials
- Need to query and validate against this table

**What Frontend Expects** (from mockup analysis):
```typescript
// Login request
{ email: string, password: string }

// Login response
{
  token: string,
  user: {
    id: string,
    email: string,
    name: string,
    role: 'super_admin' | 'global_admin' | 'venue_manager',
    venueId?: string  // if venue_manager
  }
}
```

---

## 🎯 PHASE 2A OBJECTIVES

1. ✅ Install JWT dependencies
2. ✅ Create JWT service (generate, verify, refresh tokens)
3. ✅ Create User model (from MySQL schema)
4. ✅ Implement 4 authentication endpoints
5. ✅ Create authentication dependency (protect routes)
6. ✅ Test all endpoints with Swagger UI
7. ✅ Document user roles and permissions

---

## 📁 PROJECT STRUCTURE

```
/home/ec2-user/encore-loyalty-new-system/
├── venv/                          ← Python virtual environment
└── encore_backend/
    ├── .env                       ← Environment config (exists)
    ├── main.py                    ← FastAPI app
    ├── config.py                  ← Settings
    ├── routes/
    │   ├── __init__.py
    │   ├── auth.py               ← CREATE THIS (auth endpoints)
    │   ├── health.py
    │   ├── clients.py
    │   └── ...
    ├── services/
    │   ├── jwt_service.py        ← CREATE THIS (JWT logic)
    │   └── auth_service.py       ← CREATE THIS (user validation)
    ├── models/
    │   └── user.py               ← CREATE THIS (user model)
    ├── dependencies.py            ← UPDATE THIS (add auth dependency)
    └── requirements.txt           ← UPDATE THIS (add JWT packages)
```

---

## 🛠️ DETAILED TASKS

### Task 1: Install JWT Dependencies

**Objective**: Add required packages for JWT authentication.

**File**: `encore_backend/requirements.txt`

**Add these lines**:
```txt
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.6
```

**Install**:
```bash
cd /home/ec2-user/encore-loyalty-new-system
source venv/bin/activate
cd encore_backend
pip install python-jose[cryptography] passlib[bcrypt] python-multipart
```

**Verification**:
```bash
pip list | grep -E "(jose|passlib|multipart)"
```

Expected output:
```
passlib                1.7.4
python-jose            3.3.0
python-multipart       0.0.6
```

---

### Task 2: Update Configuration

**Objective**: Add JWT configuration settings.

**File**: `encore_backend/config.py`

**Find the Settings class and add these new fields**:

```python
class Settings(BaseSettings):
    # ... existing settings ...

    # JWT Authentication Settings
    jwt_secret_key: str = "your-secret-key-change-in-production-min-32-chars"
    jwt_algorithm: str = "HS256"
    jwt_access_token_expire_minutes: int = 15  # Short-lived access tokens
    jwt_refresh_token_expire_days: int = 7     # Long-lived refresh tokens

    # Password hashing
    pwd_context_schemes: list = ["bcrypt"]
    pwd_context_deprecated: str = "auto"
```

**Important**:
- For production, `jwt_secret_key` should be set via environment variable
- Must be at least 32 characters
- Keep it secret!

**Verification**:
- [ ] Settings class updated
- [ ] No syntax errors
- [ ] Config loads correctly

---

### Task 3: Create User Model

**Objective**: Define User Pydantic model based on MySQL user table.

**File**: `encore_backend/models/user.py`

```python
"""
User models for authentication and authorization.

Author: AI Assistant
Date: 2025-11-06
"""

from pydantic import BaseModel, EmailStr
from typing import Optional, Literal
from datetime import datetime


class UserBase(BaseModel):
    """Base user model with common fields"""
    email: EmailStr
    name: str
    role: Literal['super_admin', 'global_admin', 'venue_manager']


class UserInDB(UserBase):
    """User model as stored in database"""
    id: int
    hashed_password: str
    venue_id: Optional[int] = None
    is_active: bool = True
    created_at: Optional[datetime] = None

    class Config:
        from_attributes = True


class User(UserBase):
    """User model for API responses (without password)"""
    id: int
    venue_id: Optional[int] = None
    is_active: bool = True

    class Config:
        from_attributes = True


class UserLogin(BaseModel):
    """Login request model"""
    email: EmailStr
    password: str


class Token(BaseModel):
    """JWT token response"""
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


class TokenData(BaseModel):
    """Data encoded in JWT token"""
    user_id: int
    email: str
    role: str
    venue_id: Optional[int] = None


class LoginResponse(BaseModel):
    """Complete login response"""
    token: str  # access_token
    refresh_token: str
    user: User
```

**Verification**:
- [ ] File created at `models/user.py`
- [ ] All models defined
- [ ] No import errors

---

### Task 4: Create JWT Service

**Objective**: Implement JWT token generation, verification, and refresh logic.

**File**: `encore_backend/services/jwt_service.py`

```python
"""
JWT Service for token generation and validation.

Handles:
- Access token generation (short-lived, 15 minutes)
- Refresh token generation (long-lived, 7 days)
- Token verification and decoding
- Token refresh logic

Author: AI Assistant
Date: 2025-11-06
"""

from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from config import settings
from models.user import TokenData
import logging

logger = logging.getLogger(__name__)

# Password hashing context
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


class JWTService:
    """Service for JWT token operations"""

    def __init__(self):
        self.secret_key = settings.jwt_secret_key
        self.algorithm = settings.jwt_algorithm
        self.access_token_expire = settings.jwt_access_token_expire_minutes
        self.refresh_token_expire = settings.jwt_refresh_token_expire_days

    def verify_password(self, plain_password: str, hashed_password: str) -> bool:
        """
        Verify a plain password against a hashed password.

        Args:
            plain_password: Password to verify
            hashed_password: Hashed password from database

        Returns:
            True if password matches, False otherwise
        """
        return pwd_context.verify(plain_password, hashed_password)

    def get_password_hash(self, password: str) -> str:
        """
        Hash a password for storage.

        Args:
            password: Plain text password

        Returns:
            Hashed password
        """
        return pwd_context.hash(password)

    def create_access_token(self, data: dict) -> str:
        """
        Create a short-lived access token (15 minutes).

        Args:
            data: Token payload (user_id, email, role, etc.)

        Returns:
            Encoded JWT token
        """
        to_encode = data.copy()
        expire = datetime.utcnow() + timedelta(minutes=self.access_token_expire)
        to_encode.update({
            "exp": expire,
            "type": "access"
        })

        encoded_jwt = jwt.encode(to_encode, self.secret_key, algorithm=self.algorithm)
        logger.debug(f"Created access token for user {data.get('user_id')}")
        return encoded_jwt

    def create_refresh_token(self, data: dict) -> str:
        """
        Create a long-lived refresh token (7 days).

        Args:
            data: Token payload (user_id, email)

        Returns:
            Encoded JWT refresh token
        """
        to_encode = data.copy()
        expire = datetime.utcnow() + timedelta(days=self.refresh_token_expire)
        to_encode.update({
            "exp": expire,
            "type": "refresh"
        })

        encoded_jwt = jwt.encode(to_encode, self.secret_key, algorithm=self.algorithm)
        logger.debug(f"Created refresh token for user {data.get('user_id')}")
        return encoded_jwt

    def verify_token(self, token: str, token_type: str = "access") -> Optional[TokenData]:
        """
        Verify and decode a JWT token.

        Args:
            token: JWT token to verify
            token_type: Expected token type ("access" or "refresh")

        Returns:
            TokenData if valid, None if invalid
        """
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])

            # Verify token type
            if payload.get("type") != token_type:
                logger.warning(f"Invalid token type: expected {token_type}, got {payload.get('type')}")
                return None

            # Extract token data
            user_id: int = payload.get("user_id")
            email: str = payload.get("email")
            role: str = payload.get("role")
            venue_id: Optional[int] = payload.get("venue_id")

            if user_id is None or email is None:
                logger.warning("Token missing required fields")
                return None

            token_data = TokenData(
                user_id=user_id,
                email=email,
                role=role,
                venue_id=venue_id
            )

            return token_data

        except JWTError as e:
            logger.error(f"JWT verification failed: {e}")
            return None


# Singleton instance
jwt_service = JWTService()
```

**Verification**:
- [ ] File created at `services/jwt_service.py`
- [ ] All methods implemented
- [ ] No import errors
- [ ] Can import jwt_service

---

### Task 5: Create Auth Service

**Objective**: Implement user authentication logic against MySQL database.

**File**: `encore_backend/services/auth_service.py`

```python
"""
Authentication Service for user validation.

Handles:
- User lookup in MySQL database
- Password verification
- User role and venue assignment

Author: AI Assistant
Date: 2025-11-06
"""

from typing import Optional
import logging
from models.user import User, UserInDB
from services.jwt_service import jwt_service
from db.mysql_service import MySQLService

logger = logging.getLogger(__name__)


class AuthService:
    """Service for user authentication operations"""

    def __init__(self):
        self.mysql = MySQLService()

    async def authenticate_user(self, email: str, password: str) -> Optional[User]:
        """
        Authenticate a user by email and password.

        Args:
            email: User email
            password: Plain text password

        Returns:
            User object if authentication succeeds, None otherwise
        """
        try:
            # Get user from database
            user_db = await self.get_user_by_email(email)

            if not user_db:
                logger.warning(f"User not found: {email}")
                return None

            # Verify password
            if not jwt_service.verify_password(password, user_db.hashed_password):
                logger.warning(f"Invalid password for user: {email}")
                return None

            # Check if user is active
            if not user_db.is_active:
                logger.warning(f"Inactive user attempted login: {email}")
                return None

            # Return user without password
            user = User(
                id=user_db.id,
                email=user_db.email,
                name=user_db.name,
                role=user_db.role,
                venue_id=user_db.venue_id,
                is_active=user_db.is_active
            )

            logger.info(f"User authenticated successfully: {email}")
            return user

        except Exception as e:
            logger.error(f"Authentication error for {email}: {e}")
            return None

    async def get_user_by_email(self, email: str) -> Optional[UserInDB]:
        """
        Get user from database by email.

        Args:
            email: User email address

        Returns:
            UserInDB if found, None otherwise
        """
        try:
            # TODO: Update this query based on actual MySQL table structure
            # This is a placeholder - adjust column names to match your schema
            query = """
                SELECT
                    id,
                    email,
                    name,
                    password as hashed_password,
                    role,
                    venue_id,
                    is_active,
                    created_at
                FROM users
                WHERE email = %s
                LIMIT 1
            """

            result = await self.mysql.execute_query(query, (email,))

            if not result or len(result) == 0:
                return None

            row = result[0]

            # Map database row to UserInDB model
            user_db = UserInDB(
                id=row['id'],
                email=row['email'],
                name=row['name'],
                hashed_password=row['hashed_password'],
                role=row['role'],
                venue_id=row.get('venue_id'),
                is_active=row.get('is_active', True),
                created_at=row.get('created_at')
            )

            return user_db

        except Exception as e:
            logger.error(f"Error getting user by email {email}: {e}")
            return None

    async def get_user_by_id(self, user_id: int) -> Optional[User]:
        """
        Get user from database by ID.

        Args:
            user_id: User ID

        Returns:
            User if found, None otherwise
        """
        try:
            query = """
                SELECT
                    id,
                    email,
                    name,
                    role,
                    venue_id,
                    is_active
                FROM users
                WHERE id = %s
                LIMIT 1
            """

            result = await self.mysql.execute_query(query, (user_id,))

            if not result or len(result) == 0:
                return None

            row = result[0]

            user = User(
                id=row['id'],
                email=row['email'],
                name=row['name'],
                role=row['role'],
                venue_id=row.get('venue_id'),
                is_active=row.get('is_active', True)
            )

            return user

        except Exception as e:
            logger.error(f"Error getting user by ID {user_id}: {e}")
            return None


# Singleton instance
auth_service = AuthService()
```

**IMPORTANT NOTE**:
- The SQL query assumes a `users` table exists
- **You must update the query** to match your actual MySQL schema
- Check the table name and column names in your database
- The mockup had demo users - you may need to create a users table or use existing authentication table

**Verification**:
- [ ] File created
- [ ] SQL query updated for your schema
- [ ] No import errors

---

### Task 6: Create Authentication Dependency

**Objective**: Create FastAPI dependency to protect routes that require authentication.

**File**: `encore_backend/dependencies.py`

**Add this function** (keep existing dependencies):

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from services.jwt_service import jwt_service
from services.auth_service import auth_service
from models.user import User, TokenData
import logging

logger = logging.getLogger(__name__)

# Security scheme for Swagger UI
security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> User:
    """
    Dependency to get current authenticated user from JWT token.

    Usage in routes:
        @app.get("/protected")
        async def protected_route(current_user: User = Depends(get_current_user)):
            return {"user": current_user.email}

    Args:
        credentials: JWT token from Authorization header

    Returns:
        Current authenticated User

    Raises:
        HTTPException 401 if token is invalid or user not found
    """
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        # Extract token from credentials
        token = credentials.credentials

        # Verify and decode token
        token_data: TokenData = jwt_service.verify_token(token, token_type="access")

        if token_data is None:
            logger.warning("Invalid token provided")
            raise credentials_exception

        # Get user from database
        user = await auth_service.get_user_by_id(token_data.user_id)

        if user is None:
            logger.warning(f"User not found for token: {token_data.user_id}")
            raise credentials_exception

        if not user.is_active:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Inactive user"
            )

        return user

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error in get_current_user: {e}")
        raise credentials_exception


async def get_current_active_admin(
    current_user: User = Depends(get_current_user)
) -> User:
    """
    Dependency to require admin role (super_admin or global_admin).

    Args:
        current_user: Current authenticated user

    Returns:
        User if admin, raises 403 otherwise
    """
    if current_user.role not in ['super_admin', 'global_admin']:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required"
        )
    return current_user


async def get_current_super_admin(
    current_user: User = Depends(get_current_user)
) -> User:
    """
    Dependency to require super admin role.

    Args:
        current_user: Current authenticated user

    Returns:
        User if super_admin, raises 403 otherwise
    """
    if current_user.role != 'super_admin':
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Super admin access required"
        )
    return current_user
```

**Verification**:
- [ ] Functions added to `dependencies.py`
- [ ] No import errors
- [ ] Can import dependencies

---

### Task 7: Create Authentication Routes

**Objective**: Implement 4 authentication endpoints.

**File**: `encore_backend/routes/auth.py`

```python
"""
Authentication Routes

Endpoints:
- POST /api/v1/auth/login - Login with email/password
- POST /api/v1/auth/logout - Logout (invalidate token)
- POST /api/v1/auth/refresh - Refresh access token
- GET /api/v1/auth/me - Get current user info

Author: AI Assistant
Date: 2025-11-06
"""

from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import HTTPBearer
from models.user import UserLogin, LoginResponse, User, Token
from services.auth_service import auth_service
from services.jwt_service import jwt_service
from dependencies import get_current_user
import logging

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/v1/auth", tags=["Authentication"])
security = HTTPBearer()


@router.post("/login", response_model=LoginResponse)
async def login(credentials: UserLogin):
    """
    Login with email and password.

    Returns JWT access token, refresh token, and user information.

    Args:
        credentials: Email and password

    Returns:
        LoginResponse with tokens and user data

    Raises:
        401 if credentials are invalid
    """
    # Authenticate user
    user = await auth_service.authenticate_user(
        credentials.email,
        credentials.password
    )

    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # Create token payload
    token_data = {
        "user_id": user.id,
        "email": user.email,
        "role": user.role,
        "venue_id": user.venue_id
    }

    # Generate tokens
    access_token = jwt_service.create_access_token(token_data)
    refresh_token = jwt_service.create_refresh_token(token_data)

    logger.info(f"User logged in: {user.email}")

    return LoginResponse(
        token=access_token,
        refresh_token=refresh_token,
        user=user
    )


@router.post("/logout")
async def logout(current_user: User = Depends(get_current_user)):
    """
    Logout current user.

    Note: Since we're using stateless JWT, actual invalidation would require
    a token blacklist. For MVP, this is a placeholder that confirms logout.

    Returns:
        Success message
    """
    logger.info(f"User logged out: {current_user.email}")

    return {
        "message": "Successfully logged out",
        "detail": "Please discard your access and refresh tokens"
    }


@router.post("/refresh", response_model=Token)
async def refresh_token(refresh_token: str):
    """
    Refresh access token using refresh token.

    Args:
        refresh_token: Valid refresh token

    Returns:
        New access token and refresh token

    Raises:
        401 if refresh token is invalid
    """
    # Verify refresh token
    token_data = jwt_service.verify_token(refresh_token, token_type="refresh")

    if not token_data:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid refresh token",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # Get user to ensure they still exist and are active
    user = await auth_service.get_user_by_id(token_data.user_id)

    if not user or not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User no longer active"
        )

    # Create new tokens
    new_token_data = {
        "user_id": user.id,
        "email": user.email,
        "role": user.role,
        "venue_id": user.venue_id
    }

    new_access_token = jwt_service.create_access_token(new_token_data)
    new_refresh_token = jwt_service.create_refresh_token(new_token_data)

    logger.info(f"Token refreshed for user: {user.email}")

    return Token(
        access_token=new_access_token,
        refresh_token=new_refresh_token,
        token_type="bearer"
    )


@router.get("/me", response_model=User)
async def get_current_user_info(current_user: User = Depends(get_current_user)):
    """
    Get current authenticated user information.

    Requires valid JWT access token in Authorization header.

    Returns:
        Current user data
    """
    return current_user
```

**Verification**:
- [ ] File created at `routes/auth.py`
- [ ] All 4 endpoints defined
- [ ] No syntax errors

---

### Task 8: Register Auth Routes

**Objective**: Add auth routes to the FastAPI app.

**File**: `encore_backend/routes/__init__.py`

**Update** to include auth router:

```python
from fastapi import APIRouter
from routes import health, clients, onboard, comments, auth

api_router = APIRouter()

# Include all route modules
api_router.include_router(health.router)
api_router.include_router(auth.router)      # ADD THIS LINE
api_router.include_router(clients.router)
api_router.include_router(onboard.router)
api_router.include_router(comments.router)
```

**Verification**:
- [ ] Auth router imported
- [ ] Auth router included in api_router
- [ ] No import errors

---

### Task 9: Update Requirements File

**Objective**: Document all new dependencies.

**File**: `encore_backend/requirements.txt`

**Ensure these lines exist**:
```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
pydantic-settings==2.1.0
mysql-connector-python==8.1.0
boto3==1.34.0
python-dotenv==1.0.0
paramiko==3.4.0
sshtunnel==0.4.0

# JWT Authentication (NEW)
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.6
```

---

## 🧪 TESTING

### Test 1: Start Backend with New Auth

```bash
cd /home/ec2-user/encore-loyalty-new-system
source venv/bin/activate
cd encore_backend

# Start backend
python main.py
```

**Expected**: Server starts without errors, auth endpoints registered.

---

### Test 2: Test Login Endpoint (Swagger UI)

1. Open browser: `http://localhost:8000/docs`
2. Find `POST /api/v1/auth/login`
3. Click "Try it out"
4. Enter test credentials (you need to know what's in your database):
```json
{
  "email": "admin@encore.com",
  "password": "your-password"
}
```
5. Execute

**Expected Response** (200 OK):
```json
{
  "token": "eyJ0eXAiOiJKV1QiLC...",
  "refresh_token": "eyJ0eXAiOiJKV1QiLC...",
  "user": {
    "id": 1,
    "email": "admin@encore.com",
    "name": "Admin User",
    "role": "super_admin",
    "venue_id": null,
    "is_active": true
  }
}
```

---

### Test 3: Test Protected Endpoint

1. Copy the `token` from login response
2. Click "Authorize" button at top of Swagger UI
3. Enter: `Bearer <your-token>`
4. Click "Authorize"
5. Try `GET /api/v1/auth/me`
6. Execute

**Expected Response** (200 OK):
```json
{
  "id": 1,
  "email": "admin@encore.com",
  "name": "Admin User",
  "role": "super_admin",
  "venue_id": null,
  "is_active": true
}
```

---

### Test 4: Test Token Refresh

1. In Swagger, find `POST /api/v1/auth/refresh`
2. Enter the `refresh_token` from login
3. Execute

**Expected Response** (200 OK):
```json
{
  "access_token": "new-token...",
  "refresh_token": "new-refresh-token...",
  "token_type": "bearer"
}
```

---

### Test 5: Test Invalid Token

1. Click "Authorize", enter: `Bearer invalid-token-here`
2. Try `GET /api/v1/auth/me`

**Expected Response** (401 Unauthorized):
```json
{
  "detail": "Could not validate credentials"
}
```

---

### Test 6: Test Logout

1. With valid token, try `POST /api/v1/auth/logout`

**Expected Response** (200 OK):
```json
{
  "message": "Successfully logged out",
  "detail": "Please discard your access and refresh tokens"
}
```

---

## ✅ COMPLETION CRITERIA

**Phase 2A Complete When**:

1. **Dependencies Installed**:
   - [ ] python-jose, passlib, python-multipart installed
   - [ ] No installation errors

2. **Code Created**:
   - [ ] `models/user.py` - User models defined
   - [ ] `services/jwt_service.py` - JWT service implemented
   - [ ] `services/auth_service.py` - Auth service implemented
   - [ ] `dependencies.py` - Auth dependencies added
   - [ ] `routes/auth.py` - 4 auth endpoints created

3. **Configuration**:
   - [ ] JWT settings added to `config.py`
   - [ ] Auth routes registered in `__init__.py`
   - [ ] `requirements.txt` updated

4. **Testing**:
   - [ ] Backend starts without errors
   - [ ] Login endpoint works (returns token + user)
   - [ ] Protected endpoint requires valid token
   - [ ] Invalid token returns 401
   - [ ] Token refresh works
   - [ ] Logout endpoint works

5. **Database Integration**:
   - [ ] Can authenticate against MySQL users
   - [ ] User roles properly assigned
   - [ ] Password verification works

6. **Documentation**:
   - [ ] All endpoints documented in code
   - [ ] Swagger UI shows all auth endpoints
   - [ ] Test credentials documented

---

## 🚨 TROUBLESHOOTING

### Issue: Import errors for jose or passlib

**Solution**:
```bash
pip install --upgrade python-jose[cryptography] passlib[bcrypt]
```

---

### Issue: SQL query fails (users table doesn't exist)

**Solution**:
1. Check your actual MySQL schema
2. Find the table that stores user credentials
3. Update the SQL query in `auth_service.py` to match
4. You may need to:
   - Use a different table name
   - Map different column names
   - Join multiple tables

**To check schema**:
```python
# Add this temporarily in your code
query = "SHOW TABLES"
result = await self.mysql.execute_query(query)
print("Tables:", result)

query = "DESCRIBE your_user_table"
result = await self.mysql.execute_query(query)
print("Schema:", result)
```

---

### Issue: Password hashing doesn't match

**Possible causes**:
1. Passwords in database are not bcrypt hashed
2. Need to hash existing passwords first
3. May need different hashing algorithm

**Solution**:
If your database has plain text passwords (development only!):
- Create a migration script to hash them
- Or temporarily modify `verify_password` to check plain text (NOT for production)

---

### Issue: Token verification fails

**Check**:
1. `jwt_secret_key` is set correctly
2. Token is being sent as `Bearer <token>`
3. Token hasn't expired (15 minutes for access tokens)
4. Correct algorithm (HS256)

---

## 📋 REPORTING BACK

After completing this phase, report:

```
✅ PHASE 2A COMPLETE - Backend Authentication

Dependencies:
- python-jose: ✅ Installed
- passlib: ✅ Installed
- python-multipart: ✅ Installed

Files Created:
- models/user.py: ✅
- services/jwt_service.py: ✅
- services/auth_service.py: ✅
- routes/auth.py: ✅
- Updated dependencies.py: ✅

Endpoints Working:
- POST /api/v1/auth/login: ✅ Returns token + user
- POST /api/v1/auth/logout: ✅ Confirms logout
- POST /api/v1/auth/refresh: ✅ Refreshes tokens
- GET /api/v1/auth/me: ✅ Returns current user

Authentication:
- JWT generation: ✅
- Token verification: ✅
- Password hashing: ✅
- MySQL user lookup: ✅
- Role assignment: ✅

Testing:
- All endpoints tested in Swagger: ✅
- Valid credentials work: ✅
- Invalid credentials rejected: ✅
- Protected routes require auth: ✅
- Token expiry works: ✅

Issues Found: [None / List any issues]

Database Notes:
- User table: [name]
- Schema: [details]
- Test user: [email]

Ready for Phase 2B: YES

Notes:
- Backend authentication fully functional
- JWT tokens working (15min access, 7 day refresh)
- Ready for frontend integration
- [Any special notes about your setup]
```

---

## 🎯 SUCCESS DEFINITION

**Phase 2A succeeds when**:
1. Backend has working JWT authentication
2. All 4 auth endpoints functional
3. Can login and get valid JWT token
4. Protected routes require valid token
5. Token refresh mechanism works
6. Ready for frontend to integrate

**Time**: 3-4 hours

**Next Phase**: Phase 2B - Frontend Authentication Integration

---

## 🔗 REFERENCE DOCUMENTS

**Related Files**:
- Archive Analysis: `/docs/investigation/ENCORE-ARCHIVE-ANALYSIS-REPORT.md` (MySQL schema)
- MVP Plan: `/docs/SUPERVISOR/MVP-EXECUTION-PLAN.md`
- Frontend mockup auth: `/encore_frontend/src/services/authService.ts` (what frontend expects)

**Dependencies**:
- Must complete: Backend Setup
- Parallel with: Phase 2B (Frontend Auth)
- Blocks: All other backend endpoint development

---

**READY TO START?** Copy this prompt into Cursor and begin Phase 2A!

Good luck! 🚀
