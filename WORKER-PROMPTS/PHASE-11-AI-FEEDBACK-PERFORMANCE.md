# Phase 11: AI Feedback Page Performance Optimization

## Worker Prompt for AI Assistant

**Task**: Optimize AI Feedback page loading performance
**Priority**: HIGH (User-reported issue)
**Estimated Effort**: 3-5 hours
**Risk Level**: MEDIUM (touches data fetching)

---

## 🎯 OBJECTIVE

Reduce AI Feedback page load time from **10+ seconds** to **< 2 seconds** through:
1. Server-side pagination and filtering
2. Virtual scrolling for large lists
3. Data fetching optimizations
4. Frontend caching

---

## 📋 PROBLEM ANALYSIS

### Current Bottlenecks

```
FRONTEND (AIFeedback.tsx)
├── ❌ Line 185: Fetches ALL reviews (limit=None), then filters
├── ❌ Line 305-309: Limit 200 but still too many
├── ❌ No pagination - renders all reviews at once
├── ❌ Lines 175-258: Pre-loads analysis for ALL reviews on mount
├── ❌ Complex render logic for each review card

BACKEND (clients.py → get_client_reviews)
├── ❌ Line 185: Fetches all data before filtering
├── ❌ Date filtering is client-side (inefficient)
├── ❌ No cursor-based pagination
├── ❌ DynamoDB full table scans

API CALL FLOW (Current)
1. Frontend: GET /clients/{id}/reviews?limit=200&days=7
2. Backend: Query ALL comments from DynamoDB → ~30,000 items
3. Backend: Filter by date → ~500 items
4. Backend: Apply limit → 200 items
5. Frontend: Render all 200 at once → SLOW
```

### Target Performance

| Metric | Current | Target |
|--------|---------|--------|
| Initial Load | 10-15s | < 2s |
| Page Render | 3-5s | < 500ms |
| Reviews per page | 200 | 25 (paginated) |
| Memory usage | High | Moderate |

---

## 🏗️ IMPLEMENTATION PLAN

### Part 1: Backend - Server-Side Date Filtering

**File**: `encore_backend/db/dynamodb_service.py`

Add efficient date-filtered query:

```python
def get_comments_for_restaurant_filtered(
    self,
    restaurant_id: str,
    days: int = None,
    start_date: datetime = None,
    end_date: datetime = None,
    limit: int = 25,
    last_evaluated_key: Dict = None
) -> Dict[str, Any]:
    """
    Get comments with server-side date filtering and pagination.
    
    Uses DynamoDB's FilterExpression for date filtering and
    pagination with LastEvaluatedKey.
    
    Args:
        restaurant_id: Restaurant UUID
        days: Filter to last N days
        start_date: Start date for range
        end_date: End date for range
        limit: Max items per page
        last_evaluated_key: Pagination cursor
    
    Returns:
        Dict with items, pagination token, and total
    """
    table = self.get_table()
    
    # Calculate date range
    if days:
        end_date = datetime.now()
        start_date = end_date - timedelta(days=days)
    
    # Build query
    key_condition = Key('PK').eq(f'RESTAURANT#{restaurant_id}') & \
                    Key('SK').begins_with('CUSTOMER#')
    
    # Add date filter if provided
    filter_expression = None
    if start_date:
        filter_expression = Attr('DateCreated').between(
            start_date.isoformat(),
            end_date.isoformat() if end_date else datetime.now().isoformat()
        )
    
    query_params = {
        'KeyConditionExpression': key_condition,
        'Limit': limit,
        'ScanIndexForward': False  # Newest first
    }
    
    if filter_expression:
        query_params['FilterExpression'] = filter_expression
    
    if last_evaluated_key:
        query_params['ExclusiveStartKey'] = last_evaluated_key
    
    response = table.query(**query_params)
    
    return {
        'items': response.get('Items', []),
        'last_evaluated_key': response.get('LastEvaluatedKey'),
        'has_more': 'LastEvaluatedKey' in response
    }
```

### Part 2: Backend - Paginated Reviews Endpoint

**File**: `encore_backend/routes/clients.py`

Add paginated endpoint (keep existing for backwards compatibility):

```python
@router.get("/{client_id}/reviews/paginated")
async def get_client_reviews_paginated(
    client_id: int,
    page_size: int = Query(25, ge=1, le=100, description="Items per page"),
    days: int = Query(7, description="Filter to last N days"),
    cursor: str = Query(None, description="Pagination cursor from previous request"),
    onboard_service: OnboardService = Depends(get_onboard_service)
):
    """
    Get paginated reviews with server-side filtering.
    
    Optimized endpoint for large datasets with:
    - Server-side date filtering
    - Cursor-based pagination
    - Efficient DynamoDB queries
    
    Returns:
        Reviews page with pagination cursor
    """
    import json
    import base64
    
    try:
        # Get restaurant
        restaurant = onboard_service.dynamo_service.get_restaurant_by_client_id(client_id)
        if not restaurant:
            raise HTTPException(status_code=404, detail="Restaurant not found")
        
        # Decode cursor if provided
        last_evaluated_key = None
        if cursor:
            try:
                last_evaluated_key = json.loads(base64.b64decode(cursor))
            except:
                raise HTTPException(status_code=400, detail="Invalid pagination cursor")
        
        # Get filtered and paginated comments
        result = onboard_service.dynamo_service.get_comments_for_restaurant_filtered(
            restaurant_id=restaurant.restaurant_id,
            days=days,
            limit=page_size,
            last_evaluated_key=last_evaluated_key
        )
        
        # Build next cursor
        next_cursor = None
        if result['last_evaluated_key']:
            next_cursor = base64.b64encode(
                json.dumps(result['last_evaluated_key']).encode()
            ).decode()
        
        # Format reviews for frontend
        reviews = []
        for item in result['items']:
            reviews.append({
                'comment_id': item.get('CommentId'),
                'customer_id': item.get('CustomerId'),
                'customer_name': item.get('CustomerName', 'Anonymous'),
                'customer_email': item.get('CustomerEmail'),
                'comment': item.get('CommentText', ''),
                'rating': item.get('Rating'),
                'date_created': item.get('DateCreated'),
                'status': item.get('Status', 'new'),
                'analyzed_at': item.get('AnalyzedAt'),
                'analysis_summary': item.get('AnalysisSummary'),
                'email_sent_at': item.get('EmailSentAt'),
                'email_sent_to': item.get('EmailSentTo'),
                'draft_id': item.get('DraftId')
            })
        
        return {
            'success': True,
            'client_id': client_id,
            'restaurant_id': restaurant.restaurant_id,
            'restaurant_name': restaurant.restaurant_name,
            'page_size': page_size,
            'reviews_count': len(reviews),
            'has_more': result['has_more'],
            'next_cursor': next_cursor,
            'reviews': reviews
        }
        
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error fetching paginated reviews: {e}")
        raise HTTPException(status_code=500, detail=str(e))
```

### Part 3: Frontend - Paginated Data Fetching

**File**: `encore_frontend/src/services/api/clientsApi.ts`

Add paginated API function:

```typescript
export interface PaginatedReviewsResponse {
  success: boolean;
  client_id: number;
  restaurant_id: string;
  restaurant_name: string;
  page_size: number;
  reviews_count: number;
  has_more: boolean;
  next_cursor: string | null;
  reviews: Review[];
}

export const getClientReviewsPaginated = async (
  clientId: number,
  options: {
    pageSize?: number;
    days?: number;
    cursor?: string | null;
  } = {}
): Promise<PaginatedReviewsResponse> => {
  const { pageSize = 25, days = 7, cursor = null } = options;
  
  const params: Record<string, any> = {
    page_size: pageSize,
    days: days
  };
  
  if (cursor) {
    params.cursor = cursor;
  }
  
  const response = await apiClient.get(
    `/clients/${clientId}/reviews/paginated`,
    { params }
  );
  return response.data;
};
```

### Part 4: Frontend - Paginated UI with "Load More"

**File**: `encore_frontend/src/pages/admin/AIFeedback.tsx`

Replace `loadReviews` with paginated version:

```tsx
// Add to state
const [cursor, setCursor] = useState<string | null>(null);
const [hasMore, setHasMore] = useState(false);
const [loadingMore, setLoadingMore] = useState(false);

// Replace loadReviews function
const loadReviews = async (clientId: number, isLoadMore = false) => {
  const filterOption = TIME_FILTER_OPTIONS.find(f => f.value === timeFilter);
  
  try {
    if (isLoadMore) {
      setLoadingMore(true);
    } else {
      setLoadingReviews(true);
      setReviews([]); // Clear existing on new load
      setCursor(null);
    }
    setReviewsError(null);
    
    const response = await clientsApi.getClientReviewsPaginated(clientId, {
      pageSize: 25,
      days: filterOption?.days || undefined,
      cursor: isLoadMore ? cursor : null
    });
    
    if (isLoadMore) {
      // Append to existing reviews
      setReviews(prev => [...prev, ...(response.reviews || [])]);
    } else {
      // Replace reviews
      setReviews(response.reviews || []);
    }
    
    setCursor(response.next_cursor);
    setHasMore(response.has_more);
    
  } catch (error: any) {
    setReviewsError(error.message || 'Failed to load reviews');
    if (!isLoadMore) setReviews([]);
  } finally {
    setLoadingReviews(false);
    setLoadingMore(false);
  }
};

// Add Load More button to UI (after reviews list)
{hasMore && !loadingReviews && (
  <div className="flex justify-center py-4">
    <button
      onClick={() => loadReviews(selectedClient!.client_id, true)}
      disabled={loadingMore}
      className="flex items-center px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:bg-gray-300"
    >
      {loadingMore ? (
        <>
          <ArrowPathIcon className="h-4 w-4 mr-2 animate-spin" />
          Loading more...
        </>
      ) : (
        <>
          <ChevronDownIcon className="h-4 w-4 mr-2" />
          Load More Reviews
        </>
      )}
    </button>
  </div>
)}
```

### Part 5: Frontend - Virtual Scrolling (Optional Enhancement)

For even better performance with large lists, add react-window:

```bash
npm install react-window
```

```tsx
// Add to AIFeedback.tsx
import { FixedSizeList as List } from 'react-window';

// Render virtualized list
<List
  height={600}
  itemCount={filteredReviews.length}
  itemSize={120} // Estimated row height
  width="100%"
>
  {({ index, style }) => (
    <div style={style}>
      {renderReviewRow(filteredReviews[index])}
    </div>
  )}
</List>
```

### Part 6: Remove Pre-loading of All Analysis (Quick Win)

**Current Problem** (Lines 175-258): Pre-loads stored analysis for ALL reviews on mount.

**Fix**: Only pre-load when review is expanded:

```tsx
// REMOVE the useEffect that pre-loads all analysis (lines 175-258)

// Instead, load analysis when a review is expanded
const toggleExpand = async (commentId: string) => {
  setExpandedReviews(prev => {
    const newSet = new Set(prev);
    if (newSet.has(commentId)) {
      newSet.delete(commentId);
    } else {
      newSet.add(commentId);
      
      // Lazy load analysis for this review if it has stored data
      const review = reviews.find(r => r.comment_id === commentId);
      if (review?.status !== 'new' && review?.analysis_summary && !analysisResults.has(commentId)) {
        // Pre-populate from stored summary
        setAnalysisResults(prev => {
          const newMap = new Map(prev);
          // Create minimal analysis from summary
          newMap.set(commentId, createPreloadedAnalysis(review));
          return newMap;
        });
      }
    }
    return newSet;
  });
};
```

---

## ✅ ACCEPTANCE CRITERIA

### Performance Targets

- [ ] Initial page load < 2 seconds
- [ ] Default shows 25 reviews (not 200)
- [ ] "Load More" loads next 25 reviews in < 1 second
- [ ] Date filtering happens server-side
- [ ] No pre-loading of analysis for all reviews

### Functional Requirements

- [ ] Paginated endpoint returns correct data
- [ ] Cursor-based pagination works correctly
- [ ] Date filtering returns accurate results
- [ ] Existing functionality preserved (analyze, generate response, send email)
- [ ] Status badges still display correctly

### Testing Scenarios

1. **Initial Load**
   - Select venue with 1000+ reviews
   - Time filter: Last 7 days
   - ✅ Page loads in < 2 seconds
   - ✅ Shows 25 reviews initially

2. **Load More**
   - Click "Load More"
   - ✅ Next 25 reviews append in < 1 second
   - ✅ Previous reviews remain in place

3. **Time Filter Change**
   - Change from 7 days to 30 days
   - ✅ List resets and loads new data
   - ✅ Load time < 2 seconds

4. **Expand Review**
   - Expand a reviewed review with stored analysis
   - ✅ Analysis loads from stored summary
   - ✅ No blocking API call

---

## 📁 FILES TO MODIFY

| File | Changes |
|------|---------|
| `encore_backend/db/dynamodb_service.py` | Add `get_comments_for_restaurant_filtered()` |
| `encore_backend/routes/clients.py` | Add paginated endpoint |
| `encore_frontend/src/services/api/clientsApi.ts` | Add paginated API function |
| `encore_frontend/src/pages/admin/AIFeedback.tsx` | Implement pagination, remove bulk preload |
| `package.json` | (Optional) Add react-window |
| `CHANGELOG.md` | Document changes |

---

## 🚀 IMPLEMENTATION ORDER

1. **Backend first** (safe, additive)
   - Add DynamoDB filtered query method
   - Add paginated endpoint (keep existing)

2. **Test backend**
   ```bash
   curl "http://localhost:8000/clients/57/reviews/paginated?page_size=25&days=7"
   curl "http://localhost:8000/clients/57/reviews/paginated?page_size=25&days=7&cursor=<token>"
   ```

3. **Frontend changes**
   - Add API function
   - Update AIFeedback.tsx
   - Remove bulk preload

4. **Test end-to-end**
   - Measure load times
   - Test pagination
   - Verify functionality

---

## ⚠️ IMPORTANT NOTES

1. **Backwards Compatibility**: Keep existing `/reviews` endpoint, add new `/reviews/paginated`
2. **DynamoDB Filtering**: `FilterExpression` still scans, but returns fewer items over network
3. **For true optimization**: Consider adding a GSI on `DateCreated` (future improvement)
4. **Quick Win**: Removing bulk analysis preload will give immediate improvement

---

## 📊 EXPECTED IMPROVEMENT

| Change | Time Saved |
|--------|------------|
| Server-side date filtering | ~3-5s |
| Pagination (25 vs 200 items) | ~2-3s |
| Remove bulk preload | ~1-2s |
| **Total** | **~6-10s** |

---

**Created**: 2025-11-28
**Author**: AI Supervisor
**Status**: Ready for implementation

