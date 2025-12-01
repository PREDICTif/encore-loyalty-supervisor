# Issues Tracker & Implementation Queue

**Purpose**: Comprehensive list of all documented issues and their worker prompts
**Last Updated**: 2025-11-28
**Status**: Active tracking - Phase 5 & 6 COMPLETE 🎉

---

## 📊 ISSUES OVERVIEW

```
┌─────────────────────────────────────────────────────────────┐
│ ISSUES SUMMARY                                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 🔴 CRITICAL Priority:     0 issues                          │
│ 🔴 HIGH Priority:         2 issues (ISSUE-001, ISSUE-003)   │
│ 🟡 MEDIUM Priority:       0 issues                          │
│ 🟢 LOW Priority:          0 issues                          │
│                                                             │
│ ⏳ Pending:               2 issues                          │
│ 🚧 In Progress:           0 issues                          │
│ ✅ Complete:              22 issues (incl. 4 fixed 2025-11-28) │
│                                                             │
│ 🎉 MILESTONE: Phase 5 & 6 Complete - Email Working!         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔴 HIGH PRIORITY ISSUES

### ISSUE-001: Admin Dashboard Using Mock Data

**Status**: 🔴 **DOCUMENTED - AWAITING IMPLEMENTATION**
**Priority**: HIGH
**Component**: Frontend Admin Dashboard + Backend Admin APIs
**Effort**: 12-18 hours (1.5-2 days)

**Summary**:
Admin dashboard shows hardcoded mock statistics instead of real data from Brainium API, MySQL, and DynamoDB.

**Impact**:
- Client cannot see real system metrics
- Cannot track actual AI adoption
- First impression issue when client logs in

**Files**:
- **Issue Doc**: `/docs/SUPERVISOR/ISSUES/ISSUE-001-ADMIN-DASHBOARD-MOCK-DATA.md`
- **Worker Prompt**: `/docs/SUPERVISOR/WORKER-PROMPTS/ISSUE-001-ADMIN-DASHBOARD-IMPLEMENTATION.md`

**Deliverables**:
- Backend: 3 new admin API endpoints
- Frontend: adminApi service + dashboard integration
- Tests: Backend unit/integration tests

**Dependencies**:
- ✅ Brainium API (working)
- ✅ MySQL connection (working)
- ✅ DynamoDB (working)
- ✅ Authentication (working)

**Next Steps**:
1. Review and approve issue documentation
2. Assign to developer or create Cursor thread
3. Track implementation progress
4. Test and verify accuracy
5. Update status to ✅ Complete

---

### ISSUE-003: Venue Dashboard Using Mock Data

**Status**: 🔴 **DOCUMENTED - AWAITING IMPLEMENTATION**
**Priority**: HIGH
**Component**: Frontend Venue Dashboard + Backend Venue Stats APIs
**Effort**: 9-12 hours (1-1.5 days)

**Summary**:
Venue dashboard shows mock statistics instead of real data. Venue name is hardcoded as "Bella Vista Restaurant". Similar to ISSUE-001 but for venue manager view.

**Impact**:
- Venue managers cannot see real metrics
- Hardcoded "Bella Vista Restaurant" shows for ALL venues
- Cannot track actual feedback volume and response activity

**Files**:
- **Issue Doc**: `/docs/SUPERVISOR/ISSUES/ISSUE-003-VENUE-DASHBOARD-MOCK-DATA.md`
- **Worker Prompt**: TBD

**Deliverables**:
- Backend: 2 new venue stats API endpoints
- Frontend: venueStatsApi service + VenueContext integration
- Remove hardcoded values from VenueDashboardEnhanced.tsx
- Tests: Backend unit tests

**Dependencies**:
- ✅ MySQL connection (working)
- ✅ DynamoDB (working)
- ✅ Venue impersonation (JUST FIXED 2025-11-26)

**Next Steps**:
1. Review and approve issue documentation
2. Consider implementing alongside ISSUE-001 (shared patterns)
3. Assign to developer
4. Update status to ✅ Complete

---

## ✅ RESOLVED ISSUES (22 Total)

### Resolved Issues (2025-11-28) - Email Workflow Complete 📧

#### CSS Template Parsing Bug (FIXED) ✅
- **Date**: 2025-11-28
- **Impact**: HTML email templates failed to render - CSS `{}` misinterpreted as Python format placeholders
- **Fix**: Escaped CSS braces `{}` → `{{}}` in 5 template files
- **Status**: ✅ Fixed

#### DynamoDB Draft Approval Bug (FIXED) ✅
- **Date**: 2025-11-28
- **Impact**: "Send Email" returned 400 error
- **Fix**: Added missing `:pending` value to ExpressionAttributeValues
- **Status**: ✅ Fixed

#### AWS SES Configuration Issues (FIXED) ✅
- **Date**: 2025-11-28
- **Impact**: Emails failed due to region mismatch and unverified addresses
- **Fix**: Configured verified sender/recipient in us-east-1
- **Status**: ✅ Fixed - email delivered successfully

---

### Resolved Issues (2025-11-26)

#### ISSUE-004: API Path Versioning Inconsistency (RESOLVED) ✅
- **Date**: 2025-11-26
- **Impact**: Feedback list and settings API calls failed with 404
- **Root Cause**: Backend uses mixed paths - `/api/v1/` for auth/admin, `/api/` for feedback/settings/ai. Frontend was using `/api/v1/` for all.
- **Fix**: Updated feedbackApi.ts and settingsApi.ts to use correct paths
- **Status**: ✅ Fixed - all API paths now match backend
- **Documentation**: `/docs/SUPERVISOR/ISSUES/ISSUE-004-API-PATH-INCONSISTENCY.md`

#### ISSUE-002: Browser Automation Login (RESOLVED) ✅
- **Date**: 2025-11-26
- **Impact**: Browser automation couldn't login due to React controlled inputs
- **Fix**: Added ref-based fallback, created demo user, fixed bcrypt compatibility
- **Status**: ✅ Code fixes complete (browser automation tool limitation remains - not our bug)
- **Documentation**: `/docs/SUPERVISOR/ISSUES/ISSUE-002-RESOLUTION-SUMMARY.md`

#### Impersonation localStorage Key Mismatch (FIXED) ✅
- **Date**: 2025-11-26
- **Impact**: Impersonation worked but page bounced back to /admin
- **Root Cause**: AuthContext stored tokens with wrong localStorage keys ('token' vs 'encore_auth_token')
- **Fix**: Updated AuthContext to use storage utility with correct keys
- **Status**: ✅ Tested and verified - impersonation now works end-to-end

### Critical Bugs (All Fixed)

#### BUG-007: Database Schema Mismatch (FIXED) ✅
- **Date**: 2025-11-19
- **Impact**: Login failure due to wrong column name
- **Fix**: Updated SQL query to use CONCAT(firstname, lastname) instead of fullname
- **Status**: ✅ Tested and verified

#### BUG-008: Role Validation Too Restrictive (FIXED) ✅
- **Date**: 2025-11-19
- **Impact**: Legacy roles not recognized
- **Fix**: Expanded UserBase model to include sysadmin, owner, manager, gm, regional
- **Status**: ✅ Tested and verified

#### BUG-009: Settings Access Denied (FIXED) ✅
- **Date**: 2025-11-19
- **Impact**: Sysadmin role couldn't access venue settings
- **Fix**: Added sysadmin to admin role checks in routes/settings.py
- **Status**: ✅ Tested and verified

#### BUG-010: Settings Update DynamoDB Serialization (FIXED) ✅
- **Date**: 2025-11-19
- **Impact**: Settings update returned 500 Internal Server Error
- **Fix**: Added exclude_none=True and _convert_floats_to_decimal helper
- **Status**: ✅ Tested and verified

#### BUG-011: MySQL Connection Refused (FIXED) ✅
- **Date**: 2025-11-19
- **Impact**: Phase 4 endpoints returned connection errors
- **Fix**: Unified all code to use SSHCommandMySQLService
- **Status**: ✅ Tested and verified
- **Documentation**: `docs/TESTING/MYSQL-UNIFICATION-COMPLETE.md`

#### BUG-012: Endpoint Path Mismatch (FIXED) ✅
- **Date**: 2025-11-19
- **Impact**: Tests failed due to wrong API path
- **Fix**: Corrected test to use /api/feedback not /api/v1/feedback
- **Status**: ✅ Tested and verified

#### BUG-013: Duplicate MySQL Implementation (FIXED) ✅
- **Date**: 2025-11-19
- **Impact**: Phase 4 used different MySQL connection approach
- **Fix**: Eliminated database/mysql.py, unified to SSHCommandMySQLService
- **Status**: ✅ Tested and verified

#### BUG-014: Feedback List Empty Items (FIXED) ✅
- **Date**: 2025-11-19
- **Impact**: GET /api/feedback returned empty items array despite 61K total
- **Fix**: MySQL migration - replaced SSH command with direct PyMySQL connection
- **Status**: ✅ Tested and verified (100% pass rate)
- **Documentation**: `docs/TESTING/MYSQL-MIGRATION-COMPLETE.md`

#### BUG-015: Customer Email/Name Not Retrievable (FIXED) ✅
- **Date**: 2025-11-19
- **Impact**: Client-required fields showing as null
- **Fix**: MySQL migration - direct connection supports complex JOINs
- **Status**: ✅ Tested and verified (100% pass rate)

#### BUG-016: Feedback By ID "Not Found" (FIXED) ✅
- **Date**: 2025-11-21
- **Impact**: AI response generation blocked by missing feedback
- **Fix**: MySQL migration - corrected column references in JOIN queries
- **Status**: ✅ Tested and verified (100% pass rate)
- **Performance**: 7.5x faster (1.72s → 0.23s for AI generation)

### Frontend Routing Issue (FIXED) ✅

#### Frontend Routing: Sysadmin Role Not Recognized (FIXED) ✅
- **Date**: 2025-11-23
- **Impact**: User logged in successfully but saw blank screen
- **Root Cause**: React routing only recognized "global_admin" role for admin dashboard
- **Fix**: Updated App.tsx to include sysadmin, super_admin in admin role checks
- **Status**: ✅ Tested and verified

### Frontend Error Handling (FIXED) ✅

#### Error Handler: React Crash on API Errors (FIXED) ✅
- **Date**: 2025-11-23
- **Impact**: FastAPI validation errors caused React to crash with blank screen
- **Root Cause**: Error handler didn't properly extract messages from FastAPI's error format
- **Fix**: Enhanced errorHandler.ts to handle FastAPI's array-based error format
- **Status**: ✅ Tested and verified

---

## 🎯 PRIORITY QUEUE

### Next to Address

**Priority 1**: ISSUE-001 - Admin Dashboard Mock Data (HIGH)
- **Rationale**: First impression for admin users, all dependencies satisfied
- **Assignment**: TBD
- **Timeline**: 1.5-2 days

**Priority 2**: ISSUE-003 - Venue Dashboard Mock Data (HIGH)
- **Rationale**: First impression for venue managers, can share code patterns with ISSUE-001
- **Assignment**: TBD
- **Timeline**: 1-1.5 days
- **Note**: Consider implementing alongside ISSUE-001 for efficiency

---

## 📊 RESOLVED ISSUES STATISTICS

### By Phase

- **Phase 1**: 0 issues (Foundation - clean)
- **Phase 2**: 2 bugs fixed (BUG-007, BUG-008)
- **Phase 3**: 2 bugs fixed (BUG-009, BUG-010)
- **Phase 4**: 6 bugs fixed (BUG-011 through BUG-016)
- **Phase 5**: 0 bugs (Integration testing successful after bug fixes)
- **Frontend**: 2 bugs fixed (Routing, Error Handler)

### By Severity

- **Critical**: 5 bugs (all fixed)
- **Medium**: 3 bugs (all fixed)
- **Low**: 0 bugs
- **Feature Gaps**: 1 issue (ISSUE-001)

### Resolution Time

- **Average**: 4-8 hours per critical bug
- **Longest**: MySQL Migration (BUG-014/015/016) - 3 hours total
- **Shortest**: BUG-008 (Role validation) - 30 minutes

---

## 📝 ISSUE DOCUMENTATION TEMPLATE

For future issues, use this template:

```markdown
# ISSUE-XXX: [Brief Title]

**Date Discovered**: YYYY-MM-DD
**Priority**: HIGH | MEDIUM | LOW
**Component**: Backend | Frontend | Database | Integration
**Status**: 🔴 DOCUMENTED | 🚧 IN PROGRESS | ✅ COMPLETE

## ISSUE SUMMARY
[Brief description of the issue]

## ROOT CAUSE ANALYSIS
[Technical explanation of why this is happening]

## IMPACT
- User Impact: [how it affects users]
- Business Impact: [how it affects business]
- Technical Impact: [how it affects system]

## ACCEPTANCE CRITERIA
- [ ] Criterion 1
- [ ] Criterion 2

## ESTIMATED EFFORT
[Hours/days with breakdown]

## DEPENDENCIES
- ✅ Completed dependency
- ⏳ Pending dependency

## NEXT STEPS
1. Step 1
2. Step 2
```

---

## 🔄 TRACKING PROCESS

### New Issue Discovery
1. Document issue in `/docs/SUPERVISOR/ISSUES/ISSUE-XXX-[NAME].md`
2. Create worker prompt in `/docs/SUPERVISOR/WORKER-PROMPTS/ISSUE-XXX-[NAME]-IMPLEMENTATION.md`
3. Add to this tracker with priority
4. Assign or mark as pending
5. Update CURRENT-STATUS.md

### During Implementation
1. Update status in this tracker
2. Log progress in worker prompt
3. Update DELEGATION-TRACKER.md if assigned
4. Communicate blockers immediately

### Upon Completion
1. Verify acceptance criteria met
2. Update status to ✅ Complete
3. Move to "Resolved Issues" section
4. Update CHANGELOG.md
5. Update CURRENT-STATUS.md

---

## 📂 FILE ORGANIZATION

### Issue Documentation
Location: `/docs/SUPERVISOR/ISSUES/`
Format: `ISSUE-XXX-[BRIEF-NAME].md`

### Worker Prompts
Location: `/docs/SUPERVISOR/WORKER-PROMPTS/`
Format: `ISSUE-XXX-[BRIEF-NAME]-IMPLEMENTATION.md`

### Bug Reports (Historical)
Location: Documented in CURRENT-STATUS.md and CHANGELOG.md
Format: BUG-XXX designation

---

## 🎓 LESSONS LEARNED

### From Bug Fixes

1. **MySQL Connection**: SSH tunnel approach had limitations - direct connection is better
2. **DynamoDB Serialization**: Python None and float types need special handling
3. **Role Validation**: Need to account for legacy roles in production systems
4. **Frontend Routing**: Need to handle all role types, not just expected ones
5. **Error Handling**: FastAPI returns different error formats - need robust extraction

### From Issue-001

1. **Mock vs Real**: Need to ensure ALL components use conditional API pattern
2. **Integration Gaps**: Even with working backend, frontend needs explicit integration
3. **Testing Coverage**: Need to test with real backend before calling feature "complete"

---

## 🎯 CONTINUOUS IMPROVEMENT

### Process Improvements

1. **Earlier Integration**: Start integrating frontend with backend APIs sooner
2. **Better Testing**: Test with real backend before marking features complete
3. **Code Review**: Ensure consistent patterns across all components
4. **Documentation**: Keep this tracker updated in real-time

### Quality Gates

Before marking feature complete:
- [ ] Backend endpoints tested with real data
- [ ] Frontend tested with real backend
- [ ] No mock API calls in production mode
- [ ] Error handling verified
- [ ] Performance benchmarks met

---

**Maintained By**: AI Supervisor
**Review Frequency**: Daily during active development
**Next Review**: After ISSUE-001 completion


