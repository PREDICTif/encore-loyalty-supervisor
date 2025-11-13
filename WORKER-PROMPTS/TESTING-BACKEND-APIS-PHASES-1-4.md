# Worker Prompt: Backend API Automated Testing (Phases 1-4)

**Created**: 2025-11-12  
**Assigned To**: AI Worker Thread  
**Priority**: P0 - Critical (Blocking further development)  
**Estimated Duration**: 4-6 hours  
**Delegation ID**: DEL-002

---

## 🎯 MISSION

Create comprehensive automated tests for all 18 backend API endpoints built in Phases 1-4. Test **carefully and thoroughly** to catch any bugs before advancing to Phases 5-8.

**Success Criteria**:
- All 18 endpoints tested
- Pass/fail results documented
- Bugs clearly identified with reproduction steps
- Test scripts saved for future regression testing
- Clear report delivered to Supervisor

---

## 📋 WHAT TO TEST

### Phase 2: Authentication Endpoints (4 endpoints)

#### 1. `POST /api/v1/auth/login`
**Test Cases**:
- ✅ Valid credentials (admin user)
- ✅ Valid credentials (venue manager)
- ❌ Invalid email
- ❌ Invalid password
- ❌ Missing email
- ❌ Missing password
- ❌ Malformed request body
- ⏱️ Response time < 2 seconds

**Expected Response**:
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer",
  "user": {
    "id": 1,
    "email": "admin@encore.com",
    "role": "super_admin",
    "venue_ids": []
  }
}
```

**What to Verify**:
- Status code: 200 for success, 401 for invalid credentials
- JWT tokens are valid (can decode)
- User object has correct structure
- Role matches expected (super_admin, global_admin, venue_manager)
- Token expiration times are correct (access: 15 min, refresh: 7 days)

---

#### 2. `POST /api/v1/auth/refresh`
**Test Cases**:
- ✅ Valid refresh token
- ❌ Invalid refresh token
- ❌ Expired refresh token
- ❌ Missing refresh token
- ❌ Access token instead of refresh token

**Request**:
```json
{
  "refresh_token": "eyJ..."
}
```

**Expected Response**:
```json
{
  "access_token": "eyJ...",
  "token_type": "bearer"
}
```

**What to Verify**:
- Status code: 200 for success, 401 for invalid/expired
- New access token is different from old one
- New token is valid and works for authenticated endpoints
- Token has correct expiration (15 minutes)

---

#### 3. `GET /api/v1/auth/me`
**Test Cases**:
- ✅ Valid access token (admin)
- ✅ Valid access token (venue manager)
- ❌ Missing authorization header
- ❌ Invalid token
- ❌ Expired token
- ❌ Malformed authorization header

**Request Headers**:
```
Authorization: Bearer eyJ...
```

**Expected Response**:
```json
{
  "id": 1,
  "email": "admin@encore.com",
  "role": "super_admin",
  "venue_ids": []
}
```

**What to Verify**:
- Status code: 200 for success, 401 for invalid/missing token
- User data matches logged-in user
- Role is correct
- Venue IDs are correct (empty for admin, populated for venue manager)

---

#### 4. `POST /api/v1/auth/logout`
**Test Cases**:
- ✅ Valid access token
- ❌ Missing authorization header
- ❌ Invalid token

**Request Headers**:
```
Authorization: Bearer eyJ...
```

**Expected Response**:
```json
{
  "message": "Successfully logged out"
}
```

**What to Verify**:
- Status code: 200 for success
- Token is invalidated (subsequent requests with same token fail)

---

### Phase 3: Settings Management (6 endpoints)

#### 5. `GET /api/v1/settings/venue/{venue_id}`
**Test Cases**:
- ✅ Valid venue ID (first time - should return defaults)
- ✅ Valid venue ID (after settings saved)
- ✅ Admin user accessing any venue
- ✅ Venue manager accessing their own venue
- ❌ Venue manager accessing another venue
- ❌ Invalid venue ID
- ❌ Missing authorization

**Expected Response**:
```json
{
  "venue_id": "1",
  "ai_enabled": true,
  "ai_settings": {
    "model": "claude-3-5-sonnet",
    "temperature": 0.7,
    "max_tokens": 500
  },
  "email_settings": {
    "enabled": true,
    "from_name": "Venue Name",
    "from_email": "noreply@venue.com"
  },
  "updated_at": "2025-11-12T10:30:00Z",
  "updated_by": "admin@encore.com"
}
```

**What to Verify**:
- Status code: 200 for success, 403 for unauthorized venue access, 404 for invalid venue
- Settings structure matches schema
- Default values present if first time
- Saved values present if previously updated
- Role-based access control works

---

#### 6. `PUT /api/v1/settings/venue/{venue_id}`
**Test Cases**:
- ✅ Valid update (admin user)
- ✅ Valid update (venue manager for their venue)
- ✅ Partial update (only AI settings)
- ✅ Partial update (only email settings)
- ❌ Venue manager updating another venue
- ❌ Invalid settings structure
- ❌ Invalid values (e.g., temperature > 1.0)
- ❌ Missing required fields

**Request Body**:
```json
{
  "ai_enabled": true,
  "ai_settings": {
    "temperature": 0.8,
    "max_tokens": 600
  },
  "email_settings": {
    "enabled": true,
    "from_name": "New Venue Name"
  }
}
```

**Expected Response**:
```json
{
  "venue_id": "1",
  "ai_enabled": true,
  "ai_settings": {
    "model": "claude-3-5-sonnet",
    "temperature": 0.8,
    "max_tokens": 600
  },
  "email_settings": {
    "enabled": true,
    "from_name": "New Venue Name",
    "from_email": "noreply@venue.com"
  },
  "updated_at": "2025-11-12T10:35:00Z",
  "updated_by": "admin@encore.com"
}
```

**What to Verify**:
- Status code: 200 for success, 400 for invalid data, 403 for unauthorized
- Settings actually saved to DynamoDB (read again to verify)
- Partial updates work (unchanged fields remain)
- Validation errors are clear
- `updated_at` timestamp is updated
- `updated_by` is set to current user

**Critical Test**: After updating, immediately call GET to verify persistence!

---

#### 7. `GET /api/v1/settings/venue/{venue_id}/history`
**Test Cases**:
- ✅ Venue with settings history
- ✅ Venue with no history (first time)
- ✅ Pagination works
- ❌ Unauthorized access

**Expected Response**:
```json
{
  "venue_id": "1",
  "history": [
    {
      "timestamp": "2025-11-12T10:35:00Z",
      "changed_by": "admin@encore.com",
      "changes": {
        "ai_settings.temperature": {"old": 0.7, "new": 0.8},
        "ai_settings.max_tokens": {"old": 500, "new": 600}
      }
    },
    {
      "timestamp": "2025-11-12T09:00:00Z",
      "changed_by": "admin@encore.com",
      "changes": {
        "ai_enabled": {"old": false, "new": true}
      }
    }
  ],
  "total": 2,
  "page": 1,
  "page_size": 20
}
```

**What to Verify**:
- Status code: 200
- History ordered by timestamp (newest first)
- Change tracking is accurate
- Empty array if no history

---

#### 8. `GET /api/v1/settings/global`
**Test Cases**:
- ✅ Admin user
- ❌ Venue manager (should be forbidden)
- ❌ Unauthorized

**Expected Response**:
```json
{
  "default_ai_settings": {
    "model": "claude-3-5-sonnet",
    "temperature": 0.7,
    "max_tokens": 500
  },
  "default_email_settings": {
    "enabled": true,
    "from_name": "Encore Loyalty",
    "from_email": "noreply@encore.com"
  }
}
```

**What to Verify**:
- Status code: 200 for admin, 403 for venue manager
- Global defaults present
- Role-based access enforced

---

#### 9. `PUT /api/v1/settings/global`
**Test Cases**:
- ✅ Admin user updating defaults
- ❌ Venue manager (forbidden)
- ❌ Invalid settings

**Request Body**:
```json
{
  "default_ai_settings": {
    "temperature": 0.75
  }
}
```

**Expected Response**:
```json
{
  "default_ai_settings": {
    "model": "claude-3-5-sonnet",
    "temperature": 0.75,
    "max_tokens": 500
  },
  "updated_at": "2025-11-12T10:40:00Z"
}
```

**What to Verify**:
- Status code: 200 for admin, 403 for venue manager
- Global settings updated
- Partial updates work

---

#### 10. `POST /api/v1/settings/test-notification`
**Test Cases**:
- ✅ Valid email address
- ❌ Invalid email address
- ❌ Missing email

**Request Body**:
```json
{
  "email": "test@example.com",
  "venue_id": "1"
}
```

**Expected Response**:
```json
{
  "success": true,
  "message": "Test email sent successfully"
}
```

**What to Verify**:
- Status code: 200 for success, 400 for invalid email
- Response indicates success
- (Optional: Check if email actually sent if SES is configured)

---

### Phase 4: Feedback Management (8 endpoints)

#### 11. `GET /api/v1/feedback`
**Test Cases**:
- ✅ No filters (get all)
- ✅ Filter by venue_id
- ✅ Filter by status
- ✅ Filter by sentiment
- ✅ Filter by date range
- ✅ Search by customer email or comment text
- ✅ Pagination (page 1, page 2)
- ✅ Limit/offset parameters
- ❌ Invalid filters
- ⏱️ Response time < 3 seconds

**Query Parameters**:
```
?venue_id=1&status=pending&sentiment=positive&page=1&limit=20
```

**Expected Response**:
```json
{
  "items": [
    {
      "id": 12345,
      "venue_id": 1,
      "venue_name": "Test Restaurant",
      "customer_email": "customer@example.com",
      "customer_name": "John Doe",
      "comment": "Great service!",
      "rating": 5,
      "sentiment": "positive",
      "status": "pending",
      "created_at": "2025-11-10T18:30:00Z",
      "response_generated_at": null,
      "response_sent_at": null
    }
  ],
  "total": 150,
  "page": 1,
  "page_size": 20,
  "total_pages": 8
}
```

**What to Verify**:
- Status code: 200
- Filters work correctly (verify SQL queries match filters)
- Pagination works (page 1 vs page 2 different results)
- Total count is accurate
- Data from MySQL is correct
- Venue managers only see their venues
- Admins see all venues
- Response time acceptable for large datasets

---

#### 12. `GET /api/v1/feedback/{feedback_id}`
**Test Cases**:
- ✅ Valid feedback ID (admin)
- ✅ Valid feedback ID (venue manager for their venue)
- ❌ Venue manager accessing another venue's feedback
- ❌ Invalid feedback ID
- ❌ Feedback ID that doesn't exist

**Expected Response**:
```json
{
  "id": 12345,
  "venue_id": 1,
  "venue_name": "Test Restaurant",
  "customer_email": "customer@example.com",
  "customer_name": "John Doe",
  "customer_phone": "555-1234",
  "comment": "Great service! The food was amazing.",
  "rating": 5,
  "sentiment": "positive",
  "status": "pending",
  "created_at": "2025-11-10T18:30:00Z",
  "survey_responses": [
    {
      "question": "How was the food?",
      "answer": "Excellent"
    },
    {
      "question": "Would you recommend us?",
      "answer": "Yes"
    }
  ],
  "ai_response": null,
  "response_generated_at": null,
  "response_sent_at": null,
  "history": []
}
```

**What to Verify**:
- Status code: 200 for success, 403 for unauthorized venue, 404 for not found
- Full feedback details present
- Survey responses included
- Customer data complete
- Role-based access enforced

---

#### 13. `PUT /api/v1/feedback/{feedback_id}/status`
**Test Cases**:
- ✅ Update to valid status (pending → reviewed)
- ✅ Update to valid status (reviewed → approved)
- ✅ Admin updating any feedback
- ✅ Venue manager updating their venue's feedback
- ❌ Venue manager updating another venue's feedback
- ❌ Invalid status value
- ❌ Missing status field

**Request Body**:
```json
{
  "status": "reviewed",
  "notes": "Reviewed and ready for AI response"
}
```

**Expected Response**:
```json
{
  "id": 12345,
  "status": "reviewed",
  "updated_at": "2025-11-12T10:45:00Z",
  "updated_by": "admin@encore.com"
}
```

**What to Verify**:
- Status code: 200 for success, 400 for invalid status, 403 for unauthorized
- Status actually updated in database (check DynamoDB)
- History entry created
- Timestamp updated

---

#### 14. `POST /api/v1/feedback/{feedback_id}/generate`
**Test Cases**:
- ✅ Generate for feedback without response
- ✅ Generate with custom instructions
- ❌ Feedback that already has response (should allow or warn)
- ❌ Invalid feedback ID
- ❌ AI disabled for venue

**Request Body** (optional):
```json
{
  "custom_instructions": "Make it more friendly and casual"
}
```

**Expected Response** (Phase 4 - STUB):
```json
{
  "feedback_id": 12345,
  "response": "[STUB] This is a placeholder AI response. Will be implemented in Phase 5.",
  "generated_at": "2025-11-12T10:50:00Z",
  "model_used": "claude-3-5-sonnet",
  "status": "generated"
}
```

**What to Verify**:
- Status code: 200 for success
- Response placeholder present
- Status updated to "generated"
- **NOTE**: Full AI generation in Phase 5 - this is just a stub!

---

#### 15. `POST /api/v1/feedback/{feedback_id}/regenerate`
**Test Cases**:
- ✅ Regenerate existing response
- ✅ With custom instructions
- ❌ Feedback with no previous response
- ❌ Invalid feedback ID

**Request Body**:
```json
{
  "custom_instructions": "Make it shorter and more professional"
}
```

**Expected Response** (Phase 4 - STUB):
```json
{
  "feedback_id": 12345,
  "response": "[STUB] This is a regenerated placeholder. Will be implemented in Phase 5.",
  "generated_at": "2025-11-12T10:55:00Z",
  "model_used": "claude-3-5-sonnet",
  "generation_count": 2
}
```

**What to Verify**:
- Status code: 200
- New response different from previous
- Generation count incremented
- **NOTE**: Full regeneration in Phase 5 - this is just a stub!

---

#### 16. `PUT /api/v1/feedback/{feedback_id}/response`
**Test Cases**:
- ✅ Update response text
- ✅ Clear response (empty string)
- ❌ Invalid feedback ID
- ❌ Extremely long response (test limits)

**Request Body**:
```json
{
  "response": "Thank you for your feedback! We're glad you enjoyed your experience."
}
```

**Expected Response**:
```json
{
  "feedback_id": 12345,
  "response": "Thank you for your feedback! We're glad you enjoyed your experience.",
  "updated_at": "2025-11-12T11:00:00Z",
  "updated_by": "admin@encore.com",
  "manually_edited": true
}
```

**What to Verify**:
- Status code: 200
- Response text actually saved
- `manually_edited` flag set to true
- History entry created

---

#### 17. `POST /api/v1/feedback/{feedback_id}/send`
**Test Cases**:
- ✅ Send response with valid feedback
- ❌ Send without response text
- ❌ Send with invalid email
- ❌ Email disabled for venue

**Request Body** (optional):
```json
{
  "cc_emails": ["manager@venue.com"]
}
```

**Expected Response** (Phase 4 - STUB):
```json
{
  "feedback_id": 12345,
  "email_status": "sent",
  "sent_at": "2025-11-12T11:05:00Z",
  "sent_to": "customer@example.com",
  "message": "[STUB] Email will be sent in Phase 6"
}
```

**What to Verify**:
- Status code: 200
- Status updated to "sent"
- Timestamp recorded
- **NOTE**: Actual email sending in Phase 6 - this is just a stub!

---

#### 18. `GET /api/v1/feedback/{feedback_id}/history`
**Test Cases**:
- ✅ Feedback with history
- ✅ Feedback with no history
- ❌ Invalid feedback ID

**Expected Response**:
```json
{
  "feedback_id": 12345,
  "history": [
    {
      "timestamp": "2025-11-12T11:00:00Z",
      "action": "response_updated",
      "user": "admin@encore.com",
      "details": {
        "manually_edited": true
      }
    },
    {
      "timestamp": "2025-11-12T10:45:00Z",
      "action": "status_changed",
      "user": "admin@encore.com",
      "details": {
        "old_status": "pending",
        "new_status": "reviewed"
      }
    }
  ],
  "total": 2
}
```

**What to Verify**:
- Status code: 200
- History ordered by timestamp (newest first)
- All actions logged
- User attribution correct

---

## 🧪 TESTING APPROACH

### Step 1: Environment Setup (30 minutes)

1. **Verify Backend Running**:
```bash
cd /home/ec2-user/encore-loyalty-new-system/encore_backend
source ../venv/bin/activate
python main.py
# Should start on http://localhost:8000
```

2. **Check Database Connectivity**:
- MySQL SSH tunnel active
- DynamoDB tables exist (`encore_venue_settings`, `encore_feedback_status`)

3. **Prepare Test Credentials**:
```python
# Test users from MySQL database
ADMIN_USER = {
    "email": "admin@encore.com",  # Find real admin email
    "password": "test_password"    # Find or reset password
}

VENUE_MANAGER = {
    "email": "manager@venue.com",  # Find real venue manager
    "password": "test_password",
    "venue_ids": [1]               # Their venue ID
}
```

4. **Install Test Dependencies**:
```bash
pip install pytest requests pytest-html
```

---

### Step 2: Create Test Scripts (2-3 hours)

**File Structure**:
```
/encore_backend/tests/automated/
├── test_auth.py           # Tests for endpoints 1-4
├── test_settings.py       # Tests for endpoints 5-10
├── test_feedback.py       # Tests for endpoints 11-18
├── conftest.py            # Shared fixtures (tokens, users)
├── config.py              # Test configuration
└── run_tests.sh           # Run all tests
```

**Example Test Structure** (`test_auth.py`):
```python
import pytest
import requests
from config import BASE_URL, ADMIN_USER, VENUE_MANAGER

class TestAuth:
    """Test Authentication endpoints"""
    
    def test_login_valid_admin(self):
        """Test login with valid admin credentials"""
        response = requests.post(
            f"{BASE_URL}/api/v1/auth/login",
            json={
                "email": ADMIN_USER["email"],
                "password": ADMIN_USER["password"]
            }
        )
        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert "refresh_token" in data
        assert data["user"]["role"] == "super_admin"
        
    def test_login_invalid_password(self):
        """Test login with invalid password"""
        response = requests.post(
            f"{BASE_URL}/api/v1/auth/login",
            json={
                "email": ADMIN_USER["email"],
                "password": "wrong_password"
            }
        )
        assert response.status_code == 401
        
    # ... more tests
```

**Key Testing Principles**:
1. **Test one thing at a time** - Clear test names
2. **Independent tests** - No test depends on another
3. **Clear assertions** - What exactly are we checking?
4. **Document failures** - Why did it fail?
5. **Performance tracking** - Log response times

---

### Step 3: Run Tests (1-2 hours)

**Run Tests**:
```bash
cd /home/ec2-user/encore-loyalty-new-system/encore_backend/tests/automated

# Run all tests
pytest -v --html=report.html

# Run specific test file
pytest test_auth.py -v

# Run specific test
pytest test_auth.py::TestAuth::test_login_valid_admin -v

# Run with coverage
pytest --cov=encore_backend --cov-report=html
```

**What to Capture**:
- ✅ Pass/Fail for each test
- ⏱️ Response times
- 🐛 Error messages and stack traces
- 📊 Coverage report
- 📸 Screenshots of failures (if applicable)

---

### Step 4: Document Bugs (30-60 minutes)

**For Each Bug Found**, document:

```markdown
## BUG-001: [Short Description]

**Severity**: Critical / High / Medium / Low
**Component**: Auth / Settings / Feedback
**Endpoint**: POST /api/v1/auth/login

**Description**:
Clear description of the bug.

**Steps to Reproduce**:
1. Step one
2. Step two
3. Step three

**Expected Result**:
What should happen

**Actual Result**:
What actually happened

**Error Message**:
```
Full error message/stack trace
```

**Test Code**:
```python
# Test that failed
```

**Suggested Fix** (if known):
Possible solution

**Workaround** (if exists):
Temporary fix
```

---

### Step 5: Create Test Report (30 minutes)

**Report Structure**:
```markdown
# Backend API Testing Report - Phases 1-4
**Date**: 2025-11-12
**Tester**: AI Worker
**Delegation**: DEL-002

## Executive Summary
- Total Endpoints Tested: 18
- Tests Run: XXX
- Passed: XXX (XX%)
- Failed: XXX (XX%)
- Blocked: XXX
- Duration: X hours

## Results by Phase

### Phase 2: Authentication (4 endpoints)
- ✅ POST /api/v1/auth/login - PASSED (8/8 tests)
- ❌ POST /api/v1/auth/refresh - FAILED (2/5 tests failed)
- ...

### Phase 3: Settings (6 endpoints)
...

### Phase 4: Feedback (8 endpoints)
...

## Critical Issues Found
1. [BUG-001] ...
2. [BUG-002] ...

## Performance Metrics
- Average response time: XXX ms
- Slowest endpoint: XXX (XXX ms)
- Fastest endpoint: XXX (XXX ms)

## Recommendations
1. Fix critical bugs before Phase 5
2. Optimize slow endpoints
3. ...

## Next Steps
1. Review bugs with Supervisor
2. Create fix prompts
3. Re-test after fixes
```

---

## 🎯 SUCCESS CRITERIA

**Testing Complete When**:
- ✅ All 18 endpoints have test scripts
- ✅ All tests executed successfully
- ✅ Test report generated
- ✅ Bugs documented with reproduction steps
- ✅ Performance metrics captured
- ✅ Test scripts saved for regression testing

**Quality Standards**:
- Each endpoint has at least 3 test cases (happy path, error cases, edge cases)
- All assertions are clear and meaningful
- Response times logged
- Error messages captured
- Role-based access tested

---

## 📊 DELIVERABLES

**Files to Create**:
1. `/encore_backend/tests/automated/test_auth.py`
2. `/encore_backend/tests/automated/test_settings.py`
3. `/encore_backend/tests/automated/test_feedback.py`
4. `/encore_backend/tests/automated/conftest.py`
5. `/encore_backend/tests/automated/config.py`
6. `/encore_backend/tests/automated/run_tests.sh`
7. `/encore_backend/tests/automated/README.md`

**Reports to Create**:
1. `/docs/TESTING/BACKEND-API-TEST-REPORT.md` - Main report
2. `/docs/TESTING/BUGS-FOUND.md` - Bug list
3. `/encore_backend/tests/automated/report.html` - HTML test report
4. `/encore_backend/tests/automated/coverage/` - Coverage report

---

## ⚠️ IMPORTANT NOTES

### Test Data
- **DO NOT** modify production data
- Use test venue IDs (verify which are test venues)
- Create test feedback if needed
- Clean up test data after testing

### Credentials
- Get real test credentials from Supervisor
- Do NOT hardcode passwords in tests
- Use environment variables or config file

### DynamoDB
- Tables must exist: `encore_venue_settings`, `encore_feedback_status`
- If tables don't exist, create them first
- Verify table structure matches expected schema

### MySQL
- Connection is READ-ONLY
- Do NOT attempt any INSERT/UPDATE/DELETE
- SSH tunnel must be active

### Stubs
- Phase 4 has stubs for AI generation (Phase 5) and email (Phase 6)
- Test that stubs return expected format
- Don't expect actual AI responses or emails yet

### Performance
- Flag any endpoint > 2 seconds
- Note any database query issues
- Check for N+1 query problems

---

## 🔄 REPORTING BACK TO SUPERVISOR

**When Tests Complete**, report:

```markdown
@Supervisor - Backend API Testing Complete

**Summary**:
- ✅ All 18 endpoints tested
- Tests Run: XXX
- Passed: XXX (XX%)
- Failed: XXX (XX%)
- Critical Bugs: X
- High Priority Bugs: X

**Critical Issues**:
1. [BUG-001] Brief description
2. [BUG-002] Brief description

**Files Created**:
- Test scripts: /encore_backend/tests/automated/
- Test report: /docs/TESTING/BACKEND-API-TEST-REPORT.md
- Bug list: /docs/TESTING/BUGS-FOUND.md

**Performance**:
- Average response time: XXX ms
- Slowest: XXX endpoint (XXX ms)

**Recommendation**:
[Should we fix bugs now, or proceed to frontend testing?]

**Ready for**: Next phase (frontend testing) or bug fixes
```

---

## 📚 REFERENCE MATERIALS

**Backend Code**:
- `/encore_backend/routes/auth.py` - Auth endpoints
- `/encore_backend/routes/settings.py` - Settings endpoints  
- `/encore_backend/routes/feedback.py` - Feedback endpoints
- `/encore_backend/models/` - Data models
- `/encore_backend/services/` - Business logic

**Documentation**:
- `/docs/SUPERVISOR/PHASE-2A-COMPLETION-REPORT.md` - Auth implementation
- `/docs/SUPERVISOR/PHASE-3A-COMPLETION-REPORT.md` - Settings implementation (if exists)
- `/docs/SUPERVISOR/PHASE-4A-COMPLETION-REPORT.md` - Feedback implementation (if exists)
- `/docs/plan-mockup-to-frontend/03-API-ENDPOINTS-SPECIFICATION.md` - API specs

**Database**:
- `/docs/investigation/MYSQL-DATABASE-ANALYSIS-SUMMARY.md` - MySQL schema
- `/docs/investigation/ENCORE-ARCHIVE-ANALYSIS-REPORT.md` - Legacy system analysis

---

## 🚀 START TESTING

**Ready to begin?**

1. Read this prompt thoroughly
2. Set up test environment
3. Create test scripts
4. Run tests carefully
5. Document findings
6. Report back to Supervisor

**Take your time. Be thorough. We want to catch bugs NOW, not in production.**

Good luck! 🎯

---

**Questions?** Ask Supervisor before starting.
**Blockers?** Report immediately.
**Stuck?** Don't guess - ask for clarification.

