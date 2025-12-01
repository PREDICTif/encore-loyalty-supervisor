# PHASE 7A: Analytics Backend Endpoints

## 🎯 MISSION
Create backend API endpoints that aggregate real data from MySQL and DynamoDB to power the admin and venue dashboards/analytics pages. This replaces mock data with real, computed statistics.

## 📊 CURRENT STATE ANALYSIS

### Mock Data Currently Used
The frontend currently calls `mockApi` for all dashboard/analytics data:

**Admin Dashboard** (`AdminDashboard.tsx`, `EnhancedDashboard.tsx`):
- `mockApi.getGlobalDashboardStats()` returns:
  - `totalVenues: 12`
  - `aiEnabledVenues: 8`
  - `totalFeedback: 1247`
  - `averageResponseTime: '2.3 hours'`
- `mockApi.getResponseVolumeData(days)` - daily response counts chart

**Admin Analytics** (`GlobalAnalytics.tsx`):
- `mockApi.getSystemAnalytics()` returns:
  - `totalVenues`, `activeToday`, `totalResponses`
  - `avgAPIResponseTime`, `avgAIGenerationTime`
  - `successRate`, `errorRate`
  - `usageByTier[]`

**Venue Dashboard** (`VenueDashboard.tsx`, `VenueDashboardEnhanced.tsx`):
- `mockApi.getVenueDashboardStats(venueId)` returns:
  - `todayFeedbackCount: 23`
  - `pendingResponses: 5`
  - `averageRating: 4.2`
  - `responseRate: 92`

**Venue Analytics** (`VenueAnalytics.tsx`):
- `mockApi.getVenueAnalytics(venueId, days)` - comprehensive venue stats
- `mockApi.getAIPerformanceMetrics(venueId)` - AI usage rates
- `mockApi.getSatisfactionTrends(venueId, months)` - monthly trends
- `mockApi.getSentimentAnalysis(venueId)` - themes analysis
- `mockApi.getStaffPerformance(venueId)` - team stats
- `mockApi.getBenchmarkData(venueId)` - industry comparisons

### Real Data Sources Available

**MySQL** (`DirectMySQLService`):
- `encoree3_mobile.client` table - all venues/clients
- `encoree3_mobile.review` table - all customer reviews with:
  - `rating`, `comment`, `date_entered`, `date_replied`
  - `customer_name`, `customer_email`, `customer_phone`
- Existing methods:
  - `get_all_clients()` - list all clients
  - `get_reviews_by_client(client_id)` - reviews for a venue
  - `get_review_count_since(client_id, timestamp)` - count since date

**DynamoDB** (Multiple tables):
- `restaurant_data` - Restaurant/Customer/Comment entities
  - Comment status: 'new', 'analyzed', 'draft_created', 'email_sent'
  - `analyzed_at`, `email_sent_at` timestamps
- `encore_draft_responses` - Draft responses with status
- `encore_venue_settings` - Venue AI settings

**Existing Endpoints**:
- `GET /clients/` - lists all clients with review counts
- `GET /api/sync/status` - venues with sync stats
- `GET /onboard/clients` - clients with onboarding status
- `GET /clients/{id}/reviews` - reviews for a venue

## ⚙️ IMPLEMENTATION PLAN

### 1. New Analytics Service (`services/analytics_service.py`)

```python
class AnalyticsService:
    """
    Service for computing analytics from MySQL and DynamoDB data.
    All queries are read-only and optimized for dashboard display.
    """
    
    def __init__(self):
        self.mysql = DirectMySQLService()
        self.dynamo = RestaurantDynamoDBService()
    
    # === GLOBAL ADMIN STATS ===
    
    def get_global_dashboard_stats(self) -> Dict[str, Any]:
        """
        Stats for admin dashboard cards.
        Returns:
            {
                "total_venues": int,          # All clients in MySQL
                "ai_enabled_venues": int,     # Onboarded in DynamoDB
                "total_feedback": int,        # Total reviews across all venues
                "total_feedback_today": int,  # Reviews entered today
                "avg_rating": float,          # Average rating across all
                "emails_sent_today": int,     # Emails sent today
                "drafts_pending": int,        # Pending review drafts
            }
        """
        pass
    
    def get_response_volume_data(self, days: int = 7) -> List[Dict]:
        """
        Daily response volume for charts.
        Returns: [{"date": "2025-11-28", "value": 45, "label": "Fri"}, ...]
        """
        pass
    
    def get_system_analytics(self) -> Dict[str, Any]:
        """
        Comprehensive system-wide analytics for GlobalAnalytics page.
        Returns:
            {
                "total_venues": int,
                "active_venues": int,           # Onboarded + has recent activity
                "total_responses": int,         # Total reviews
                "avg_response_time_ms": int,    # Can compute from analysis timestamps
                "success_rate": float,          # Analyzed / Total
                "error_rate": float,            # Failed analyses / Total
                "venues_by_status": {
                    "onboarded": int,
                    "not_onboarded": int,
                },
                "top_venues": [                 # By feedback volume
                    {"name": str, "count": int}, ...
                ]
            }
        """
        pass
    
    # === VENUE-SPECIFIC STATS ===
    
    def get_venue_dashboard_stats(self, client_id: int) -> Dict[str, Any]:
        """
        Stats for venue dashboard cards.
        Returns:
            {
                "today_feedback_count": int,
                "pending_responses": int,       # Drafts pending review
                "average_rating": float,
                "total_feedback": int,
                "analyzed_count": int,
                "emails_sent_count": int,
            }
        """
        pass
    
    def get_venue_analytics(self, client_id: int, days: int = 30) -> Dict[str, Any]:
        """
        Comprehensive venue analytics.
        Returns:
            {
                "metrics": {
                    "response_rate": float,
                    "avg_response_time_min": int,
                    "ai_accuracy": float,       # Not computable now - placeholder
                    "satisfaction": float,      # Average rating
                },
                "rating_distribution": {1: int, 2: int, 3: int, 4: int, 5: int},
                "feedback_by_day": [{"date": str, "count": int}, ...],
                "status_breakdown": {
                    "new": int,
                    "analyzed": int,
                    "draft_created": int,
                    "email_sent": int,
                },
                "alert_levels": {
                    "critical": int,
                    "high": int,
                    "medium": int,
                    "low": int,
                    "none": int,
                }
            }
        """
        pass
```

### 2. New Analytics Routes (`routes/analytics.py`)

```python
router = APIRouter(prefix="/api/analytics", tags=["analytics"])

@router.get("/global/dashboard")
async def get_global_dashboard():
    """Global admin dashboard stats"""
    pass

@router.get("/global/system")
async def get_system_analytics():
    """Comprehensive system analytics"""
    pass

@router.get("/global/volume")
async def get_response_volume(days: int = 7):
    """Daily response volume for charts"""
    pass

@router.get("/venue/{client_id}/dashboard")
async def get_venue_dashboard(client_id: int):
    """Venue-specific dashboard stats"""
    pass

@router.get("/venue/{client_id}/analytics")
async def get_venue_analytics(client_id: int, days: int = 30):
    """Comprehensive venue analytics"""
    pass
```

### 3. Required Database Methods to Add

**`DirectMySQLService`** (if not already present):
```python
def get_reviews_since(self, client_id: int, since: datetime) -> List[Dict]:
    """Get reviews for a client since a specific date"""
    pass

def get_reviews_today(self, client_id: int = None) -> List[Dict]:
    """Get today's reviews, optionally filtered by client"""
    pass

def get_review_stats_aggregate(self) -> Dict[str, Any]:
    """Aggregate stats across all reviews"""
    pass

def get_rating_distribution(self, client_id: int, days: int = 30) -> Dict[int, int]:
    """Get rating distribution for a venue"""
    pass

def get_reviews_by_date_range(self, client_id: int, start: datetime, end: datetime) -> List[Dict]:
    """Get reviews within a date range"""
    pass
```

**`RestaurantDynamoDBService`** (if not already present):
```python
def count_comments_by_status(self, restaurant_id: str = None) -> Dict[str, int]:
    """Count comments grouped by status"""
    pass

def count_emails_sent_today(self, restaurant_id: str = None) -> int:
    """Count emails sent today"""
    pass

def get_alert_level_distribution(self, restaurant_id: str) -> Dict[str, int]:
    """Count comments by alert level"""
    pass
```

**`DraftResponseService`**:
```python
def count_pending_drafts(self, client_id: int = None) -> int:
    """Count drafts with status='pending_review'"""
    pass
```

## 📁 FILES TO CREATE/MODIFY

### New Files
- `encore_backend/services/analytics_service.py` - Core analytics service
- `encore_backend/routes/analytics.py` - API endpoints

### Modified Files
- `encore_backend/main.py` - Register analytics router
- `encore_backend/db/direct_mysql_service.py` - Add aggregate query methods
- `encore_backend/db/dynamodb_service.py` - Add count/aggregate methods
- `encore_backend/services/draft_response_service.py` - Add count methods

## ✅ ACCEPTANCE CRITERIA

### Backend Endpoints Working
- [ ] `GET /api/analytics/global/dashboard` returns real stats
- [ ] `GET /api/analytics/global/system` returns system-wide analytics
- [ ] `GET /api/analytics/global/volume?days=7` returns daily counts
- [ ] `GET /api/analytics/venue/{client_id}/dashboard` returns venue stats
- [ ] `GET /api/analytics/venue/{client_id}/analytics?days=30` returns detailed analytics

### Data Accuracy
- [ ] Total venues count matches MySQL `client` table
- [ ] AI-enabled count matches DynamoDB onboarded restaurants
- [ ] Today's feedback count is accurate
- [ ] Rating distribution matches actual reviews
- [ ] Pending drafts count matches DraftResponseService

### Performance
- [ ] Global dashboard loads in < 2 seconds
- [ ] Venue dashboard loads in < 1 second
- [ ] Queries are optimized (use COUNT, aggregates where possible)

## 🧪 TESTING

```bash
# Test global dashboard
curl http://localhost:8000/api/analytics/global/dashboard | python3 -m json.tool

# Test system analytics
curl http://localhost:8000/api/analytics/global/system | python3 -m json.tool

# Test volume data
curl "http://localhost:8000/api/analytics/global/volume?days=7" | python3 -m json.tool

# Test venue dashboard (client_id=57 for Opus Ocean Grille)
curl http://localhost:8000/api/analytics/venue/57/dashboard | python3 -m json.tool

# Test venue analytics
curl "http://localhost:8000/api/analytics/venue/57/analytics?days=30" | python3 -m json.tool
```

## ⏱️ ESTIMATED EFFORT
- Analytics Service: 3-4 hours
- Database Methods: 2-3 hours
- API Routes: 1 hour
- Testing: 1-2 hours
- **Total: 7-10 hours**

## 📚 REFERENCES
- `encore_backend/db/direct_mysql_service.py` - MySQL connection
- `encore_backend/db/dynamodb_service.py` - DynamoDB operations
- `encore_backend/services/draft_response_service.py` - Draft operations
- `encore_frontend/src/services/mockApi.ts` - Expected data structures (lines 28-50, 483-500, 1124-1160, 1318-1340)

## ⚠️ IMPORTANT NOTES

1. **Read-Only MySQL**: Never write to MySQL - read-only access
2. **Performance**: Use SQL aggregates (COUNT, AVG, SUM) instead of fetching all rows
3. **Date Handling**: MySQL dates are in the legacy system's timezone
4. **Placeholders OK**: Some metrics (like "AI Accuracy") can't be computed yet - use reasonable placeholders
5. **Backwards Compatible**: Return similar structure to mock data so frontend changes are minimal

