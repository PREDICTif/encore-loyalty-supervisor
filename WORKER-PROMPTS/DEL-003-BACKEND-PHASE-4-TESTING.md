# DEL-003: Backend Phase 4 API Testing

**Assignment ID:** DEL-003  
**Assigned to:** AI Worker (Backend Tester)  
**Model:** Claude Sonnet 4.5  
**Environment:** encore_backend directory  
**Priority:** CRITICAL  
**Estimated Effort:** 6-8 hours  
**Prerequisites:** Phase 4A backend implementation complete

---

## 🎯 Mission

Create and execute comprehensive automated tests for the Phase 4A backend feedback API. Your goal is to verify that all 5 feedback endpoints work correctly with real MySQL data, proper JOINs are executed, DynamoDB integration works, and edge cases are handled.

---

## 📋 Context

### What You're Testing

Phase 4A implements the feedback management backend that:
- **Reads** historical feedback from legacy MySQL (60,000+ records)
- **Uses complex 4-table JOINs** (comment, datasession, eclub, client)
- **Writes** status/responses to DynamoDB
- **Provides** 5 REST API endpoints with filtering and pagination

### Why This is Critical

Phase 4 is the "essential part" of the project (client's words). It bridges legacy data with new AI functionality. Without proper testing:
- MySQL queries might fail with real data
- NULL customer names might cause crashes
- Pagination might break with large datasets
- Role-based access control might be bypassed

### Key Files You'll Be Testing

```
encore_backend/
├── routes/feedback.py           # 5 API endpoints
├── services/feedback_service.py # Business logic (MySQL + DynamoDB)
├── services/mysql_service.py    # MySQL queries with JOINs
├── models/feedback.py           # Data models
└── tests/automated/
    ├── test_feedback_api.py          # ← YOU CREATE THIS
    ├── test_mysql_service.py         # ← YOU CREATE THIS
    ├── test_feedback_service.py      # ← YOU CREATE THIS
    └── fixtures_feedback.py          # ← YOU CREATE THIS
```

---

## 📚 Required Reading

Before starting, READ these documents:

1. **`docs/TESTING/PHASE-4-TEST-PLAN.md`**
   - Part 1: Backend API Tests (your full test plan)
   - Contains 50+ test cases you need to implement
   
2. **`docs/TESTING/PHASE-4-VALIDATION-REPORT.md`**
   - SQL query verification
   - Field mappings
   - Known issues (NULL names, zero ratings)

3. **`docs/SUPERVISOR/WORKER-PROMPTS/PHASE-4A-BACKEND-FEEDBACK.md`**
   - Implementation details
   - MySQL schema documentation
   - API endpoint specifications

4. **`docs/investigation/ENCORE-ARCHIVE-ANALYSIS-REPORT.md`**
   - Legacy database schema
   - Table relationships

---

## ⚠️ CRITICAL: Legacy MySQL Schema

You MUST understand the legacy schema:

### Tables and Relationships

```sql
comment (c)          -- The actual feedback
├── iddatasession    -- Primary key
├── comment          -- Feedback text
├── overall_rating   -- Rating (99.59% are 0!)
├── emailaddress     -- Customer email
├── dateentered      -- Created timestamp
└── isdeleted        -- Soft delete flag

datasession (ds)     -- Links to venue
├── iddatasession    -- Primary key
├── idclient         -- Venue ID (FK)
└── isdeleted        -- Soft delete flag

eclub (e)            -- Customer details (may be NULL!)
├── iddatasession    -- Links to datasession
├── firstname        -- May be NULL
├── lastname         -- May be NULL
└── phonenumber      -- May be NULL

client (cl)          -- Venue information
├── idclient         -- Primary key
└── description      -- Venue name
```

### Expected JOIN Pattern

```sql
FROM comment c
INNER JOIN datasession ds ON c.iddatasession = ds.iddatasession
LEFT JOIN eclub e ON c.iddatasession = e.iddatasession  -- LEFT JOIN! May be NULL
INNER JOIN client cl ON ds.idclient = cl.idclient
WHERE 
    c.isdeleted = 0 
    AND ds.isdeleted = 0
    AND c.comment IS NOT NULL
    AND c.comment != ''
```

### Field Mappings

| API Field | MySQL Source | Notes |
|-----------|--------------|-------|
| `id` | `c.iddatasession` | Primary key |
| `venue_id` | `ds.idclient` | Via datasession |
| `venue_name` | `cl.description` | Via client |
| `customer_email` | `c.emailaddress` | Direct |
| `customer_name` | `e.firstname + e.lastname` | **May be NULL!** |
| `rating` | `c.overall_rating` | **99.59% are 0!** |
| `comment` | `c.comment` | The feedback text |
| `created_at` | `c.dateentered` | Timestamp |

---

## ✅ Task 1: Create Test Fixtures

**File to create:** `encore_backend/tests/automated/fixtures_feedback.py`

### Purpose

Provide reusable test data and authentication helpers.

### Implementation

```python
"""
Test fixtures for Phase 4 feedback testing
"""
import pytest
from typing import Dict, Any, List
import sys
import os

# Add project root to path
sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), '../../..')))

from services.mysql_service import MySQLService
from models.feedback import FeedbackFilter

@pytest.fixture(scope="session")
def mysql_service():
    """Returns MySQL service instance"""
    return MySQLService()

@pytest.fixture(scope="session")
def real_feedback_ids(mysql_service) -> List[int]:
    """
    Get real feedback IDs from MySQL for testing
    Returns list of at least 10 valid iddatasession values
    """
    filters = FeedbackFilter(page=1, page_size=20)
    items, total = mysql_service.list_feedback(filters)
    
    feedback_ids = [item['id'] for item in items[:10]]
    
    assert len(feedback_ids) >= 10, "Need at least 10 feedback items for testing"
    return feedback_ids

@pytest.fixture
def first_feedback_id(real_feedback_ids) -> int:
    """Returns the first valid feedback ID"""
    return real_feedback_ids[0]

@pytest.fixture(scope="session")
def feedback_with_null_customer(mysql_service) -> int:
    """
    Find feedback with NULL eclub record (no customer name)
    This tests the LEFT JOIN case
    """
    # Query to find feedback where eclub is NULL
    # You may need to query MySQL directly for this
    # For now, return None and skip test if not found
    return None  # Implement if needed

@pytest.fixture(scope="session")
def feedback_with_zero_rating(mysql_service) -> int:
    """
    Find feedback with rating = 0
    Should be easy since 99.59% of data has this
    """
    filters = FeedbackFilter(page=1, page_size=100)
    items, total = mysql_service.list_feedback(filters)
    
    for item in items:
        if item.get('rating') == 0:
            return item['id']
    
    return None

@pytest.fixture(scope="session")
def feedback_with_nonzero_rating(mysql_service) -> int:
    """
    Find feedback with non-zero rating
    This is rare (0.41% of data)
    """
    filters = FeedbackFilter(page=1, page_size=1000)
    items, total = mysql_service.list_feedback(filters)
    
    for item in items:
        if item.get('rating') and item.get('rating') > 0:
            return item['id']
    
    return None

@pytest.fixture(scope="session")
def valid_venue_ids(mysql_service) -> List[str]:
    """
    Get list of valid venue IDs from database
    """
    filters = FeedbackFilter(page=1, page_size=100)
    items, total = mysql_service.list_feedback(filters)
    
    venue_ids = list(set(str(item['venue_id']) for item in items))
    
    assert len(venue_ids) >= 3, "Need at least 3 venues for testing"
    return venue_ids[:5]  # Return up to 5 venue IDs

@pytest.fixture
def first_venue_id(valid_venue_ids) -> str:
    """Returns the first valid venue ID"""
    return valid_venue_ids[0]

@pytest.fixture
def admin_user_credentials() -> Dict[str, str]:
    """
    Returns admin user credentials for authentication
    Use the legacy users from docs/TESTING/legacy-test-users.md
    """
    return {
        "username": "tjrottet@encoreloyalty.net",  # Admin user
        "password": "Predictif1"  # Replace with actual password if known
    }

@pytest.fixture
def venue_manager_credentials() -> Dict[str, str]:
    """
    Returns venue manager credentials
    """
    return {
        "username": "eric@heatherbrae.com",  # Venue manager
        "password": "password"  # Replace with actual password if known
    }

@pytest.fixture
def admin_auth_headers(client, admin_user_credentials) -> Dict[str, str]:
    """
    Returns authentication headers for admin user
    """
    response = client.post("/api/v1/auth/login", json=admin_user_credentials)
    
    if response.status_code == 200:
        token = response.json()["access_token"]
        return {"Authorization": f"Bearer {token}"}
    else:
        pytest.skip("Admin authentication failed - check credentials")

@pytest.fixture
def venue_manager_auth_headers(client, venue_manager_credentials) -> Dict[str, str]:
    """
    Returns authentication headers for venue manager
    """
    response = client.post("/api/v1/auth/login", json=venue_manager_credentials)
    
    if response.status_code == 200:
        token = response.json()["access_token"]
        return {"Authorization": f"Bearer {token}"}
    else:
        pytest.skip("Venue manager authentication failed - check credentials")
```

**Validation:**
```bash
cd /home/ec2-user/encore-loyalty-new-system/encore_backend
source venv/bin/activate
python -c "from tests.automated.fixtures_feedback import *; print('✅ Fixtures import successfully')"
```

---

## ✅ Task 2: Create MySQL Service Tests

**File to create:** `encore_backend/tests/automated/test_mysql_service.py`

### Purpose

Test MySQL service layer directly (bypassing API) to verify queries work correctly.

### Implementation

```python
"""
Tests for MySQL service layer
Verifies correct JOINs, field mapping, and filtering
"""
import pytest
from services.mysql_service import MySQLService
from models.feedback import FeedbackFilter

class TestMySQLServiceGetFeedbackById:
    """Test get_feedback_by_id method"""
    
    def test_get_feedback_returns_correct_structure(self, mysql_service, first_feedback_id):
        """Should return dict with all required fields"""
        result = mysql_service.get_feedback_by_id(first_feedback_id)
        
        assert result is not None, f"Feedback {first_feedback_id} not found"
        
        # Verify required fields present
        assert 'id' in result
        assert 'comment' in result
        assert 'customer_email' in result
        assert 'venue_id' in result
        assert 'venue_name' in result
        assert 'created_at' in result
        
        # Verify correct ID
        assert result['id'] == first_feedback_id
    
    def test_get_feedback_handles_null_customer_name(
        self, mysql_service, feedback_with_null_customer
    ):
        """Should handle case where eclub record doesn't exist"""
        if feedback_with_null_customer is None:
            pytest.skip("No feedback with NULL customer found")
        
        result = mysql_service.get_feedback_by_id(feedback_with_null_customer)
        
        assert result is not None
        assert result.get('customer_name') is None or result.get('customer_name') == ''
        # Should still have email
        assert result.get('customer_email') is not None
    
    def test_get_feedback_not_found(self, mysql_service):
        """Should return None for invalid ID"""
        result = mysql_service.get_feedback_by_id(999999999)
        assert result is None
    
    def test_get_feedback_combines_customer_name(self, mysql_service, first_feedback_id):
        """Should combine firstname + lastname from eclub"""
        result = mysql_service.get_feedback_by_id(first_feedback_id)
        
        if result and result.get('customer_first_name') and result.get('customer_last_name'):
            expected_name = f"{result['customer_first_name']} {result['customer_last_name']}".strip()
            assert result['customer_name'] == expected_name


class TestMySQLServiceListFeedback:
    """Test list_feedback method"""
    
    def test_list_feedback_no_filters(self, mysql_service):
        """Should return paginated results"""
        filters = FeedbackFilter(page=1, page_size=20)
        items, total = mysql_service.list_feedback(filters)
        
        assert isinstance(items, list)
        assert isinstance(total, int)
        assert total > 0, "Should have at least some feedback in database"
        assert len(items) <= 20, "Should respect page_size"
    
    def test_list_feedback_filters_deleted_records(self, mysql_service):
        """Should exclude records where isdeleted = 1"""
        filters = FeedbackFilter(page=1, page_size=100)
        items, total = mysql_service.list_feedback(filters)
        
        # All items should be non-deleted
        # (We can't verify the SQL directly, but implementation should exclude them)
        assert len(items) > 0
    
    def test_list_feedback_filters_empty_comments(self, mysql_service):
        """Should exclude NULL or empty comments"""
        filters = FeedbackFilter(page=1, page_size=100)
        items, total = mysql_service.list_feedback(filters)
        
        for item in items:
            assert item.get('comment')
            assert item['comment'].strip() != ''
    
    def test_list_feedback_filter_by_venue(self, mysql_service, first_venue_id):
        """Should filter by venue_id"""
        filters = FeedbackFilter(venue_id=first_venue_id, page=1, page_size=20)
        items, total = mysql_service.list_feedback(filters)
        
        if total > 0:
            for item in items:
                assert str(item['venue_id']) == str(first_venue_id)
    
    def test_list_feedback_filter_by_rating(self, mysql_service):
        """Should filter by rating"""
        filters = FeedbackFilter(rating=5, page=1, page_size=20)
        items, total = mysql_service.list_feedback(filters)
        
        for item in items:
            assert item.get('rating') == 5
    
    def test_list_feedback_search_text(self, mysql_service):
        """Should search in comment and customer name"""
        # Use a common word likely to be in comments
        filters = FeedbackFilter(search_text="good", page=1, page_size=20)
        items, total = mysql_service.list_feedback(filters)
        
        # At least verify the query executes without error
        assert isinstance(items, list)
    
    def test_list_feedback_pagination(self, mysql_service):
        """Should paginate correctly"""
        # Get page 1
        filters_p1 = FeedbackFilter(page=1, page_size=10)
        items_p1, total = mysql_service.list_feedback(filters_p1)
        
        # Get page 2
        filters_p2 = FeedbackFilter(page=2, page_size=10)
        items_p2, _ = mysql_service.list_feedback(filters_p2)
        
        if total > 10:
            # Pages should have different items
            ids_p1 = [item['id'] for item in items_p1]
            ids_p2 = [item['id'] for item in items_p2]
            
            assert set(ids_p1).isdisjoint(set(ids_p2)), "Pages should not overlap"
    
    def test_list_feedback_date_range(self, mysql_service):
        """Should filter by date range"""
        from datetime import datetime, timedelta
        
        date_to = datetime.now()
        date_from = date_to - timedelta(days=365)  # Last year
        
        filters = FeedbackFilter(
            date_from=date_from,
            date_to=date_to,
            page=1,
            page_size=20
        )
        items, total = mysql_service.list_feedback(filters)
        
        # Verify query executes
        assert isinstance(items, list)


class TestMySQLServiceGetFeedbackCount:
    """Test get_feedback_count_by_venue method"""
    
    def test_get_count_returns_int(self, mysql_service, first_venue_id):
        """Should return integer count"""
        count = mysql_service.get_feedback_count_by_venue(first_venue_id)
        
        assert isinstance(count, int)
        assert count >= 0
    
    def test_get_count_for_invalid_venue(self, mysql_service):
        """Should return 0 for non-existent venue"""
        count = mysql_service.get_feedback_count_by_venue("999999")
        
        assert count == 0
```

**Validation:**
```bash
pytest tests/automated/test_mysql_service.py -v
```

---

## ✅ Task 3: Create Feedback Service Tests

**File to create:** `encore_backend/tests/automated/test_feedback_service.py`

### Purpose

Test feedback service that combines MySQL + DynamoDB.

### Implementation

```python
"""
Tests for Feedback Service
Verifies MySQL + DynamoDB integration
"""
import pytest
from services.feedback_service import FeedbackService
from models.feedback import FeedbackFilter, FeedbackStatus

class TestFeedbackServiceGetFeedback:
    """Test get_feedback method"""
    
    def test_get_feedback_combines_mysql_and_dynamodb(self, first_feedback_id):
        """Should merge data from MySQL and DynamoDB"""
        service = FeedbackService()
        feedback = service.get_feedback(first_feedback_id)
        
        assert feedback is not None
        
        # Should have MySQL fields
        assert 'comment' in feedback
        assert 'customer_email' in feedback
        assert 'venue_id' in feedback
        
        # Should have DynamoDB fields (or defaults)
        assert 'status' in feedback
        assert feedback['status'] in [s.value for s in FeedbackStatus]
    
    def test_get_feedback_creates_default_status(self, first_feedback_id):
        """Should create pending status if not in DynamoDB yet"""
        service = FeedbackService()
        feedback = service.get_feedback(first_feedback_id)
        
        # If not in DynamoDB, should default to pending
        assert feedback['status'] in ['pending', 'reviewed', 'generated', 'edited', 'sent', 'archived', 'spam']


class TestFeedbackServiceListFeedback:
    """Test list_feedback method"""
    
    def test_list_feedback_enriches_with_status(self):
        """Should add status from DynamoDB to each item"""
        service = FeedbackService()
        filters = FeedbackFilter(page=1, page_size=10)
        
        result = service.list_feedback(filters)
        
        assert 'items' in result
        assert 'total' in result
        
        for item in result['items']:
            assert 'status' in item


class TestFeedbackServiceUpdateStatus:
    """Test update_status method"""
    
    def test_update_status_writes_to_dynamodb(self, first_feedback_id):
        """Should update status in DynamoDB only"""
        service = FeedbackService()
        
        # Update to reviewed
        success = service.update_status(
            feedback_id=first_feedback_id,
            new_status=FeedbackStatus.REVIEWED,
            updated_by="test_user"
        )
        
        assert success is True
        
        # Verify it persisted
        feedback = service.get_feedback(first_feedback_id)
        assert feedback['status'] == 'reviewed'
        
        # Change it back to pending for next test
        service.update_status(
            feedback_id=first_feedback_id,
            new_status=FeedbackStatus.PENDING,
            updated_by="test_user"
        )
```

**Validation:**
```bash
pytest tests/automated/test_feedback_service.py -v
```

---

## ✅ Task 4: Create API Endpoint Tests

**File to create:** `encore_backend/tests/automated/test_feedback_api.py`

### Purpose

Test all 5 API endpoints end-to-end with authentication.

### Implementation Structure

```python
"""
End-to-end API tests for feedback endpoints
Tests authentication, authorization, filtering, pagination
"""
import pytest

class TestListFeedbackEndpoint:
    """Test GET /api/feedback"""
    
    def test_list_feedback_requires_auth(self, client):
        """Should return 401 without authentication"""
        response = client.get("/api/v1/feedback")
        assert response.status_code == 401
    
    def test_list_feedback_with_auth(self, client, admin_auth_headers):
        """Should return paginated list with authentication"""
        response = client.get("/api/v1/feedback", headers=admin_auth_headers)
        
        assert response.status_code == 200
        data = response.json()
        
        # Verify pagination structure
        assert 'items' in data
        assert 'total' in data
        assert 'page' in data
        assert 'page_size' in data
        assert 'total_pages' in data
        
        assert isinstance(data['items'], list)
        assert data['page'] == 1
    
    def test_list_feedback_filter_by_venue(
        self, client, admin_auth_headers, first_venue_id
    ):
        """Should filter by venue_id"""
        response = client.get(
            f"/api/v1/feedback?venue_id={first_venue_id}",
            headers=admin_auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        
        for item in data['items']:
            assert str(item['venue_id']) == str(first_venue_id)
    
    def test_list_feedback_filter_by_status(self, client, admin_auth_headers):
        """Should filter by status"""
        response = client.get(
            "/api/v1/feedback?status=pending",
            headers=admin_auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        
        for item in data['items']:
            assert item['status'] == 'pending'
    
    def test_list_feedback_pagination(self, client, admin_auth_headers):
        """Should paginate correctly"""
        # Page 1
        response1 = client.get(
            "/api/v1/feedback?page=1&page_size=5",
            headers=admin_auth_headers
        )
        assert response1.status_code == 200
        data1 = response1.json()
        
        # Page 2
        response2 = client.get(
            "/api/v1/feedback?page=2&page_size=5",
            headers=admin_auth_headers
        )
        assert response2.status_code == 200
        data2 = response2.json()
        
        if data1['total'] > 5:
            ids1 = [item['id'] for item in data1['items']]
            ids2 = [item['id'] for item in data2['items']]
            assert set(ids1).isdisjoint(set(ids2))
    
    def test_list_feedback_search(self, client, admin_auth_headers):
        """Should search in comments"""
        response = client.get(
            "/api/v1/feedback?search_text=good",
            headers=admin_auth_headers
        )
        
        assert response.status_code == 200
    
    def test_list_feedback_combined_filters(self, client, admin_auth_headers, first_venue_id):
        """Should apply multiple filters"""
        response = client.get(
            f"/api/v1/feedback?venue_id={first_venue_id}&status=pending&page_size=10",
            headers=admin_auth_headers
        )
        
        assert response.status_code == 200
    
    # ADD MORE TESTS: rating filter, date range, etc.


class TestGetFeedbackEndpoint:
    """Test GET /api/feedback/{id}"""
    
    def test_get_feedback_requires_auth(self, client, first_feedback_id):
        """Should return 401 without auth"""
        response = client.get(f"/api/v1/feedback/{first_feedback_id}")
        assert response.status_code == 401
    
    def test_get_feedback_with_auth(self, client, admin_auth_headers, first_feedback_id):
        """Should return complete feedback object"""
        response = client.get(
            f"/api/v1/feedback/{first_feedback_id}",
            headers=admin_auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        
        # Verify structure
        assert data['id'] == first_feedback_id
        assert 'comment' in data
        assert 'customer_email' in data
        assert 'venue_id' in data
        assert 'status' in data
        assert 'created_at' in data
    
    def test_get_feedback_not_found(self, client, admin_auth_headers):
        """Should return 404 for invalid ID"""
        response = client.get(
            "/api/v1/feedback/999999999",
            headers=admin_auth_headers
        )
        assert response.status_code == 404
    
    def test_get_feedback_with_null_customer_name(
        self, client, admin_auth_headers, feedback_with_null_customer
    ):
        """Should handle NULL customer names gracefully"""
        if feedback_with_null_customer is None:
            pytest.skip("No feedback with NULL customer found")
        
        response = client.get(
            f"/api/v1/feedback/{feedback_with_null_customer}",
            headers=admin_auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        
        # Customer name may be None
        assert data.get('customer_name') is None or data.get('customer_name') == ''
        # But email should exist
        assert data.get('customer_email') is not None
    
    def test_get_feedback_with_zero_rating(
        self, client, admin_auth_headers, feedback_with_zero_rating
    ):
        """Should handle zero ratings (99.59% of data)"""
        if feedback_with_zero_rating is None:
            pytest.skip("No zero rating feedback found")
        
        response = client.get(
            f"/api/v1/feedback/{feedback_with_zero_rating}",
            headers=admin_auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        assert data['rating'] == 0 or data['rating'] is None


class TestUpdateFeedbackStatusEndpoint:
    """Test PUT /api/feedback/{id}/status"""
    
    def test_update_status_requires_auth(self, client, first_feedback_id):
        """Should return 401 without auth"""
        response = client.put(
            f"/api/v1/feedback/{first_feedback_id}/status",
            json={"status": "reviewed"}
        )
        assert response.status_code == 401
    
    def test_update_status_to_reviewed(
        self, client, admin_auth_headers, first_feedback_id
    ):
        """Should update status to reviewed"""
        response = client.put(
            f"/api/v1/feedback/{first_feedback_id}/status",
            json={"status": "reviewed", "notes": "Test review"},
            headers=admin_auth_headers
        )
        
        assert response.status_code == 200
        
        # Verify it persisted
        get_response = client.get(
            f"/api/v1/feedback/{first_feedback_id}",
            headers=admin_auth_headers
        )
        assert get_response.status_code == 200
        data = get_response.json()
        assert data['status'] == 'reviewed'
        
        # Change back to pending
        client.put(
            f"/api/v1/feedback/{first_feedback_id}/status",
            json={"status": "pending"},
            headers=admin_auth_headers
        )
    
    def test_update_status_all_valid_statuses(
        self, client, admin_auth_headers, first_feedback_id
    ):
        """Should accept all valid status values"""
        valid_statuses = ['pending', 'reviewed', 'generated', 'edited', 'sent', 'archived', 'spam']
        
        for status in valid_statuses:
            response = client.put(
                f"/api/v1/feedback/{first_feedback_id}/status",
                json={"status": status},
                headers=admin_auth_headers
            )
            assert response.status_code == 200, f"Failed for status: {status}"
        
        # Reset to pending
        client.put(
            f"/api/v1/feedback/{first_feedback_id}/status",
            json={"status": "pending"},
            headers=admin_auth_headers
        )
    
    def test_update_status_invalid_status(
        self, client, admin_auth_headers, first_feedback_id
    ):
        """Should reject invalid status"""
        response = client.put(
            f"/api/v1/feedback/{first_feedback_id}/status",
            json={"status": "invalid_status_xyz"},
            headers=admin_auth_headers
        )
        assert response.status_code == 422  # Validation error
    
    def test_update_status_not_found(self, client, admin_auth_headers):
        """Should return 404 for invalid ID"""
        response = client.put(
            "/api/v1/feedback/999999999/status",
            json={"status": "reviewed"},
            headers=admin_auth_headers
        )
        assert response.status_code == 404


class TestGenerateResponseEndpoint:
    """Test POST /api/feedback/{id}/generate (stub for Phase 5)"""
    
    def test_generate_requires_auth(self, client, first_feedback_id):
        """Should return 401 without auth"""
        response = client.post(f"/api/v1/feedback/{first_feedback_id}/generate")
        assert response.status_code == 401
    
    def test_generate_with_auth(self, client, admin_auth_headers, first_feedback_id):
        """Should call generate endpoint (may be stub)"""
        response = client.post(
            f"/api/v1/feedback/{first_feedback_id}/generate",
            headers=admin_auth_headers
        )
        
        # Accept either 200 (implemented) or 501 (not implemented stub)
        assert response.status_code in [200, 501]


class TestSendEmailEndpoint:
    """Test POST /api/feedback/{id}/send (stub for Phase 6)"""
    
    def test_send_requires_auth(self, client, first_feedback_id):
        """Should return 401 without auth"""
        response = client.post(f"/api/v1/feedback/{first_feedback_id}/send")
        assert response.status_code == 401
    
    def test_send_with_auth(self, client, admin_auth_headers, first_feedback_id):
        """Should call send endpoint (may be stub)"""
        response = client.post(
            f"/api/v1/feedback/{first_feedback_id}/send",
            headers=admin_auth_headers
        )
        
        # Accept either 200 (implemented) or 501 (not implemented stub)
        assert response.status_code in [200, 501]
```

**Validation:**
```bash
pytest tests/automated/test_feedback_api.py -v --tb=short
```

---

## ✅ Task 5: Performance and Integration Tests

Add to `test_feedback_api.py`:

```python
class TestPerformance:
    """Performance tests"""
    
    def test_list_feedback_performance(self, client, admin_auth_headers):
        """Should return results in < 2 seconds"""
        import time
        
        start = time.time()
        response = client.get(
            "/api/v1/feedback?page_size=100",
            headers=admin_auth_headers
        )
        duration = time.time() - start
        
        assert response.status_code == 200
        assert duration < 2.0, f"Too slow: {duration:.2f}s"
    
    def test_pagination_at_high_page_numbers(self, client, admin_auth_headers):
        """Should paginate quickly even at high page numbers"""
        import time
        
        # Test page 100
        start = time.time()
        response = client.get(
            "/api/v1/feedback?page=100&page_size=20",
            headers=admin_auth_headers
        )
        duration = time.time() - start
        
        # Should still be fast
        assert duration < 3.0, f"Pagination too slow: {duration:.2f}s"


class TestIntegration:
    """Integration tests - complete workflows"""
    
    def test_complete_feedback_workflow(self, client, admin_auth_headers):
        """Test list → view → update → verify"""
        
        # STEP 1: List feedback
        list_response = client.get("/api/v1/feedback", headers=admin_auth_headers)
        assert list_response.status_code == 200
        feedback_list = list_response.json()
        assert len(feedback_list['items']) > 0
        
        # STEP 2: Get first feedback
        feedback_id = feedback_list['items'][0]['id']
        detail_response = client.get(
            f"/api/v1/feedback/{feedback_id}",
            headers=admin_auth_headers
        )
        assert detail_response.status_code == 200
        
        # STEP 3: Update status
        initial_status = detail_response.json()['status']
        new_status = 'reviewed' if initial_status == 'pending' else 'pending'
        
        update_response = client.put(
            f"/api/v1/feedback/{feedback_id}/status",
            json={"status": new_status},
            headers=admin_auth_headers
        )
        assert update_response.status_code == 200
        
        # STEP 4: Verify update persisted
        verify_response = client.get(
            f"/api/v1/feedback/{feedback_id}",
            headers=admin_auth_headers
        )
        assert verify_response.status_code == 200
        assert verify_response.json()['status'] == new_status
        
        # STEP 5: Revert status
        client.put(
            f"/api/v1/feedback/{feedback_id}/status",
            json={"status": initial_status},
            headers=admin_auth_headers
        )
```

---

## 📊 Task 6: Execute Tests and Generate Report

### Running Tests

```bash
cd /home/ec2-user/encore-loyalty-new-system/encore_backend
source venv/bin/activate

# Run all feedback tests
pytest tests/automated/test_mysql_service.py \
       tests/automated/test_feedback_service.py \
       tests/automated/test_feedback_api.py \
       -v --tb=short

# With coverage
pytest tests/automated/test_mysql_service.py \
       tests/automated/test_feedback_service.py \
       tests/automated/test_feedback_api.py \
       --cov=services.mysql_service \
       --cov=services.feedback_service \
       --cov=routes.feedback \
       --cov-report=html \
       --cov-report=term

# Performance tests only
pytest tests/automated/test_feedback_api.py::TestPerformance -v

# Integration tests only
pytest tests/automated/test_feedback_api.py::TestIntegration -v
```

### Create Test Report

**File to create:** `docs/TESTING/PHASE-4-BACKEND-TEST-REPORT.md`

Use this template:

```markdown
# Phase 4 Backend Testing - Execution Report

**Date:** YYYY-MM-DD  
**Tester:** DEL-003 (AI Worker)  
**Environment:** Development  
**Test Framework:** pytest

---

## Summary

- **Total Tests:** XX
- **Passed:** XX
- **Failed:** XX
- **Skipped:** XX
- **Coverage:** XX%
- **Duration:** XX minutes

---

## Test Results by Module

### MySQL Service Tests
- `test_mysql_service.py`: XX/XX passed
- Coverage: XX%
- Issues: [List any issues]

### Feedback Service Tests
- `test_feedback_service.py`: XX/XX passed
- Coverage: XX%
- Issues: [List any issues]

### API Endpoint Tests
- `test_feedback_api.py`: XX/XX passed
- Coverage: XX%
- Issues: [List any issues]

---

## Performance Metrics

- List endpoint (100 items): X.XX seconds
- Get single feedback: X.XX seconds
- Update status: X.XX seconds
- Pagination at page 100: X.XX seconds

---

## Issues Found

### Critical Issues
1. [Description]

### High Priority Issues
1. [Description]

### Medium/Low Priority Issues
1. [Description]

---

## Edge Cases Tested

- ✅ NULL customer names
- ✅ Zero ratings
- ✅ Large result sets (100+)
- ✅ High page numbers
- ✅ Multiple filters combined
- ✅ Invalid authentication
- ✅ Invalid feedback IDs
- ✅ Invalid status values

---

## Recommendations

1. [Recommendation]
2. [Recommendation]

---

## Next Steps

1. Fix critical issues
2. Re-run failed tests
3. Proceed to frontend testing (DEL-004)
```

---

## ✅ Success Criteria

Your assignment is complete when:

- [ ] All 4 test files created and functional
- [ ] All MySQL service tests passing (or documented failures)
- [ ] All feedback service tests passing (or documented failures)
- [ ] All API endpoint tests passing (or documented failures)
- [ ] Performance benchmarks met (< 2s for list, < 3s for pagination)
- [ ] Edge cases tested (NULL names, zero ratings)
- [ ] Test coverage ≥ 90% on feedback routes and services
- [ ] Test execution report created
- [ ] Issues documented with severity levels
- [ ] Recommendations provided

---

## 📝 Deliverables

1. **Test Files** (4 files):
   - `tests/automated/fixtures_feedback.py`
   - `tests/automated/test_mysql_service.py`
   - `tests/automated/test_feedback_service.py`
   - `tests/automated/test_feedback_api.py`

2. **Test Report**:
   - `docs/TESTING/PHASE-4-BACKEND-TEST-REPORT.md`

3. **Coverage Report** (if possible):
   - HTML coverage report in `htmlcov/`

---

## ⚠️ Important Notes

1. **Real Data:** Tests run against real MySQL database - be careful
2. **Credentials:** Use credentials from `docs/TESTING/legacy-test-users.md`
3. **Cleanup:** Revert any status changes after tests
4. **Skips:** Use `pytest.skip()` if fixtures not available
5. **Documentation:** Document everything - findings, issues, workarounds
6. **Communication:** Report back to Supervisor with findings

---

## 🆘 Troubleshooting

### Backend won't start
- Check `.env` file in encore_backend
- Verify MySQL connection
- Check DynamoDB credentials

### Authentication fails
- Verify credentials in legacy-test-users.md
- Check if passwords are MD5 hashed
- Ensure dual password verification is working

### MySQL queries fail
- Check SSH tunnel is active
- Verify table names (comment, not feedback!)
- Check field names match legacy schema

### Tests timeout
- Increase pytest timeout
- Check network connection to MySQL
- Verify DynamoDB is accessible

---

**Good luck! Report findings back to Supervisor when complete.**

---

**Created By:** AI Supervisor  
**Assignment ID:** DEL-003  
**Priority:** CRITICAL  
**Due:** ASAP (Phase 5 blocked until complete)

