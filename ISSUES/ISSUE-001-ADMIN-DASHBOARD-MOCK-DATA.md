# ISSUE-001: Admin Dashboard Using Mock Data

**Date Discovered**: 2025-11-23
**Priority**: HIGH
**Component**: Frontend - Admin Dashboard
**Status**: 🔴 **DOCUMENTED - AWAITING IMPLEMENTATION**

---

## 📋 ISSUE SUMMARY

The Admin Dashboard (`EnhancedDashboard` component) is **hardcoded to use mock API data** even when the application is configured to use the real backend (`REACT_APP_USE_MOCK_API=false`).

### Current Behavior
- User successfully authenticates using **real backend** credentials
- Dashboard displays statistics:
  - **Total Venues**: 12 (Mock)
  - **AI-Enabled Venues**: 8 (Mock)
  - **Responses Today**: Mock data
  - **Avg Response Time**: Mock data
  - **Recent Activity**: Mock data

### Expected Behavior
- Dashboard should fetch real statistics from the backend API
- Display actual venue counts from Brainium API integration
- Show real feedback statistics from MySQL database
- Display actual AI usage metrics from DynamoDB

---

## 🐛 ROOT CAUSE ANALYSIS

### Location of Issue
**File**: `encore_frontend/src/pages/admin/EnhancedDashboard.tsx`
**Lines**: 32-35

### Code Analysis
```typescript
const [statsData, chartData] = await Promise.all([
    mockApi.getGlobalDashboardStats(),  // ❌ Hardcoded to mock
    mockApi.getResponseVolumeData(chartPeriod),  // ❌ Hardcoded to mock
]);
```

### Why This Happened
1. The dashboard was built during Phase 1 with mock data
2. No real admin statistics API endpoints were implemented yet
3. The component was never refactored to use the conditional API service pattern (like authentication does)
4. Other components (Settings, Feedback, AI) correctly use the conditional pattern via `API_CONFIG.USE_MOCK_API`

---

## 🔧 TECHNICAL DETAILS

### Missing Backend Endpoints

The following admin statistics endpoints need to be implemented:

#### 1. **GET /api/admin/stats**
**Purpose**: Global system statistics
**Response**:
```json
{
  "totalVenues": 85,
  "aiEnabledVenues": 12,
  "totalFeedback": 61803,
  "feedbackToday": 156,
  "averageResponseTime": "2.3 hours",
  "systemHealth": "healthy"
}
```

#### 2. **GET /api/admin/chart-data**
**Purpose**: Historical response volume for charting
**Query Parameters**:
- `period`: 7 | 30 | 90 (days)

**Response**:
```json
{
  "data": [
    {"date": "2025-11-17", "count": 145},
    {"date": "2025-11-18", "count": 156},
    ...
  ]
}
```

#### 3. **GET /api/admin/recent-activity**
**Purpose**: Recent system activities and alerts
**Query Parameters**:
- `limit`: number (default 10)

**Response**:
```json
{
  "activities": [
    {
      "id": "act_123",
      "type": "venue_added" | "ai_enabled" | "alert_triggered" | "feedback_generated",
      "title": "New venue added",
      "description": "Venue XYZ has been provisioned",
      "severity": "success" | "info" | "warning" | "error",
      "timestamp": "2025-11-23T10:30:00Z",
      "venue_id": "venue_123",
      "venue_name": "Restaurant ABC"
    }
  ]
}
```

### Missing Frontend Service

Need to create: `encore_frontend/src/services/api/adminApi.ts`

Following the same pattern as:
- `authApi.ts` ✅ (exists)
- `settingsApi.ts` ✅ (exists)
- `feedbackApi.ts` ✅ (exists)
- `aiApi.ts` ✅ (exists)

---

## 📝 IMPLEMENTATION REQUIREMENTS

### Backend Development

**Files to Create/Modify**:
1. `encore_backend/routes/admin.py` (new file)
   - Implement statistics aggregation endpoints
   - Query Brainium API for venue counts
   - Query MySQL for feedback statistics
   - Query DynamoDB for AI usage metrics
   
2. `encore_backend/services/admin_service.py` (new file)
   - Business logic for statistics calculation
   - Integration with multiple data sources:
     - Brainium API (venue data)
     - MySQL (feedback data)
     - DynamoDB (AI settings and responses)

3. `encore_backend/models/admin.py` (new file)
   - Pydantic models for admin statistics
   - Request/response schemas

### Frontend Development

**Files to Create/Modify**:
1. `encore_frontend/src/services/api/adminApi.ts` (new file)
   - Implement API client for admin endpoints
   - Follow same pattern as existing API services
   - Support both mock and real API (conditional export)

2. `encore_frontend/src/services/api/index.ts` (modify)
   - Export adminApi service

3. `encore_frontend/src/pages/admin/EnhancedDashboard.tsx` (modify)
   - Replace `mockApi` calls with conditional `adminApi`
   - Handle loading and error states
   - Format data for display

---

## 🎯 ACCEPTANCE CRITERIA

### Backend
- [ ] `/api/admin/stats` endpoint returns accurate venue count from Brainium API
- [ ] `/api/admin/stats` returns accurate feedback counts from MySQL
- [ ] `/api/admin/chart-data` returns historical data for past N days
- [ ] `/api/admin/recent-activity` returns recent system events
- [ ] All endpoints return data in < 2 seconds
- [ ] Proper error handling for all data source failures
- [ ] Unit tests for admin service logic
- [ ] Integration tests for API endpoints

### Frontend
- [ ] `adminApi.ts` created following existing patterns
- [ ] Conditional export based on `REACT_APP_USE_MOCK_API`
- [ ] Dashboard component updated to use real API
- [ ] Loading states properly displayed
- [ ] Error states properly handled
- [ ] Numbers update in real-time when data changes
- [ ] No console errors
- [ ] TypeScript compilation clean

### Testing
- [ ] Manual verification: Real venue count matches Brainium API
- [ ] Manual verification: Feedback count matches MySQL count
- [ ] Manual verification: AI-enabled count matches DynamoDB
- [ ] Test both mock and real API modes work correctly
- [ ] Verify dashboard loads in < 3 seconds

---

## 📖 IMPLEMENTATION NOTES

### Data Sources for Statistics

#### Total Venues (from Brainium API)
```python
# Query Brainium API /api/v1/venues
# Count total venues owned by the account
```

#### AI-Enabled Venues (from DynamoDB)
```python
# Query encore_venue_settings table
# Count venues where settings.ai_generation_enabled = true
```

#### Total Feedback (from MySQL)
```python
# Query: SELECT COUNT(*) FROM comment
# Can be cached/updated periodically
```

#### Feedback Today (from MySQL)
```python
# Query: SELECT COUNT(*) FROM comment 
# WHERE DATE(created_at) = CURDATE()
```

#### Average Response Time (from DynamoDB)
```python
# Query encore_feedback_responses table
# Calculate avg time between feedback_created and response_generated
# Format as human-readable (e.g., "2.3 hours")
```

### Performance Considerations

- **Cache Strategy**: Statistics can be cached for 5-15 minutes
- **Parallel Queries**: Use async/await to query all data sources simultaneously
- **Fallback Values**: If a data source fails, show "N/A" instead of crashing
- **Background Updates**: Consider WebSocket updates for real-time stats

---

## 🔄 RELATED ISSUES

- **BUG-014**: Feedback query empty items (FIXED via MySQL migration)
- **BUG-015**: Customer data not retrievable (FIXED via MySQL migration)
- **BUG-016**: Feedback by ID returns "Not Found" (FIXED via MySQL migration)

All critical blockers are now resolved. This issue is purely about integrating the frontend with working backend capabilities.

---

## 📅 ESTIMATED EFFORT

- **Backend Development**: 8-12 hours
  - Admin routes: 3-4 hours
  - Admin service logic: 3-4 hours
  - Models and schemas: 1-2 hours
  - Testing: 2-3 hours

- **Frontend Development**: 4-6 hours
  - adminApi.ts creation: 2-3 hours
  - Dashboard component update: 1-2 hours
  - Testing and debugging: 1-2 hours

- **Total**: 12-18 hours (1.5-2 days for focused developer)

---

## 💡 PRIORITY JUSTIFICATION

**Why HIGH Priority**:
1. First impression for client - admin logs in and sees mock data
2. Statistics are critical for understanding system performance
3. Blocks accurate assessment of AI rollout progress
4. Other features (Settings, Feedback, AI) are working with real backend

**Impact of NOT Fixing**:
- Client cannot see real system metrics
- Cannot track actual AI adoption across venues
- False sense of system activity
- Reduces confidence in overall implementation

**Recommendation**: Address immediately after Phase 5 integration testing is complete, OR in parallel if separate developer available.

---

## 🔗 DEPENDENCIES

**Requires**:
- ✅ Brainium API integration (complete)
- ✅ MySQL connection (complete)
- ✅ DynamoDB setup (complete)
- ✅ Authentication system (complete)

**Blocks**:
- Production readiness
- Client demo/acceptance
- Accurate system monitoring

---

## 📄 NEXT STEPS

1. Review and approve this issue documentation
2. Prioritize against remaining Phase 5 work
3. Assign to developer (or create Cursor thread with prompt below)
4. Track implementation progress
5. Test and verify real data accuracy
6. Update CURRENT-STATUS.md upon completion

---

**Issue Owner**: TBD
**Reviewer**: AI Supervisor
**Last Updated**: 2025-11-23

