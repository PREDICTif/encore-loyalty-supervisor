# Phase 4: Feedback System Architecture Mapping

**Created**: 2025-11-12  
**Purpose**: Document exact data architecture for Phase 4 implementation  
**Critical**: This is the core of the project - must be implemented correctly

---

## 🎯 **Core Principle**

**We are AUGMENTING the legacy system, NOT replacing it.**

- ✅ Read historical feedback from MySQL (READ-ONLY)
- ✅ Add new AI capabilities via DynamoDB (READ/WRITE)
- ❌ NEVER modify MySQL data

---

## 📊 **Data Architecture**

### **MySQL Legacy (READ-ONLY)** - Historical Feedback

#### Primary Table: `comment`

```sql
-- The main feedback table
comment (
  iddatasession INT PRIMARY KEY,        -- Unique feedback ID
  comment VARCHAR(2300),                -- Customer comment text
  overall_rating INT,                   -- 1-5 rating (NOTE: 99.59% are 0)
  emailaddress VARCHAR,                 -- Customer email
  dateentered DATETIME,                 -- When comment was submitted
  reply_details TEXT,                   -- JSON array of historical replies
  bcc TEXT,                            -- BCC emails for notifications
  replied TINYINT(1),                  -- Whether replied (0/1)
  datereplied DATETIME,                -- When reply was sent
  IsActive TINYINT(1),                 -- Active status
  isdeleted TINYINT(1)                 -- Soft delete flag
)
```

#### Related Tables (for JOINs):

```sql
-- Session/Venue linkage
datasession (
  iddatasession INT PRIMARY KEY,
  idclient INT,                        -- Venue/restaurant ID
  idlocation INT,                      -- Specific location ID
  iddevice VARCHAR(255),               -- Device used
  dateentered DATETIME,
  ipaddress VARCHAR(255),
  isactive TINYINT(1),
  isdeleted TINYINT(1)
)

-- Customer contact info
eclub (
  iddatasession INT PRIMARY KEY,       -- Links to comment
  emailaddress VARCHAR,
  firstname VARCHAR,
  lastname VARCHAR,
  phonenumber VARCHAR,
  zipcode VARCHAR,
  dateentered DATETIME
)

-- Venue/Restaurant info
client (
  idclient INT PRIMARY KEY,
  description VARCHAR,                 -- Restaurant name
  emailaddress VARCHAR,
  firstname VARCHAR,
  lastname VARCHAR,
  status TINYINT(1)
)

-- Location details
location (
  idlocation INT PRIMARY KEY,
  idclient INT,
  locationname VARCHAR,
  address VARCHAR,
  city VARCHAR,
  state VARCHAR,
  zip VARCHAR
)
```

---

### **Complete SQL Query for Feedback**

```sql
SELECT 
    -- Feedback ID and core data
    c.iddatasession AS id,
    c.comment AS comment_text,
    c.overall_rating AS rating,
    c.emailaddress AS customer_email,
    c.dateentered AS created_at,
    c.reply_details AS legacy_replies,
    c.replied AS has_legacy_reply,
    c.datereplied AS legacy_reply_date,
    
    -- Venue information
    ds.idclient AS venue_id,
    cl.description AS venue_name,
    
    -- Location information
    ds.idlocation AS location_id,
    loc.locationname AS location_name,
    
    -- Customer information
    e.firstname AS customer_first_name,
    e.lastname AS customer_last_name,
    e.phonenumber AS customer_phone,
    
    -- Metadata
    ds.dateentered AS session_date,
    ds.device AS device_type,
    ds.browser AS browser_type,
    ds.ipaddress AS ip_address

FROM comment c
INNER JOIN datasession ds ON c.iddatasession = ds.iddatasession
LEFT JOIN eclub e ON c.iddatasession = e.iddatasession
INNER JOIN client cl ON ds.idclient = cl.idclient
LEFT JOIN location loc ON ds.idlocation = loc.idlocation

WHERE 
    c.isdeleted = 0
    AND ds.isdeleted = 0
    AND c.comment IS NOT NULL
    AND c.comment != ''
    
ORDER BY c.dateentered DESC
```

---

### **DynamoDB Augmentation (READ/WRITE)** - New AI Functionality

We store NEW data in DynamoDB, keyed by the MySQL `iddatasession`:

#### Table: `encore_feedback_status`

```
PK: FEEDBACK#{iddatasession}
SK: METADATA

Attributes:
{
  feedback_id: iddatasession (from MySQL),
  venue_id: idclient (from MySQL),
  
  // NEW AI-DRIVEN FIELDS (we add these)
  status: "pending" | "reviewed" | "generated" | "edited" | "sent" | "archived" | "spam",
  sentiment: "positive" | "neutral" | "negative" | "mixed" | null,
  
  // AI Response tracking
  ai_response_id: uuid | null,
  ai_response_text: string | null,
  ai_model: "claude-3-5-sonnet" | null,
  ai_generated_at: ISO timestamp | null,
  
  // Action tracking
  last_updated: ISO timestamp,
  last_action_by: user_email,
  action_history: [
    {
      timestamp: ISO timestamp,
      action: "status_changed" | "response_generated" | "response_edited" | "email_sent",
      user: user_email,
      details: {}
    }
  ],
  
  // Metadata
  response_count: number,      // How many times response generated
  email_sent_count: number,    // How many times email sent
  manually_edited: boolean,    // If response was manually edited
  
  // Timestamps
  created_at: ISO timestamp,
  updated_at: ISO timestamp
}
```

---

## 🔄 **Data Flow**

### Reading Feedback (GET endpoints)

```
1. Query MySQL for feedback data (historical)
   └─ SELECT with JOINs to get complete feedback

2. Query DynamoDB for status/AI data (augmented)
   └─ PK: FEEDBACK#{iddatasession}

3. MERGE results:
   └─ MySQL data (comment, rating, customer, venue)
   └─ DynamoDB data (status, sentiment, AI response)
   
4. Return combined Feedback object
```

### Updating Feedback (PUT endpoints)

```
1. READ from MySQL to verify feedback exists
   └─ Ensure iddatasession is valid

2. WRITE to DynamoDB only
   └─ Update status, sentiment, AI response
   └─ Add to action_history
   └─ Update timestamps
   
3. NEVER write to MySQL
```

### Creating AI Response (POST endpoints)

```
1. READ feedback from MySQL (get context)

2. Generate AI response using AWS Bedrock

3. WRITE to DynamoDB:
   └─ Store AI response text
   └─ Update status to "generated"
   └─ Increment response_count
   └─ Add to action_history
   
4. Return response
```

---

## 📋 **Field Mapping**

### MySQL → API Model

| MySQL Field | API Model Field | Notes |
|-------------|----------------|-------|
| `c.iddatasession` | `id` | Primary identifier |
| `c.comment` | `comment` | Customer feedback text |
| `c.overall_rating` | `rating` | 1-5 stars (99.59% are 0 - data quality issue) |
| `c.emailaddress` | `customer_email` | From comment table |
| `e.emailaddress` | `customer_email` | From eclub table (fallback) |
| `e.firstname` | `customer_first_name` | |
| `e.lastname` | `customer_last_name` | |
| `e.phonenumber` | `customer_phone` | |
| `c.dateentered` | `created_at` | When feedback submitted |
| `ds.idclient` | `venue_id` | Restaurant identifier |
| `cl.description` | `venue_name` | Restaurant name |
| `ds.idlocation` | `location_id` | Specific location |
| `loc.locationname` | `location_name` | Location name |

### DynamoDB → API Model

| DynamoDB Field | API Model Field | Type | Default |
|----------------|----------------|------|---------|
| `status` | `status` | Enum | "pending" |
| `sentiment` | `sentiment` | Enum | null |
| `ai_response_text` | `ai_response` | String | null |
| `ai_generated_at` | `response_generated_at` | DateTime | null |
| `last_updated` | `last_updated` | DateTime | - |
| `last_action_by` | `last_action_by` | String | - |
| `response_count` | `response_count` | Int | 0 |
| `email_sent_count` | `email_sent_count` | Int | 0 |

---

## 🔑 **Key Design Decisions**

### 1. Why `iddatasession` as Primary Key?

- **MySQL**: Uses `iddatasession` as PK for comment table
- **DynamoDB**: We key by same ID to maintain linkage
- **Benefit**: Direct 1:1 mapping between systems

### 2. Why Separate DynamoDB Table?

- **Read-only MySQL**: Can't add new columns to comment table
- **Flexible schema**: Can add AI fields without MySQL constraints
- **Performance**: Fast DynamoDB reads for status/AI data
- **Migration**: Legacy system continues to work unchanged

### 3. Status Lifecycle

```
pending → reviewed → generated → edited → sent → archived
                 ↘           ↗
                   spam
```

- **pending**: New feedback, not processed
- **reviewed**: Manager reviewed manually
- **generated**: AI response created
- **edited**: Response manually modified
- **sent**: Email sent to customer
- **archived**: Closed/completed
- **spam**: Marked as spam

### 4. Sentiment Analysis

Stored in DynamoDB, not MySQL:
- **positive**: Happy customer
- **neutral**: Neutral feedback
- **negative**: Unhappy customer
- **mixed**: Both positive and negative elements
- **null**: Not yet analyzed

---

## ⚠️ **Critical Implementation Notes**

### DO:
- ✅ Read all historical feedback from MySQL
- ✅ Write all status/AI data to DynamoDB
- ✅ Use `iddatasession` as the linking key
- ✅ Handle NULL ratings (99.59% are 0 in MySQL)
- ✅ Merge MySQL + DynamoDB data in response
- ✅ Use JOINs to get complete customer/venue info

### DON'T:
- ❌ Write to MySQL (READ-ONLY)
- ❌ Assume ratings exist (they're mostly 0)
- ❌ Query DynamoDB first (MySQL is source of truth for feedback)
- ❌ Create new tables in MySQL
- ❌ Modify legacy comment.reply_details field

---

## 📝 **API Endpoint Requirements**

### 1. GET /api/v1/feedback
- Query MySQL with filters (venue_id, date range, search)
- For each feedback, lookup DynamoDB status
- Merge and return paginated list

### 2. GET /api/v1/feedback/{id}
- Query MySQL for feedback details (with JOINs)
- Query DynamoDB for status/AI data
- Merge and return single feedback object

### 3. PUT /api/v1/feedback/{id}/status
- Verify feedback exists in MySQL
- Update status in DynamoDB only
- Add to action_history
- Return updated feedback

### 4. POST /api/v1/feedback/{id}/generate
- Read feedback from MySQL (for context)
- Call AWS Bedrock for AI generation
- Store response in DynamoDB
- Update status to "generated"
- Return AI response

### 5. POST /api/v1/feedback/{id}/regenerate
- Read feedback + previous response from DynamoDB
- Call AWS Bedrock with custom instructions
- Update response in DynamoDB
- Increment response_count
- Return new response

### 6. PUT /api/v1/feedback/{id}/response
- Update AI response text in DynamoDB
- Set manually_edited = true
- Update status to "edited"
- Return updated feedback

### 7. POST /api/v1/feedback/{id}/send
- Read response from DynamoDB
- Send email via AWS SES
- Update status to "sent"
- Increment email_sent_count
- Return success

### 8. GET /api/v1/feedback/{id}/history
- Read action_history from DynamoDB
- Return chronological list of actions

---

## 🧪 **Testing Considerations**

### Data Quality Issues

**Rating Problem** (from archive analysis):
- 99.59% of ratings in MySQL are 0
- This is a bug in legacy data collection
- Our API should handle null/0 ratings gracefully
- Don't assume rating exists

**Customer Data**:
- 50% duplicate customers (by email)
- Not all feedback has customer info in eclub
- Must handle missing firstname/lastname

### Sample Test Data

```sql
-- Find feedback with good data for testing
SELECT 
    c.iddatasession,
    c.comment,
    c.overall_rating,
    e.firstname,
    e.lastname,
    cl.description
FROM comment c
INNER JOIN datasession ds ON c.iddatasession = ds.iddatasession
LEFT JOIN eclub e ON c.iddatasession = e.iddatasession
INNER JOIN client cl ON ds.idclient = cl.idclient
WHERE 
    c.comment IS NOT NULL 
    AND c.comment != ''
    AND c.overall_rating > 0
    AND e.firstname IS NOT NULL
LIMIT 10
```

---

## 🔍 **Example: Complete Feedback Object**

```json
{
  // From MySQL (historical data)
  "id": 123456,
  "comment": "Great food but slow service",
  "rating": 3,
  "customer_email": "john@example.com",
  "customer_name": "John Doe",
  "customer_phone": "555-1234",
  "venue_id": 5,
  "venue_name": "Downtown Bistro",
  "location_id": 12,
  "location_name": "Main Street",
  "created_at": "2023-10-15T18:30:00Z",
  "has_legacy_reply": true,
  "legacy_reply_date": "2023-10-16T10:00:00Z",
  
  // From DynamoDB (AI augmentation)
  "status": "sent",
  "sentiment": "mixed",
  "ai_response": "Thank you for your feedback John! We're thrilled you enjoyed the food...",
  "response_generated_at": "2023-10-16T09:00:00Z",
  "last_updated": "2023-10-16T10:05:00Z",
  "last_action_by": "manager@bistro.com",
  "response_count": 1,
  "email_sent_count": 1,
  "manually_edited": false
}
```

---

## ✅ **Validation Checklist**

Before implementing, verify:

- [ ] MySQL queries use correct table names (comment, datasession, eclub, client)
- [ ] All JOINs are correct (INNER vs LEFT)
- [ ] DynamoDB keys use FEEDBACK#{iddatasession} format
- [ ] Status enum matches defined values
- [ ] All endpoints handle missing data gracefully
- [ ] No writes to MySQL anywhere
- [ ] Action history is logged for all changes
- [ ] Response counts are incremented correctly
- [ ] Timestamps use ISO 8601 format
- [ ] Error messages are clear and actionable

---

**This document is the source of truth for Phase 4 implementation.**

**Last Updated**: 2025-11-12  
**Status**: Architecture locked - ready for implementation

