# MVP Execution Plan
## Encore Loyalty AI Feedback System

**Created**: 2025-10-28
**Start Date**: 2025-10-29 (Tomorrow)
**MVP Target**: 2025-11-19 (3 weeks)
**Team**: PREDICTif + AI Agents
**Scope**: MVP Features Only

---

## 🎯 MISSION

Build and deploy MVP of AI-powered feedback management system in **3 weeks** using AI-assisted development.

**MVP Delivers**:
- JWT Authentication
- Feedback list & detail
- AI response generation (Bedrock)
- Email sending (SES)
- Basic settings
- Venue provisioning
- Simple analytics

---

## 📅 3-WEEK TIMELINE

```
Week 1 (Oct 29 - Nov 5): Backend P1 + Frontend Foundation
  ├── Backend: Auth, Settings, Feedback endpoints
  ├── Frontend: API client, Auth flow, Core structure
  └── Milestone: Can login and see feedback list

Week 2 (Nov 6 - Nov 12): Backend P2 + Frontend Integration
  ├── Backend: AI generation, Email sending, Analytics
  ├── Frontend: All pages integrated, Real API calls
  └── Milestone: Can generate AI response and send email

Week 3 (Nov 13 - Nov 19): Testing + Bug Fixes + Deploy
  ├── Integration testing
  ├── Bug fixes
  ├── Staging deployment
  └── Milestone: MVP deployed to staging, ready for UAT
```

---

## 🚀 DAY 1 - TOMORROW (Oct 29)

### Morning (4 hours)

**Backend Track** (AI-assisted):
1. **Setup Authentication Foundation** (2 hours)
   - Create JWT service (generate, verify, refresh)
   - Create user model (integrate with MySQL user table)
   - Implement password hashing (bcrypt)
   - **AI Agent Task**: Generate boilerplate JWT service

2. **Create Auth Endpoints** (2 hours)
   - POST `/api/auth/login` (validate, return JWT)
   - POST `/api/auth/refresh` (refresh token)
   - GET `/api/auth/user` (get current user)
   - POST `/api/auth/logout` (invalidate token)
   - **AI Agent Task**: Generate FastAPI auth routes

**Frontend Track** (Parallel):
1. **Rename & Setup** (1 hour)
   - Copy `mockup` → `encore_frontend`
   - Update package.json name
   - Create `.env` files (dev, staging, production)
   - **AI Agent Task**: Setup environment configuration

2. **Create API Client** (1 hour)
   - Setup axios with interceptors
   - Create auth service (login, logout, refresh)
   - Add JWT token management (localStorage)
   - **AI Agent Task**: Generate axios API client

3. **Feature Flag Implementation** (30 min)
   - Add `REACT_APP_USE_MOCK_API` flag
   - Create API service switcher
   - Test toggle between mock and real

4. **Update AuthContext** (30 min)
   - Replace mock auth with real API calls
   - Add token refresh logic
   - Handle auth errors

### Afternoon (4 hours)

**Backend Track**:
1. **Venues Endpoint** (2 hours)
   - GET `/api/venues` (aggregate Brainium + DynamoDB)
   - GET `/api/venues/{id}` (single venue)
   - Test with real Brainium API
   - **AI Agent Task**: Generate venue aggregation logic

2. **Basic Settings Endpoints** (2 hours)
   - GET `/api/settings/venue/{id}` (from DynamoDB)
   - PUT `/api/settings/venue/{id}` (save to DynamoDB)
   - Default settings structure
   - **AI Agent Task**: Generate CRUD operations

**Frontend Track**:
1. **Test Authentication Flow** (1 hour)
   - Test login with real API
   - Test token refresh
   - Test protected routes
   - Fix any integration issues

2. **Update Venue List** (1 hour)
   - Call real `/api/venues` endpoint
   - Handle loading/error states
   - Test with mock toggle

3. **Documentation** (30 min)
   - Document what was built
   - Note any blockers
   - Update supervisor status

### End of Day 1 Deliverables

**Backend**:
- ✅ JWT authentication working
- ✅ 4 auth endpoints functional
- ✅ 2 venue endpoints functional
- ✅ 2 settings endpoints functional
- **Total: 8 endpoints**

**Frontend**:
- ✅ encore_frontend setup complete
- ✅ API client configured
- ✅ Feature flags working
- ✅ Login flow integrated with real API

**Test**: Can login from frontend and get JWT token

---

## 🔧 AI AGENT DELEGATION STRATEGY

### How to Use AI Agents Effectively

**Cursor/Claude Tasks** (Give AI these):
1. Generate boilerplate code (models, CRUD)
2. Write API endpoint implementations
3. Create TypeScript interfaces
4. Generate test cases
5. Write database queries

**Human Focus** (You do these):
1. Architecture decisions
2. Integration between components
3. Testing and validation
4. Bug fixing
5. Code review

### Example Prompts for AI Agents

**Backend Auth Service**:
```
Create a FastAPI JWT authentication service with:
- JWT token generation (15 min expiry, 7 day refresh)
- Token verification middleware
- Password hashing with bcrypt
- User model from MySQL (see schema below)
- Login, refresh, logout endpoints
Use the user schema from ENCORE-ARCHIVE-ANALYSIS-REPORT.md
```

**Frontend API Client**:
```
Create an axios API client for React with:
- Base URL from environment variable
- JWT token interceptor (add token to headers)
- Auto-refresh on 401 errors
- Error handling and logging
- TypeScript interfaces for all requests
See mockApi.ts for interface reference
```

---

## 📋 WEEK 1 DETAILED BREAKDOWN

### Day 1 (Oct 29) - Auth & Foundation
- Backend: Auth (4 endpoints), Venues (2), Settings (2) = 8 endpoints
- Frontend: Setup, API client, Auth integration
- **Milestone**: Can login and see venue list

### Day 2 (Oct 30) - Feedback Basics
- Backend: GET /feedback (list), GET /feedback/{id} (detail), PUT /feedback/{id}/status
- Frontend: Feedback list page with real data, Feedback detail page
- **Milestone**: Can view feedback from database

### Day 3 (Oct 31) - Settings & Profile
- Backend: Complete settings endpoints (get, update all types)
- Frontend: Settings pages integrated, User profile
- **Milestone**: Can update settings

### Day 4 (Nov 1) - AI Response Generation Part 1
- Backend: AWS Bedrock integration, Customer profiling
- Frontend: Generate button, Loading states
- **Milestone**: Can generate AI response (but not save/send yet)

### Day 5 (Nov 2) - AI Response Generation Part 2 & Email
- Backend: POST /feedback/{id}/response (save), POST /feedback/{id}/send (email via SES)
- Frontend: Edit response, Send confirmation modal
- **Milestone**: Can generate, edit, and send AI responses

**Week 1 End**: 15-20 endpoints complete, Frontend 80% integrated

---

## 📋 WEEK 2 DETAILED BREAKDOWN

### Days 6-7 (Nov 3-4) - Weekend Buffer / Catch-up
- Fix Week 1 issues
- Complete any pending endpoints
- Polish UI integration

### Day 8 (Nov 6) - Analytics Basics
- Backend: Basic analytics endpoints (metrics, charts)
- Frontend: Dashboard with real metrics
- **Milestone**: Dashboard shows real data

### Day 9 (Nov 7) - Provisioning
- Backend: POST /venues/{id}/provision, POST /venues/{id}/deprovision
- Frontend: AI Provisioning page functional
- **Milestone**: Can enable/disable AI for venues

### Day 10 (Nov 8) - Polish & Testing
- Integration testing all workflows
- Bug fixes
- Error handling improvements

### Day 11 (Nov 9) - More Testing
- Cross-browser testing
- Performance optimization
- Security review

### Day 12 (Nov 10) - Deploy to Staging
- Setup staging environment
- Deploy backend and frontend
- Test in staging

**Week 2 End**: MVP feature-complete in staging

---

## 📋 WEEK 3 - TESTING & DEPLOYMENT

### Day 13-15 (Nov 13-15) - QA & Bug Fixes
- User testing
- Fix all critical bugs
- Performance tuning

### Day 16-17 (Nov 16-17) - UAT
- Client testing
- Final fixes
- Documentation

### Day 18-19 (Nov 18-19) - Production Deploy
- Production deployment
- Monitoring setup
- Go-live

**Week 3 End**: MVP in production

---

## 🎯 SUCCESS METRICS

### Technical
- ✅ All P1 endpoints working (15 endpoints)
- ✅ All P2 core endpoints working (8 endpoints)
- ✅ Frontend integrated with real APIs
- ✅ JWT authentication functional
- ✅ AI generation working (Bedrock)
- ✅ Email sending working (SES)
- ✅ No critical bugs

### Functional
- ✅ Can login as admin or venue manager
- ✅ Can view feedback from database
- ✅ Can generate AI responses
- ✅ Can send responses via email
- ✅ Can update settings
- ✅ Can enable/disable AI for venues
- ✅ Dashboard shows metrics

### Performance
- ✅ API responses < 2 seconds
- ✅ Page loads < 3 seconds
- ✅ No console errors
- ✅ Mobile responsive

---

## 🚨 RISKS & MITIGATION

### High Risks

**Risk**: AWS Bedrock API issues
- **Mitigation**: Test Bedrock early (Day 4), Have fallback prompt ready

**Risk**: MySQL SSH tunnel issues
- **Mitigation**: Already validated working, Test Day 1

**Risk**: JWT token refresh complexity
- **Mitigation**: Use proven pattern, Test thoroughly Day 1

### Medium Risks

**Risk**: Brainium API null fields
- **Mitigation**: Handle in encore_backend aggregation layer

**Risk**: Feature creep during development
- **Mitigation**: Stick to MVP scope, Log v2.0 requests

**Risk**: Time underestimation
- **Mitigation**: Buffer days in schedule, Cut scope if needed

---

## 📞 DAILY COORDINATION

### Daily Standup (15 min)
- What completed yesterday
- What's the plan today
- Any blockers

### End of Day (5 min)
- Update SUPERVISOR/CURRENT-STATUS.md
- Note any decisions made
- Flag any risks

---

## 🔗 REFERENCES

**Key Documents**:
- Backend Requirements: `/docs/plan-mockup-to-frontend/04-BACKEND-REQUIREMENTS.md`
- Archive Analysis: `/docs/investigation/ENCORE-ARCHIVE-ANALYSIS-REPORT.md`
- Q&A (Updated): `/docs/plan-mockup-to-frontend/05-QUESTIONS-AND-CLARIFICATIONS.md`
- Mockup Context: `/mockup/.context.md`

**Code Locations**:
- Backend: `/encore_backend`
- Frontend: `/encore_frontend` (will be created from `/mockup`)
- MySQL Config: `/encore_backend/config.py`

---

## ✅ PRE-START CHECKLIST

Before starting tomorrow:

**Environment**:
- [ ] encore_backend runs locally
- [ ] MySQL SSH access working
- [ ] AWS credentials configured (Bedrock, SES, DynamoDB)
- [ ] Brainium API accessible

**Documentation**:
- [ ] Read ENCORE-ARCHIVE-ANALYSIS-REPORT.md (key sections)
- [ ] Review MVP scope (Q5.1)
- [ ] Understand database schema (Section 1 of report)

**Tools**:
- [ ] Cursor/Claude ready for AI assistance
- [ ] Terminal/IDE setup
- [ ] Git repository ready

---

**STATUS**: ✅ READY TO START
**NEXT ACTION**: Begin Day 1 tasks tomorrow morning
**ESTIMATED MVP COMPLETION**: November 19, 2025 (3 weeks)

🚀 **LET'S BUILD!**

