# Phase 9: Background Jobs & Scheduled Tasks

**Status**: 🔴 **NOT IMPLEMENTED - CRITICAL FOR PRODUCTION**
**Priority**: P0 - BLOCKS PRODUCTION DEPLOYMENT
**Created**: 2025-11-26
**Estimated Effort**: 2-3 days (16-24 hours)

---

## 📋 EXECUTIVE SUMMARY

### The Problem

The Encore Loyalty AI Feedback System currently operates entirely in **request-response mode**. Without background jobs:

1. ❌ **Dashboard statistics** require manual computation
2. ❌ **Alerts** are not automatically detected
3. ❌ **AI responses** must be manually triggered one-by-one
4. ❌ **Auto-send emails** don't happen automatically
5. ❌ **Statistics** become stale immediately
6. ❌ **No periodic reports** are generated

### The Solution

Implement a **Background Job System** that:
1. Periodically computes dashboard statistics
2. Detects alerts from new feedback (low ratings, critical keywords)
3. Auto-generates AI responses for new feedback
4. Auto-sends approved responses after configurable delay
5. Generates daily/weekly summary reports
6. Cleans up old processed data

---

## 🎯 REQUIRED JOBS

### Job 1: Statistics Aggregator (Every 5-15 minutes)

**Purpose**: Compute dashboard statistics for all venues

**Tasks**:
- Count new feedback per venue (today, this week, this month)
- Count pending AI responses per venue
- Count sent emails per venue
- Calculate average ratings
- Calculate response rates
- Calculate average response time
- Store computed stats in DynamoDB

**Data Sources**:
- MySQL: Feedback counts, ratings
- DynamoDB: AI responses, email status

**Storage**:
```python
# DynamoDB table: encore_computed_stats
{
    "venue_id": 57,
    "computed_at": "2025-11-26T14:00:00Z",
    "stats": {
        "feedback_today": 23,
        "feedback_this_week": 156,
        "pending_responses": 5,
        "sent_today": 8,
        "average_rating": 4.2,
        "response_rate": 0.85,
        "avg_response_time_hours": 2.3
    }
}
```

---

### Job 2: Alert Detector (Every 5 minutes)

**Purpose**: Scan new feedback and generate alerts

**Triggers**:
- Low rating (1-2 stars)
- Critical keywords ("food poisoning", "sick", "roach", "hair", "rude", etc.)
- High-value customer negative feedback
- Repeated complaints from same customer

**Actions**:
- Create alert in DynamoDB
- Optionally send real-time notification (webhook/email to manager)

**Alert Schema**:
```python
# DynamoDB table: encore_venue_alerts
{
    "alert_id": "alert_uuid",
    "venue_id": 57,
    "feedback_id": 456981,
    "level": "critical",  # critical, high, medium, low, info
    "title": "Critical: Food poisoning mention",
    "description": "Customer mentioned food poisoning...",
    "detected_at": "2025-11-26T14:05:00Z",
    "is_read": False,
    "acknowledged_by": None,
    "acknowledged_at": None
}
```

---

### Job 3: Auto AI Response Generator (Every 10-15 minutes)

**Purpose**: Automatically generate AI responses for new feedback

**Logic**:
1. Find feedback without AI responses (status = 'new' or 'pending')
2. Check venue settings: Is AI auto-generation enabled?
3. Generate AI response via Bedrock
4. Store response in DynamoDB with status = 'generated'
5. If auto-send is enabled AND delay is 0, mark for sending

**Settings to Check**:
```python
venue_settings = {
    "ai_generation_enabled": True,
    "auto_generate_responses": True,  # Auto-generate without manual trigger
    "auto_send_enabled": True,
    "auto_send_delay_minutes": 30,    # Wait 30 min before sending
    "require_approval": False          # If True, status = 'pending_approval'
}
```

---

### Job 4: Auto Email Sender (Every 5 minutes)

**Purpose**: Send approved/auto-approved responses via email

**Logic**:
1. Find responses with status = 'approved' or 'auto_approved'
2. Check if delay period has passed (scheduled_send_time)
3. Send email via SES
4. Update status to 'sent'
5. Log email in history

**Queue Table** (DynamoDB: `encore_email_queue`):
```python
{
    "queue_id": "queue_uuid",
    "feedback_id": 456981,
    "response_id": "response_uuid",
    "venue_id": 57,
    "customer_email": "john@example.com",
    "scheduled_send_time": "2025-11-26T14:30:00Z",
    "status": "pending",  # pending, sent, failed, cancelled
    "created_at": "2025-11-26T14:00:00Z",
    "sent_at": None,
    "error": None
}
```

---

### Job 5: Daily Summary Report (Daily at 6 AM)

**Purpose**: Generate daily summary for venue managers

**Report Content**:
- Total feedback received yesterday
- Responses sent
- Average response time
- Critical alerts
- Customer satisfaction score
- Top issues mentioned

**Delivery**:
- Email to venue managers (if notification enabled)
- Store in DynamoDB for dashboard display

---

### Job 6: Weekly Cleanup (Weekly on Sunday)

**Purpose**: Maintain system health

**Tasks**:
- Archive old alerts (> 90 days)
- Clean up failed email queue items (> 30 days)
- Recalculate long-term statistics
- Check for orphaned records

---

## 🏗️ ARCHITECTURE OPTIONS

### Option A: FastAPI BackgroundTasks (Current - Limited)

**What We Have**:
- FastAPI's `BackgroundTasks` for immediate async processing
- Used in `comments.py` for post-submit processing

**Limitations**:
- Only triggers on HTTP requests
- No scheduling capability
- No persistence (lost on restart)
- No retry mechanism

**Verdict**: ❌ Not suitable for scheduled jobs

---

### Option B: APScheduler (Recommended for MVP)

**What It Is**:
- Python scheduling library
- Can run in-process with FastAPI
- Supports cron-like scheduling
- Persistent job storage (DynamoDB/Redis)

**Implementation**:
```python
# scheduler/scheduler.py
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.jobstores.dynamodb import DynamoDBJobStore

scheduler = AsyncIOScheduler()

# Configure job store
scheduler.add_jobstore(
    DynamoDBJobStore(region_name='us-east-1', table_name='encore_scheduler_jobs'),
    'default'
)

# Add jobs
scheduler.add_job(
    compute_statistics,
    'interval',
    minutes=5,
    id='stats_aggregator'
)

scheduler.add_job(
    detect_alerts,
    'interval',
    minutes=5,
    id='alert_detector'
)

scheduler.add_job(
    auto_generate_responses,
    'interval',
    minutes=10,
    id='ai_generator'
)

scheduler.add_job(
    send_pending_emails,
    'interval',
    minutes=5,
    id='email_sender'
)

scheduler.add_job(
    generate_daily_report,
    'cron',
    hour=6,
    minute=0,
    id='daily_report'
)

# Start with FastAPI
def start_scheduler():
    scheduler.start()
```

**Pros**:
- Easy to implement
- Runs in same process as FastAPI
- Good for MVP

**Cons**:
- Single point of failure
- Scaling requires coordination

**Verdict**: ✅ Best for MVP (1-2 days implementation)

---

### Option C: Celery + Redis (Production Scale)

**What It Is**:
- Distributed task queue
- Separate worker processes
- Redis as message broker

**When to Use**:
- High volume (>1000 feedback/day)
- Need horizontal scaling
- Want separate worker deployment

**Verdict**: 🔄 Consider for production scale (future)

---

### Option D: AWS Lambda + EventBridge (Serverless)

**What It Is**:
- Serverless scheduled functions
- AWS EventBridge for scheduling
- Pay per invocation

**When to Use**:
- Want serverless architecture
- Variable load
- Cost optimization

**Verdict**: 🔄 Consider for AWS-native deployment (future)

---

## 📝 IMPLEMENTATION PLAN

### Phase 9A: Core Scheduler Infrastructure (4-6 hours)

**Tasks**:
1. Install APScheduler: `pip install apscheduler`
2. Create scheduler module: `encore_backend/scheduler/`
3. Configure job persistence (DynamoDB table)
4. Integrate with FastAPI startup/shutdown
5. Add health check endpoint for scheduler

**Files to Create**:
- `encore_backend/scheduler/__init__.py`
- `encore_backend/scheduler/scheduler.py`
- `encore_backend/scheduler/config.py`

---

### Phase 9B: Statistics Aggregator Job (3-4 hours)

**Tasks**:
1. Create stats aggregation service
2. Implement MySQL queries for counts
3. Implement DynamoDB queries for responses
4. Create DynamoDB table for computed stats
5. Update dashboard APIs to read from computed stats

**Files to Create**:
- `encore_backend/scheduler/jobs/stats_aggregator.py`
- `encore_backend/services/stats_service.py`

---

### Phase 9C: Alert Detection Job (3-4 hours)

**Tasks**:
1. Define alert detection rules
2. Create keyword detection service
3. Create alert storage service
4. Implement notification service (optional)
5. Update frontend to display real alerts

**Files to Create**:
- `encore_backend/scheduler/jobs/alert_detector.py`
- `encore_backend/services/alert_service.py`

---

### Phase 9D: Auto AI Response Job (3-4 hours)

**Tasks**:
1. Identify pending feedback without responses
2. Check venue auto-generation settings
3. Queue for AI generation
4. Handle rate limiting (Bedrock has quotas)
5. Update response status

**Files to Create**:
- `encore_backend/scheduler/jobs/ai_generator.py`

---

### Phase 9E: Auto Email Sender Job (2-3 hours)

**Tasks**:
1. Create email queue table
2. Process pending emails
3. Handle send failures and retries
4. Update email status

**Files to Create**:
- `encore_backend/scheduler/jobs/email_sender.py`

---

### Phase 9F: Testing & Monitoring (2-3 hours)

**Tasks**:
1. Unit tests for each job
2. Integration tests for scheduler
3. Add logging and metrics
4. Create admin endpoint to view job status
5. Document configuration

---

## 🗄️ DATABASE CHANGES

### New DynamoDB Tables Required

#### 1. `encore_computed_stats`
- Stores pre-computed dashboard statistics
- Partition Key: `venue_id`
- Sort Key: `computed_at`

#### 2. `encore_venue_alerts`
- Stores detected alerts
- Partition Key: `venue_id`
- Sort Key: `detected_at`
- GSI: `alert_id` for direct lookup

#### 3. `encore_email_queue`
- Email sending queue
- Partition Key: `venue_id`
- Sort Key: `scheduled_send_time`
- GSI: `status` for finding pending emails

#### 4. `encore_scheduler_jobs` (if using DynamoDB job store)
- APScheduler job persistence
- Stores next run times, job state

---

## ⚙️ CONFIGURATION

### Environment Variables

```env
# Scheduler Configuration
SCHEDULER_ENABLED=true
SCHEDULER_STATS_INTERVAL_MINUTES=5
SCHEDULER_ALERT_INTERVAL_MINUTES=5
SCHEDULER_AI_INTERVAL_MINUTES=10
SCHEDULER_EMAIL_INTERVAL_MINUTES=5
SCHEDULER_DAILY_REPORT_HOUR=6

# Auto-processing
AUTO_GENERATE_AI_RESPONSES=true
AUTO_SEND_EMAILS=true
DEFAULT_AUTO_SEND_DELAY_MINUTES=30

# Rate Limits
MAX_AI_GENERATIONS_PER_MINUTE=10
MAX_EMAILS_PER_MINUTE=20
```

---

## 🎯 SUCCESS CRITERIA

### Must Have (MVP)
- [ ] Statistics job computes venue stats every 5 minutes
- [ ] Dashboards show stats from computed table (not real-time queries)
- [ ] Alerts are detected within 5 minutes of feedback submission
- [ ] Venue dashboard shows real alerts
- [ ] Jobs survive server restart (persistence)

### Should Have
- [ ] Auto AI generation for new feedback
- [ ] Auto email sending after delay
- [ ] Daily summary reports

### Nice to Have
- [ ] Real-time notifications (WebSocket/webhook)
- [ ] Admin interface to manage jobs
- [ ] Job execution history and metrics

---

## 📅 WHEN TO IMPLEMENT

### Recommendation: **After Phase 6 (Email Integration), Before Production**

**Rationale**:
1. Phase 6 provides email sending capability
2. Background jobs need email service to work
3. Cannot go to production without background jobs
4. Dashboard data will be stale without stats job
5. Alerts won't work without detection job

### Proposed Timeline

```
Current:  Phase 5 ✅ → Phase 6 → Phase 7 → Phase 8 → Deploy

Revised:  Phase 5 ✅ → Phase 6 → Phase 9 (NEW) → Phase 7 → Phase 8 → Deploy
                              ↑
                    Insert Background Jobs here
```

### Time Investment

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| Phase 9A (Infrastructure) | 4-6 hours | None |
| Phase 9B (Stats Job) | 3-4 hours | 9A |
| Phase 9C (Alerts Job) | 3-4 hours | 9A |
| Phase 9D (AI Job) | 3-4 hours | 9A, Phase 5 |
| Phase 9E (Email Job) | 2-3 hours | 9A, Phase 6 |
| Phase 9F (Testing) | 2-3 hours | All above |
| **Total** | **16-24 hours** | **2-3 days** |

---

## 🔗 DEPENDENCIES

**Requires**:
- ✅ Phase 5: AI Response Generation (for auto AI job)
- ⏳ Phase 6: Email Integration (for auto email job)
- ✅ MySQL connection (for stats queries)
- ✅ DynamoDB (for storing computed data)

**Blocks**:
- Production deployment
- Real dashboard data
- Automated workflows
- Client acceptance (they expect automation)

---

## 📚 RELATED DOCUMENTATION

**Existing Planning Docs**:
- `docs/plan/03-PREDICTif-Team-Code-Examples.md` - Contains `AutoReplyScheduler` example (line 1006+)
- `docs/plan/02-Brainium-Team-Integration-Plan-Complete.md` - Contains PHP auto-send logic

**Issues This Resolves**:
- ISSUE-001: Admin Dashboard Mock Data (stats job provides real data)
- ISSUE-003: Venue Dashboard Mock Data (stats job provides real data)

---

## 🚀 NEXT STEPS

1. **Review and approve** this plan
2. **Decide on timing**: After Phase 6 or parallel?
3. **Create worker prompts** for Phase 9A-9F
4. **Update PHASE-ROADMAP.md** to include Phase 9
5. **Create DynamoDB tables** for computed stats, alerts, email queue
6. **Implement and test** each job incrementally

---

**Document Owner**: AI Supervisor
**Review Status**: Pending User Review
**Last Updated**: 2025-11-26

