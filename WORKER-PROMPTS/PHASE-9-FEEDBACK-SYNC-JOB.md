# Phase 9: Feedback Sync Job Implementation

## Worker Prompt for AI Assistant

**Task**: Implement a MySQL → DynamoDB feedback synchronization system
**Priority**: HIGH
**Estimated Effort**: 6-8 hours
**Dependencies**: Phase 5 & 6 complete (AI + Email working)

---

## 🎯 OBJECTIVE

Create a synchronization system that:
1. Copies **new** customer feedback from MySQL (legacy) to DynamoDB (new system)
2. Can be triggered manually from the UI (Global Settings page)
3. **Does NOT create duplicates** - idempotent operation
4. **Does NOT overwrite** DynamoDB-only data (AI responses, drafts, etc.)
5. Tracks sync history and provides status visibility

---

## 📋 CONTEXT

### Why This Is Needed

```
CURRENT PROBLEM:
┌────────────────────────────────────────────────────────────────┐
│ New feedback arrives in MySQL daily                            │
│              ↓                                                 │
│ AI Feedback page reads from DynamoDB (default)                 │
│              ↓                                                 │
│ DynamoDB only has data from ONE-TIME onboarding                │
│              ↓                                                 │
│ ⚠️ NEW FEEDBACK IS INVISIBLE IN AI FEEDBACK PAGE!             │
└────────────────────────────────────────────────────────────────┘

SOLUTION:
┌────────────────────────────────────────────────────────────────┐
│ Periodic/Manual sync job                                       │
│              ↓                                                 │
│ Query MySQL for feedback NEWER than last_sync_timestamp        │
│              ↓                                                 │
│ Create new Customer/Comment entities in DynamoDB               │
│              ↓                                                 │
│ ✅ New feedback visible in AI Feedback page!                   │
└────────────────────────────────────────────────────────────────┘
```

### Current Data Structure

**MySQL Tables** (Legacy - Source):
- `client` - Restaurant/venue info
- `datasession` - Feedback sessions
- `comment` - Customer comments/reviews
- `eclub` - Customer info (email, name, phone)

**DynamoDB Tables** (New System - Target):
- `restaurant_data` - Restaurant/Customer/Comment entities (hierarchical)
- `encore_venue_settings` - Venue AI settings
- `encore_draft_responses` - AI-generated drafts awaiting review
- `encore_feedback_responses` - Approved AI responses

### Key Files to Reference

```
encore_backend/
├── db/
│   ├── direct_mysql_service.py    # MySQL queries (use this)
│   └── dynamodb_service.py        # DynamoDB operations
├── services/
│   └── onboard_service.py         # Existing migration logic (reference)
├── routes/
│   ├── onboard.py                 # Onboarding API (reference)
│   └── clients.py                 # Client/reviews API
└── models/
    └── restaurant.py              # Restaurant/Customer/Comment models
```

---

## 🏗️ IMPLEMENTATION PLAN

### Part 1: Backend - Sync Service

**File**: `encore_backend/services/feedback_sync_service.py`

```python
"""
Feedback Sync Service

Synchronizes new customer feedback from MySQL to DynamoDB.
Designed for incremental updates without duplicates.
"""

from datetime import datetime
from typing import Dict, Any, List, Optional
import logging

from db.direct_mysql_service import DirectMySQLService
from db.dynamodb_service import RestaurantDynamoDBService
from models.restaurant import Customer, Comment

logger = logging.getLogger(__name__)

class FeedbackSyncService:
    """
    Service for synchronizing feedback data from MySQL to DynamoDB.
    
    Features:
    - Incremental sync (only new data since last sync)
    - Idempotent (safe to run multiple times)
    - Preserves existing DynamoDB data
    - Tracks sync metadata
    """
    
    def __init__(
        self,
        mysql_service: DirectMySQLService,
        dynamo_service: RestaurantDynamoDBService
    ):
        self.mysql = mysql_service
        self.dynamo = dynamo_service
    
    async def sync_venue(self, client_id: int) -> Dict[str, Any]:
        """
        Sync feedback for a single venue.
        
        Args:
            client_id: MySQL client ID
            
        Returns:
            Sync result with statistics
        """
        # Implementation here
        pass
    
    async def sync_all_venues(self) -> Dict[str, Any]:
        """
        Sync feedback for all onboarded venues.
        
        Returns:
            Combined sync results
        """
        pass
    
    def _get_last_sync_timestamp(self, restaurant_id: str) -> Optional[datetime]:
        """Get the last sync timestamp for a restaurant."""
        pass
    
    def _update_sync_timestamp(self, restaurant_id: str, timestamp: datetime):
        """Update the sync timestamp after successful sync."""
        pass
    
    def _comment_exists(self, restaurant_id: str, original_review_id: int) -> bool:
        """Check if a comment already exists (by original MySQL ID)."""
        pass
```

### Part 2: Backend - Sync API Routes

**File**: `encore_backend/routes/sync.py`

```python
"""
Feedback Sync API Routes

Endpoints for triggering and monitoring feedback synchronization.
"""

from fastapi import APIRouter, Depends, HTTPException, BackgroundTasks
from typing import Optional

router = APIRouter(prefix="/api/sync", tags=["sync"])

@router.post("/{client_id}")
async def sync_venue_feedback(
    client_id: int,
    background_tasks: BackgroundTasks
):
    """
    Trigger sync for a specific venue.
    
    - Runs in background
    - Returns immediately with job ID
    - Check status via GET /api/sync/status/{job_id}
    """
    pass

@router.post("/all")
async def sync_all_feedback(
    background_tasks: BackgroundTasks
):
    """
    Trigger sync for all onboarded venues.
    """
    pass

@router.get("/status")
async def get_sync_status():
    """
    Get overall sync status for all venues.
    
    Returns:
    - Last sync time per venue
    - Sync in progress indicator
    - Recent sync history
    """
    pass

@router.get("/status/{client_id}")
async def get_venue_sync_status(client_id: int):
    """
    Get sync status for a specific venue.
    """
    pass

@router.get("/history")
async def get_sync_history(limit: int = 10):
    """
    Get recent sync job history.
    """
    pass
```

### Part 3: Backend - Sync Metadata Storage

**Option A**: Add to existing `restaurant_data` table
```python
# Add to Restaurant entity
last_sync_at: Optional[datetime]
last_sync_stats: Optional[Dict]  # {new_customers: 5, new_comments: 23}
```

**Option B**: Create new `encore_sync_metadata` table
```
PK: SYNC#{restaurant_id}
SK: METADATA
last_sync_at: ISO timestamp
last_sync_stats: {...}
sync_history: [...last 10 syncs...]
```

**Recommendation**: Option A is simpler, use existing Restaurant entity.

### Part 4: Frontend - Sync UI

**File**: `encore_frontend/src/services/api/syncApi.ts`

```typescript
/**
 * Sync API Service
 */

export interface SyncStatus {
  client_id: number;
  restaurant_name: string;
  last_sync_at: string | null;
  last_sync_stats: {
    new_customers: number;
    new_comments: number;
    errors: number;
  } | null;
  sync_in_progress: boolean;
}

export interface SyncResult {
  success: boolean;
  client_id: number;
  new_customers: number;
  new_comments: number;
  skipped_duplicates: number;
  errors: string[];
  duration_ms: number;
}

export const syncVenue = async (clientId: number): Promise<{ job_id: string }>;
export const syncAllVenues = async (): Promise<{ job_id: string }>;
export const getSyncStatus = async (): Promise<SyncStatus[]>;
export const getVenueSyncStatus = async (clientId: number): Promise<SyncStatus>;
```

**File**: Add to `encore_frontend/src/pages/admin/GlobalSettings.tsx`

Add a new **"Data Synchronization"** section/card with:

```
┌─────────────────────────────────────────────────────────────────┐
│  📊 Data Synchronization                                        │
│                                                                 │
│  [🔄 Sync All Venues]  ← Primary action button (blue)           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ Venue               │ Last Sync        │ Status   │ Action  ││
│  ├─────────────────────┼──────────────────┼──────────┼─────────┤│
│  │ Opus Ocean Grille   │ 2 hours ago      │ ✅ Synced│ [Sync]  ││
│  │ Bella Vista         │ Never            │ ⚠️ Needed│ [Sync]  ││
│  │ Downtown Bistro     │ 1 day ago        │ ✅ Synced│ [Sync]  ││
│  └─────────────────────┴──────────────────┴──────────┴─────────┘│
│                                                                 │
│  📋 Recent Sync History                                         │
│  • Nov 28, 2:30 PM - Opus Ocean Grille: 15 new comments synced  │
│  • Nov 28, 2:30 PM - Bella Vista: 8 new comments synced         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**UI Components Required**:
1. **"Sync All Venues" Button** - Triggers `POST /api/sync/all`
   - Shows loading spinner while syncing
   - Displays success/error toast notification

2. **Venues Table** - Shows all onboarded venues with:
   - Venue name
   - Last sync timestamp (relative time: "2 hours ago", "Never")
   - Status indicator (✅ Synced, ⚠️ Needs Sync, 🔄 Syncing...)
   - Per-venue "Sync" button - Triggers `POST /api/sync/{client_id}`

3. **Sync History** - Last 5-10 sync operations with:
   - Timestamp
   - Venue name
   - Results (X new comments, X new customers)

4. **Loading States**:
   - Disable buttons while sync in progress
   - Show spinner on active sync button
   - Update table status to "🔄 Syncing..." during operation

---

## 🔑 KEY IMPLEMENTATION DETAILS

### 1. Duplicate Prevention (Critical!)

```python
def _comment_exists(self, restaurant_id: str, original_review_id: int) -> bool:
    """
    Check if comment already exists in DynamoDB.
    
    We store original_review_id in Comment entity during onboarding.
    This field links back to MySQL comment.id.
    """
    # Query DynamoDB for comments with this original_review_id
    # The Comment model already has: original_review_id: Optional[int]
    
    # Option 1: Query by original_review_id (if indexed)
    # Option 2: Scan restaurant's comments (slower but works)
    pass
```

### 2. Customer Deduplication

```python
def _find_or_create_customer(
    self,
    restaurant_id: str,
    email: str,
    phone: str,
    name: str
) -> Customer:
    """
    Find existing customer or create new one.
    
    Deduplication logic:
    1. Match by email (primary)
    2. Match by phone (secondary)
    3. Create new if no match
    """
    # Use existing get_or_create_customer from dynamo_service
    pass
```

### 3. Preserve Existing Data

```python
def _sync_comment(
    self,
    restaurant_id: str,
    customer_id: str,
    mysql_comment: Dict
) -> bool:
    """
    Create comment in DynamoDB if it doesn't exist.
    
    IMPORTANT: Never update existing comments!
    They may have AI responses attached.
    """
    # Check if exists
    if self._comment_exists(restaurant_id, mysql_comment['id']):
        return False  # Skip, already synced
    
    # Create new comment
    comment = Comment(
        comment_id=str(uuid.uuid4()),
        comment_text=mysql_comment['comment_text'],
        rating=mysql_comment['rating'],
        date_created=mysql_comment['date_entered'],
        original_review_id=mysql_comment['id']  # Link to MySQL
    )
    
    self.dynamo.create_comment(restaurant_id, customer_id, comment)
    return True  # New comment created
```

### 4. Incremental Sync Query

```python
def _get_new_mysql_reviews(
    self,
    client_id: int,
    since: datetime
) -> List[Dict]:
    """
    Get MySQL reviews created after the given timestamp.
    
    Query structure:
    SELECT c.*, e.emailaddress, e.firstname, e.lastname, e.phonenumber
    FROM comment c
    JOIN datasession d ON c.iddatasession = d.iddatasession
    LEFT JOIN eclub e ON d.ideclub = e.ideclub
    WHERE d.idclient = {client_id}
      AND c.date_entered > '{since}'
    ORDER BY c.date_entered ASC
    """
    # Use DirectMySQLService
    pass
```

---

## 📁 FILES TO CREATE/MODIFY

### New Files

| File | Purpose |
|------|---------|
| `encore_backend/services/feedback_sync_service.py` | Core sync logic |
| `encore_backend/routes/sync.py` | Sync API endpoints |
| `encore_frontend/src/services/api/syncApi.ts` | Frontend API client |

### Files to Modify

| File | Changes |
|------|---------|
| `encore_backend/main.py` | Register sync router |
| `encore_backend/models/restaurant.py` | Add `last_sync_at`, `last_sync_stats` to Restaurant |
| `encore_backend/db/dynamodb_service.py` | Add `update_restaurant_sync_metadata()` method |
| `encore_backend/db/direct_mysql_service.py` | Add `get_reviews_since()` method |
| `encore_frontend/src/pages/admin/GlobalSettings.tsx` | Add sync UI section |
| `CHANGELOG.md` | Document new feature |

---

## ✅ ACCEPTANCE CRITERIA

### Functional Requirements

- [ ] Manual sync can be triggered from Global Settings page
- [ ] Sync can target single venue or all venues
- [ ] New MySQL feedback appears in AI Feedback page after sync
- [ ] Duplicates are NOT created (idempotent)
- [ ] Existing DynamoDB data is NOT overwritten
- [ ] Sync status and history are visible in UI
- [ ] Errors are logged and reported

### Technical Requirements

- [ ] Uses existing `DirectMySQLService` for MySQL queries
- [ ] Uses existing `RestaurantDynamoDBService` for DynamoDB operations
- [ ] Background task execution (doesn't block API response)
- [ ] Tracks `original_review_id` for duplicate detection
- [ ] Stores `last_sync_at` per restaurant for incremental sync
- [ ] Proper error handling and logging

### Testing

- [ ] Sync creates new comments correctly
- [ ] Sync skips existing comments (no duplicates)
- [ ] Sync finds/creates customers correctly
- [ ] Sync updates restaurant statistics
- [ ] UI shows correct sync status
- [ ] Error cases handled gracefully

---

## 🧪 TESTING SCENARIOS

### Scenario 1: First Sync After Onboarding
```
Given: Venue was onboarded 2 days ago
And: 15 new reviews in MySQL since then
When: Sync is triggered
Then: 15 new comments created in DynamoDB
And: last_sync_at is updated
```

### Scenario 2: Repeated Sync (Idempotency)
```
Given: Sync was run successfully
When: Sync is triggered again immediately
Then: 0 new comments created
And: No duplicates
And: No errors
```

### Scenario 3: Partial New Data
```
Given: 50 reviews in MySQL total
And: 35 already synced
And: 15 new since last sync
When: Sync is triggered
Then: Only 15 new comments created
```

### Scenario 4: Customer Deduplication
```
Given: Customer john@email.com submitted 3 new reviews
When: Sync is triggered
Then: Only 1 customer record exists
And: 3 comments linked to that customer
```

---

## 📚 REFERENCE CODE

### Existing Onboarding Logic (Reference)

```python
# From services/onboard_service.py - REFERENCE ONLY
# The sync service should follow similar patterns

def migrate_client_to_dynamodb(self, client_id: int) -> Dict[str, Any]:
    """
    Existing full migration - creates ALL data.
    Sync service should do incremental updates only.
    """
    # Get all reviews from MySQL
    reviews = self.mysql.get_enriched_reviews_by_client(client_id)
    
    # Create restaurant
    restaurant = self._create_restaurant(client_data)
    
    # Process reviews with customer deduplication
    for review in reviews:
        customer = self._get_or_create_customer(...)
        comment = self._create_comment(...)
```

### MySQL Query for Reviews (Reference)

```python
# From db/direct_mysql_service.py
# Use this as base for get_reviews_since()

def get_enriched_reviews_by_client(self, client_id: int) -> List[EnrichedReview]:
    query = """
        SELECT 
            c.idcomment as review_id,
            c.comment as comment_text,
            c.rating,
            c.date_entered,
            c.date_replied,
            c.reply as reply_text,
            e.emailaddress as customer_email,
            e.firstname,
            e.lastname,
            e.phonenumber as customer_phone,
            d.device_info,
            d.session_info
        FROM comment c
        JOIN datasession d ON c.iddatasession = d.iddatasession
        LEFT JOIN eclub e ON d.ideclub = e.ideclub
        WHERE d.idclient = %s
        ORDER BY c.date_entered DESC
    """
```

---

## 🚀 GETTING STARTED

1. **Read existing code**:
   - `services/onboard_service.py` - Understand migration patterns
   - `db/dynamodb_service.py` - Understand DynamoDB operations
   - `db/direct_mysql_service.py` - Understand MySQL queries

2. **Start with backend**:
   - Create `FeedbackSyncService` class
   - Add `sync_venue()` method
   - Add `get_reviews_since()` to MySQL service
   - Test with curl

3. **Add API routes**:
   - Create `routes/sync.py`
   - Register in `main.py`
   - Test endpoints

4. **Add frontend**:
   - Create `syncApi.ts`
   - Add UI to Global Settings
   - Test full flow

5. **Test thoroughly**:
   - Run sync multiple times
   - Verify no duplicates
   - Verify new data appears in AI Feedback

---

## ⚠️ IMPORTANT NOTES

1. **DO NOT modify existing onboard logic** - sync is separate
2. **Always check for duplicates** before creating comments
3. **Never update existing comments** - they may have AI responses
4. **Use background tasks** for sync operations
5. **Log everything** for debugging
6. **Update CHANGELOG.md** when complete

---

## 📞 QUESTIONS?

If you encounter issues:
1. Check existing code in `onboard_service.py` for patterns
2. Review DynamoDB table structure in `restaurant_data`
3. Test MySQL queries manually before implementing
4. Ask for clarification on business requirements

---

**Created**: 2025-11-28
**Author**: AI Supervisor
**Status**: Ready for implementation

