# PHASE 7B: Analytics Frontend Integration

## 🎯 MISSION
Replace mock API calls in all dashboard and analytics pages with real API calls to the new analytics endpoints created in Phase 7A.

## ⚠️ PREREQUISITE
**Phase 7A must be completed first.** The backend endpoints at `/api/analytics/*` must be working.

## 📊 PAGES TO UPDATE

### 1. Admin Dashboard Pages

#### `AdminDashboard.tsx`
**Current**: `mockApi.getGlobalDashboardStats()`
**Replace with**: `GET /api/analytics/global/dashboard`

**Changes needed**:
```typescript
// Before
import { mockApi } from '../services/mockApi';
const data = await mockApi.getGlobalDashboardStats();

// After
import { analyticsApi } from '../services/api/analyticsApi';
const data = await analyticsApi.getGlobalDashboard();
```

#### `EnhancedDashboard.tsx`
**Current**: 
- `mockApi.getGlobalDashboardStats()`
- `mockApi.getResponseVolumeData(chartPeriod)`

**Replace with**:
- `GET /api/analytics/global/dashboard`
- `GET /api/analytics/global/volume?days={chartPeriod}`

**Additional changes**:
- "Top Venues by Activity" section - use data from `/api/analytics/global/system`
- "Venue Distribution" section - use real onboarded vs not-onboarded counts
- "Response Performance" section - use real success rate

#### `GlobalAnalytics.tsx`
**Current**: `mockApi.getSystemAnalytics()`
**Replace with**: `GET /api/analytics/global/system`

**Changes needed**:
- Update metric cards to use real data
- "Usage by Tier" section may need placeholder or removal (no tier system currently)
- "System Health Metrics" - use available data, placeholder for unavailable

### 2. Venue Dashboard Pages

#### `VenueDashboard.tsx`
**Current**: `mockApi.getVenueDashboardStats(venueId)`
**Replace with**: `GET /api/analytics/venue/{client_id}/dashboard`

**Note**: This is a simpler dashboard - may be deprecated in favor of `VenueDashboardEnhanced.tsx`

#### `VenueDashboardEnhanced.tsx`
**Current**: Uses `VenueContext.dashboardStats` which calls mock API
**Replace with**: Direct API call to `/api/analytics/venue/{client_id}/dashboard`

**Additional changes**:
- "Sent Today" card - use `emails_sent_count` from API
- Recent Alerts section - consider using `/comments/restaurant/{client_id}/management-alerts`

### 3. Venue Analytics Page

#### `VenueAnalytics.tsx`
**Current**: Multiple mock API calls:
- `mockApi.getVenueAnalytics(venueId, dateRange)`
- `mockApi.getAIPerformanceMetrics(venueId)`
- `mockApi.getSatisfactionTrends(venueId, 12)`
- `mockApi.getSentimentAnalysis(venueId)`
- `mockApi.getStaffPerformance(venueId)`
- `mockApi.getBenchmarkData(venueId)`

**Replace with**: `GET /api/analytics/venue/{client_id}/analytics?days={dateRange}`

**Note**: Not all sections have real data equivalents. Some sections may need:
- **AI Performance Metrics**: Use `status_breakdown` for usage approximation
- **Sentiment Analysis**: May need placeholder or future AI enhancement
- **Staff Performance**: No data available - hide or remove section
- **Industry Benchmarks**: No data available - hide or remove section

## 📁 NEW FILE TO CREATE

### `services/api/analyticsApi.ts`

```typescript
import { apiClient } from '../apiClient';

// === Types ===

export interface GlobalDashboardStats {
    total_venues: number;
    ai_enabled_venues: number;
    total_feedback: number;
    total_feedback_today: number;
    avg_rating: number;
    emails_sent_today: number;
    drafts_pending: number;
}

export interface SystemAnalytics {
    total_venues: number;
    active_venues: number;
    total_responses: number;
    avg_response_time_ms: number;
    success_rate: number;
    error_rate: number;
    venues_by_status: {
        onboarded: number;
        not_onboarded: number;
    };
    top_venues: Array<{ name: string; count: number }>;
}

export interface VolumeDataPoint {
    date: string;
    value: number;
    label: string;
}

export interface VenueDashboardStats {
    today_feedback_count: number;
    pending_responses: number;
    average_rating: number;
    total_feedback: number;
    analyzed_count: number;
    emails_sent_count: number;
}

export interface VenueAnalytics {
    metrics: {
        response_rate: number;
        avg_response_time_min: number;
        ai_accuracy: number;
        satisfaction: number;
    };
    rating_distribution: Record<number, number>;
    feedback_by_day: Array<{ date: string; count: number }>;
    status_breakdown: {
        new: number;
        analyzed: number;
        draft_created: number;
        email_sent: number;
    };
    alert_levels: {
        critical: number;
        high: number;
        medium: number;
        low: number;
        none: number;
    };
}

// === API Functions ===

export const getGlobalDashboard = async (): Promise<GlobalDashboardStats> => {
    const response = await apiClient.get('/api/analytics/global/dashboard');
    return response.data;
};

export const getSystemAnalytics = async (): Promise<SystemAnalytics> => {
    const response = await apiClient.get('/api/analytics/global/system');
    return response.data;
};

export const getVolumeData = async (days: number = 7): Promise<VolumeDataPoint[]> => {
    const response = await apiClient.get(`/api/analytics/global/volume?days=${days}`);
    return response.data;
};

export const getVenueDashboard = async (clientId: number): Promise<VenueDashboardStats> => {
    const response = await apiClient.get(`/api/analytics/venue/${clientId}/dashboard`);
    return response.data;
};

export const getVenueAnalytics = async (clientId: number, days: number = 30): Promise<VenueAnalytics> => {
    const response = await apiClient.get(`/api/analytics/venue/${clientId}/analytics?days=${days}`);
    return response.data;
};

export default {
    getGlobalDashboard,
    getSystemAnalytics,
    getVolumeData,
    getVenueDashboard,
    getVenueAnalytics,
};
```

## 🔄 UPDATE STRATEGY

### Step 1: Create Analytics API Service
Create `analyticsApi.ts` with all type definitions and API functions.

### Step 2: Update Admin Dashboard (`EnhancedDashboard.tsx`)
This is the main admin dashboard. Changes:
1. Import `analyticsApi` instead of `mockApi`
2. Update `loadData()` to use new endpoints
3. Map response fields to existing UI (may need minor adjustments)
4. Update "Top Venues" to use real data
5. Keep VenueImpersonation component unchanged

### Step 3: Update Global Analytics (`GlobalAnalytics.tsx`)
1. Import `analyticsApi`
2. Replace `mockApi.getSystemAnalytics()` call
3. Update UI to handle new data structure
4. **Handle missing sections gracefully**:
   - "Usage by Tier" - hide or show simplified version
   - Keep Custom Report Builder UI (it's mostly UI, can be made functional later)

### Step 4: Update Venue Dashboard (`VenueDashboardEnhanced.tsx`)
1. Import `analyticsApi`
2. Replace VenueContext usage with direct API call
3. Get `client_id` from user context
4. Map response to existing stat cards

### Step 5: Update Venue Analytics (`VenueAnalytics.tsx`)
1. Import `analyticsApi`
2. Replace all mock API calls with single `getVenueAnalytics()`
3. **Simplify sections without data**:
   - Remove or placeholder: Staff Performance, Industry Benchmarks
   - Simplify: Sentiment Analysis (use alert levels as proxy)
   - Keep: Rating Distribution, Response metrics, Alert breakdown

## ✅ ACCEPTANCE CRITERIA

### Admin Pages
- [ ] `EnhancedDashboard` shows real venue counts
- [ ] Total feedback count is accurate
- [ ] Response volume chart shows real data
- [ ] Top venues list shows actual top venues by review count
- [ ] No errors in console

### Venue Pages
- [ ] `VenueDashboardEnhanced` shows real stats for logged-in venue
- [ ] Today's feedback count is accurate
- [ ] Average rating is accurate
- [ ] Works for impersonated venues

### Analytics Page
- [ ] Rating distribution chart shows real data
- [ ] Status breakdown shows real counts
- [ ] Unavailable sections are hidden or show appropriate message

### No Regressions
- [ ] Login still works
- [ ] Navigation still works
- [ ] Impersonation still works
- [ ] Other pages unaffected

## 🧪 TESTING

1. **Login as Global Admin** (admin@encore.com / password123)
2. **Check Admin Dashboard**:
   - Verify venue counts match `/onboard/clients`
   - Verify chart loads without errors
   - Check console for API calls
3. **Check Global Analytics**:
   - Verify metrics cards have real values
   - Verify no console errors
4. **Impersonate a Venue** (e.g., Opus Ocean Grille)
5. **Check Venue Dashboard**:
   - Verify today's count, rating, etc.
6. **Check Venue Analytics**:
   - Verify rating distribution chart
   - Verify status breakdown

## ⏱️ ESTIMATED EFFORT
- Analytics API Service: 30 minutes
- Admin Dashboard: 1-2 hours
- Global Analytics: 1-2 hours
- Venue Dashboard: 1 hour
- Venue Analytics: 2-3 hours (most changes needed)
- Testing: 1 hour
- **Total: 6-9 hours**

## 📚 REFERENCES
- Phase 7A worker prompt (for endpoint contracts)
- `encore_frontend/src/services/mockApi.ts` (current data structures)
- `encore_frontend/src/services/apiClient.ts` (how to make API calls)
- `encore_frontend/src/pages/admin/EnhancedDashboard.tsx` (main admin dashboard)
- `encore_frontend/src/pages/venue/VenueAnalytics.tsx` (most complex page)

## ⚠️ IMPORTANT NOTES

1. **Graceful Degradation**: If an API call fails, show error message, don't crash
2. **Loading States**: Keep existing loading spinners
3. **Remove Unused Sections**: Better to hide than show fake data
4. **Keep Mock Export**: Keep mockApi for reference, don't delete yet
5. **Type Safety**: Ensure TypeScript types match backend response

