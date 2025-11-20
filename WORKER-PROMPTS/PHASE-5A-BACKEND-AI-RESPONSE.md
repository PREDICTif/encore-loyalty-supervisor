# Phase 5A: Backend AI Response Generation

## 🎯 OBJECTIVE
Implement complete backend infrastructure for AI-powered feedback response generation using AWS Bedrock (Claude 3.5 Sonnet), including API endpoints, DynamoDB storage, and prompt engineering.

## 📋 CONTEXT

### Current State (Phase 4 Complete)
- ✅ Phase 1-4 tested and working
- ✅ Feedback system operational (61,803 items accessible)
- ✅ AWS Bedrock service already configured (`services/bedrock_service.py`)
- ✅ DynamoDB infrastructure in place
- ⚠️ Customer email/name fields currently null (BUG-015, deferred)

### What Already Exists
1. **Bedrock Service** (`services/bedrock_service.py`):
   - Claude 3.5 Sonnet integration
   - Customer profiling capabilities
   - Rate limiting and error handling
   - S3 storage integration

2. **Comment Analysis** (`routes/comments.py`):
   - AI sentiment analysis
   - Alert generation logic
   - Background task processing

3. **DynamoDB Service** (`db/dynamodb_service.py`):
   - Single-table design
   - Customer/Comment entities

### What Needs to Be Built
1. **New DynamoDB Table**: `encore_feedback_responses`
2. **AI Response Service**: Dedicated service for feedback responses
3. **API Endpoints**: 3 new endpoints for generation/retrieval
4. **Prompt Engineering**: Templates for high-quality responses
5. **Testing Infrastructure**: Comprehensive test suite

## 🏗️ ARCHITECTURE

### DynamoDB Schema for AI Responses
```python
Table: encore_feedback_responses
PK (Partition Key): feedback_id
SK (Sort Key): version_timestamp (e.g., "v1_2025-11-19T10:30:00Z")

Attributes:
- feedback_id: str (from MySQL feedback)
- version_timestamp: str (sortable timestamp)
- response_text: str (generated response)
- generation_metadata: dict
  - model_id: str (e.g., "claude-3.5-sonnet")
  - prompt_version: str (e.g., "v1.0")
  - temperature: float
  - custom_instructions: str (if any)
  - generation_time_ms: int
- sentiment_analysis: dict
  - sentiment: str (positive/neutral/negative)
  - satisfaction_score: float (0-1)
  - key_issues: list[str]
- status: str (draft/approved/sent)
- created_at: str (ISO timestamp)
- created_by: str (user_id)
- approved_at: str (ISO timestamp, optional)
- approved_by: str (user_id, optional)
- sent_at: str (ISO timestamp, optional)
```

### API Endpoints

#### 1. POST /api/ai/generate-response/{feedback_id}
```python
Request Body (optional):
{
  "custom_instructions": "Be extra apologetic about the wait time",
  "temperature": 0.7,  # Optional, default 0.7
  "regenerate": false  # Set true to create new version
}

Response (201 Created):
{
  "success": true,
  "feedback_id": "12345",
  "version": "v1_2025-11-19T10:30:00Z",
  "response_text": "Dear Valued Customer...",
  "sentiment_analysis": {
    "sentiment": "negative",
    "satisfaction_score": 0.3,
    "key_issues": ["long wait", "cold food"]
  },
  "generation_time_ms": 2500,
  "status": "draft"
}
```

#### 2. POST /api/ai/regenerate/{feedback_id}
```python
Request Body:
{
  "custom_instructions": "Mention our new express service",
  "previous_version": "v1_2025-11-19T10:30:00Z"  # Optional
}

Response: Same as generate-response
```

#### 3. GET /api/ai/response/{feedback_id}
```python
Query Parameters:
- version: str (optional, defaults to latest)
- include_history: bool (optional, default false)

Response (200 OK):
{
  "feedback_id": "12345",
  "current_version": {
    "version": "v2_2025-11-19T10:45:00Z",
    "response_text": "...",
    "sentiment_analysis": {...},
    "status": "approved",
    "generation_metadata": {...}
  },
  "history": [  # Only if include_history=true
    {
      "version": "v1_2025-11-19T10:30:00Z",
      "created_at": "2025-11-19T10:30:00Z",
      "status": "superseded"
    }
  ]
}
```

## 📝 IMPLEMENTATION TASKS

### Task 1: Create DynamoDB Table and Models (1 hour)

1. **Create DynamoDB table creation script**:
   ```bash
   encore_backend/scripts/create_ai_response_tables.py
   ```

2. **Create Pydantic models**:
   ```python
   # In models/ai_response.py
   class AIResponseGenerationRequest(BaseModel)
   class AIResponseMetadata(BaseModel)
   class SentimentAnalysis(BaseModel)
   class AIResponse(BaseModel)
   class AIResponseHistory(BaseModel)
   ```

3. **Extend DynamoDB service**:
   - Add methods for response CRUD operations
   - Support versioning and history queries

### Task 2: Create AI Response Service (2-3 hours)

Create `services/ai_response_service.py`:

```python
class AIResponseService:
    def __init__(self, 
                 bedrock_service: BedrockCustomerProfileService,
                 dynamodb_service: RestaurantDynamoDBService,
                 mysql_service: MySQLService):
        self.bedrock = bedrock_service
        self.dynamodb = dynamodb_service
        self.mysql = mysql_service
    
    async def generate_response(self, feedback_id: str, 
                              custom_instructions: Optional[str] = None,
                              temperature: float = 0.7) -> AIResponse:
        # 1. Fetch feedback from MySQL
        # 2. Build context (venue settings, customer history if available)
        # 3. Create prompt using template
        # 4. Call Bedrock for generation
        # 5. Parse response and perform sentiment analysis
        # 6. Store in DynamoDB with version
        # 7. Return response
    
    def _build_prompt(self, feedback: dict, context: dict, 
                     custom_instructions: Optional[str] = None) -> str:
        # Sophisticated prompt engineering
        # Include venue context, feedback details, custom instructions
        
    async def get_response(self, feedback_id: str, 
                          version: Optional[str] = None) -> AIResponse:
        # Retrieve from DynamoDB
        
    async def get_response_history(self, feedback_id: str) -> List[AIResponse]:
        # Query all versions
```

### Task 3: Implement Prompt Engineering (2 hours)

Create sophisticated prompt templates:

```python
RESPONSE_GENERATION_PROMPT_V1 = """
You are a professional customer service representative for {venue_name}.
Your task is to craft a personalized response to customer feedback.

VENUE CONTEXT:
- Restaurant: {venue_name}
- Type: {venue_type}
- Location: {venue_location}

FEEDBACK DETAILS:
- Customer: {customer_greeting}  # "Dear Valued Customer" since we don't have names
- Rating: {rating}/5
- Comment: {comment}
- Date: {visit_date}

AI BEHAVIOR SETTINGS:
- Response Tone: {response_tone}
- Response Length: {response_length}
- Manager Name: {manager_name}
- Special Instructions: {custom_instructions}

RESPONSE GUIDELINES:
1. Acknowledge the feedback sincerely
2. Address specific concerns mentioned
3. For ratings 3 or below:
   - Apologize for the experience
   - Offer to make it right
   - Invite them back
4. For ratings 4-5:
   - Thank them enthusiastically
   - Reinforce what they enjoyed
   - Encourage return visits
5. Keep tone {response_tone} and length {response_length}
6. Sign with manager name: {manager_name}
7. DO NOT use these phrases: {prohibited_phrases}

Generate a response that feels personal and genuine:
"""
```

### Task 4: Create API Routes (1 hour)

Create `routes/ai.py`:

```python
from fastapi import APIRouter, Depends, HTTPException
from typing import Optional

router = APIRouter(prefix="/api/ai", tags=["ai"])

@router.post("/generate-response/{feedback_id}", 
            response_model=AIGenerationResponse,
            status_code=201)
async def generate_ai_response(
    feedback_id: str,
    request: Optional[AIResponseGenerationRequest] = None,
    current_user: User = Depends(get_current_user),
    ai_service: AIResponseService = Depends(get_ai_response_service)
):
    """Generate AI response for feedback"""
    # Check permissions
    # Call service
    # Return response

@router.post("/regenerate/{feedback_id}",
            response_model=AIGenerationResponse)
async def regenerate_ai_response(
    feedback_id: str,
    request: AIRegenerationRequest,
    current_user: User = Depends(get_current_user),
    ai_service: AIResponseService = Depends(get_ai_response_service)
):
    """Regenerate response with new instructions"""
    # Validate previous version exists
    # Generate new version
    # Return response

@router.get("/response/{feedback_id}",
           response_model=AIResponseDetail)
async def get_ai_response(
    feedback_id: str,
    version: Optional[str] = None,
    include_history: bool = False,
    current_user: User = Depends(get_current_user),
    ai_service: AIResponseService = Depends(get_ai_response_service)
):
    """Get AI response for feedback"""
    # Fetch response
    # Include history if requested
    # Return detailed response
```

### Task 5: Update Dependencies and Configuration (30 min)

1. **Update `dependencies.py`**:
   ```python
   def get_ai_response_service(
       bedrock: BedrockCustomerProfileService = Depends(get_bedrock_service),
       dynamodb: RestaurantDynamoDBService = Depends(get_dynamodb_service),
       mysql: MySQLService = Depends(get_mysql_service)
   ) -> AIResponseService:
       return AIResponseService(bedrock, dynamodb, mysql)
   ```

2. **Update `.env.example`**:
   ```env
   # AI Response Generation
   AI_RESPONSE_TABLE=encore_feedback_responses
   AI_TEMPERATURE_DEFAULT=0.7
   AI_MODEL_ID=anthropic.claude-3-5-sonnet-20241022-v2:0
   ```

3. **Update main.py** to include AI routes

### Task 6: Create Comprehensive Tests (2 hours)

Create `tests/test_ai_response.py`:

```python
class TestAIResponseGeneration:
    def test_generate_response_success(self):
        # Test successful generation
    
    def test_generate_response_with_custom_instructions(self):
        # Test custom instructions
    
    def test_regenerate_response(self):
        # Test regeneration creates new version
    
    def test_get_response_latest(self):
        # Test fetching latest version
    
    def test_get_response_specific_version(self):
        # Test fetching specific version
    
    def test_get_response_history(self):
        # Test history retrieval
    
    def test_sentiment_analysis_negative(self):
        # Test negative feedback handling
    
    def test_sentiment_analysis_positive(self):
        # Test positive feedback handling
    
    def test_rate_limiting(self):
        # Test concurrent request limits
    
    def test_error_handling(self):
        # Test Bedrock failures gracefully handled
```

## 🧪 TESTING REQUIREMENTS

### Unit Tests (Minimum 10)
1. ✅ Response generation success
2. ✅ Custom instructions applied
3. ✅ Version management
4. ✅ History tracking
5. ✅ Sentiment analysis accuracy
6. ✅ Error handling
7. ✅ Rate limiting
8. ✅ DynamoDB operations
9. ✅ Prompt building
10. ✅ Permission checks

### Integration Tests
1. End-to-end response generation
2. Bedrock API integration
3. DynamoDB persistence
4. MySQL data retrieval

### Manual Testing Checklist
- [ ] Generate response for negative feedback
- [ ] Generate response for positive feedback
- [ ] Regenerate with custom instructions
- [ ] View response history
- [ ] Test rate limiting
- [ ] Test error scenarios

## ⚡ PERFORMANCE REQUIREMENTS

- Response generation: < 5 seconds (target: 2-3s)
- API response time: < 500ms for retrieval
- Concurrent requests: Handle 5 simultaneous generations
- DynamoDB queries: Use efficient indexes

## 🛡️ ERROR HANDLING

1. **Bedrock Failures**: Fallback to template response
2. **Rate Limiting**: Return 429 with retry-after header
3. **Invalid Feedback ID**: Return 404
4. **DynamoDB Errors**: Log and return 500
5. **Permission Denied**: Return 403

## 📋 DELIVERABLES

1. ✅ DynamoDB table created and tested
2. ✅ AI Response Service fully implemented
3. ✅ 3 API endpoints working
4. ✅ 10+ unit tests passing
5. ✅ Integration tests passing
6. ✅ API documentation updated
7. ✅ Error handling comprehensive
8. ✅ Performance requirements met

## 🚀 SUCCESS CRITERIA

- [ ] Can generate AI response in < 5 seconds
- [ ] Response quality is professional and contextual
- [ ] Regeneration creates new versions
- [ ] History tracking works correctly
- [ ] All tests pass (minimum 10)
- [ ] Error scenarios handled gracefully
- [ ] API documentation complete

## 📝 NOTES

- **BUG-015 Workaround**: Use "Dear Valued Customer" for now since customer names are null
- **Model Selection**: Use Claude 3.5 Sonnet for best quality
- **Temperature**: Default 0.7 for balance of creativity and consistency
- **Rate Limiting**: Respect Bedrock API limits (5 concurrent)

## 🔗 REFERENCES

- Existing Bedrock service: `services/bedrock_service.py`
- DynamoDB service: `db/dynamodb_service.py`
- Settings models: `models/settings.py`
- Comment analysis: `routes/comments.py`

---

**Estimated Time**: 3-4 days
**Dependencies**: AWS Bedrock enabled, DynamoDB configured
**Next Phase**: Phase 5B (Frontend Integration)
