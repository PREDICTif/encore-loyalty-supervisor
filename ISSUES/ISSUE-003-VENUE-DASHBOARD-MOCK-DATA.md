# ISSUE-003: Venue Dashboard Using Mock Data

**Date Discovered**: 2025-11-26
**Priority**: HIGH
**Component**: Frontend - Venue Dashboard
**Status**: 🔴 **DOCUMENTED - AWAITING IMPLEMENTATION**

---

## 📋 ISSUE SUMMARY

The Venue Dashboard (`VenueDashboardEnhanced` component) is **using the mock API** even when the application is configured to use the real backend (`REACT_APP_USE_MOCK_API=false`). This is similar to ISSUE-001 for the Admin Dashboard.

### Current Behavior
- User successfully impersonates a venue (e.g., "Opus Ocean Grille")
- Dashboard displays statistics from **mock API**:
  - **New Feedback**: 23 (Mock via mockApi.getVenueDashboardStats)
  - **Pending**: 5 (Mock)
  - **Sent Today**: 8 (HARDCODED in component!)
  - **Avg Rating**: 4.2 (Mock)
  - **Recent Alerts**: Mock data (e.g., "Food poisoning mention", "Low Rating: 2-star review")
- **Venue Name**: Hardcoded as "Bella Vista Restaurant" instead of actual impersonated venue

### Expected Behavior
- Dashboard should fetch real statistics for the impersonated venue from the backend API
- Display actual feedback counts from MySQL database
- Show real alerts/notifications
- Display actual venue name from auth context

---

## 🐛 ROOT CAUSE ANALYSIS

### Location of Issue

**Primary File**: `encore_frontend/src/contexts/VenueContext.tsx`
**Lines**: 79, 162-163

**Secondary File**: `encore_frontend/src/pages/venue/VenueDashboardEnhanced.tsx`
**Lines**: 104-105 (hardcoded values), 132 (hardcoded venue name)

### Code Analysis

**VenueContext.tsx** - All data from mockApi:
```typescript
// Line 79
const venueSettings = await mockApi.getVenueSettings(venueId);

// Lines 162-163
const [stats, alertsData] = await Promise.all([
  mockApi.getVenueDashboardStats(venueId),  // ❌ Hardcoded to mock
  mockApi.getVenueAlerts(venueId),          // ❌ Hardcoded to mock
]);
```

**VenueDashboardEnhanced.tsx** - Hardcoded values:
```typescript
// Lines 104-105 - "Sent Today" is hardcoded
{
  title: 'Sent Today',
  value: 8,  // ❌ HARDCODED!
  change: 67,
  // ...
}

// Line 132 - Venue name is hardcoded
<p className="text-gray-600 mt-1">Bella Vista Restaurant</p>  // ❌ HARDCODED!
```

### Why This Happened
1. The venue dashboard was built during mockup phase with mock data
2. VenueContext was designed to work with mockApi only
3. No real venue statistics API endpoints were implemented
4. Component has some hardcoded values that were never parameterized

---

## 🔧 TECHNICAL DETAILS

### Missing/Needed Backend Endpoints

The following venue-specific endpoints need to be implemented or connected:

#### 1. **GET /api/venue/{venue_id}/stats**
**Purpose**: Venue-specific dashboard statistics
**Response**:
```json
{
  "todayFeedbackCount": 23,
  "pendingResponses": 5,
  "sentToday": 8,
  "averageRating": 4.2,
  "ratingTrend": -0.1,
  "feedbackTrend": 20,
  "sentTrend": 67
}
```

#### 2. **GET /api/venue/{venue_id}/alerts**
**Purpose**: Recent alerts for the venue
**Response**:
```json
{
  "alerts": [
    {
      "id": "alert_123",
      "level": "critical" | "high" | "medium" | "low" | "info",
      "title": "Critical: Food poisoning mention",
      "feedbackId": "456789",
      "time": "2025-11-26T10:30:00Z",
      "isRead": false
    }
  ]
}
```

### Existing Backend Resources

The good news is we already have working infrastructure:

#### Feedback Statistics (MySQL)
We can calculate these from existing feedback data:
```python
# New feedback today for venue
SELECT COUNT(*) FROM comment c 
INNER JOIN datasession ds ON c.iddatasession = ds.iddatasession
WHERE ds.idclient = {venue_id} 
AND DATE(c.dateentered) = CURDATE()

# Average rating for venue
SELECT AVG(overall_rating) FROM comment c
INNER JOIN datasession ds ON c.iddatasession = ds.iddatasession
WHERE ds.idclient = {venue_id}
```

#### AI Response Status (DynamoDB)
We can query `encore_feedback_responses` table for:
- Count of pending responses
- Count of sent responses today

---

## 📝 IMPLEMENTATION REQUIREMENTS

### Backend Development

**Files to Create/Modify**:

1. `encore_backend/routes/venue_stats.py` (new file)
   - Implement `/api/venue/{venue_id}/stats` endpoint
   - Implement `/api/venue/{venue_id}/alerts` endpoint
   - Aggregate data from MySQL and DynamoDB
   
2. `encore_backend/services/venue_stats_service.py` (new file)
   - Calculate venue-specific statistics
   - Query MySQL for feedback counts and ratings
   - Query DynamoDB for AI response status
   - Generate alerts based on feedback content (low ratings, critical keywords)

3. `encore_backend/models/venue_stats.py` (new file)
   - Pydantic models for venue statistics
   - Alert models

### Frontend Development

**Files to Create/Modify**:

1. `encore_frontend/src/services/api/venueStatsApi.ts` (new file)
   - API client for venue statistics endpoints
   - Follow existing patterns (authApi, feedbackApi, etc.)

2. `encore_frontend/src/contexts/VenueContext.tsx` (modify)
   - Replace `mockApi` calls with conditional API (check `API_CONFIG.USE_MOCK_API`)
   - Import and use venueStatsApi when not in mock mode

3. `encore_frontend/src/pages/venue/VenueDashboardEnhanced.tsx` (modify)
   - Remove hardcoded values (Sent Today: 8, venue name)
   - Use `dashboardStats` for all values
   - Use `user.venue_name` from auth context for venue name display

---

## 🎯 ACCEPTANCE CRITERIA

### Backend
- [ ] `/api/venue/{venue_id}/stats` returns real feedback counts
- [ ] `/api/venue/{venue_id}/stats` returns real average rating
- [ ] `/api/venue/{venue_id}/alerts` returns alerts based on recent feedback
- [ ] All endpoints return data in < 2 seconds
- [ ] Proper error handling
- [ ] Unit tests for statistics calculation

### Frontend
- [ ] VenueContext uses real API when `REACT_APP_USE_MOCK_API=false`
- [ ] Dashboard shows actual venue name from auth context
- [ ] All statistics are from API (no hardcoded values)
- [ ] Loading states properly displayed
- [ ] Error states properly handled
- [ ] No console errors
- [ ] TypeScript compilation clean

### Testing
- [ ] Manual verification: Feedback counts match MySQL query
- [ ] Manual verification: Average rating is accurate
- [ ] Test both mock and real API modes
- [ ] Verify dashboard loads in < 3 seconds

---

## 📊 MOCK DATA vs REAL DATA MAPPING

| Field | Current (Mock/Hardcoded) | Source for Real Data |
|-------|-------------------------|----------------------|
| Venue Name | "Bella Vista Restaurant" | `user.venue_name` from AuthContext |
| New Feedback | 23 (mock) | MySQL: COUNT feedback today for venue |
| Pending | 5 (mock) | DynamoDB: COUNT responses with status='pending' |
| Sent Today | 8 (HARDCODED) | DynamoDB: COUNT responses with status='sent' today |
| Avg Rating | 4.2 (mock) | MySQL: AVG(overall_rating) for venue |
| Feedback Trend | 20% (hardcoded) | Calculate: today vs yesterday |
| Rating Trend | -0.1 (hardcoded) | Calculate: this week vs last week |
| Alerts | Mock alerts | Generate from: low ratings, keywords in feedback |

---

## 📅 ESTIMATED EFFORT

- **Backend Development**: 6-8 hours
  - Venue stats route: 2-3 hours
  - Statistics service: 2-3 hours
  - Alerts generation logic: 2-3 hours

- **Frontend Development**: 3-4 hours
  - venueStatsApi.ts: 1 hour
  - VenueContext update: 1-2 hours
  - Dashboard component fixes: 1 hour

- **Total**: 9-12 hours (1-1.5 days for focused developer)

---

## 💡 PRIORITY JUSTIFICATION

**Why HIGH Priority**:
1. Venue managers will see obviously fake data after impersonation
2. "Bella Vista Restaurant" showing for all venues is confusing
3. Statistics are critical for venue managers to understand their performance
4. Other venue features (Feedback List) work with real data - inconsistent

**Impact of NOT Fixing**:
- Venue managers cannot see real metrics
- False sense of feedback volume and response activity
- Reduces confidence in the system
- Client demo will show obvious mock data

**Recommendation**: Address in Phase 6 or 7, alongside ISSUE-001 (Admin Dashboard). These can share some infrastructure (stats calculation patterns).

---

## 🔗 DEPENDENCIES

**Requires**:
- ✅ MySQL connection (complete)
- ✅ DynamoDB setup (complete)
- ✅ Venue impersonation (JUST FIXED - working!)
- ✅ Authentication system (complete)

**Blocks**:
- Production readiness for venue managers
- Accurate venue performance monitoring
- Client acceptance testing

---

## 🔄 RELATED ISSUES

- **ISSUE-001**: Admin Dashboard Mock Data (similar issue, different dashboard)
- **Impersonation Bug**: FIXED 2025-11-26 (localStorage key mismatch)

Both ISSUE-001 and ISSUE-003 can potentially share:
- Statistics calculation service patterns
- API response structures
- Frontend conditional API patterns

---

## 📄 NEXT STEPS

1. Review and approve this issue documentation
2. Consider addressing alongside ISSUE-001 for efficiency
3. Prioritize based on client needs (admin vs venue manager views)
4. Create implementation prompt for developer
5. Test with real impersonation flow
6. Update CURRENT-STATUS.md upon completion

---

**Issue Owner**: TBD
**Reviewer**: AI Supervisor
**Last Updated**: 2025-11-26

