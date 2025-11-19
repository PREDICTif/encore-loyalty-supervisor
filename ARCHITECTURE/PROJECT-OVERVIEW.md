# CLAUDE.md - AI Assistant Context for Encore Loyalty AI Feedback System

## 🎯 Project Overview

**Project**: Encore Loyalty AI Feedback System
**Purpose**: Transform restaurant customer feedback into AI-powered, personalized responses at scale
**Status**: 🧪 Testing Phase - Phases 1-4 complete and tested (65% overall)
**Timeline**: 3-4 weeks to completion (target: Nov 26 - Dec 2, 2025)

### Key Objectives

- Process 60,000+ customer feedback items from 85 restaurant venues
- Generate personalized AI responses using AWS Bedrock (Claude 3.5 Sonnet)
- Send automated email responses via AWS SES
- Provide analytics and venue management dashboard
- Enable/disable AI per venue with granular settings control

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     React Frontend (TypeScript)              │
│                      /encore_frontend                        │
│  - Authentication (JWT)                                      │
│  - Settings Management                                       │
│  - Feedback System                                          │
│  - Admin Dashboard                                          │
└────────────────────────┬────────────────────────────────────┘
                         │ REST API
┌────────────────────────┴────────────────────────────────-────┐
│                    FastAPI Backend (Python)                  │
│                      /encore_backend                         │
│  - JWT Authentication                                        │
│  - Business Logic                                            │
│  - API Endpoints (18+ implemented, 20+ pending)              │
└────────┬──────────┬──────────┬──────────┬────────────---─────┘
         │          │          │          │
    ┌────┴───┐ ┌───┴────┐ ┌──┴───┐ ┌────┴────┐
    │ MySQL  │ │DynamoDB│ │ SES  │ │ Bedrock │
    │(Legacy)│ │(Status)│ │(Email)│ │  (AI)   │
    └────────┘ └────────┘ └──────┘ └─────────┘
```

## 📁 Directory Structure

```
encore-loyalty-new-system/
├── CLAUDE.md                    # 👈 You are here - Main AI context
├── docs/
│   ├── SUPERVISOR/              # 🎯 AI Supervisor framework & prompts
│   │   ├── CURRENT-STATUS.md    # ⭐ ALWAYS CHECK THIS FIRST
│   │   ├── PHASE-ROADMAP.md     # Development phases & timeline
│   │   └── WORKER-PROMPTS/      # Phase-specific implementation prompts
│   ├── plan-mockup-to-frontend/ # Conversion strategy
│   └── investigation/           # Database & API analysis
├── encore_backend/              # FastAPI backend (Python)
│   ├── models/                  # Pydantic models
│   ├── services/                # Business logic
│   ├── routes/                  # API endpoints
│   ├── database/                # MySQL & DynamoDB connections
│   └── main.py                  # Application entry
├── encore_frontend/             # React frontend (TypeScript)
│   ├── src/
│   │   ├── api/                 # API service layer
│   │   ├── components/          # React components
│   │   ├── pages/              # Page components
│   │   ├── hooks/              # Custom React hooks
│   │   ├── types/              # TypeScript interfaces
│   │   └── mockData/           # Mock data for development
│   └── .env.local              # Environment config
├── encore_archive/              # Archived code (mockup-backup, old versions)
└── venv/                        # Python virtual environment
```

## 🔧 Tech Stack

### Backend (Python)

- **Framework**: FastAPI
- **Authentication**: JWT (python-jose)
- **Databases**:
  - MySQL (legacy data, read-only via SSH tunnel)
  - DynamoDB (settings, status, responses)
- **AWS Services**:
  - Bedrock (Claude 3.5 Sonnet for AI)
  - SES (email sending)
  - S3 (customer profiles)
- **Dependencies**: See `requirements.txt`

### Frontend (TypeScript/React)

- **Framework**: React 18 with TypeScript
- **State**: React Context + hooks
- **API Client**: Axios with interceptors
- **UI**: Custom components (no UI library)
- **Auth**: JWT with auto-refresh
- **Dependencies**: axios, jwt-decode, react-query, react-toastify

### External APIs

- **Brainium Venues API**: Restaurant/venue data
  - ✅ Provisioning fields issue RESOLVED
  - ✅ Location filtering issue RESOLVED

## 🚀 Quick Start Commands

```bash
# Backend
cd /home/ec2-user/encore-loyalty-new-system
source venv/bin/activate
cd encore_backend
python main.py  # Runs on http://localhost:8000

# Frontend
cd /home/ec2-user/encore-loyalty-new-system/encore_frontend
npm start  # Runs on http://localhost:3000

# Unified Management (if scripts exist)
./START.sh   # Starts both backend and frontend
./STOP.sh    # Stops both services
./STATUS.sh  # Check service status
```

## 📋 Development Workflow

### IMPORTANT: AI Supervisor Framework

This project uses a **SUPERVISOR framework** for AI-assisted development:

1. **ALWAYS check** `/docs/SUPERVISOR/CURRENT-STATUS.md` first
2. **Use worker prompts** from `/docs/SUPERVISOR/WORKER-PROMPTS/` for specific phases
3. **Follow the roadmap** in `/docs/SUPERVISOR/PHASE-ROADMAP.md`

### Current Development Status

**Completed & Tested Phases**:

- ✅ Phase 1: Foundation Setup
- ✅ Phase 2: Authentication (JWT, 5 endpoints) - **Fully tested and working** (30 tests passing)
- ✅ Phase 3: Settings Management (DynamoDB, 4 endpoints) - **Fully working** (BUG-010 fixed!)
- ✅ Phase 4: Feedback System (MySQL integration, 5 endpoints) - **CORE FUNCTIONALITY WORKING** ⚠️
  - **CRITICAL FIX (2025-11-19)**: Unified MySQL connection - eliminated duplicate code (BUG-011, BUG-013)
  - **CRITICAL FIX (2025-11-19)**: Settings Update endpoint - fixed DynamoDB serialization (BUG-010)
  - **CRITICAL FIX (2025-11-19)**: Feedback query SSH limitations - analyzed PHP source, removed problematic columns (BUG-014)
  - **🔴 CRITICAL ISSUE (BUG-015)**: Customer email/name fields NOT retrievable via current SSH MySQL approach
    - **Impact**: Client REQUIRES these fields in list AND detail views
    - **Status**: Deferred to after Phase 5 - must fix before production
    - **Current**: All customer fields set to `null`
  - **Status**: Feedback list working (61,803 items), pagination working, filters working

**In Progress**:

- 📝 Phase 4 remaining endpoints testing (detail view, stats)
- 📝 Planning Phase 5: AI Response Generation

**Pending Phases**:

- ⏳ Phase 5: AI Response Generation (Bedrock)
- ⏳ Phase 6: Email Integration (SES)
- ⏳ Phase 7: Analytics & Reporting
- ⏳ Phase 8: Admin & Provisioning
- ⏳ Phase 9: Conversation Engineering (NEW)

## 🔑 Key Patterns & Conventions

### Backend Conventions (Python)

```python
# File naming: snake_case
# Class naming: PascalCase
# Function naming: snake_case
# Constants: UPPER_SNAKE_CASE

# Standard service pattern
class ServiceName:
    def __init__(self):
        self.db = get_connection()

    def operation(self, params) -> ReturnType:
        """Docstring describing operation"""
        pass

# Standard route pattern
@router.get("/endpoint", response_model=Model)
async def endpoint_name(
    param: Type,
    current_user: User = Depends(get_current_user)
):
    """API endpoint docstring"""
    pass
```

### Frontend Conventions (TypeScript)

```typescript
// File naming: camelCase.tsx/ts
// Component naming: PascalCase
// Function naming: camelCase
// Constants: UPPER_SNAKE_CASE

// Standard API service pattern
class ApiService {
  async operation(params: Type): Promise<ReturnType> {
    if (API_CONFIG.USE_MOCK_API) {
      // Return mock data
    }
    return apiClient.method('/endpoint', params);
  }
}

// Standard hook pattern
export const useFeature = (params): UseFeatureReturn => {
  const [state, setState] = useState();
  // Hook logic
  return { state, operations };
};
```

### Mock API Toggle

Frontend supports toggling between mock and real API:

```env
# .env.local
REACT_APP_USE_MOCK_API=true  # Use mock data
REACT_APP_USE_MOCK_API=false # Use real backend
```

## 🗄️ Database Schema

### MySQL (Legacy - Read Only)

```sql
-- Main tables
feedback (id, venue_id, customer_email, comment, rating, created_at)
users (id, email, password_hash, role, venue_ids)
venues (id, name, location, active)
```

### DynamoDB Tables

- `encore_venue_settings` - AI and email settings per venue
- `encore_feedback_status` - Feedback processing status
- `encore_feedback_responses` - AI-generated responses

## 🔐 Authentication Flow

1. User logs in with email/password
2. Backend validates against MySQL users table
3. Returns JWT access token (15 min) + refresh token (7 days)
4. Frontend stores tokens and adds to API requests
5. Auto-refresh on 401 errors
6. Protected routes require valid JWT

## 🧪 Testing Guidelines

### Backend Testing

```bash
cd encore_backend
python tests/test_auth_api.py
python tests/test_settings_api.py
python tests/test_feedback_api.py
```

### Frontend Testing

```bash
cd encore_frontend
npm test                    # Unit tests
npm run build              # Build check
REACT_APP_USE_MOCK_API=true npm start  # Test with mock
REACT_APP_USE_MOCK_API=false npm start # Test with backend
```

### Integration Testing Checklist

- [ ] Authentication flow (login → protected route → logout)
- [ ] Settings persistence (update → reload → verify)
- [ ] Feedback list (filter → paginate → detail view)
- [ ] Error handling (network errors, 401, 403, 500)
- [ ] Mock/Real API toggle

## ⚠️ CRITICAL: MySQL Connection Pattern (Updated 2025-11-19)

**ALL MySQL connections MUST use `SSHCommandMySQLService`**

### How to Use MySQL in Your Code

```python
from db.ssh_command_mysql_service import SSHCommandMySQLService

class YourService:
    def __init__(self):
        ssh_config = {
            'host': '52.14.57.81',
            'port': 22,
            'username': 'ubuntu',
            'key_file': 'key/Encoreepad_ppk.pem'
        }
        mysql_config = {
            'user': 'root',
            'password': 'bOuDCUvd8',
            'database': 'encoree3_mobile'
        }
        self.mysql = SSHCommandMySQLService(ssh_config, mysql_config)
    
    def query(self):
        results = self.mysql._execute_mysql_query("SELECT * FROM table")
        return results
```

### What Was Fixed (2025-11-19)

- **Problem**: Phase 4 used duplicate `database/mysql.py` with different approach
- **Solution**: Refactored to use `SSHCommandMySQLService` everywhere
- **Result**: Single unified MySQL connection, no configuration mismatches
- **Documentation**: `docs/TESTING/MYSQL-UNIFICATION-COMPLETE.md`

---

## 🚨 Important Context & Gotchas

### Environment Variables

- React uses `REACT_APP_` prefix for all env vars
- No shell substitution in React (e.g., `${VAR}` doesn't work)
- See `/encore_frontend/ENV-VARIABLES-GUIDE.md` for deployment

### AWS Services

- **Bedrock**: Must enable in `.env` with `BEDROCK_ENABLED=true`
- **SES**: Email addresses must be verified in sandbox mode
- **DynamoDB**: Tables must be created before use
- **EC2**: Already configured with `aws configure`

### MySQL Connection (UNIFIED APPROACH - 2025-11-19)

**IMPORTANT**: All MySQL connections now use `SSHCommandMySQLService` (single unified approach)

- **File**: `db/ssh_command_mysql_service.py`
- **Connection**: SSH tunnel to legacy database (via command execution)
- **Access**: READ-ONLY - never write to MySQL
- **Configuration**: Hardcoded in service classes (not in .env)
  ```python
  ssh_config = {
      'host': '52.14.57.81',
      'port': 22,
      'username': 'ubuntu',
      'key_file': 'key/Encoreepad_ppk.pem'
  }
  mysql_config = {
      'user': 'root',
      'password': 'bOuDCUvd8',
      'database': 'encoree3_mobile'
  }
  ```

**Used By**:
- ✅ Authentication system (`services/auth_service.py`)
- ✅ Phase 4 Feedback system (`services/mysql_service.py`)

**DO NOT** create new MySQL connection implementations!
**See**: `docs/TESTING/MYSQL-UNIFICATION-COMPLETE.md` for details

### Brainium API

- ✅ All known issues resolved by Brainium team
- Provides venue/restaurant data
- Used in Phase 8 for provisioning

## 📚 Key Documentation

**Must Read**:

1. `/docs/SUPERVISOR/CURRENT-STATUS.md` - Current project status
2. `/docs/SUPERVISOR/PHASE-ROADMAP.md` - Development roadmap
3. `/docs/SUPERVISOR/WORKER-PROMPTS/` - Implementation prompts

**Reference**:

- `/docs/investigation/MYSQL-DATABASE-ANALYSIS-SUMMARY.md` - Database insights
- `/docs/investigation/VENUES-API-VALIDATION-REPORT.md` - Brainium API details
- `/docs/plan-mockup-to-frontend/` - Conversion strategy
- `/encore_archive/mockup-backup/.context.md` - Original mockup context (archived)

## 🔄 Version Control

```bash
# Project uses git
git status
git add .
git commit -m "feat: description"
git push

# Sync supervisor docs (if script exists)
./sync-supervisor.sh push
```

## 🆘 Common Issues & Solutions

### Backend Won't Start

- Check `.env` file exists in `/encore_backend`
- Verify MySQL SSH key permissions: `chmod 600 /path/to/key`
- Check AWS credentials: `aws sts get-caller-identity`

### Frontend API Errors

- Verify backend is running on port 8000
- Check `REACT_APP_API_BASE_URL` in `.env.local`
- Toggle `REACT_APP_USE_MOCK_API` for debugging

### DynamoDB Errors

- Run table creation scripts in `/encore_backend/scripts/`
- Verify AWS credentials and region

### JWT Token Issues

- Check token expiry (15 min for access, 7 days for refresh)
- Verify `JWT_SECRET_KEY` matches between frontend and backend
- Check localStorage for stored tokens

## 💡 AI Assistant Guidelines

When working on this project:

1. **Always check** `/docs/SUPERVISOR/CURRENT-STATUS.md` first
2. **Use existing patterns** - don't reinvent the wheel
3. **Maintain mock API support** - essential for development
4. **Follow the phases** - they build on each other
5. **Test incrementally** - verify each component works
6. **Document changes** - update status and roadmap
7. **Use worker prompts** - they contain detailed instructions

## 🎯 Current Priority

**TODAY**: Testing Phase 1-4

- Backend endpoints validation
- Frontend integration testing
- Bug documentation and fixes

**NEXT**: Phase 5 (AI Generation) after testing complete

---

*Last Updated: 2025-11-06*
*Project Lead: AI Supervisor Framework*
*Status: 58% Complete - Testing Phase*

**For detailed status, see `/docs/SUPERVISOR/CURRENT-STATUS.md`**
