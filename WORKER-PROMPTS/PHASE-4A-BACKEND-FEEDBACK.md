# Phase 4A: Backend Feedback System

**Assigned to:** AI Worker (Backend Developer)
**Model:** Claude Sonnet 4.5
**Environment:** encore_backend directory
**Prerequisites:**
- Phase 2A completed (Authentication working)
- Phase 3A completed (Settings API working)
- MySQL connection established via SSH tunnel

## 📋 Overview

Implement the **Feedback Management** system for the Encore Loyalty backend. This includes reading feedback from the legacy MySQL database, storing AI responses in DynamoDB, managing feedback status, and tracking feedback history.

## 🎯 Objectives

1. Create feedback models (MySQL read + DynamoDB write)
2. Implement MySQL database service for reading feedback
3. Implement DynamoDB service for storing responses and status
4. Create feedback API endpoints with filtering and pagination
5. Add feedback status management
6. Implement action history tracking
7. Test all feedback operations

## 📁 Directory Structure

```
encore_backend/
├── models/
│   ├── feedback.py              # ← CREATE
│   └── feedback_response.py     # ← CREATE
├── services/
│   ├── mysql_service.py         # ← CREATE (read feedback from MySQL)
│   ├── feedback_service.py      # ← CREATE (business logic)
│   └── feedback_history.py      # ← CREATE (track changes)
├── routes/
│   └── feedback.py              # ← CREATE
├── database/
│   └── mysql.py                 # ← CREATE (MySQL connection)
└── main.py                      # ← UPDATE (register feedback routes)
```

## ✅ Task Checklist

### Task 1: Create Feedback Models (`models/feedback.py`)

**Purpose:** Define Pydantic models for feedback data

**Create file:** `encore_backend/models/feedback.py`

```python
"""
Feedback models - reads from MySQL, writes status to DynamoDB
"""
from typing import Optional, List
from pydantic import BaseModel, Field
from datetime import datetime
from enum import Enum

class FeedbackStatus(str, Enum):
    """Feedback processing status"""
    PENDING = "pending"           # New feedback, not processed
    REVIEWED = "reviewed"         # Manually reviewed
    GENERATED = "generated"       # AI response generated
    EDITED = "edited"             # Response manually edited
    SENT = "sent"                 # Response sent via email
    ARCHIVED = "archived"         # Archived/closed
    SPAM = "spam"                 # Marked as spam

class Sentiment(str, Enum):
    """Sentiment analysis results"""
    POSITIVE = "positive"
    NEUTRAL = "neutral"
    NEGATIVE = "negative"
    MIXED = "mixed"

class Feedback(BaseModel):
    """
    Feedback model - combines data from MySQL and DynamoDB

    MySQL fields: id, venue_id, customer_email, customer_name,
                  rating, comment, created_at (from Encore legacy)
    DynamoDB fields: status, ai_response_id, sentiment, last_action
    """
    # From MySQL (read-only)
    id: int
    venue_id: str
    venue_name: Optional[str] = None
    customer_email: str
    customer_name: Optional[str] = None
    rating: Optional[int] = Field(None, ge=1, le=5)  # 1-5 stars
    comment: str
    created_at: datetime

    # From DynamoDB (writable)
    status: FeedbackStatus = FeedbackStatus.PENDING
    sentiment: Optional[Sentiment] = None
    ai_response_id: Optional[str] = None
    last_updated: Optional[datetime] = None
    last_action_by: Optional[str] = None

    # Metadata
    response_count: int = 0  # Number of times response generated
    email_sent_count: int = 0  # Number of times email sent

    class Config:
        json_encoders = {
            datetime: lambda v: v.isoformat()
        }
        use_enum_values = True

class FeedbackListItem(BaseModel):
    """Simplified feedback for list views"""
    id: int
    venue_id: str
    venue_name: Optional[str]
    customer_name: Optional[str]
    rating: Optional[int]
    comment_preview: str  # First 100 chars
    status: FeedbackStatus
    sentiment: Optional[Sentiment]
    created_at: datetime
    has_response: bool

class FeedbackFilter(BaseModel):
    """Filter parameters for feedback queries"""
    venue_id: Optional[str] = None
    status: Optional[FeedbackStatus] = None
    sentiment: Optional[Sentiment] = None
    rating: Optional[int] = None
    date_from: Optional[datetime] = None
    date_to: Optional[datetime] = None
    search_text: Optional[str] = None  # Search in comment or customer name

    # Pagination
    page: int = Field(default=1, ge=1)
    page_size: int = Field(default=20, ge=1, le=100)

class FeedbackListResponse(BaseModel):
    """Paginated feedback list response"""
    items: List[FeedbackListItem]
    total: int
    page: int
    page_size: int
    total_pages: int

class UpdateFeedbackStatusRequest(BaseModel):
    """Request to update feedback status"""
    status: FeedbackStatus
    notes: Optional[str] = None
```

**Validation:**
```bash
cd /home/ec2-user/encore-loyalty-new-system
source venv/bin/activate
python -c "from encore_backend.models.feedback import Feedback, FeedbackStatus; print('✅ Feedback models imported successfully')"
```

---

### Task 2: Create Feedback Response Models (`models/feedback_response.py`)

**Purpose:** Models for AI-generated responses stored in DynamoDB

**Create file:** `encore_backend/models/feedback_response.py`

```python
"""
Feedback response models - AI generated responses stored in DynamoDB
"""
from typing import Optional, Dict, Any
from pydantic import BaseModel
from datetime import datetime
from enum import Enum

class ResponseGenerationMethod(str, Enum):
    """How the response was generated"""
    AI_GENERATED = "ai_generated"      # Generated by AI
    AI_REGENERATED = "ai_regenerated"  # Regenerated with custom instructions
    MANUAL = "manual"                  # Manually written

class FeedbackResponse(BaseModel):
    """
    AI-generated or manual response to feedback
    Stored in DynamoDB
    """
    response_id: str  # UUID
    feedback_id: int  # References MySQL feedback.id
    venue_id: str

    # Response content
    response_text: str
    generation_method: ResponseGenerationMethod

    # AI generation details (if applicable)
    ai_model: Optional[str] = None  # e.g., "claude-3-5-sonnet-20241022"
    ai_temperature: Optional[float] = None
    ai_max_tokens: Optional[int] = None
    custom_instructions: Optional[str] = None  # For regeneration

    # Customer profile used
    customer_profile: Optional[Dict[str, Any]] = None

    # Metadata
    created_at: datetime
    created_by: str  # User email who generated/wrote it
    edited_at: Optional[datetime] = None
    edited_by: Optional[str] = None

    # Email sending
    email_sent: bool = False
    email_sent_at: Optional[datetime] = None
    email_status: Optional[str] = None  # "sent", "failed", "bounced"

    class Config:
        json_encoders = {
            datetime: lambda v: v.isoformat()
        }

class CreateResponseRequest(BaseModel):
    """Request to create/update a response"""
    response_text: str
    generation_method: ResponseGenerationMethod = ResponseGenerationMethod.MANUAL
    custom_instructions: Optional[str] = None

class RegenerateResponseRequest(BaseModel):
    """Request to regenerate AI response with custom instructions"""
    custom_instructions: str
    temperature: Optional[float] = None
    max_tokens: Optional[int] = None
```

---

### Task 3: Create MySQL Database Service (`database/mysql.py`)

**Purpose:** MySQL connection manager with SSH tunnel support

**Create file:** `encore_backend/database/mysql.py`

```python
"""
MySQL database connection via SSH tunnel
Reads feedback from legacy Encore database (read-only)
"""
import os
import mysql.connector
from mysql.connector import Error
from typing import Optional
from sshtunnel import SSHTunnelForwarder

class MySQLConnection:
    """MySQL connection manager with SSH tunnel"""

    def __init__(self):
        # SSH tunnel configuration
        self.ssh_host = os.getenv('MYSQL_SSH_HOST')
        self.ssh_port = int(os.getenv('MYSQL_SSH_PORT', '22'))
        self.ssh_user = os.getenv('MYSQL_SSH_USER')
        self.ssh_key_path = os.getenv('MYSQL_SSH_KEY_PATH')

        # MySQL configuration
        self.mysql_host = os.getenv('MYSQL_HOST', 'localhost')
        self.mysql_port = int(os.getenv('MYSQL_PORT', '3306'))
        self.mysql_user = os.getenv('MYSQL_USER')
        self.mysql_password = os.getenv('MYSQL_PASSWORD')
        self.mysql_database = os.getenv('MYSQL_DATABASE')

        self.tunnel: Optional[SSHTunnelForwarder] = None
        self.connection = None

    def __enter__(self):
        """Context manager entry - establish connection"""
        try:
            # Start SSH tunnel if configured
            if self.ssh_host and self.ssh_key_path:
                print(f"Starting SSH tunnel to {self.ssh_host}...")
                self.tunnel = SSHTunnelForwarder(
                    (self.ssh_host, self.ssh_port),
                    ssh_username=self.ssh_user,
                    ssh_pkey=self.ssh_key_path,
                    remote_bind_address=(self.mysql_host, self.mysql_port)
                )
                self.tunnel.start()

                # Connect via tunnel
                local_port = self.tunnel.local_bind_port
                self.connection = mysql.connector.connect(
                    host='127.0.0.1',
                    port=local_port,
                    user=self.mysql_user,
                    password=self.mysql_password,
                    database=self.mysql_database
                )
            else:
                # Direct connection (no tunnel)
                self.connection = mysql.connector.connect(
                    host=self.mysql_host,
                    port=self.mysql_port,
                    user=self.mysql_user,
                    password=self.mysql_password,
                    database=self.mysql_database
                )

            print("✅ MySQL connection established")
            return self.connection

        except Error as e:
            print(f"❌ Error connecting to MySQL: {e}")
            raise

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Context manager exit - close connection"""
        if self.connection:
            self.connection.close()
            print("MySQL connection closed")

        if self.tunnel:
            self.tunnel.stop()
            print("SSH tunnel closed")

def get_mysql_connection():
    """Get MySQL connection context manager"""
    return MySQLConnection()
```

**Note:** Make sure `sshtunnel` is installed:
```bash
pip install sshtunnel mysql-connector-python
```

---

### Task 4: Create MySQL Service (`services/mysql_service.py`)

**Purpose:** Read feedback data from MySQL

**Create file:** `encore_backend/services/mysql_service.py`

```python
"""
MySQL service for reading feedback from legacy Encore database
"""
from typing import List, Optional, Dict, Any, Tuple
from datetime import datetime
from database.mysql import get_mysql_connection
from models.feedback import Feedback, FeedbackFilter, FeedbackListItem, Sentiment

class MySQLService:
    """Service for reading feedback from MySQL"""

    def get_feedback_by_id(self, feedback_id: int) -> Optional[Dict[str, Any]]:
        """
        Get single feedback by ID from MySQL

        Returns raw dict from database
        """
        with get_mysql_connection() as conn:
            cursor = conn.cursor(dictionary=True)

            # Adjust query based on actual table schema
            # This is a template - update field names as needed
            query = """
                SELECT
                    id,
                    venue_id,
                    customer_email,
                    customer_name,
                    rating,
                    comment,
                    created_at
                FROM feedback
                WHERE id = %s
            """

            cursor.execute(query, (feedback_id,))
            result = cursor.fetchone()
            cursor.close()

            return result

    def list_feedback(
        self,
        filters: FeedbackFilter
    ) -> Tuple[List[Dict[str, Any]], int]:
        """
        List feedback with filters and pagination

        Returns (items, total_count)
        """
        with get_mysql_connection() as conn:
            cursor = conn.cursor(dictionary=True)

            # Build WHERE clause based on filters
            where_clauses = []
            params = []

            if filters.venue_id:
                where_clauses.append("venue_id = %s")
                params.append(filters.venue_id)

            if filters.rating:
                where_clauses.append("rating = %s")
                params.append(filters.rating)

            if filters.date_from:
                where_clauses.append("created_at >= %s")
                params.append(filters.date_from)

            if filters.date_to:
                where_clauses.append("created_at <= %s")
                params.append(filters.date_to)

            if filters.search_text:
                where_clauses.append(
                    "(comment LIKE %s OR customer_name LIKE %s)"
                )
                search_pattern = f"%{filters.search_text}%"
                params.extend([search_pattern, search_pattern])

            # Build WHERE clause
            where_sql = ""
            if where_clauses:
                where_sql = "WHERE " + " AND ".join(where_clauses)

            # Get total count
            count_query = f"SELECT COUNT(*) as total FROM feedback {where_sql}"
            cursor.execute(count_query, params)
            total = cursor.fetchone()['total']

            # Get paginated results
            offset = (filters.page - 1) * filters.page_size

            query = f"""
                SELECT
                    id,
                    venue_id,
                    customer_email,
                    customer_name,
                    rating,
                    comment,
                    created_at
                FROM feedback
                {where_sql}
                ORDER BY created_at DESC
                LIMIT %s OFFSET %s
            """

            cursor.execute(query, params + [filters.page_size, offset])
            items = cursor.fetchall()
            cursor.close()

            return items, total

    def get_feedback_count_by_venue(self, venue_id: str) -> int:
        """Get total feedback count for a venue"""
        with get_mysql_connection() as conn:
            cursor = conn.cursor()

            query = "SELECT COUNT(*) FROM feedback WHERE venue_id = %s"
            cursor.execute(query, (venue_id,))
            count = cursor.fetchone()[0]
            cursor.close()

            return count
```

---

### Task 5: Create Feedback Service (`services/feedback_service.py`)

**Purpose:** Business logic combining MySQL reads with DynamoDB writes

**Create file:** `encore_backend/services/feedback_service.py`

```python
"""
Feedback service - combines MySQL (read) with DynamoDB (write)
"""
import boto3
from typing import Optional, List
from datetime import datetime
import uuid
from botocore.exceptions import ClientError

from services.mysql_service import MySQLService
from models.feedback import (
    Feedback,
    FeedbackListItem,
    FeedbackFilter,
    FeedbackListResponse,
    FeedbackStatus,
    Sentiment
)
from models.feedback_response import FeedbackResponse

class FeedbackService:
    """Service for feedback management"""

    def __init__(self):
        self.mysql = MySQLService()
        self.dynamodb = boto3.resource('dynamodb')
        self.status_table = self.dynamodb.Table('encore_feedback_status')
        self.response_table = self.dynamodb.Table('encore_feedback_responses')

    def get_feedback(self, feedback_id: int) -> Optional[Feedback]:
        """
        Get complete feedback with status and response data

        Combines:
        - MySQL: feedback content
        - DynamoDB: status, response metadata
        """
        # Get from MySQL
        mysql_data = self.mysql.get_feedback_by_id(feedback_id)
        if not mysql_data:
            return None

        # Get status from DynamoDB
        try:
            status_response = self.status_table.get_item(
                Key={'feedback_id': feedback_id}
            )

            if 'Item' in status_response:
                status_data = status_response['Item']
            else:
                # Default status if not in DynamoDB
                status_data = {
                    'status': FeedbackStatus.PENDING.value,
                    'sentiment': None,
                    'ai_response_id': None,
                    'last_updated': None,
                    'last_action_by': None,
                    'response_count': 0,
                    'email_sent_count': 0
                }
        except ClientError:
            status_data = {
                'status': FeedbackStatus.PENDING.value,
                'sentiment': None,
                'ai_response_id': None
            }

        # Combine data
        return Feedback(
            **mysql_data,
            status=status_data.get('status', FeedbackStatus.PENDING),
            sentiment=status_data.get('sentiment'),
            ai_response_id=status_data.get('ai_response_id'),
            last_updated=status_data.get('last_updated'),
            last_action_by=status_data.get('last_action_by'),
            response_count=status_data.get('response_count', 0),
            email_sent_count=status_data.get('email_sent_count', 0)
        )

    def list_feedback(
        self,
        filters: FeedbackFilter
    ) -> FeedbackListResponse:
        """
        List feedback with filters and pagination
        """
        # Get from MySQL
        mysql_items, total = self.mysql.list_feedback(filters)

        # Enrich with DynamoDB status
        feedback_ids = [item['id'] for item in mysql_items]
        status_map = self._get_status_batch(feedback_ids)

        # Build list items
        items = []
        for mysql_item in mysql_items:
            feedback_id = mysql_item['id']
            status_data = status_map.get(feedback_id, {})

            # Apply status filter if specified
            status = status_data.get('status', FeedbackStatus.PENDING)
            if filters.status and status != filters.status:
                continue

            # Apply sentiment filter if specified
            sentiment = status_data.get('sentiment')
            if filters.sentiment and sentiment != filters.sentiment:
                continue

            # Create list item
            comment_preview = mysql_item['comment'][:100]
            if len(mysql_item['comment']) > 100:
                comment_preview += "..."

            items.append(FeedbackListItem(
                id=feedback_id,
                venue_id=mysql_item['venue_id'],
                venue_name=mysql_item.get('venue_name'),
                customer_name=mysql_item.get('customer_name'),
                rating=mysql_item.get('rating'),
                comment_preview=comment_preview,
                status=status,
                sentiment=sentiment,
                created_at=mysql_item['created_at'],
                has_response=bool(status_data.get('ai_response_id'))
            ))

        # Calculate pagination
        total_pages = (total + filters.page_size - 1) // filters.page_size

        return FeedbackListResponse(
            items=items,
            total=total,
            page=filters.page,
            page_size=filters.page_size,
            total_pages=total_pages
        )

    def update_status(
        self,
        feedback_id: int,
        status: FeedbackStatus,
        updated_by: str,
        notes: Optional[str] = None
    ) -> bool:
        """Update feedback status in DynamoDB"""
        try:
            self.status_table.update_item(
                Key={'feedback_id': feedback_id},
                UpdateExpression="""
                    SET #status = :status,
                        last_updated = :updated,
                        last_action_by = :by
                """,
                ExpressionAttributeNames={
                    '#status': 'status'
                },
                ExpressionAttributeValues={
                    ':status': status.value,
                    ':updated': datetime.utcnow().isoformat(),
                    ':by': updated_by
                }
            )

            # TODO: Add to history table

            return True
        except ClientError as e:
            print(f"Error updating status: {e}")
            return False

    def _get_status_batch(self, feedback_ids: List[int]) -> dict:
        """Get status for multiple feedback IDs"""
        if not feedback_ids:
            return {}

        try:
            # Batch get from DynamoDB
            keys = [{'feedback_id': fid} for fid in feedback_ids]
            response = self.dynamodb.batch_get_item(
                RequestItems={
                    'encore_feedback_status': {
                        'Keys': keys
                    }
                }
            )

            # Build map
            status_map = {}
            for item in response.get('Responses', {}).get('encore_feedback_status', []):
                status_map[item['feedback_id']] = item

            return status_map
        except ClientError:
            return {}
```

---

### Task 6: Create Feedback Routes (`routes/feedback.py`)

**Purpose:** API endpoints for feedback management

**Create file:** `encore_backend/routes/feedback.py`

```python
"""
Feedback API routes
"""
from fastapi import APIRouter, Depends, HTTPException, status, Query
from typing import Optional
from datetime import datetime

from models.user import User
from models.feedback import (
    Feedback,
    FeedbackFilter,
    FeedbackListResponse,
    FeedbackStatus,
    Sentiment,
    UpdateFeedbackStatusRequest
)
from services.feedback_service import FeedbackService
from services.jwt_service import get_current_user

router = APIRouter(prefix="/api/feedback", tags=["feedback"])
feedback_service = FeedbackService()

@router.get("", response_model=FeedbackListResponse)
async def list_feedback(
    venue_id: Optional[str] = Query(None),
    status: Optional[FeedbackStatus] = Query(None),
    sentiment: Optional[Sentiment] = Query(None),
    rating: Optional[int] = Query(None, ge=1, le=5),
    date_from: Optional[datetime] = Query(None),
    date_to: Optional[datetime] = Query(None),
    search_text: Optional[str] = Query(None),
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    current_user: User = Depends(get_current_user)
):
    """
    List feedback with filters and pagination

    Filters:
    - venue_id: Filter by specific venue
    - status: Filter by processing status
    - sentiment: Filter by sentiment
    - rating: Filter by star rating (1-5)
    - date_from/date_to: Date range filter
    - search_text: Search in comment or customer name
    - page/page_size: Pagination
    """
    # Build filter
    filters = FeedbackFilter(
        venue_id=venue_id,
        status=status,
        sentiment=sentiment,
        rating=rating,
        date_from=date_from,
        date_to=date_to,
        search_text=search_text,
        page=page,
        page_size=page_size
    )

    # Check venue access (non-admins can only see their venues)
    if current_user.role != "admin" and venue_id:
        if venue_id not in current_user.venue_ids:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Access denied to this venue"
            )

    return feedback_service.list_feedback(filters)

@router.get("/{feedback_id}", response_model=Feedback)
async def get_feedback(
    feedback_id: int,
    current_user: User = Depends(get_current_user)
):
    """Get single feedback by ID"""
    feedback = feedback_service.get_feedback(feedback_id)

    if not feedback:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Feedback {feedback_id} not found"
        )

    # Check venue access
    if current_user.role != "admin":
        if feedback.venue_id not in current_user.venue_ids:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Access denied to this feedback"
            )

    return feedback

@router.put("/{feedback_id}/status")
async def update_feedback_status(
    feedback_id: int,
    request: UpdateFeedbackStatusRequest,
    current_user: User = Depends(get_current_user)
):
    """Update feedback status"""
    # Get feedback to check access
    feedback = feedback_service.get_feedback(feedback_id)
    if not feedback:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Feedback {feedback_id} not found"
        )

    # Check venue access
    if current_user.role != "admin":
        if feedback.venue_id not in current_user.venue_ids:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Access denied to this feedback"
            )

    # Update status
    success = feedback_service.update_status(
        feedback_id=feedback_id,
        status=request.status,
        updated_by=current_user.email,
        notes=request.notes
    )

    if not success:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Failed to update status"
        )

    return {"success": True, "message": "Status updated successfully"}

@router.post("/{feedback_id}/generate")
async def generate_response(
    feedback_id: int,
    current_user: User = Depends(get_current_user)
):
    """
    Generate AI response for feedback
    (Stub for now - will be implemented in Phase 5)
    """
    feedback = feedback_service.get_feedback(feedback_id)
    if not feedback:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Feedback {feedback_id} not found"
        )

    # Check venue access
    if current_user.role != "admin":
        if feedback.venue_id not in current_user.venue_ids:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Access denied"
            )

    # TODO: Implement in Phase 5
    return {
        "success": False,
        "message": "AI generation not yet implemented (Phase 5)"
    }

@router.post("/{feedback_id}/send")
async def send_response(
    feedback_id: int,
    current_user: User = Depends(get_current_user)
):
    """
    Send response via email
    (Stub for now - will be implemented in Phase 6)
    """
    feedback = feedback_service.get_feedback(feedback_id)
    if not feedback:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Feedback {feedback_id} not found"
        )

    # Check venue access
    if current_user.role != "admin":
        if feedback.venue_id not in current_user.venue_ids:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Access denied"
            )

    # TODO: Implement in Phase 6
    return {
        "success": False,
        "message": "Email sending not yet implemented (Phase 6)"
    }
```

---

### Task 7: Create DynamoDB Tables

**Purpose:** Create tables for feedback status and responses

**Create script:** `encore_backend/scripts/create_feedback_tables.py`

```python
"""
Create DynamoDB tables for feedback system
"""
import boto3
from botocore.exceptions import ClientError

def create_feedback_tables():
    """Create DynamoDB tables for feedback status and responses"""
    dynamodb = boto3.resource('dynamodb')

    tables_to_create = [
        {
            'TableName': 'encore_feedback_status',
            'KeySchema': [
                {'AttributeName': 'feedback_id', 'KeyType': 'HASH'}
            ],
            'AttributeDefinitions': [
                {'AttributeName': 'feedback_id', 'AttributeType': 'N'}
            ]
        },
        {
            'TableName': 'encore_feedback_responses',
            'KeySchema': [
                {'AttributeName': 'response_id', 'KeyType': 'HASH'}
            ],
            'AttributeDefinitions': [
                {'AttributeName': 'response_id', 'AttributeType': 'S'},
                {'AttributeName': 'feedback_id', 'AttributeType': 'N'}
            ],
            'GlobalSecondaryIndexes': [
                {
                    'IndexName': 'feedback_id_index',
                    'KeySchema': [
                        {'AttributeName': 'feedback_id', 'KeyType': 'HASH'}
                    ],
                    'Projection': {'ProjectionType': 'ALL'}
                }
            ]
        }
    ]

    existing_tables = [table.name for table in dynamodb.tables.all()]

    for table_config in tables_to_create:
        table_name = table_config['TableName']

        if table_name in existing_tables:
            print(f"✅ Table {table_name} already exists")
            continue

        try:
            # Add billing mode
            table_config['BillingMode'] = 'PAY_PER_REQUEST'

            table = dynamodb.create_table(**table_config)
            table.meta.client.get_waiter('table_exists').wait(TableName=table_name)

            print(f"✅ Table {table_name} created successfully")
        except ClientError as e:
            print(f"❌ Error creating table {table_name}: {e}")

if __name__ == "__main__":
    create_feedback_tables()
```

---

### Task 8: Register Routes and Test

**Update `main.py`:**
```python
from routes.feedback import router as feedback_router

# Register feedback routes
app.include_router(feedback_router)
```

**Test:**
```bash
# Create tables
python encore_backend/scripts/create_feedback_tables.py

# Start backend
cd encore_backend
python main.py

# Test endpoints
curl http://localhost:8000/docs
# Should see /api/feedback/* endpoints
```

---

## 🔍 Verification

- [ ] All models import without errors
- [ ] MySQL connection works (test with real DB credentials)
- [ ] DynamoDB tables created
- [ ] Feedback routes registered
- [ ] Can list feedback via API
- [ ] Can get single feedback via API
- [ ] Can update feedback status
- [ ] Filtering and pagination work

---

## 📝 Important Notes

1. **MySQL Schema:** Update queries in `mysql_service.py` to match actual table schema
2. **SSH Tunnel:** Test SSH connection separately before integrating
3. **Read-Only:** MySQL access is read-only - never write to legacy DB
4. **Status Storage:** All status updates go to DynamoDB only
5. **Performance:** Consider caching frequently accessed data

---

## ✅ Completion Criteria

- [ ] Can read feedback from MySQL database
- [ ] Can list feedback with filters (venue, status, date, etc.)
- [ ] Can get single feedback details
- [ ] Can update feedback status (stored in DynamoDB)
- [ ] Pagination works correctly
- [ ] Search functionality works
- [ ] Authentication and authorization enforced
- [ ] All endpoints documented in Swagger UI

---

## 🔄 Handoff to Phase 4B

**Status to report:**
- ✅ Feedback models created
- ✅ MySQL integration working
- ✅ DynamoDB tables created
- ✅ Feedback API endpoints functional
- ✅ Filtering and pagination implemented

**Next steps:**
- Integrate feedback API with React frontend
- Create feedback list page
- Create feedback detail page
- Add filtering UI
- Test end-to-end flow
