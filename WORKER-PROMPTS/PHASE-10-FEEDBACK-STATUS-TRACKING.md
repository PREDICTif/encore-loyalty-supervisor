# Phase 10: Feedback Status Tracking & Persistence

## Worker Prompt for AI Assistant

**Task**: Persist AI analysis and email status on feedback entries
**Priority**: HIGH (Critical UX issue)
**Estimated Effort**: 4-6 hours
**Dependencies**: Phase 9 complete (Sync Job)

---

## 🎯 OBJECTIVE

Make feedback analysis and email status **persistent** so users can:
1. See which feedback entries have been analyzed (even after page reload)
2. See which feedback entries have had emails sent
3. View history of actions taken on each feedback entry

---

## 📋 PROBLEM STATEMENT

### Current Issue

```
User analyzes feedback on AI Feedback page
       ↓
Analysis stored in React state (useState)
       ↓
User navigates away or refreshes
       ↓
❌ ALL ANALYSIS DATA LOST

User sends email for feedback
       ↓
Email sent via SES
       ↓
❌ NO RECORD - Can't see what happened
```

### User Expectation

- When I analyze feedback, I should see it's analyzed even after leaving the page
- When I send an email, I should see "Email sent on Nov 28" on that feedback
- I should see status indicators: ⏳ Pending, 🔍 Analyzed, 📧 Email Sent

---

## 🏗️ IMPLEMENTATION PLAN

### Part 1: Backend - Add Status Fields to Comment Model

**File**: `encore_backend/db/models.py`

Add these fields to the `Comment` dataclass:

```python
@dataclass
class Comment:
    # ... existing fields ...
    
    # NEW: Status tracking fields
    status: str = 'new'  # 'new' | 'analyzed' | 'draft_created' | 'email_sent'
    analyzed_at: Optional[datetime] = None
    analysis_summary: Optional[str] = None  # JSON string with key insights
    email_sent_at: Optional[datetime] = None
    email_sent_to: Optional[str] = None
    draft_id: Optional[str] = None  # Link to draft if exists
    
    # Ensure these are included in to_dynamodb_item()
```

**Update `to_dynamodb_item()` method** to include new fields:

```python
def to_dynamodb_item(self) -> Dict[str, Any]:
    # ... existing code ...
    
    # Add new status fields
    if self.status:
        item['Status'] = self.status
    if self.analyzed_at:
        item['AnalyzedAt'] = self.analyzed_at.isoformat()
    if self.analysis_summary:
        item['AnalysisSummary'] = self.analysis_summary
    if self.email_sent_at:
        item['EmailSentAt'] = self.email_sent_at.isoformat()
    if self.email_sent_to:
        item['EmailSentTo'] = self.email_sent_to
    if self.draft_id:
        item['DraftId'] = self.draft_id
    
    return item
```

**Add `from_dynamodb_item()` class method** to parse these fields:

```python
@classmethod
def from_dynamodb_item(cls, item: Dict[str, Any]) -> 'Comment':
    """Create Comment from DynamoDB item."""
    return cls(
        comment_id=item.get('CommentId', ''),
        customer_id=item.get('CustomerId', ''),
        restaurant_id=item.get('RestaurantId', ''),
        comment_text=item.get('CommentText', ''),
        rating=item.get('Rating'),
        date_created=datetime.fromisoformat(item['DateCreated']) if item.get('DateCreated') else None,
        date_replied=datetime.fromisoformat(item['DateReplied']) if item.get('DateReplied') else None,
        reply_text=item.get('ReplyText'),
        device_info=item.get('DeviceInfo'),
        session_info=item.get('SessionInfo'),
        original_review_id=item.get('OriginalReviewId'),
        # New fields
        status=item.get('Status', 'new'),
        analyzed_at=datetime.fromisoformat(item['AnalyzedAt']) if item.get('AnalyzedAt') else None,
        analysis_summary=item.get('AnalysisSummary'),
        email_sent_at=datetime.fromisoformat(item['EmailSentAt']) if item.get('EmailSentAt') else None,
        email_sent_to=item.get('EmailSentTo'),
        draft_id=item.get('DraftId')
    )
```

---

### Part 2: Backend - Add DynamoDB Update Methods

**File**: `encore_backend/db/dynamodb_service.py`

Add methods to update comment status:

```python
def update_comment_status(
    self,
    restaurant_id: str,
    customer_id: str,
    comment_id: str,
    status: str,
    **kwargs
) -> bool:
    """
    Update comment status and tracking fields.
    
    Args:
        restaurant_id: Restaurant UUID
        customer_id: Customer UUID
        comment_id: Comment UUID
        status: New status ('analyzed', 'draft_created', 'email_sent')
        **kwargs: Additional fields to update (analyzed_at, email_sent_at, etc.)
    
    Returns:
        bool: Success status
    """
    table = self.get_table()
    
    update_expr = "SET #status = :status"
    expr_names = {'#status': 'Status'}
    expr_values = {':status': status}
    
    if kwargs.get('analyzed_at'):
        update_expr += ", AnalyzedAt = :analyzed_at"
        expr_values[':analyzed_at'] = kwargs['analyzed_at'].isoformat()
    
    if kwargs.get('analysis_summary'):
        update_expr += ", AnalysisSummary = :analysis_summary"
        expr_values[':analysis_summary'] = kwargs['analysis_summary']
    
    if kwargs.get('email_sent_at'):
        update_expr += ", EmailSentAt = :email_sent_at"
        expr_values[':email_sent_at'] = kwargs['email_sent_at'].isoformat()
    
    if kwargs.get('email_sent_to'):
        update_expr += ", EmailSentTo = :email_sent_to"
        expr_values[':email_sent_to'] = kwargs['email_sent_to']
    
    if kwargs.get('draft_id'):
        update_expr += ", DraftId = :draft_id"
        expr_values[':draft_id'] = kwargs['draft_id']
    
    try:
        table.update_item(
            Key={
                'PK': f'RESTAURANT#{restaurant_id}',
                'SK': f'CUSTOMER#{customer_id}#COMMENT#{comment_id}'
            },
            UpdateExpression=update_expr,
            ExpressionAttributeNames=expr_names,
            ExpressionAttributeValues=expr_values
        )
        return True
    except Exception as e:
        logger.error(f"Failed to update comment status: {e}")
        return False


def get_comment_by_id(
    self,
    restaurant_id: str,
    customer_id: str,
    comment_id: str
) -> Optional[Comment]:
    """
    Get a specific comment by its IDs.
    
    Args:
        restaurant_id: Restaurant UUID
        customer_id: Customer UUID
        comment_id: Comment UUID
    
    Returns:
        Comment object or None if not found
    """
    table = self.get_table()
    
    try:
        response = table.get_item(
            Key={
                'PK': f'RESTAURANT#{restaurant_id}',
                'SK': f'CUSTOMER#{customer_id}#COMMENT#{comment_id}'
            }
        )
        
        if 'Item' in response:
            return Comment.from_dynamodb_item(response['Item'])
        return None
    except Exception as e:
        logger.error(f"Failed to get comment: {e}")
        return None
```

---

### Part 3: Backend - Update Analyze Endpoint to Persist

**File**: `encore_backend/routes/comments.py`

Modify the `analyze_existing_comment` endpoint to save analysis status:

```python
@router.post("/analyze-comment")
async def analyze_existing_comment(
    request: AnalyzeExistingCommentRequest,
    onboard_service: OnboardService = Depends(get_onboard_service),
    bedrock_service: BedrockCustomerProfileService = Depends(get_bedrock_service)
):
    """
    Analyze an existing comment and PERSIST the analysis status.
    
    This endpoint now saves the analysis result to DynamoDB so it
    persists across page reloads.
    """
    # ... existing analysis code ...
    
    # After successful analysis, update comment status
    if result.get('success'):
        try:
            # Create analysis summary (compact version for storage)
            analysis_summary = json.dumps({
                'sentiment': result.get('analysis', {}).get('comment_analysis', {}).get('sentiment'),
                'urgency': result.get('alerts', {}).get('urgency_level'),
                'key_topics': result.get('analysis', {}).get('comment_analysis', {}).get('key_topics', [])[:3],
                'alert_required': result.get('alerts', {}).get('alert_required', False)
            })
            
            # Update comment in DynamoDB
            onboard_service.dynamo_service.update_comment_status(
                restaurant_id=request.restaurant_id,
                customer_id=request.customer_id,
                comment_id=request.comment_id,
                status='analyzed',
                analyzed_at=datetime.now(),
                analysis_summary=analysis_summary
            )
            
            # Include status in response
            result['status_updated'] = True
            
        except Exception as status_error:
            logger.warning(f"Failed to update comment status: {status_error}")
            result['status_updated'] = False
    
    return result
```

---

### Part 4: Backend - Update Email Sending to Track

**File**: `encore_backend/routes/comments.py`

In the `send_draft_response` endpoint, update comment status when email is sent:

```python
@router.post("/draft-response/{draft_id}/send")
async def send_draft_response(
    draft_id: str,
    # ... existing params ...
):
    """Send an approved draft response via email."""
    
    # ... existing email sending code ...
    
    # After successful email send, update comment status
    if email_result.get('success'):
        try:
            # Get draft details to find comment
            draft = draft_service.get_draft(draft_id)
            
            if draft:
                onboard_service.dynamo_service.update_comment_status(
                    restaurant_id=draft['restaurant_id'],
                    customer_id=draft['customer_id'],
                    comment_id=draft['comment_id'],
                    status='email_sent',
                    email_sent_at=datetime.now(),
                    email_sent_to=draft['customer_email']
                )
                
        except Exception as status_error:
            logger.warning(f"Failed to update comment email status: {status_error}")
    
    return email_result
```

---

### Part 5: Backend - Include Status in Reviews Response

**File**: `encore_backend/services/onboard_service.py`

Update `get_client_reviews_new_structure` to include status fields:

```python
def get_client_reviews_new_structure(self, client_id: int, limit: int = None) -> Dict[str, Any]:
    """Get reviews with status information."""
    
    # ... existing code to get comments ...
    
    # Ensure status fields are included in response
    for review in reviews:
        review['status'] = review.get('status', 'new')
        review['analyzed_at'] = review.get('analyzed_at')
        review['analysis_summary'] = review.get('analysis_summary')
        review['email_sent_at'] = review.get('email_sent_at')
        review['email_sent_to'] = review.get('email_sent_to')
        review['draft_id'] = review.get('draft_id')
    
    return result
```

---

### Part 6: Frontend - Update Types

**File**: `encore_frontend/src/services/api/clientsApi.ts`

Add status fields to Review type:

```typescript
export interface Review {
  // ... existing fields ...
  
  // NEW: Status tracking fields
  status: 'new' | 'analyzed' | 'draft_created' | 'email_sent';
  analyzed_at: string | null;
  analysis_summary: {
    sentiment: string | null;
    urgency: string | null;
    key_topics: string[];
    alert_required: boolean;
  } | null;
  email_sent_at: string | null;
  email_sent_to: string | null;
  draft_id: string | null;
}
```

---

### Part 7: Frontend - Show Status Indicators

**File**: `encore_frontend/src/pages/admin/AIFeedback.tsx`

Add status badges to the review list:

```tsx
// Helper function to render status badge
const renderStatusBadge = (review: Review) => {
  if (review.status === 'email_sent') {
    return (
      <span className="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium bg-green-100 text-green-800">
        <CheckCircleIcon className="h-3 w-3 mr-1" />
        Email Sent {review.email_sent_at ? formatRelativeTime(review.email_sent_at) : ''}
      </span>
    );
  }
  
  if (review.status === 'draft_created') {
    return (
      <span className="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium bg-yellow-100 text-yellow-800">
        <DocumentTextIcon className="h-3 w-3 mr-1" />
        Draft Pending
      </span>
    );
  }
  
  if (review.status === 'analyzed') {
    return (
      <span className="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium bg-blue-100 text-blue-800">
        <MagnifyingGlassIcon className="h-3 w-3 mr-1" />
        Analyzed {review.analyzed_at ? formatRelativeTime(review.analyzed_at) : ''}
      </span>
    );
  }
  
  return (
    <span className="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium bg-gray-100 text-gray-600">
      <ClockIcon className="h-3 w-3 mr-1" />
      New
    </span>
  );
};

// Use in the review row
<div className="flex items-center space-x-2">
  {renderStatusBadge(review)}
  {/* ... rest of review content ... */}
</div>
```

---

### Part 8: Frontend - Pre-load Analysis if Available

When a review has `analysis_summary`, pre-populate the analysis display:

```tsx
// In useEffect or when reviews load
useEffect(() => {
  // Pre-load analysis results from stored data
  const preloadedAnalysis = new Map<string, AIAnalysisResult>();
  
  reviews.forEach(review => {
    if (review.status === 'analyzed' && review.analysis_summary) {
      try {
        const summary = typeof review.analysis_summary === 'string' 
          ? JSON.parse(review.analysis_summary) 
          : review.analysis_summary;
        
        // Create a minimal AIAnalysisResult from stored summary
        preloadedAnalysis.set(review.comment_id, {
          success: true,
          analysis: {
            comment_analysis: {
              sentiment: summary.sentiment,
              key_topics: summary.key_topics || []
            }
          },
          alerts: {
            urgency_level: summary.urgency,
            alert_required: summary.alert_required
          },
          // Mark as loaded from storage (not fresh analysis)
          loaded_from_storage: true
        });
      } catch (e) {
        console.warn('Failed to parse stored analysis:', e);
      }
    }
  });
  
  if (preloadedAnalysis.size > 0) {
    setAnalysisResults(prev => new Map([...prev, ...preloadedAnalysis]));
  }
}, [reviews]);
```

---

## ✅ ACCEPTANCE CRITERIA

### Functional Requirements

- [ ] Analyzing a comment updates its status to 'analyzed' in DynamoDB
- [ ] Status persists after page reload
- [ ] Sending email updates status to 'email_sent' with timestamp
- [ ] Review list shows status badges (New, Analyzed, Email Sent)
- [ ] Previously analyzed comments show analysis summary without re-analyzing
- [ ] Email history is visible (when sent, to whom)

### Technical Requirements

- [ ] Comment model has new status fields
- [ ] DynamoDB update methods work correctly
- [ ] Frontend types match backend response
- [ ] No breaking changes to existing functionality
- [ ] Error handling for status update failures (non-blocking)

### Testing Scenarios

1. **Analysis Persistence**
   - Analyze a comment
   - Refresh page
   - ✅ Comment should show "Analyzed" badge
   - ✅ Analysis summary should be visible

2. **Email Tracking**
   - Send email for a comment
   - Refresh page
   - ✅ Comment should show "Email Sent" badge with timestamp

3. **New Comments**
   - View a never-analyzed comment
   - ✅ Should show "New" badge

4. **Error Handling**
   - If status update fails, analysis should still work
   - ✅ Non-blocking - just log warning

---

## 📁 FILES TO MODIFY

| File | Changes |
|------|---------|
| `encore_backend/db/models.py` | Add status fields to Comment, update to_dynamodb_item(), add from_dynamodb_item() |
| `encore_backend/db/dynamodb_service.py` | Add update_comment_status(), get_comment_by_id() |
| `encore_backend/routes/comments.py` | Update analyze_existing_comment to persist, update send_draft_response to track |
| `encore_backend/services/onboard_service.py` | Include status in reviews response |
| `encore_frontend/src/services/api/clientsApi.ts` | Add status fields to Review interface |
| `encore_frontend/src/pages/admin/AIFeedback.tsx` | Add status badges, pre-load stored analysis |
| `CHANGELOG.md` | Document changes |

---

## 🚀 GETTING STARTED

1. **Start with backend models** - Add fields to Comment class
2. **Add DynamoDB methods** - update_comment_status, get_comment_by_id
3. **Update endpoints** - Persist analysis and email status
4. **Test backend** - Verify status updates via curl
5. **Update frontend** - Add types and UI components
6. **Test end-to-end** - Full workflow in browser

---

## ⚠️ IMPORTANT NOTES

1. **Non-breaking change** - Existing comments without status fields should work (default to 'new')
2. **Status updates are non-blocking** - If update fails, main operation should still succeed
3. **Analysis summary is compact** - Only store key fields, not full analysis
4. **Backwards compatible** - Old comments still display, just show as 'new'

---

**Created**: 2025-11-28
**Author**: AI Supervisor
**Status**: Ready for implementation

