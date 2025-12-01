# WORKER PROMPT: ISSUE-001 - Admin Dashboard Real Data Integration

**Issue Reference**: `ISSUE-001-ADMIN-DASHBOARD-MOCK-DATA.md`
**Priority**: HIGH
**Estimated Effort**: 12-18 hours
**Skills Required**: FastAPI, React/TypeScript, DynamoDB, MySQL
**Target Completion**: 1.5-2 days

---

## 🎯 TASK OBJECTIVE

Implement backend API endpoints and frontend integration to replace **mock data** with **real statistics** in the Admin Dashboard.

### Current State
- ✅ User authentication working with real backend
- ❌ Admin Dashboard showing hardcoded mock numbers
- ✅ All required data sources are working (MySQL, DynamoDB, Brainium API)

### Desired State
- ✅ Admin Dashboard displays real venue counts from Brainium API
- ✅ Real feedback statistics from MySQL database
- ✅ Actual AI usage metrics from DynamoDB
- ✅ Real-time or cached statistics refresh

---

## 📦 DELIVERABLES

### Backend Deliverables

1. **Admin API Routes** (`encore_backend/routes/admin.py`)
   - 3 new endpoints for statistics
   - Proper authentication and authorization
   - Error handling and logging

2. **Admin Service** (`encore_backend/services/admin_service.py`)
   - Business logic for data aggregation
   - Integration with Brainium API, MySQL, DynamoDB
   - Caching strategy (5-15 min TTL)

3. **Admin Models** (`encore_backend/models/admin.py`)
   - Pydantic request/response schemas
   - Type definitions for statistics

4. **Tests** (`encore_backend/tests/test_admin_api.py`)
   - Unit tests for admin service
   - Integration tests for API endpoints
   - Minimum 80% coverage

### Frontend Deliverables

1. **Admin API Client** (`encore_frontend/src/services/api/adminApi.ts`)
   - API client following existing patterns
   - Mock and real implementations
   - Error handling

2. **Updated Dashboard** (`encore_frontend/src/pages/admin/EnhancedDashboard.tsx`)
   - Replace mockApi calls with adminApi
   - Proper loading and error states
   - Real-time data refresh

3. **Type Definitions** (update `encore_frontend/src/types/index.ts`)
   - Ensure frontend types match backend models

---

## 🔧 TECHNICAL IMPLEMENTATION GUIDE

### PART 1: Backend Implementation (8-12 hours)

#### Step 1.1: Create Admin Models (1-2 hours)

**File**: `encore_backend/models/admin.py`

```python
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime

class DashboardStats(BaseModel):
    """Global dashboard statistics"""
    total_venues: int
    ai_enabled_venues: int
    total_feedback: int
    feedback_today: int
    average_response_time: str  # Human readable: "2.3 hours"
    system_health: str  # "healthy", "warning", "error"
    last_updated: datetime

class ChartDataPoint(BaseModel):
    """Single data point for charts"""
    date: str  # ISO date string
    count: int

class ChartData(BaseModel):
    """Chart data response"""
    data: List[ChartDataPoint]
    period: int  # Number of days

class Activity(BaseModel):
    """Recent activity item"""
    id: str
    type: str  # "venue_added", "ai_enabled", "alert_triggered", etc.
    title: str
    description: str
    severity: str  # "success", "info", "warning", "error"
    timestamp: datetime
    venue_id: Optional[str] = None
    venue_name: Optional[str] = None

class RecentActivitiesResponse(BaseModel):
    """Recent activities response"""
    activities: List[Activity]
    total: int
```

#### Step 1.2: Create Admin Service (3-4 hours)

**File**: `encore_backend/services/admin_service.py`

```python
import logging
from datetime import datetime, timedelta
from typing import Dict, Any, List
from db.direct_mysql_service import DirectMySQLService
from db.dynamodb_service import RestaurantDynamoDBService
from services.brainium_service import BrainiumAPIService
from models.admin import DashboardStats, ChartDataPoint, ChartData, Activity

logger = logging.getLogger(__name__)

class AdminService:
    """Service for admin dashboard statistics and data"""
    
    def __init__(
        self,
        mysql_service: DirectMySQLService,
        dynamodb_service: RestaurantDynamoDBService,
        brainium_service: BrainiumAPIService
    ):
        self.mysql = mysql_service
        self.dynamodb = dynamodb_service
        self.brainium = brainium_service
        self._cache = {}
        self._cache_ttl = 300  # 5 minutes
    
    async def get_dashboard_stats(self) -> DashboardStats:
        """
        Get global dashboard statistics.
        
        Data Sources:
        - Total Venues: Brainium API
        - AI-Enabled Venues: DynamoDB (count where ai_generation_enabled = true)
        - Total Feedback: MySQL (SELECT COUNT(*) FROM comment)
        - Feedback Today: MySQL (WHERE DATE(created_at) = CURDATE())
        - Avg Response Time: DynamoDB (calculate from encore_feedback_responses)
        
        Returns:
            DashboardStats: Global statistics
        """
        # Check cache first
        cache_key = 'dashboard_stats'
        if cache_key in self._cache:
            cached_data, cached_time = self._cache[cache_key]
            if (datetime.now() - cached_time).seconds < self._cache_ttl:
                return cached_data
        
        # Fetch data from all sources in parallel
        try:
            # TODO: Implement parallel queries
            total_venues = await self._get_total_venues()
            ai_enabled_venues = await self._get_ai_enabled_count()
            total_feedback = await self._get_total_feedback()
            feedback_today = await self._get_feedback_today()
            avg_response_time = await self._calculate_avg_response_time()
            
            stats = DashboardStats(
                total_venues=total_venues,
                ai_enabled_venues=ai_enabled_venues,
                total_feedback=total_feedback,
                feedback_today=feedback_today,
                average_response_time=avg_response_time,
                system_health="healthy",
                last_updated=datetime.now()
            )
            
            # Cache the result
            self._cache[cache_key] = (stats, datetime.now())
            
            return stats
            
        except Exception as e:
            logger.error(f"Error fetching dashboard stats: {e}")
            # Return fallback values on error
            return self._get_fallback_stats()
    
    async def _get_total_venues(self) -> int:
        """Get total venue count from Brainium API"""
        try:
            venues = await self.brainium.get_venues()
            return len(venues)
        except Exception as e:
            logger.error(f"Error getting total venues: {e}")
            return 0
    
    async def _get_ai_enabled_count(self) -> int:
        """Count AI-enabled venues from DynamoDB"""
        try:
            # Query encore_venue_settings table
            # Count where ai_generation_enabled = true
            # TODO: Implement DynamoDB query
            return 0
        except Exception as e:
            logger.error(f"Error getting AI-enabled count: {e}")
            return 0
    
    async def _get_total_feedback(self) -> int:
        """Get total feedback count from MySQL"""
        try:
            query = "SELECT COUNT(*) as count FROM comment"
            result = self.mysql._execute_mysql_query(query)
            return result[0]['count'] if result else 0
        except Exception as e:
            logger.error(f"Error getting total feedback: {e}")
            return 0
    
    async def _get_feedback_today(self) -> int:
        """Get today's feedback count from MySQL"""
        try:
            query = """
                SELECT COUNT(*) as count 
                FROM comment 
                WHERE DATE(created_at) = CURDATE()
            """
            result = self.mysql._execute_mysql_query(query)
            return result[0]['count'] if result else 0
        except Exception as e:
            logger.error(f"Error getting today's feedback: {e}")
            return 0
    
    async def _calculate_avg_response_time(self) -> str:
        """Calculate average response time from DynamoDB"""
        try:
            # Query encore_feedback_responses
            # Calculate avg time between feedback_created and response_generated
            # Format as human readable (e.g., "2.3 hours")
            # TODO: Implement calculation
            return "N/A"
        except Exception as e:
            logger.error(f"Error calculating avg response time: {e}")
            return "N/A"
    
    def _get_fallback_stats(self) -> DashboardStats:
        """Return fallback stats on error"""
        return DashboardStats(
            total_venues=0,
            ai_enabled_venues=0,
            total_feedback=0,
            feedback_today=0,
            average_response_time="N/A",
            system_health="error",
            last_updated=datetime.now()
        )
    
    async def get_chart_data(self, period: int = 7) -> ChartData:
        """
        Get response volume chart data for specified period.
        
        Args:
            period: Number of days to include (7, 30, or 90)
            
        Returns:
            ChartData: Historical response volume
        """
        try:
            # Query MySQL for feedback counts by day
            # TODO: Implement SQL query with date grouping
            data_points = []
            return ChartData(data=data_points, period=period)
        except Exception as e:
            logger.error(f"Error fetching chart data: {e}")
            return ChartData(data=[], period=period)
    
    async def get_recent_activities(self, limit: int = 10) -> List[Activity]:
        """
        Get recent system activities.
        
        Args:
            limit: Number of activities to return
            
        Returns:
            List of Activity objects
        """
        # TODO: Implement activity tracking
        # For now, return empty list
        return []
```

#### Step 1.3: Create Admin Routes (3-4 hours)

**File**: `encore_backend/routes/admin.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from dependencies import get_current_user, get_mysql_service, get_dynamodb_service, get_brainium_service
from services.admin_service import AdminService
from models.admin import DashboardStats, ChartData, RecentActivitiesResponse
from models.user import User
from db.direct_mysql_service import DirectMySQLService
from db.dynamodb_service import RestaurantDynamoDBService
from services.brainium_service import BrainiumAPIService

router = APIRouter(prefix="/api/admin", tags=["admin"])

def get_admin_service(
    mysql_service: DirectMySQLService = Depends(get_mysql_service),
    dynamodb_service: RestaurantDynamoDBService = Depends(get_dynamodb_service),
    brainium_service: BrainiumAPIService = Depends(get_brainium_service)
) -> AdminService:
    """Dependency injection for AdminService"""
    return AdminService(mysql_service, dynamodb_service, brainium_service)

@router.get("/stats", response_model=DashboardStats)
async def get_dashboard_stats(
    current_user: User = Depends(get_current_user),
    admin_service: AdminService = Depends(get_admin_service)
):
    """
    Get global dashboard statistics.
    
    Requires admin role (global_admin, super_admin, or sysadmin).
    
    Returns:
        DashboardStats: Global system statistics
    """
    # Check admin permission
    if current_user.role not in ["global_admin", "super_admin", "sysadmin"]:
        raise HTTPException(status_code=403, detail="Admin access required")
    
    return await admin_service.get_dashboard_stats()

@router.get("/chart-data", response_model=ChartData)
async def get_chart_data(
    period: int = 7,
    current_user: User = Depends(get_current_user),
    admin_service: AdminService = Depends(get_admin_service)
):
    """
    Get response volume chart data.
    
    Args:
        period: Number of days to include (7, 30, or 90)
    
    Returns:
        ChartData: Historical response volume
    """
    if current_user.role not in ["global_admin", "super_admin", "sysadmin"]:
        raise HTTPException(status_code=403, detail="Admin access required")
    
    if period not in [7, 30, 90]:
        raise HTTPException(status_code=400, detail="Period must be 7, 30, or 90 days")
    
    return await admin_service.get_chart_data(period)

@router.get("/recent-activity", response_model=RecentActivitiesResponse)
async def get_recent_activities(
    limit: int = 10,
    current_user: User = Depends(get_current_user),
    admin_service: AdminService = Depends(get_admin_service)
):
    """
    Get recent system activities and alerts.
    
    Args:
        limit: Number of activities to return (default 10, max 50)
    
    Returns:
        RecentActivitiesResponse: List of recent activities
    """
    if current_user.role not in ["global_admin", "super_admin", "sysadmin"]:
        raise HTTPException(status_code=403, detail="Admin access required")
    
    if limit > 50:
        limit = 50
    
    activities = await admin_service.get_recent_activities(limit)
    return RecentActivitiesResponse(activities=activities, total=len(activities))
```

#### Step 1.4: Register Admin Router (5 minutes)

**File**: `encore_backend/routes/__init__.py` (modify)

```python
from fastapi import APIRouter
from routes import auth, settings, feedback, ai, admin  # Add admin

api_router = APIRouter()

# Register all routers
api_router.include_router(auth.router)
api_router.include_router(settings.router)
api_router.include_router(feedback.router)
api_router.include_router(ai.router)
api_router.include_router(admin.router)  # ← ADD THIS LINE
```

---

### PART 2: Frontend Implementation (4-6 hours)

#### Step 2.1: Create Admin API Client (2-3 hours)

**File**: `encore_frontend/src/services/api/adminApi.ts`

```typescript
/**
 * Admin API Service
 *
 * Handles all admin-related API calls for dashboard statistics,
 * system monitoring, and administrative functions.
 */

import { apiClient } from './client';
import { API_CONFIG } from '../../config';
import { mockApi } from '../mockApi';
import {
  DashboardStats,
  ChartDataPoint,
  Activity
} from '../../types';

class AdminApiService {
  /**
   * Get global dashboard statistics
   */
  async getDashboardStats(): Promise<DashboardStats> {
    // Use mock in development if configured
    if (API_CONFIG.USE_MOCK_API) {
      return mockApi.getGlobalDashboardStats();
    }

    const response = await apiClient.get<DashboardStats>('/api/admin/stats');
    return response.data;
  }

  /**
   * Get response volume chart data
   */
  async getChartData(period: 7 | 30 | 90): Promise<ChartDataPoint[]> {
    if (API_CONFIG.USE_MOCK_API) {
      return mockApi.getResponseVolumeData(period);
    }

    const response = await apiClient.get<{ data: ChartDataPoint[] }>(
      '/api/admin/chart-data',
      { params: { period } }
    );
    return response.data.data;
  }

  /**
   * Get recent system activities
   */
  async getRecentActivities(limit: number = 10): Promise<Activity[]> {
    if (API_CONFIG.USE_MOCK_API) {
      return mockApi.getAdminRecentActivity();
    }

    const response = await apiClient.get<{ activities: Activity[] }>(
      '/api/admin/recent-activity',
      { params: { limit } }
    );
    return response.data.activities;
  }
}

export const adminApi = new AdminApiService();
```

#### Step 2.2: Update Dashboard Component (1-2 hours)

**File**: `encore_frontend/src/pages/admin/EnhancedDashboard.tsx`

**FIND**:
```typescript
const [statsData, chartData] = await Promise.all([
    mockApi.getGlobalDashboardStats(),
    mockApi.getResponseVolumeData(chartPeriod),
]);
```

**REPLACE WITH**:
```typescript
import { adminApi } from '../../services/api/adminApi';

// ... in loadData function:
const [statsData, chartData] = await Promise.all([
    adminApi.getDashboardStats(),
    adminApi.getChartData(chartPeriod),
]);
```

**Also update Recent Activities**:

**FIND**:
```typescript
const { activities, loading: adminLoading } = useAdmin();
```

**REPLACE WITH**:
```typescript
const [activities, setActivities] = useState<Activity[]>([]);
const [loading, setLoading] = useState(true);

useEffect(() => {
    const loadData = async () => {
        try {
            setLoading(true);
            const [statsData, chartData, activitiesData] = await Promise.all([
                adminApi.getDashboardStats(),
                adminApi.getChartData(chartPeriod),
                adminApi.getRecentActivities(),
            ]);
            setStats(statsData);
            setChartData(chartData);
            setActivities(activitiesData);
        } catch (error) {
            console.error('Failed to load dashboard data:', error);
        } finally {
            setLoading(false);
        }
    };

    loadData();
}, [chartPeriod]);
```

#### Step 2.3: Update Type Definitions (30 min)

**File**: `encore_frontend/src/types/index.ts`

Ensure frontend types match backend Pydantic models exactly:

```typescript
export interface DashboardStats {
  totalVenues: number;
  aiEnabledVenues: number;
  totalFeedback: number;
  feedbackToday: number;
  averageResponseTime: string;
  systemHealth: 'healthy' | 'warning' | 'error';
  lastUpdated: string;
}

export interface ChartDataPoint {
  date: string;  // ISO date string
  count: number;
}

export interface Activity {
  id: string;
  type: 'venue_added' | 'ai_enabled' | 'alert_triggered' | 'feedback_generated';
  title: string;
  description: string;
  severity: 'success' | 'info' | 'warning' | 'error';
  timestamp: string;
  venue_id?: string;
  venue_name?: string;
}
```

---

## 🧪 TESTING REQUIREMENTS

### Backend Tests

**File**: `encore_backend/tests/test_admin_api.py`

```python
import pytest
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_get_dashboard_stats_requires_auth():
    """Test that stats endpoint requires authentication"""
    response = client.get("/api/admin/stats")
    assert response.status_code == 401

def test_get_dashboard_stats_requires_admin_role():
    """Test that stats endpoint requires admin role"""
    # TODO: Login as venue_manager
    # Attempt to access /api/admin/stats
    # Assert 403 Forbidden

def test_get_dashboard_stats_success():
    """Test successful stats retrieval"""
    # TODO: Login as admin
    # Get /api/admin/stats
    # Assert 200 OK
    # Verify response structure matches DashboardStats model

def test_chart_data_validates_period():
    """Test that chart data validates period parameter"""
    # TODO: Login as admin
    # Try period=15 (invalid)
    # Assert 400 Bad Request

def test_recent_activities_respects_limit():
    """Test that activities endpoint respects limit parameter"""
    # TODO: Login as admin
    # Request with limit=5
    # Assert response has max 5 items
```

### Frontend Testing

1. **Manual Testing Checklist**:
   - [ ] Toggle `REACT_APP_USE_MOCK_API` between true/false
   - [ ] Verify mock mode shows mock data
   - [ ] Verify real mode shows actual data
   - [ ] Check that venue counts match Brainium API
   - [ ] Verify feedback counts match MySQL
   - [ ] Test loading states display correctly
   - [ ] Test error handling (disconnect backend)

2. **Automated Tests**:
   - Update existing component tests to handle real API
   - Add tests for adminApi service
   - Test error boundary behavior

---

## 📋 IMPLEMENTATION CHECKLIST

### Phase 1: Backend (Day 1)
- [ ] Create `models/admin.py` with Pydantic schemas
- [ ] Implement `AdminService` in `services/admin_service.py`
- [ ] Create `/api/admin/stats` endpoint
- [ ] Create `/api/admin/chart-data` endpoint
- [ ] Create `/api/admin/recent-activity` endpoint
- [ ] Register admin router in `routes/__init__.py`
- [ ] Write unit tests for AdminService
- [ ] Write integration tests for API endpoints
- [ ] Test manually using curl/Postman
- [ ] Verify response times < 2 seconds

### Phase 2: Frontend (Day 1-2)
- [ ] Create `services/api/adminApi.ts`
- [ ] Implement conditional export (mock vs real)
- [ ] Update `types/index.ts` with matching interfaces
- [ ] Modify `EnhancedDashboard.tsx` to use adminApi
- [ ] Test with `REACT_APP_USE_MOCK_API=true` (mock mode)
- [ ] Test with `REACT_APP_USE_MOCK_API=false` (real mode)
- [ ] Verify no console errors
- [ ] Verify TypeScript compilation clean
- [ ] Test loading and error states
- [ ] Verify data accuracy against source APIs

### Phase 3: Integration Testing (Day 2)
- [ ] End-to-end test: Login → Dashboard → Verify real data
- [ ] Compare venue count with Brainium API
- [ ] Compare feedback count with MySQL
- [ ] Compare AI-enabled count with DynamoDB
- [ ] Test chart data accuracy
- [ ] Test dashboard refresh/reload
- [ ] Test error recovery scenarios
- [ ] Performance testing (< 3 second load time)

---

## 🎓 TECHNICAL REFERENCE

### Existing Patterns to Follow

#### Backend Service Pattern
See: `encore_backend/services/settings_service.py`
- Dependency injection via FastAPI
- Proper error handling and logging
- Type hints for all methods
- Docstrings for all public methods

#### Backend Route Pattern
See: `encore_backend/routes/auth.py`
- Router with prefix and tags
- Dependency injection
- Response models
- Error handling
- Docstrings

#### Frontend API Pattern
See: `encore_frontend/src/services/api/authApi.ts`
- Axios client usage
- Error handling with logger
- Type definitions from `types/index.ts`
- Conditional export for mock vs real

---

## 🚨 GOTCHAS & CONSIDERATIONS

### Backend

1. **Brainium API Rate Limiting**: Cache venue data (rarely changes)
2. **MySQL Query Performance**: Feedback counts can be slow on 61K rows
   - Consider adding index on `created_at`
   - Cache total count (update every 15 min)
3. **DynamoDB Scan**: AI-enabled count requires scanning
   - Implement secondary index if performance issue
4. **Error Handling**: Each data source can fail independently
   - Return partial data if some sources fail
   - Set `system_health` to "warning" if issues detected

### Frontend

1. **Loading States**: Dashboard should show loading skeletons while fetching
2. **Error States**: Display friendly error message if API fails
3. **Data Freshness**: Consider auto-refresh every 30 seconds
4. **Type Safety**: Ensure backend response matches TypeScript interfaces exactly
5. **Decimal Formatting**: Handle large numbers (61K+) with proper formatting

---

## 📚 REQUIRED READING

Before starting, review these files:

### Backend
- `/encore_backend/services/settings_service.py` (service pattern)
- `/encore_backend/routes/auth.py` (route pattern)
- `/encore_backend/services/brainium_service.py` (external API integration)
- `/encore_backend/db/direct_mysql_service.py` (MySQL queries)

### Frontend
- `/encore_frontend/src/services/api/authApi.ts` (API client pattern)
- `/encore_frontend/src/services/api/settingsApi.ts` (another API example)
- `/encore_frontend/src/pages/admin/EnhancedDashboard.tsx` (component to modify)
- `/encore_frontend/src/config/api.config.ts` (configuration pattern)

### Documentation
- `/docs/SUPERVISOR/ISSUES/ISSUE-001-ADMIN-DASHBOARD-MOCK-DATA.md` (issue details)
- `/docs/TESTING/MYSQL-MIGRATION-COMPLETE.md` (database patterns)

---

## ✅ DEFINITION OF DONE

### Code Complete When:
- [ ] All backend endpoints implemented and tested
- [ ] Frontend adminApi created and integrated
- [ ] Dashboard displays real data in both modes
- [ ] No TypeScript errors
- [ ] No console errors
- [ ] All tests passing (backend + frontend)

### Ready for Review When:
- [ ] Self-review completed using checklist
- [ ] Manual testing completed successfully
- [ ] Code documented (docstrings, comments)
- [ ] CHANGELOG.md updated
- [ ] Ready for supervisor review

### Accepted When:
- [ ] Supervisor review passed
- [ ] Integration tests passed
- [ ] Performance benchmarks met
- [ ] Client sign-off (if required)

---

## 📝 DELIVERABLE ARTIFACTS

### Code Files
1. `encore_backend/models/admin.py`
2. `encore_backend/services/admin_service.py`
3. `encore_backend/routes/admin.py`
4. `encore_backend/tests/test_admin_api.py`
5. `encore_frontend/src/services/api/adminApi.ts`
6. `encore_frontend/src/pages/admin/EnhancedDashboard.tsx` (modified)

### Documentation Updates
1. Update `CHANGELOG.md` with implementation details
2. Create implementation notes if needed
3. Update API documentation
4. Add to test coverage reports

---

## 🎬 GETTING STARTED

### Step-by-Step Start

1. **Read this entire prompt carefully**
2. **Review required reading files**
3. **Understand the data flow**:
   ```
   Brainium API → AdminService → /api/admin/stats → adminApi → Dashboard
   MySQL → AdminService → /api/admin/stats → adminApi → Dashboard  
   DynamoDB → AdminService → /api/admin/stats → adminApi → Dashboard
   ```
4. **Start with backend** (easier to test independently)
5. **Test backend endpoints thoroughly** before frontend work
6. **Then integrate frontend**
7. **Test end-to-end**

### Questions to Ask Before Starting

- [ ] Do I understand the data sources for each statistic?
- [ ] Have I reviewed the existing service patterns?
- [ ] Do I know how to test the endpoints?
- [ ] Am I clear on the acceptance criteria?
- [ ] Do I have access to all required documentation?

---

## 💡 HELPFUL HINTS

### Backend Development

- **Brainium API**: Already working, see `services/brainium_service.py`
- **MySQL Queries**: Use `DirectMySQLService`, see `services/mysql_service.py` 
- **DynamoDB**: Use `RestaurantDynamoDBService`, see `services/settings_service.py`
- **Parallel Queries**: Use `asyncio.gather()` for better performance
- **Caching**: Simple dict cache with TTL (5-15 min) is sufficient

### Frontend Development

- **API Client**: Copy pattern from `authApi.ts` or `settingsApi.ts`
- **Mock Integration**: Ensure mock mode still works for development
- **Error Handling**: Use logger and show user-friendly messages
- **Loading States**: Show spinners while loading
- **Type Safety**: Run `npm run type-check` frequently

### Testing

- **Backend**: Use FastAPI TestClient, see existing tests
- **Frontend**: Manual testing is sufficient for this task
- **Integration**: Test with real credentials against real APIs
- **Performance**: Check with browser dev tools (Network tab)

---

## 🎯 SUCCESS INDICATORS

You'll know you're done when:

1. **Admin Dashboard Shows Real Numbers**:
   - Venue count matches Brainium API query
   - Feedback count matches MySQL count
   - AI-enabled count is calculated from DynamoDB

2. **Both Modes Work**:
   - `REACT_APP_USE_MOCK_API=true` shows mock data
   - `REACT_APP_USE_MOCK_API=false` shows real data

3. **No Errors**:
   - Backend endpoints return 200 OK
   - Frontend console has no errors
   - TypeScript builds cleanly

4. **Good Performance**:
   - Dashboard loads in < 3 seconds
   - API responses in < 2 seconds

---

## 📞 COMMUNICATION

### Progress Updates

Please provide updates in this format:

```markdown
**Date**: YYYY-MM-DD
**Phase**: Backend / Frontend / Testing
**Status**: In Progress / Blocked / Complete
**Completed**: 
- Backend models created
- Admin service implemented

**Next Steps**:
- Create API routes
- Test endpoints

**Blockers**: None / [describe blocker]
**Questions**: [any questions for supervisor]
```

### Final Report

When complete, please provide:

```markdown
**Implementation Complete**: ISSUE-001

**What Was Built**:
- 3 backend API endpoints
- Frontend adminApi service
- Dashboard integration

**Test Results**:
- Backend tests: X/X passing
- Manual testing: ✅ All scenarios passed
- Performance: Dashboard loads in X.X seconds

**Known Issues**: None / [list any remaining issues]

**Next Steps**: Ready for supervisor review
```

---

## 🚀 YOU CAN DO THIS!

This is a straightforward task following well-established patterns. The hard work (data sources, auth, infrastructure) is already done. This is just connecting the dots!

**Key Success Factors**:
1. Follow existing patterns closely
2. Test backend thoroughly before frontend work
3. Use conditional API pattern (mock vs real)
4. Don't overthink it - it's simpler than it looks
5. Ask for help if you get stuck

**Estimated Timeline**: 1.5-2 days for focused developer

Good luck! 🎉

---

**Created**: 2025-11-23
**Issue**: ISSUE-001
**Priority**: HIGH
**Estimated Effort**: 12-18 hours
**Status**: Ready for assignment

