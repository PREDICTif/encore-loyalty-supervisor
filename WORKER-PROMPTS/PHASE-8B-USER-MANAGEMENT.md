# Phase 8B: Application User Management (SaaS Model)

**Created**: 2025-11-28
**Priority**: HIGH
**Estimated Time**: 9-13 hours total (3 sub-phases)
**Dependencies**: Phase 8 complete ✅

---

## 🎯 Objective

Replace the incorrect MySQL `user` table authentication with a proper DynamoDB-based application user management system for a multi-tenant SaaS application.

**Current Problem**: The app incorrectly authenticates against MySQL `user` table which contains restaurant staff (customers), not application administrators.

**Solution**: Create dedicated `encore_app_users` DynamoDB table with proper role-based access control.

---

## 📋 User Roles & Permissions

### Global Admin (Encore Staff)
- Provision/deprovision venues
- Global settings management
- View ALL feedback across all venues
- Generate AI responses for any venue
- Impersonate as venue admins
- View global analytics
- Create/manage ALL users (global admins + venue admins)
- System monitoring

### Venue Admin (Restaurant Managers)
- View feedback for their assigned venue ONLY
- Generate AI responses for their venue ONLY
- Manage venue-level settings
- View venue-level analytics
- Create other venue admins for SAME venue only

---

## Phase 8B-1: Database & Backend (4-6 hours)

### Step 1: Create DynamoDB Table Script

**File**: `encore_backend/scripts/create_app_users_table.py`

```python
#!/usr/bin/env python3
"""
Create the encore_app_users DynamoDB table for application user management.
"""
import boto3
import sys
import os

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
from config import settings

def create_table():
    dynamodb = boto3.resource(
        'dynamodb',
        region_name=settings.aws_region,
        aws_access_key_id=settings.aws_access_key_id,
        aws_secret_access_key=settings.aws_secret_access_key
    )
    
    table_name = 'encore_app_users'
    
    # Check if table exists
    existing_tables = [t.name for t in dynamodb.tables.all()]
    if table_name in existing_tables:
        print(f"Table {table_name} already exists")
        return
    
    # Create table with GSIs for email and venue_id lookups
    table = dynamodb.create_table(
        TableName=table_name,
        KeySchema=[
            {'AttributeName': 'user_id', 'KeyType': 'HASH'}
        ],
        AttributeDefinitions=[
            {'AttributeName': 'user_id', 'AttributeType': 'S'},
            {'AttributeName': 'email', 'AttributeType': 'S'},
            {'AttributeName': 'venue_id', 'AttributeType': 'N'},
            {'AttributeName': 'role', 'AttributeType': 'S'}
        ],
        GlobalSecondaryIndexes=[
            {
                'IndexName': 'email-index',
                'KeySchema': [{'AttributeName': 'email', 'KeyType': 'HASH'}],
                'Projection': {'ProjectionType': 'ALL'},
                'ProvisionedThroughput': {'ReadCapacityUnits': 5, 'WriteCapacityUnits': 5}
            },
            {
                'IndexName': 'venue_id-index',
                'KeySchema': [
                    {'AttributeName': 'venue_id', 'KeyType': 'HASH'},
                    {'AttributeName': 'role', 'KeyType': 'RANGE'}
                ],
                'Projection': {'ProjectionType': 'ALL'},
                'ProvisionedThroughput': {'ReadCapacityUnits': 5, 'WriteCapacityUnits': 5}
            }
        ],
        ProvisionedThroughput={'ReadCapacityUnits': 5, 'WriteCapacityUnits': 5}
    )
    
    print(f"Creating table {table_name}...")
    table.wait_until_exists()
    print(f"Table {table_name} created successfully!")

if __name__ == '__main__':
    create_table()
```

### Step 2: Create AppUserService

**File**: `encore_backend/services/app_user_service.py`

```python
"""
Application User Service - Manages users in DynamoDB for the SaaS application.
"""
import boto3
import uuid
import bcrypt
from datetime import datetime
from typing import Optional, List, Dict, Any
from boto3.dynamodb.conditions import Key
from decimal import Decimal

from config import settings


# Role-based permissions
ROLE_PERMISSIONS = {
    "global_admin": [
        "provision_venues",
        "global_settings",
        "view_all_feedback",
        "generate_responses",
        "impersonate",
        "global_analytics",
        "manage_all_users",
        "system_monitoring",
        "venue_settings"
    ],
    "venue_admin": [
        "view_venue_feedback",
        "generate_responses",
        "venue_settings",
        "venue_analytics",
        "manage_venue_users"
    ]
}


class AppUserService:
    """Service for managing application users in DynamoDB."""
    
    def __init__(self):
        self.dynamodb = boto3.resource(
            'dynamodb',
            region_name=settings.aws_region,
            aws_access_key_id=settings.aws_access_key_id,
            aws_secret_access_key=settings.aws_secret_access_key
        )
        self.table = self.dynamodb.Table('encore_app_users')
    
    def _hash_password(self, password: str) -> str:
        """Hash password using bcrypt."""
        return bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt()).decode('utf-8')
    
    def _verify_password(self, password: str, hashed: str) -> bool:
        """Verify password against hash."""
        return bcrypt.checkpw(password.encode('utf-8'), hashed.encode('utf-8'))
    
    def _get_permissions(self, role: str) -> List[str]:
        """Get permissions for a role."""
        return ROLE_PERMISSIONS.get(role, [])
    
    def _serialize_user(self, item: Dict) -> Dict:
        """Convert DynamoDB item to user dict."""
        return {
            "user_id": item.get("user_id"),
            "email": item.get("email"),
            "name": item.get("name"),
            "role": item.get("role"),
            "venue_id": int(item["venue_id"]) if item.get("venue_id") else None,
            "venue_name": item.get("venue_name"),
            "is_active": item.get("is_active", True),
            "created_at": item.get("created_at"),
            "created_by": item.get("created_by"),
            "last_login": item.get("last_login"),
            "permissions": self._get_permissions(item.get("role", ""))
        }
    
    # ==================== CRUD Operations ====================
    
    def create_user(
        self,
        email: str,
        password: str,
        name: str,
        role: str,
        venue_id: Optional[int] = None,
        venue_name: Optional[str] = None,
        created_by: Optional[str] = None
    ) -> Dict:
        """Create a new application user."""
        # Validate role
        if role not in ["global_admin", "venue_admin"]:
            raise ValueError(f"Invalid role: {role}")
        
        # Venue admin must have venue_id
        if role == "venue_admin" and not venue_id:
            raise ValueError("Venue admin must have a venue_id")
        
        # Global admin should not have venue_id
        if role == "global_admin":
            venue_id = None
            venue_name = None
        
        # Check if email already exists
        existing = self.get_user_by_email(email)
        if existing:
            raise ValueError(f"User with email {email} already exists")
        
        user_id = str(uuid.uuid4())
        now = datetime.utcnow().isoformat()
        
        item = {
            "user_id": user_id,
            "email": email.lower().strip(),
            "password_hash": self._hash_password(password),
            "name": name,
            "role": role,
            "is_active": True,
            "created_at": now,
            "created_by": created_by,
            "last_login": None
        }
        
        # Add venue info for venue admins
        if venue_id:
            item["venue_id"] = venue_id
            item["venue_name"] = venue_name
        
        self.table.put_item(Item=item)
        
        return self._serialize_user(item)
    
    def get_user_by_id(self, user_id: str) -> Optional[Dict]:
        """Get user by ID."""
        response = self.table.get_item(Key={"user_id": user_id})
        item = response.get("Item")
        return self._serialize_user(item) if item else None
    
    def get_user_by_email(self, email: str) -> Optional[Dict]:
        """Get user by email (for login)."""
        response = self.table.query(
            IndexName="email-index",
            KeyConditionExpression=Key("email").eq(email.lower().strip())
        )
        items = response.get("Items", [])
        if items:
            return self._serialize_user(items[0])
        return None
    
    def get_user_with_password(self, email: str) -> Optional[Dict]:
        """Get user with password hash (for authentication)."""
        response = self.table.query(
            IndexName="email-index",
            KeyConditionExpression=Key("email").eq(email.lower().strip())
        )
        items = response.get("Items", [])
        if items:
            item = items[0]
            user = self._serialize_user(item)
            user["password_hash"] = item.get("password_hash")
            return user
        return None
    
    def get_all_users(self) -> List[Dict]:
        """Get all application users."""
        response = self.table.scan()
        items = response.get("Items", [])
        return [self._serialize_user(item) for item in items]
    
    def get_users_by_venue(self, venue_id: int) -> List[Dict]:
        """Get all users for a specific venue."""
        response = self.table.query(
            IndexName="venue_id-index",
            KeyConditionExpression=Key("venue_id").eq(venue_id)
        )
        items = response.get("Items", [])
        return [self._serialize_user(item) for item in items]
    
    def get_global_admins(self) -> List[Dict]:
        """Get all global admin users."""
        response = self.table.scan(
            FilterExpression="role = :role",
            ExpressionAttributeValues={":role": "global_admin"}
        )
        items = response.get("Items", [])
        return [self._serialize_user(item) for item in items]
    
    def update_user(
        self,
        user_id: str,
        name: Optional[str] = None,
        role: Optional[str] = None,
        venue_id: Optional[int] = None,
        venue_name: Optional[str] = None,
        is_active: Optional[bool] = None
    ) -> Optional[Dict]:
        """Update user details."""
        update_expr_parts = []
        expr_values = {}
        expr_names = {}
        
        if name is not None:
            update_expr_parts.append("#name = :name")
            expr_values[":name"] = name
            expr_names["#name"] = "name"
        
        if role is not None:
            if role not in ["global_admin", "venue_admin"]:
                raise ValueError(f"Invalid role: {role}")
            update_expr_parts.append("#role = :role")
            expr_values[":role"] = role
            expr_names["#role"] = "role"
        
        if venue_id is not None:
            update_expr_parts.append("venue_id = :venue_id")
            expr_values[":venue_id"] = venue_id
        
        if venue_name is not None:
            update_expr_parts.append("venue_name = :venue_name")
            expr_values[":venue_name"] = venue_name
        
        if is_active is not None:
            update_expr_parts.append("is_active = :is_active")
            expr_values[":is_active"] = is_active
        
        if not update_expr_parts:
            return self.get_user_by_id(user_id)
        
        response = self.table.update_item(
            Key={"user_id": user_id},
            UpdateExpression="SET " + ", ".join(update_expr_parts),
            ExpressionAttributeValues=expr_values,
            ExpressionAttributeNames=expr_names if expr_names else None,
            ReturnValues="ALL_NEW"
        )
        
        return self._serialize_user(response.get("Attributes", {}))
    
    def update_password(self, user_id: str, new_password: str) -> bool:
        """Update user password."""
        self.table.update_item(
            Key={"user_id": user_id},
            UpdateExpression="SET password_hash = :hash",
            ExpressionAttributeValues={":hash": self._hash_password(new_password)}
        )
        return True
    
    def update_last_login(self, user_id: str) -> None:
        """Update last login timestamp."""
        self.table.update_item(
            Key={"user_id": user_id},
            UpdateExpression="SET last_login = :now",
            ExpressionAttributeValues={":now": datetime.utcnow().isoformat()}
        )
    
    def delete_user(self, user_id: str) -> bool:
        """Delete a user (or soft delete by setting is_active=False)."""
        # Soft delete - set is_active to False
        self.table.update_item(
            Key={"user_id": user_id},
            UpdateExpression="SET is_active = :inactive",
            ExpressionAttributeValues={":inactive": False}
        )
        return True
    
    def hard_delete_user(self, user_id: str) -> bool:
        """Permanently delete a user."""
        self.table.delete_item(Key={"user_id": user_id})
        return True
    
    # ==================== Authentication ====================
    
    def authenticate(self, email: str, password: str) -> Optional[Dict]:
        """Authenticate user and return user data if valid."""
        user = self.get_user_with_password(email)
        
        if not user:
            return None
        
        if not user.get("is_active", True):
            return None
        
        if not self._verify_password(password, user.get("password_hash", "")):
            return None
        
        # Update last login
        self.update_last_login(user["user_id"])
        
        # Remove password hash before returning
        user.pop("password_hash", None)
        
        return user
    
    # ==================== Statistics ====================
    
    def get_user_stats(self) -> Dict:
        """Get user statistics."""
        all_users = self.get_all_users()
        active_users = [u for u in all_users if u.get("is_active", True)]
        global_admins = [u for u in active_users if u.get("role") == "global_admin"]
        venue_admins = [u for u in active_users if u.get("role") == "venue_admin"]
        
        return {
            "total_users": len(all_users),
            "active_users": len(active_users),
            "global_admins": len(global_admins),
            "venue_admins": len(venue_admins)
        }
```

### Step 3: Create Seed Script

**File**: `encore_backend/scripts/seed_app_users.py`

```python
#!/usr/bin/env python3
"""
Seed initial application users.
"""
import sys
import os

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from services.app_user_service import AppUserService

def seed_users():
    service = AppUserService()
    
    # Initial global admins
    initial_admins = [
        {
            "email": "tjrottet@encoreloyalty.net",
            "password": "password222",
            "name": "TJ Rottet",
            "role": "global_admin"
        },
        {
            "email": "admin@encore.com",
            "password": "password123",
            "name": "Encore Admin",
            "role": "global_admin"
        },
        {
            "email": "marian@dumitrascu.net",
            "password": "password123",
            "name": "Marian Dumitrascu",
            "role": "global_admin"
        }
    ]
    
    for admin in initial_admins:
        try:
            user = service.create_user(
                email=admin["email"],
                password=admin["password"],
                name=admin["name"],
                role=admin["role"],
                created_by="system_seed"
            )
            print(f"✅ Created: {user['email']} ({user['role']})")
        except ValueError as e:
            print(f"⚠️  Skipped: {admin['email']} - {e}")

if __name__ == '__main__':
    seed_users()
```

### Step 4: Update AuthService

**File**: `encore_backend/services/auth_service.py`

Update the `AuthService` to use `AppUserService` instead of MySQL:

```python
# Add to imports
from services.app_user_service import AppUserService

class AuthService:
    def __init__(self):
        self.app_user_service = AppUserService()
        # Keep mysql_service for restaurant data if needed
    
    def authenticate(self, email: str, password: str) -> Optional[Dict]:
        """Authenticate using DynamoDB app users."""
        return self.app_user_service.authenticate(email, password)
    
    def get_user_by_id(self, user_id: str) -> Optional[Dict]:
        """Get user by ID from DynamoDB."""
        return self.app_user_service.get_user_by_id(user_id)
```

### Step 5: Create User Management API Endpoints

**File**: `encore_backend/routes/users.py`

```python
"""
User Management API endpoints.
"""
from fastapi import APIRouter, HTTPException, Depends, status
from pydantic import BaseModel, EmailStr
from typing import Optional, List

from services.app_user_service import AppUserService
from dependencies import get_current_user
from models.user import User

router = APIRouter(prefix="/api/v1/users", tags=["users"])

# ==================== Models ====================

class CreateUserRequest(BaseModel):
    email: EmailStr
    password: str
    name: str
    role: str  # "global_admin" or "venue_admin"
    venue_id: Optional[int] = None
    venue_name: Optional[str] = None

class UpdateUserRequest(BaseModel):
    name: Optional[str] = None
    role: Optional[str] = None
    venue_id: Optional[int] = None
    venue_name: Optional[str] = None
    is_active: Optional[bool] = None

class ChangePasswordRequest(BaseModel):
    new_password: str

class UserResponse(BaseModel):
    user_id: str
    email: str
    name: str
    role: str
    venue_id: Optional[int]
    venue_name: Optional[str]
    is_active: bool
    created_at: Optional[str]
    last_login: Optional[str]
    permissions: List[str]

class UsersListResponse(BaseModel):
    users: List[UserResponse]
    total: int

class UserStatsResponse(BaseModel):
    total_users: int
    active_users: int
    global_admins: int
    venue_admins: int

# ==================== Helper Functions ====================

def get_user_service() -> AppUserService:
    return AppUserService()

def require_global_admin(current_user: User = Depends(get_current_user)):
    """Require global admin role."""
    if current_user.role != "global_admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Global admin access required"
        )
    return current_user

def require_admin(current_user: User = Depends(get_current_user)):
    """Require any admin role."""
    if current_user.role not in ["global_admin", "venue_admin"]:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required"
        )
    return current_user

# ==================== Endpoints ====================

@router.get("", response_model=UsersListResponse)
async def list_users(
    venue_id: Optional[int] = None,
    current_user: User = Depends(require_global_admin),
    service: AppUserService = Depends(get_user_service)
):
    """List all users (global admin only)."""
    if venue_id:
        users = service.get_users_by_venue(venue_id)
    else:
        users = service.get_all_users()
    
    # Filter to active users only
    active_users = [u for u in users if u.get("is_active", True)]
    
    return UsersListResponse(users=active_users, total=len(active_users))

@router.get("/stats", response_model=UserStatsResponse)
async def get_user_stats(
    current_user: User = Depends(require_global_admin),
    service: AppUserService = Depends(get_user_service)
):
    """Get user statistics."""
    return service.get_user_stats()

@router.get("/me", response_model=UserResponse)
async def get_current_user_info(
    current_user: User = Depends(get_current_user),
    service: AppUserService = Depends(get_user_service)
):
    """Get current logged-in user info."""
    user = service.get_user_by_id(current_user.id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: str,
    current_user: User = Depends(require_admin),
    service: AppUserService = Depends(get_user_service)
):
    """Get user by ID."""
    # Venue admins can only view users from their venue
    if current_user.role == "venue_admin":
        user = service.get_user_by_id(user_id)
        if user and user.get("venue_id") != current_user.venue_id:
            raise HTTPException(status_code=403, detail="Cannot view users from other venues")
    
    user = service.get_user_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@router.post("", response_model=UserResponse, status_code=201)
async def create_user(
    request: CreateUserRequest,
    current_user: User = Depends(require_admin),
    service: AppUserService = Depends(get_user_service)
):
    """Create a new user."""
    # Venue admins can only create venue_admin for their own venue
    if current_user.role == "venue_admin":
        if request.role != "venue_admin":
            raise HTTPException(status_code=403, detail="Venue admins can only create venue admins")
        if request.venue_id != current_user.venue_id:
            raise HTTPException(status_code=403, detail="Cannot create users for other venues")
    
    try:
        user = service.create_user(
            email=request.email,
            password=request.password,
            name=request.name,
            role=request.role,
            venue_id=request.venue_id,
            venue_name=request.venue_name,
            created_by=current_user.id
        )
        return user
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.put("/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: str,
    request: UpdateUserRequest,
    current_user: User = Depends(require_admin),
    service: AppUserService = Depends(get_user_service)
):
    """Update user details."""
    # Check permissions
    target_user = service.get_user_by_id(user_id)
    if not target_user:
        raise HTTPException(status_code=404, detail="User not found")
    
    if current_user.role == "venue_admin":
        if target_user.get("venue_id") != current_user.venue_id:
            raise HTTPException(status_code=403, detail="Cannot update users from other venues")
        if request.role and request.role != "venue_admin":
            raise HTTPException(status_code=403, detail="Cannot change role to global_admin")
    
    try:
        user = service.update_user(
            user_id=user_id,
            name=request.name,
            role=request.role,
            venue_id=request.venue_id,
            venue_name=request.venue_name,
            is_active=request.is_active
        )
        return user
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.put("/{user_id}/password")
async def change_user_password(
    user_id: str,
    request: ChangePasswordRequest,
    current_user: User = Depends(require_admin),
    service: AppUserService = Depends(get_user_service)
):
    """Change user password."""
    # Users can change their own password, admins can change any
    if current_user.id != user_id and current_user.role != "global_admin":
        raise HTTPException(status_code=403, detail="Cannot change other user's password")
    
    service.update_password(user_id, request.new_password)
    return {"message": "Password updated successfully"}

@router.delete("/{user_id}")
async def delete_user(
    user_id: str,
    current_user: User = Depends(require_global_admin),
    service: AppUserService = Depends(get_user_service)
):
    """Soft delete a user (global admin only)."""
    # Prevent self-deletion
    if user_id == current_user.id:
        raise HTTPException(status_code=400, detail="Cannot delete yourself")
    
    service.delete_user(user_id)
    return {"message": "User deactivated successfully"}
```

### Step 6: Register Router

**File**: `encore_backend/routes/__init__.py`

Add to the router registration:
```python
from routes.users import router as users_router
# ...
app.include_router(users_router)
```

### Step 7: Update User Model

**File**: `encore_backend/models/user.py`

Ensure the User model supports the new structure:
```python
class User(BaseModel):
    id: str  # Now UUID string, not int
    email: str
    name: str
    role: str  # "global_admin" or "venue_admin"
    venue_id: Optional[int] = None
    venue_name: Optional[str] = None
    is_active: bool = True
    permissions: List[str] = []
    # For impersonation
    is_impersonation: bool = False
    impersonated_by: Optional[str] = None
```

---

## Testing Requirements

### 1. Table Creation
```bash
cd encore_backend
python scripts/create_app_users_table.py
```

### 2. Seed Users
```bash
python scripts/seed_app_users.py
```

### 3. API Tests
```bash
# Get token
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"tjrottet@encoreloyalty.net","password":"password222"}' | jq -r '.token')

# List users
curl -s http://localhost:8000/api/v1/users \
  -H "Authorization: Bearer $TOKEN" | jq

# Get stats
curl -s http://localhost:8000/api/v1/users/stats \
  -H "Authorization: Bearer $TOKEN" | jq

# Create venue admin
curl -s -X POST http://localhost:8000/api/v1/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@venue.com",
    "password": "test123",
    "name": "Test Venue Admin",
    "role": "venue_admin",
    "venue_id": 57,
    "venue_name": "Opus Ocean Grille"
  }' | jq
```

---

## Success Criteria

- [ ] DynamoDB table `encore_app_users` created
- [ ] Initial admin users seeded
- [ ] Login works with new user pool
- [ ] CRUD endpoints functional
- [ ] Role-based permissions enforced
- [ ] Impersonation still works
- [ ] Frontend User Management shows new users

---

## Files to Create/Modify

| File | Action |
|------|--------|
| `scripts/create_app_users_table.py` | CREATE |
| `services/app_user_service.py` | CREATE |
| `scripts/seed_app_users.py` | CREATE |
| `routes/users.py` | CREATE |
| `services/auth_service.py` | MODIFY |
| `models/user.py` | MODIFY |
| `routes/__init__.py` | MODIFY |
| `dependencies.py` | MODIFY |

---

## Phase 8B-2 & 8B-3 (Frontend & Testing)

After backend is complete, proceed with:
- Frontend User Management page update
- Add/Edit/Delete user modals
- Integration testing
- Documentation update

