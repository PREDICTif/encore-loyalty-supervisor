# Phase 3A: Backend Settings Management

**Assigned to:** AI Worker (Backend Developer)
**Model:** Claude Sonnet 4.5
**Environment:** encore_backend directory
**Prerequisites:** Phase 2A completed (Authentication system working)

## 📋 Overview

Implement the **Settings Management** system for the Encore Loyalty backend. This includes venue settings CRUD operations, AI configuration, and settings validation.

## 🎯 Objectives

1. Create settings models and database schema
2. Implement settings service with business logic
3. Create settings API endpoints
4. Add settings validation and defaults
5. Integrate with authentication (protected endpoints)
6. Test all settings operations

## 📁 Directory Structure

```
encore_backend/
├── models/
│   └── settings.py          # ← CREATE
├── services/
│   └── settings_service.py  # ← CREATE
├── routes/
│   └── settings.py          # ← CREATE
├── database/
│   └── dynamodb.py          # ← UPDATE (add settings table operations)
└── main.py                  # ← UPDATE (register settings routes)
```

## ✅ Task Checklist

### Task 1: Create Settings Models (`models/settings.py`)

**Purpose:** Define Pydantic models for venue settings

**Create file:** `encore_backend/models/settings.py`

```python
"""
Settings models for venue configuration
"""
from typing import Optional, Dict, Any
from pydantic import BaseModel, Field, validator
from datetime import datetime

class AISettings(BaseModel):
    """AI-specific settings for a venue"""
    enabled: bool = False
    model_version: str = "claude-3-5-sonnet-20241022"
    temperature: float = Field(default=0.7, ge=0.0, le=1.0)
    max_tokens: int = Field(default=500, ge=100, le=2000)
    system_prompt: Optional[str] = None
    auto_respond: bool = False

    @validator('temperature')
    def validate_temperature(cls, v):
        if not 0.0 <= v <= 1.0:
            raise ValueError('Temperature must be between 0.0 and 1.0')
        return v

class EmailSettings(BaseModel):
    """Email-specific settings for a venue"""
    enabled: bool = False
    from_email: Optional[str] = None
    reply_to: Optional[str] = None
    signature: Optional[str] = None
    auto_send: bool = False

    @validator('from_email', 'reply_to')
    def validate_email(cls, v):
        if v and '@' not in v:
            raise ValueError('Invalid email format')
        return v

class NotificationSettings(BaseModel):
    """Notification preferences for a venue"""
    email_on_feedback: bool = True
    email_on_response: bool = False
    slack_webhook: Optional[str] = None

class VenueSettings(BaseModel):
    """Complete settings for a venue"""
    venue_id: str
    venue_name: str
    ai_settings: AISettings = Field(default_factory=AISettings)
    email_settings: EmailSettings = Field(default_factory=EmailSettings)
    notification_settings: NotificationSettings = Field(default_factory=NotificationSettings)
    custom_fields: Dict[str, Any] = Field(default_factory=dict)
    created_at: Optional[datetime] = None
    updated_at: Optional[datetime] = None
    updated_by: Optional[str] = None

    class Config:
        json_encoders = {
            datetime: lambda v: v.isoformat()
        }

class UpdateVenueSettingsRequest(BaseModel):
    """Request model for updating venue settings"""
    ai_settings: Optional[AISettings] = None
    email_settings: Optional[EmailSettings] = None
    notification_settings: Optional[NotificationSettings] = None
    custom_fields: Optional[Dict[str, Any]] = None

class VenueSettingsResponse(BaseModel):
    """Response model for venue settings"""
    success: bool
    message: str
    settings: Optional[VenueSettings] = None
```

**Validation:**
```bash
cd /home/ec2-user/encore-loyalty-new-system
source venv/bin/activate
python -c "from encore_backend.models.settings import VenueSettings, AISettings; print('✅ Settings models imported successfully')"
```

---

### Task 2: Create Settings Service (`services/settings_service.py`)

**Purpose:** Business logic for settings management

**Create file:** `encore_backend/services/settings_service.py`

```python
"""
Settings service for venue configuration management
"""
import boto3
from typing import Optional, Dict, Any
from datetime import datetime
from botocore.exceptions import ClientError
from models.settings import (
    VenueSettings,
    AISettings,
    EmailSettings,
    NotificationSettings,
    UpdateVenueSettingsRequest
)

class SettingsService:
    """Service for managing venue settings in DynamoDB"""

    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        self.table_name = 'encore_venue_settings'
        self.table = self.dynamodb.Table(self.table_name)

    def get_settings(self, venue_id: str) -> Optional[VenueSettings]:
        """
        Retrieve settings for a venue

        Args:
            venue_id: Venue identifier

        Returns:
            VenueSettings object or None if not found
        """
        try:
            response = self.table.get_item(Key={'venue_id': venue_id})

            if 'Item' not in response:
                # Return default settings if none exist
                return self._create_default_settings(venue_id)

            item = response['Item']

            # Parse nested settings
            return VenueSettings(
                venue_id=item['venue_id'],
                venue_name=item.get('venue_name', 'Unknown Venue'),
                ai_settings=AISettings(**item.get('ai_settings', {})),
                email_settings=EmailSettings(**item.get('email_settings', {})),
                notification_settings=NotificationSettings(**item.get('notification_settings', {})),
                custom_fields=item.get('custom_fields', {}),
                created_at=item.get('created_at'),
                updated_at=item.get('updated_at'),
                updated_by=item.get('updated_by')
            )

        except ClientError as e:
            print(f"Error retrieving settings: {e}")
            return None

    def update_settings(
        self,
        venue_id: str,
        venue_name: str,
        updates: UpdateVenueSettingsRequest,
        updated_by: str
    ) -> Optional[VenueSettings]:
        """
        Update venue settings

        Args:
            venue_id: Venue identifier
            venue_name: Venue name
            updates: Settings to update
            updated_by: User email who made the update

        Returns:
            Updated VenueSettings object or None on error
        """
        try:
            # Get existing settings or create default
            existing = self.get_settings(venue_id)
            if not existing:
                existing = self._create_default_settings(venue_id, venue_name)

            # Build update expression
            update_expr_parts = []
            expr_attr_values = {}

            # Update AI settings
            if updates.ai_settings:
                update_expr_parts.append("ai_settings = :ai")
                expr_attr_values[':ai'] = updates.ai_settings.dict()

            # Update email settings
            if updates.email_settings:
                update_expr_parts.append("email_settings = :email")
                expr_attr_values[':email'] = updates.email_settings.dict()

            # Update notification settings
            if updates.notification_settings:
                update_expr_parts.append("notification_settings = :notif")
                expr_attr_values[':notif'] = updates.notification_settings.dict()

            # Update custom fields
            if updates.custom_fields:
                update_expr_parts.append("custom_fields = :custom")
                expr_attr_values[':custom'] = updates.custom_fields

            # Always update metadata
            update_expr_parts.extend([
                "updated_at = :updated_at",
                "updated_by = :updated_by"
            ])
            expr_attr_values[':updated_at'] = datetime.utcnow().isoformat()
            expr_attr_values[':updated_by'] = updated_by

            # Create full update expression
            update_expr = "SET " + ", ".join(update_expr_parts)

            # Execute update
            response = self.table.update_item(
                Key={'venue_id': venue_id},
                UpdateExpression=update_expr,
                ExpressionAttributeValues=expr_attr_values,
                ReturnValues='ALL_NEW'
            )

            # Return updated settings
            item = response['Attributes']
            return VenueSettings(
                venue_id=item['venue_id'],
                venue_name=item.get('venue_name', venue_name),
                ai_settings=AISettings(**item.get('ai_settings', {})),
                email_settings=EmailSettings(**item.get('email_settings', {})),
                notification_settings=NotificationSettings(**item.get('notification_settings', {})),
                custom_fields=item.get('custom_fields', {}),
                created_at=item.get('created_at'),
                updated_at=item.get('updated_at'),
                updated_by=item.get('updated_by')
            )

        except ClientError as e:
            print(f"Error updating settings: {e}")
            return None

    def create_settings(
        self,
        venue_id: str,
        venue_name: str,
        created_by: str
    ) -> Optional[VenueSettings]:
        """
        Create default settings for a new venue

        Args:
            venue_id: Venue identifier
            venue_name: Venue name
            created_by: User email who created the settings

        Returns:
            Created VenueSettings object or None on error
        """
        try:
            settings = self._create_default_settings(venue_id, venue_name)

            item = {
                'venue_id': venue_id,
                'venue_name': venue_name,
                'ai_settings': settings.ai_settings.dict(),
                'email_settings': settings.email_settings.dict(),
                'notification_settings': settings.notification_settings.dict(),
                'custom_fields': {},
                'created_at': datetime.utcnow().isoformat(),
                'updated_at': datetime.utcnow().isoformat(),
                'updated_by': created_by
            }

            self.table.put_item(Item=item)
            return settings

        except ClientError as e:
            print(f"Error creating settings: {e}")
            return None

    def delete_settings(self, venue_id: str) -> bool:
        """
        Delete settings for a venue (use with caution)

        Args:
            venue_id: Venue identifier

        Returns:
            True if successful, False otherwise
        """
        try:
            self.table.delete_item(Key={'venue_id': venue_id})
            return True
        except ClientError as e:
            print(f"Error deleting settings: {e}")
            return False

    def _create_default_settings(
        self,
        venue_id: str,
        venue_name: str = "Unknown Venue"
    ) -> VenueSettings:
        """Create default settings for a venue"""
        return VenueSettings(
            venue_id=venue_id,
            venue_name=venue_name,
            ai_settings=AISettings(),
            email_settings=EmailSettings(),
            notification_settings=NotificationSettings(),
            custom_fields={},
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow()
        )
```

**Validation:**
```bash
python -c "from encore_backend.services.settings_service import SettingsService; print('✅ Settings service imported successfully')"
```

---

### Task 3: Create Settings Routes (`routes/settings.py`)

**Purpose:** API endpoints for settings management

**Create file:** `encore_backend/routes/settings.py`

```python
"""
Settings API routes
"""
from fastapi import APIRouter, Depends, HTTPException, status
from typing import Dict, Any
from models.settings import (
    VenueSettings,
    UpdateVenueSettingsRequest,
    VenueSettingsResponse
)
from models.user import User
from services.settings_service import SettingsService
from services.jwt_service import get_current_user

router = APIRouter(prefix="/api/settings", tags=["settings"])
settings_service = SettingsService()

@router.get("/{venue_id}", response_model=VenueSettings)
async def get_venue_settings(
    venue_id: str,
    current_user: User = Depends(get_current_user)
):
    """
    Get settings for a specific venue

    Requires authentication.
    """
    # Check if user has access to this venue
    if not _user_has_venue_access(current_user, venue_id):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Access denied to this venue"
        )

    settings = settings_service.get_settings(venue_id)

    if not settings:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Settings not found for venue {venue_id}"
        )

    return settings

@router.put("/{venue_id}", response_model=VenueSettingsResponse)
async def update_venue_settings(
    venue_id: str,
    updates: UpdateVenueSettingsRequest,
    current_user: User = Depends(get_current_user)
):
    """
    Update settings for a specific venue

    Requires authentication. Only updates provided fields.
    """
    # Check if user has access to this venue
    if not _user_has_venue_access(current_user, venue_id):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Access denied to this venue"
        )

    # Get venue name from user's venues
    venue_name = _get_venue_name(current_user, venue_id)

    # Update settings
    updated_settings = settings_service.update_settings(
        venue_id=venue_id,
        venue_name=venue_name,
        updates=updates,
        updated_by=current_user.email
    )

    if not updated_settings:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Failed to update settings"
        )

    return VenueSettingsResponse(
        success=True,
        message="Settings updated successfully",
        settings=updated_settings
    )

@router.post("/{venue_id}", response_model=VenueSettingsResponse)
async def create_venue_settings(
    venue_id: str,
    venue_name: str,
    current_user: User = Depends(get_current_user)
):
    """
    Create default settings for a new venue

    Requires authentication and admin role.
    """
    # Only admins can create new venue settings
    if current_user.role != "admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Only admins can create venue settings"
        )

    # Check if settings already exist
    existing = settings_service.get_settings(venue_id)
    if existing:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Settings already exist for this venue"
        )

    # Create settings
    new_settings = settings_service.create_settings(
        venue_id=venue_id,
        venue_name=venue_name,
        created_by=current_user.email
    )

    if not new_settings:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Failed to create settings"
        )

    return VenueSettingsResponse(
        success=True,
        message="Settings created successfully",
        settings=new_settings
    )

@router.delete("/{venue_id}")
async def delete_venue_settings(
    venue_id: str,
    current_user: User = Depends(get_current_user)
):
    """
    Delete settings for a venue (admin only)

    Use with caution - this is permanent.
    """
    # Only admins can delete settings
    if current_user.role != "admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Only admins can delete venue settings"
        )

    success = settings_service.delete_settings(venue_id)

    if not success:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Failed to delete settings"
        )

    return {"success": True, "message": "Settings deleted successfully"}

# Helper functions
def _user_has_venue_access(user: User, venue_id: str) -> bool:
    """Check if user has access to a specific venue"""
    if user.role == "admin":
        return True
    return venue_id in user.venue_ids

def _get_venue_name(user: User, venue_id: str) -> str:
    """Get venue name from user's venues list"""
    # In a real system, this would query the venues database
    # For now, return a default name
    return f"Venue {venue_id}"
```

**Validation:**
```bash
python -c "from encore_backend.routes.settings import router; print('✅ Settings routes imported successfully')"
```

---

### Task 4: Register Settings Routes in Main App

**Purpose:** Make settings endpoints accessible

**Update file:** `encore_backend/main.py`

Add the import:
```python
from routes.settings import router as settings_router
```

Register the router (add after auth router):
```python
# Register settings routes
app.include_router(settings_router)
```

**Validation:**
```bash
cd encore_backend
python -c "from main import app; print('✅ Main app with settings routes loaded successfully')"
```

---

### Task 5: Create DynamoDB Table

**Purpose:** Initialize settings table in DynamoDB

**Create script:** `encore_backend/scripts/create_settings_table.py`

```python
"""
Create DynamoDB table for venue settings
"""
import boto3
from botocore.exceptions import ClientError

def create_settings_table():
    """Create encore_venue_settings table in DynamoDB"""
    dynamodb = boto3.resource('dynamodb')

    table_name = 'encore_venue_settings'

    try:
        # Check if table exists
        existing_tables = [table.name for table in dynamodb.tables.all()]
        if table_name in existing_tables:
            print(f"✅ Table {table_name} already exists")
            return

        # Create table
        table = dynamodb.create_table(
            TableName=table_name,
            KeySchema=[
                {
                    'AttributeName': 'venue_id',
                    'KeyType': 'HASH'  # Partition key
                }
            ],
            AttributeDefinitions=[
                {
                    'AttributeName': 'venue_id',
                    'AttributeType': 'S'
                }
            ],
            BillingMode='PAY_PER_REQUEST'  # On-demand pricing
        )

        # Wait for table to be created
        table.meta.client.get_waiter('table_exists').wait(TableName=table_name)

        print(f"✅ Table {table_name} created successfully")
        print(f"   Status: {table.table_status}")

    except ClientError as e:
        print(f"❌ Error creating table: {e}")
        return False

    return True

if __name__ == "__main__":
    create_settings_table()
```

**Run script:**
```bash
cd /home/ec2-user/encore-loyalty-new-system
source venv/bin/activate
python encore_backend/scripts/create_settings_table.py
```

---

### Task 6: Test Settings API

**Purpose:** Verify all endpoints work correctly

**Test script:** `encore_backend/tests/test_settings_api.py`

```python
"""
Manual test script for Settings API
Run with: python encore_backend/tests/test_settings_api.py
"""
import requests
import json

BASE_URL = "http://localhost:8000"

def test_settings_flow():
    """Test complete settings workflow"""

    print("\n=== Testing Settings API ===\n")

    # Step 1: Login to get token
    print("1. Logging in...")
    login_response = requests.post(
        f"{BASE_URL}/api/auth/login",
        json={
            "email": "admin@encore.com",
            "password": "admin123"
        }
    )

    if login_response.status_code != 200:
        print(f"❌ Login failed: {login_response.text}")
        return

    token = login_response.json()["access_token"]
    headers = {"Authorization": f"Bearer {token}"}
    print("✅ Login successful")

    # Step 2: Get settings (should return defaults if none exist)
    print("\n2. Getting venue settings...")
    venue_id = "test_venue_001"

    get_response = requests.get(
        f"{BASE_URL}/api/settings/{venue_id}",
        headers=headers
    )

    print(f"Status: {get_response.status_code}")
    print(f"Response: {json.dumps(get_response.json(), indent=2)}")

    # Step 3: Update AI settings
    print("\n3. Updating AI settings...")
    update_response = requests.put(
        f"{BASE_URL}/api/settings/{venue_id}",
        headers=headers,
        json={
            "ai_settings": {
                "enabled": True,
                "temperature": 0.8,
                "max_tokens": 750,
                "auto_respond": True
            }
        }
    )

    print(f"Status: {update_response.status_code}")
    print(f"Response: {json.dumps(update_response.json(), indent=2)}")

    # Step 4: Update email settings
    print("\n4. Updating email settings...")
    update_response = requests.put(
        f"{BASE_URL}/api/settings/{venue_id}",
        headers=headers,
        json={
            "email_settings": {
                "enabled": True,
                "from_email": "noreply@encore.com",
                "signature": "Thanks,\\nThe Encore Team"
            }
        }
    )

    print(f"Status: {update_response.status_code}")
    print(f"Response: {json.dumps(update_response.json(), indent=2)}")

    # Step 5: Get updated settings
    print("\n5. Getting updated settings...")
    final_response = requests.get(
        f"{BASE_URL}/api/settings/{venue_id}",
        headers=headers
    )

    print(f"Status: {final_response.status_code}")
    print(f"Response: {json.dumps(final_response.json(), indent=2)}")

    print("\n=== Settings API Test Complete ===\n")

if __name__ == "__main__":
    # Make sure backend is running on port 8000
    try:
        test_settings_flow()
    except requests.exceptions.ConnectionError:
        print("❌ Cannot connect to backend. Make sure it's running on port 8000")
```

**Run tests:**
```bash
# Terminal 1: Start backend
cd /home/ec2-user/encore-loyalty-new-system
source venv/bin/activate
cd encore_backend
python main.py

# Terminal 2: Run tests
cd /home/ec2-user/encore-loyalty-new-system
source venv/bin/activate
python encore_backend/tests/test_settings_api.py
```

---

## 🔍 Verification

After completing all tasks, verify:

1. **Models import correctly:**
   ```bash
   python -c "from encore_backend.models.settings import VenueSettings; print('✅ OK')"
   ```

2. **Service imports correctly:**
   ```bash
   python -c "from encore_backend.services.settings_service import SettingsService; print('✅ OK')"
   ```

3. **Routes import correctly:**
   ```bash
   python -c "from encore_backend.routes.settings import router; print('✅ OK')"
   ```

4. **DynamoDB table exists:**
   ```bash
   aws dynamodb describe-table --table-name encore_venue_settings
   ```

5. **API endpoints accessible:**
   ```bash
   curl http://localhost:8000/docs
   # Should see /api/settings/* endpoints in Swagger UI
   ```

6. **Manual test passes:**
   ```bash
   python encore_backend/tests/test_settings_api.py
   # Should complete without errors
   ```

---

## 📝 API Endpoints Summary

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| GET | `/api/settings/{venue_id}` | Get venue settings | Yes |
| PUT | `/api/settings/{venue_id}` | Update venue settings | Yes |
| POST | `/api/settings/{venue_id}` | Create venue settings | Yes (Admin) |
| DELETE | `/api/settings/{venue_id}` | Delete venue settings | Yes (Admin) |

---

## 🚨 Important Notes

1. **DynamoDB Table:** Must be created before testing (Task 5)
2. **Authentication:** All endpoints require valid JWT token from Phase 2A
3. **Venue Access:** Users can only access settings for their assigned venues (except admins)
4. **Default Settings:** If no settings exist, defaults are returned automatically
5. **Validation:** Pydantic models validate all input data (temperature 0-1, email format, etc.)

---

## ✅ Completion Criteria

- [ ] All files created with no syntax errors
- [ ] All imports work correctly
- [ ] DynamoDB table `encore_venue_settings` created
- [ ] Settings routes registered in main app
- [ ] Backend starts without errors
- [ ] Swagger UI shows all settings endpoints
- [ ] Manual test script runs successfully
- [ ] Can GET, PUT settings for a venue via API

---

## 🔄 Handoff to Phase 3B

Once backend is complete and tested, proceed to **Phase 3B: Frontend Settings Integration**.

**Status to report:**
- ✅ Settings models created
- ✅ Settings service implemented
- ✅ Settings API endpoints working
- ✅ DynamoDB table created
- ✅ Manual tests passing

**Next steps:**
- Integrate settings API with React frontend
- Create Settings page UI
- Add settings state management
- Test end-to-end settings flow
