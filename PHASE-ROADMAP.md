# Phase Roadmap - Complete Development Plan
## Encore Loyalty AI Feedback System

**Created**: November 5, 2025
**Status**: Active Development - Phase 2B Complete
**Timeline**: 3-4 weeks for MVP
**Last Updated**: 2025-11-06 (Phase 2B Complete)

---

## 📊 OVERVIEW

This document maps all development work to specific phases, showing what endpoints and features will be built when.

**Total Phases**: 8 phases
**Current Status**: Phase 2B complete (Frontend Authentication)
**Parallel Work**: Backend and Frontend can work simultaneously

---

## 🗺️ COMPLETE PHASE MAP

### ✅ Phase 0: Planning & Setup (COMPLETE)

**Duration**: Completed October 22-28, 2025

**Deliverables**:
- [x] Complete planning documentation
- [x] MVP scope defined
- [x] Backend infrastructure analyzed
- [x] Database schema documented
- [x] Architecture decisions made
- [x] Supervisor framework established

**Documents**:
- `/docs/plan-mockup-to-frontend/` - Complete conversion plan
- `/docs/SUPERVISOR/` - Supervisor framework
- `/docs/investigation/ENCORE-ARCHIVE-ANALYSIS-REPORT.md` - 3,312-line analysis

---

### ✅ Phase 1: Foundation Setup (COMPLETE)

**Duration**: 2-3 hours
**Status**: Completed November 5, 2025

**Frontend Track**:
- [x] Rename `mockup` → `encore_frontend`
- [x] Install dependencies (axios, jwt-decode, react-query, react-toastify)
- [x] Create environment files (.env.development, .env.production, .env.local.example)
- [x] Create config module (`src/config/api.config.ts`)
- [x] Create utility modules (errorHandler, logger, storage)
- [x] Update .gitignore
- [x] Verify app still runs with mock data

**Backend Track**:
- [x] Create `.env` file
- [x] Verify SSH key permissions
- [x] Start backend server
- [x] Test health endpoints
- [x] Create unified management scripts (START.sh, STOP.sh, STATUS.sh)

**Endpoints Available After Phase 1**:
- None (infrastructure only)

**Prompt**: `/docs/SUPERVISOR/WORKER-PROMPTS/PHASE-1-FOUNDATION-SETUP.md`

**Success Criteria**:
- Frontend infrastructure ready
- Backend running on port 8000
- Both services manageable via unified scripts

---

### ✅ Phase 2: Authentication (COMPLETE)

**Duration**: 4-6 hours (parallel tracks)
**Start Date**: November 6, 2025
**Completion Date**: November 6, 2025
**Priority**: P1 - Critical

#### ✅ Phase 2A: Backend Authentication (COMPLETE)

**Tasks**:
- [x] Install JWT dependencies (python-jose, passlib, python-multipart)
- [x] Create User model (`models/user.py`)
- [x] Create JWT service (`services/jwt_service.py`)
- [x] Create Auth service (`services/auth_service.py`)
- [x] Create auth dependency (`dependencies.py`)
- [x] Create auth routes (`routes/auth.py`)

**Endpoints Created** (4 endpoints):
- `POST /api/v1/auth/login` - Login with email/password
- `POST /api/v1/auth/logout` - Logout user
- `POST /api/v1/auth/refresh` - Refresh access token
- `GET /api/v1/auth/me` - Get current user info

**Completion Report**: `/docs/SUPERVISOR/PHASE-2A-COMPLETION-REPORT.md`

#### ✅ Phase 2B: Frontend Authentication (COMPLETE - Nov 6, 2025)

**Tasks**:
- [x] Create auth API service (`src/services/api/authApi.ts`)
- [x] Create API client with JWT interceptors (`src/services/apiClient.ts`)
- [x] Create real auth service (`src/services/realAuthService.ts`)
- [x] Create mock auth service (`src/services/mockAuthService.ts`)
- [x] Update auth service toggle (`src/services/authService.ts`)
- [x] Update AuthContext with real API integration
- [x] Update storage utilities (refresh token methods)
- [x] Update Login page error handling
- [x] Test build compilation

**Files Created**: 6 new files
**Files Updated**: 5 files
**TypeScript Errors**: 0
**Linter Errors**: 0
**Build Status**: ✅ Compiled successfully

**Completion Report**: `/docs/SUPERVISOR/PHASE-2B-COMPLETION-REPORT.md`

**Success Criteria**: ✅ ALL MET
- [x] Can login from frontend with real credentials
- [x] JWT tokens stored and refreshed automatically
- [x] Protected routes require authentication
- [x] Mock API toggle still works
- [x] Token refresh transparent to user
- [x] Error handling comprehensive

---

### ⚙️ Phase 3: Settings Management (Week 1)

**Duration**: 6-8 hours
**Start After**: Phase 2 complete
**Priority**: P1 - Core MVP feature

#### Phase 3A: Backend Settings Endpoints

**Endpoints to Create** (6 endpoints):
- `GET /api/v1/settings/global` - Get global AI settings
- `PUT /api/v1/settings/global` - Update global AI settings
- `GET /api/v1/settings/venue/{id}` - Get venue-specific settings
- `PUT /api/v1/settings/venue/{id}` - Update venue settings
- `GET /api/v1/settings/venue/{id}/history` - Get settings change history
- `POST /api/v1/settings/test-notification` - Test notification settings

**Database**:
- Store settings in DynamoDB
- Track settings history
- Default settings cascade (global → venue → user)

**Models**:
- `GlobalAISettings`
- `VenueAISettings`
- `SettingsHistory`

#### Phase 3B: Frontend Settings Integration

**Tasks**:
- Create settings API service
- Update settings pages to call real API
- Implement settings form validation
- Add settings history display
- Test notification feature

**Pages to Update**:
- Global Admin: AI Settings page
- Venue Manager: AI Configuration (3 tabs)

**Success Criteria**:
- Can view and update global settings
- Can view and update venue settings
- Settings persist to DynamoDB
- History tracked properly

---

### 📋 Phase 4: Feedback Management (Week 1-2)

**Duration**: 8-10 hours
**Start After**: Phase 3 complete
**Priority**: P1 - Core MVP feature

#### Phase 4A: Backend Feedback Endpoints

**Endpoints to Create** (8 endpoints):
- `GET /api/v1/feedback` - List feedback (filtered, paginated)
- `GET /api/v1/feedback/{id}` - Get single feedback detail
- `PUT /api/v1/feedback/{id}/status` - Update feedback status
- `POST /api/v1/feedback/{id}/generate` - Generate AI response (stub for now)
- `POST /api/v1/feedback/{id}/regenerate` - Regenerate response with instructions
- `PUT /api/v1/feedback/{id}/response` - Update response text manually
- `POST /api/v1/feedback/{id}/send` - Send response via email (stub for now)
- `GET /api/v1/feedback/{id}/history` - Get feedback action history

**Database**:
- Query feedback from MySQL (read-only)
- Store AI responses in DynamoDB
- Track feedback status changes
- Store action history

**Models**:
- `Feedback`
- `FeedbackResponse`
- `FeedbackHistory`

**Filters**:
- By venue
- By status (pending, generated, sent, etc.)
- By date range
- By sentiment
- By rating

#### Phase 4B: Frontend Feedback Integration

**Tasks**:
- Create feedback API service
- Update feedback list page
- Update feedback detail page
- Implement status updates
- Add filtering and search
- Add pagination

**Pages to Update**:
- Venue Manager: Feedback List
- Venue Manager: Feedback Detail

**Success Criteria**:
- Can view list of feedback from database
- Can filter and search feedback
- Can view feedback details
- Can update feedback status
- Pagination works

---

### 🤖 Phase 5: AI Response Generation (Week 2)

**Duration**: 8-10 hours
**Start After**: Phase 4 complete
**Priority**: P1 - Core MVP feature

#### Phase 5A: Backend AI Integration

**Tasks**:
- Implement AWS Bedrock integration
- Create customer profiling service
- Implement AI prompt engineering
- Create response generation service
- Store generated responses in DynamoDB
- Handle regeneration with custom instructions

**Endpoints to Update**:
- `POST /api/v1/feedback/{id}/generate` - Actually call Bedrock
- `POST /api/v1/feedback/{id}/regenerate` - Regenerate with instructions

**AWS Services**:
- AWS Bedrock (Claude 3.5 Sonnet)
- S3 (customer profiles)

**Models**:
- `CustomerProfile`
- `AIResponse`

**Configuration**:
- Enable in `.env`: `BEDROCK_ENABLED=true`
- Configure model: `anthropic.claude-3-5-sonnet-20241022-v2:0`

#### Phase 5B: Frontend AI Features

**Tasks**:
- Add "Generate Response" button
- Add loading states for AI generation
- Display generated response
- Add "Regenerate" with custom instructions
- Add response editing
- Show generation history

**Success Criteria**:
- Can generate AI response for feedback
- Response appears in reasonable time (<10 seconds)
- Can regenerate with custom instructions
- Can edit generated response
- Response quality is good

---

### 📧 Phase 6: Email Integration (Week 2)

**Duration**: 4-6 hours
**Start After**: Phase 5 complete
**Priority**: P1 - Core MVP feature

#### Phase 6A: Backend Email Service

**Tasks**:
- Implement AWS SES integration
- Create email templates
- Implement send response logic
- Track email status (sent, failed, bounced)
- Store email history

**Endpoints to Update**:
- `POST /api/v1/feedback/{id}/send` - Actually send via SES

**AWS Services**:
- AWS SES (email sending)

**Configuration**:
- Enable in `.env`: `SES_ENABLED=true`
- Configure sender email
- Verify email addresses in SES

#### Phase 6B: Frontend Email Features

**Tasks**:
- Add "Send Response" button
- Add send confirmation modal
- Display email status
- Show send history
- Handle send errors

**Success Criteria**:
- Can send response via email
- Email actually arrives in inbox
- Send status tracked
- Errors handled gracefully

---

### 📊 Phase 7: Analytics & Reporting (Week 2-3)

**Duration**: 6-8 hours
**Start After**: Phase 6 complete
**Priority**: P2 - Important but not blocking

#### Phase 7A: Backend Analytics Endpoints

**Endpoints to Create** (6 endpoints):
- `GET /api/v1/analytics/venue/{id}` - Get venue analytics
- `GET /api/v1/analytics/global` - Get global analytics
- `GET /api/v1/analytics/venue/{id}/performance` - Performance metrics
- `GET /api/v1/analytics/venue/{id}/satisfaction` - Satisfaction trends
- `GET /api/v1/analytics/venue/{id}/sentiment` - Sentiment analysis
- `POST /api/v1/analytics/export` - Export analytics data

**Metrics to Calculate**:
- Total feedback count
- Response rate
- Average response time
- Sentiment distribution
- Satisfaction scores
- AI usage statistics

**Database**:
- Aggregate from MySQL and DynamoDB
- Cache computed metrics

#### Phase 7B: Frontend Analytics Dashboard

**Tasks**:
- Create analytics API service
- Update dashboard with real data
- Implement charts (Recharts)
- Add date range filters
- Add export functionality

**Pages to Update**:
- Global Admin: Analytics Dashboard
- Venue Manager: Analytics Dashboard

**Success Criteria**:
- Dashboard shows real metrics
- Charts render correctly
- Data updates when filtered
- Export works

---

### 🏢 Phase 8: Venue Management & Admin (Week 3)

**Duration**: 6-8 hours
**Start After**: Phase 7 complete
**Priority**: P2 - Admin features

#### Phase 8A: Backend Admin Endpoints

**Endpoints to Create** (10 endpoints):
- `GET /api/v1/venues` - List all venues
- `GET /api/v1/venues/{id}` - Get venue details
- `PUT /api/v1/venues/{id}` - Update venue
- `POST /api/v1/venues/{id}/enable-ai` - Enable AI for venue
- `POST /api/v1/venues/{id}/disable-ai` - Disable AI for venue
- `GET /api/v1/admin/users` - List all users
- `POST /api/v1/admin/users` - Create user
- `PUT /api/v1/admin/users/{id}` - Update user
- `GET /api/v1/admin/activity` - Get activity log
- `GET /api/v1/admin/system-health` - System health check

**Integration**:
- Brainium Venues API (external)
- MySQL users table
- DynamoDB for AI settings

#### Phase 8B: Frontend Admin Features

**Tasks**:
- Create admin API service
- Create venue management pages
- Create user management pages
- Add system health monitoring
- Add activity log

**Pages to Update**:
- Global Admin: AI Provisioning
- Global Admin: User Management
- Global Admin: System Monitoring

**Success Criteria**:
- Can provision AI for venues
- Can manage users
- System health visible
- Activity log works

---

## 📋 ENDPOINT SUMMARY

### Total Endpoints to Build: ~42 endpoints

**Authentication** (4 endpoints):
- Phase 2: Login, Logout, Refresh, Get User

**Settings** (6 endpoints):
- Phase 3: Get/Update Global, Get/Update Venue, History, Test

**Feedback** (8 endpoints):
- Phase 4: List, Get, Update Status, Generate, Regenerate, Update Response, Send, History

**Analytics** (6 endpoints):
- Phase 7: Venue Analytics, Global Analytics, Performance, Satisfaction, Sentiment, Export

**Venues** (5 endpoints):
- Phase 8: List, Get, Update, Enable AI, Disable AI

**Admin** (5 endpoints):
- Phase 8: List Users, Create User, Update User, Activity Log, System Health

**Existing** (8 endpoints - already built):
- Health Check
- Clients List
- Client Reviews
- Onboard Client
- Onboard Status
- Submit Comment
- List Restaurants
- (Root endpoint)

---

## 🗓️ TIMELINE

### Week 1 (Nov 5-11)

**Day 1** (Nov 5): Phase 1 Foundation
- Frontend: Infrastructure setup
- Backend: Server running + scripts

**Day 2** (Nov 6): Phase 2 Authentication
- Backend: 4 auth endpoints
- Frontend: JWT integration

**Day 3** (Nov 7): Phase 3 Settings (Part 1)
- Backend: Settings endpoints
- Frontend: Settings pages

**Day 4** (Nov 8): Phase 3 Settings (Part 2)
- Complete settings integration
- Testing

**Day 5** (Nov 9): Phase 4 Feedback (Part 1)
- Backend: Feedback endpoints
- Frontend: List & detail pages

**Weekend** (Nov 9-10): Buffer/Catch-up

### Week 2 (Nov 11-17)

**Day 6** (Nov 11): Phase 4 Feedback (Part 2)
- Complete feedback integration
- Testing

**Day 7** (Nov 12): Phase 5 AI Generation
- Backend: Bedrock integration
- Frontend: Generate UI

**Day 8** (Nov 13): Phase 6 Email
- Backend: SES integration
- Frontend: Send UI

**Day 9** (Nov 14): Phase 7 Analytics (Part 1)
- Backend: Analytics endpoints
- Frontend: Dashboard

**Day 10** (Nov 15): Phase 7 Analytics (Part 2)
- Complete analytics
- Testing

**Weekend** (Nov 16-17): Buffer/Testing

### Week 3 (Nov 18-24)

**Day 11** (Nov 18): Phase 8 Admin (Part 1)
- Backend: Admin endpoints
- Frontend: Admin pages

**Day 12** (Nov 19): Phase 8 Admin (Part 2)
- Complete admin features
- Testing

**Days 13-15** (Nov 20-22): Integration Testing
- End-to-end testing
- Bug fixes
- Performance optimization

**Days 16-17** (Nov 23-24): Deployment
- Staging deployment
- Production deployment
- Documentation

**MVP Complete**: November 24, 2025

---

## 🎯 MILESTONES

**Milestone 1**: Basic Infrastructure (End of Day 1)
- ✅ Frontend and backend running
- ✅ Can start/stop/check status

**Milestone 2**: Authentication Working (End of Day 2)
- 🎯 Can login from frontend
- 🎯 JWT tokens working
- 🎯 Protected routes secure

**Milestone 3**: Settings & Feedback (End of Week 1)
- 🎯 Can configure AI settings
- 🎯 Can view feedback list
- 🎯 Can view feedback details

**Milestone 4**: AI Features (Mid Week 2)
- 🎯 Can generate AI responses
- 🎯 Can send responses via email
- 🎯 Core workflow complete

**Milestone 5**: Full Feature Set (End of Week 2)
- 🎯 Analytics dashboard working
- 🎯 All core features functional
- 🎯 Ready for admin features

**Milestone 6**: MVP Complete (End of Week 3)
- 🎯 All admin features working
- 🎯 System fully functional
- 🎯 Deployed to staging
- 🎯 Ready for UAT

---

## 📁 PROMPTS CREATED

**Available Now**:
1. ✅ Phase 1: Foundation Setup
   - `/docs/SUPERVISOR/WORKER-PROMPTS/PHASE-1-FOUNDATION-SETUP.md`
2. ✅ Backend Setup
   - `/docs/SUPERVISOR/WORKER-PROMPTS/BACKEND-SETUP-AND-START.md`
3. ✅ Phase 2A: Backend Authentication
   - `/docs/SUPERVISOR/WORKER-PROMPTS/PHASE-2A-BACKEND-AUTHENTICATION.md`
4. ✅ Phase 2B: Frontend Authentication
   - `/docs/SUPERVISOR/WORKER-PROMPTS/PHASE-2B-FRONTEND-AUTHENTICATION.md`
5. ✅ Phase 3A: Backend Settings
   - `/docs/SUPERVISOR/WORKER-PROMPTS/PHASE-3A-BACKEND-SETTINGS.md`
6. ✅ Phase 3B: Frontend Settings
   - `/docs/SUPERVISOR/WORKER-PROMPTS/PHASE-3B-FRONTEND-SETTINGS.md`
7. ✅ Phase 4A: Backend Feedback
   - `/docs/SUPERVISOR/WORKER-PROMPTS/PHASE-4A-BACKEND-FEEDBACK.md`
8. ✅ Phase 4B: Frontend Feedback
   - `/docs/SUPERVISOR/WORKER-PROMPTS/PHASE-4B-FRONTEND-FEEDBACK.md`

**To Be Created**:
- Phase 5A: Backend AI Integration
- Phase 5B: Frontend AI Features
- Phase 6A: Backend Email
- Phase 6B: Frontend Email
- Phase 7A: Backend Analytics
- Phase 7B: Frontend Analytics
- Phase 8A: Backend Admin
- Phase 8B: Frontend Admin

---

## 🔄 PARALLEL WORK STRATEGY

**Backend and Frontend can work in parallel**:

```
Week 1:
  Backend Dev  →  [Auth] → [Settings] → [Feedback]
  Frontend Dev →  [Auth] → [Settings] → [Feedback]

Week 2:
  Backend Dev  →  [AI] → [Email] → [Analytics]
  Frontend Dev →  [AI] → [Email] → [Analytics]

Week 3:
  Backend Dev  →  [Admin] → [Testing] → [Deploy]
  Frontend Dev →  [Admin] → [Testing] → [Deploy]
```

**Requirements for Parallel**:
1. API contracts defined upfront (types, endpoints)
2. Frontend can use mock data while backend builds
3. Regular sync points (end of each day)
4. Clear interface definitions

---

## 📊 PROGRESS TRACKING

**Phase Completion**:
- Phase 0: ✅ 100% Complete
- Phase 1: 🔄 In Progress (80%)
- Phase 2: ⏳ Not Started
- Phase 3: ⏳ Not Started
- Phase 4: ⏳ Not Started
- Phase 5: ⏳ Not Started
- Phase 6: ⏳ Not Started
- Phase 7: ⏳ Not Started
- Phase 8: ⏳ Not Started

**Overall**: ~35% (Planning + Phase 1 partial)

---

## 🎓 USING THIS ROADMAP

**For Planning**:
- See what's coming next
- Understand dependencies
- Estimate timelines

**For Execution**:
- Follow phases in order
- Use provided prompts
- Check off completed items

**For Reporting**:
- Track progress against timeline
- Identify blockers early
- Communicate status

---

**Last Updated**: 2025-11-05
**Next Review**: End of Week 1 (Nov 11)
**Status**: Active - In Execution Phase
