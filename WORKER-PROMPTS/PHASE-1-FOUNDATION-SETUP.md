# Phase 1: Foundation Setup - Worker Prompt
## Mockup to Frontend Conversion - Encore Loyalty AI Feedback System

**Assigned To**: Cursor Chat (Claude Sonnet 4.5)
**Phase**: 1 of 8 - Foundation Setup
**Duration**: 2-3 hours
**Date**: November 5, 2025
**Priority**: P1 - Critical (blocks all other phases)

---

## 🎯 MISSION

Transform the complete React mockup (`./mockup`) into a production-ready frontend (`./encore_frontend`) with proper infrastructure for real API integration.

**This Phase**: Set up the foundation - rename project, install dependencies, create configuration files, and establish utility infrastructure.

---

## 📋 CONTEXT

### What You're Working With

**Current State**:
- `./mockup` - Complete React TypeScript application (100% functional)
  - 29 components, 8,000+ lines of code
  - Uses mock data (localStorage)
  - Mock authentication
  - Tailwind CSS, React Router, Recharts
  - All features validated and working

**Target State**:
- `./encore_frontend` - Production frontend
  - Same UI/UX (zero visual changes)
  - Real API integration (axios + JWT)
  - Environment-based configuration
  - Production-grade error handling
  - Proper logging and monitoring

**Backend API**:
- FastAPI at `./encore_backend`
- Will provide 40+ endpoints
- JWT authentication
- Base URL: `http://localhost:8000` (dev), TBD (production)

**External APIs**:
- Brainium Venues API: `https://www.encore-ai.com/report/api`

---

## 🎯 PHASE 1 OBJECTIVES

1. ✅ Rename `mockup` → `encore_frontend` (preserve git history)
2. ✅ Install new dependencies (axios, jwt-decode, etc.)
3. ✅ Create environment configuration files
4. ✅ Create API configuration module
5. ✅ Create utility modules (error handling, logging, storage)
6. ✅ Update .gitignore for security
7. ✅ Verify app still runs with existing mock data

**Critical**: After this phase, the app must still run exactly as before, just with new infrastructure in place.

---

## 📁 PROJECT STRUCTURE

```
/home/ec2-user/encore-loyalty-new-system/
├── mockup/                    ← Convert this
│   ├── src/
│   │   ├── services/
│   │   │   ├── mockApi.ts    ← Will replace with real API
│   │   │   └── authService.ts ← Will update with JWT
│   │   ├── components/
│   │   ├── contexts/
│   │   ├── pages/
│   │   └── types/
│   └── package.json
├── encore_backend/            ← Backend (already exists)
└── docs/
    └── plan-mockup-to-frontend/  ← Your reference docs
        ├── 00-START-HERE.md
        ├── 01-DETAILED-IMPLEMENTATION-PLAN.md
        └── 04-BACKEND-REQUIREMENTS.md
```

---

## 🛠️ DETAILED TASKS

### Task 1.1: Rename Directory (Preserve Git History)

**Objective**: Rename `mockup` to `encore_frontend` while preserving git history.

**Commands**:
```bash
cd /home/ec2-user/encore-loyalty-new-system

# Create backup first (safety)
cp -r mockup mockup-backup

# Rename (git will track this)
git mv mockup encore_frontend

# Verify
ls -la | grep encore_frontend

# Stage the rename
git add -A
```

**Verification**:
- [ ] `encore_frontend` directory exists
- [ ] Git shows rename (not delete + add)
- [ ] Backup exists at `mockup-backup`

---

### Task 1.2: Update package.json

**File**: `encore_frontend/package.json`

**Change**:
```json
{
  "name": "encore_frontend",  // Changed from "mockup"
  "version": "0.1.0",
  "private": true,
  // ... rest unchanged
}
```

**Verification**:
- [ ] Name changed to "encore_frontend"
- [ ] Version and other fields unchanged

---

### Task 1.3: Install New Dependencies

**Commands**:
```bash
cd encore_frontend

# Core dependencies (required)
npm install axios@^1.6.0
npm install jwt-decode@^4.0.0

# Optional but recommended
npm install react-query@^3.39.3
npm install react-toastify@^9.1.3

# Verify installation
npm list axios jwt-decode react-query react-toastify
```

**Purpose**:
- `axios`: HTTP client for API calls
- `jwt-decode`: Decode JWT tokens client-side
- `react-query`: API state management (caching, refetching)
- `react-toastify`: Toast notifications for user feedback

**Verification**:
- [ ] All packages installed without errors
- [ ] `package.json` updated with new dependencies
- [ ] `package-lock.json` regenerated

---

### Task 1.4: Create Environment Files

**File 1**: `encore_frontend/.env.development`
```env
# encore_backend API (local development)
REACT_APP_API_BASE_URL=http://localhost:8000
REACT_APP_API_TIMEOUT=30000

# Brainium Venues API
REACT_APP_BRAINIUM_API_URL=https://www.encore-ai.com/report/api
REACT_APP_BRAINIUM_API_KEY=

# Feature Flags
REACT_APP_ENABLE_ANALYTICS=true
REACT_APP_ENABLE_DEBUG_LOGGING=true
REACT_APP_USE_MOCK_API=true

# App Configuration
REACT_APP_VERSION=1.0.0
REACT_APP_ENVIRONMENT=development
```

**File 2**: `encore_frontend/.env.production`
```env
# encore_backend API (production)
REACT_APP_API_BASE_URL=https://api.predictif.com
REACT_APP_API_TIMEOUT=30000

# Brainium Venues API
REACT_APP_BRAINIUM_API_URL=https://www.encore-ai.com/report/api
# IMPORTANT: Set this via environment variable in your deployment pipeline
# DO NOT commit the actual API key to git
# React does NOT support ${VAR} substitution - leave empty and set via CI/CD
REACT_APP_BRAINIUM_API_KEY=

# Feature Flags
REACT_APP_ENABLE_ANALYTICS=true
REACT_APP_ENABLE_DEBUG_LOGGING=false
REACT_APP_USE_MOCK_API=false

# App Configuration
REACT_APP_VERSION=1.0.0
REACT_APP_ENVIRONMENT=production
```

**File 3**: `encore_frontend/.env.local.example`
```env
# Copy this file to .env.local and fill in your values
# .env.local is gitignored and overrides other env files

# Local Development Overrides
REACT_APP_API_BASE_URL=http://localhost:8000
REACT_APP_BRAINIUM_API_KEY=your-brainium-api-key-here

# Toggle Mock API (useful during backend development)
# Set to false to use real API, true to use mockApi.ts
REACT_APP_USE_MOCK_API=true
```

**Important Notes**:
- `.env.development` - Used during `npm start`
- `.env.production` - Used during `npm run build`
- `.env.local.example` - Template for developers (not used directly)
- Developers should copy `.env.local.example` → `.env.local` for local overrides
- `.env.local` should be in `.gitignore` (don't commit API keys)

**⚠️ CRITICAL - Environment Variables in Production**:
- React does **NOT** support shell variable substitution (e.g., `${VAR}`)
- Do NOT use syntax like `REACT_APP_API_KEY=${SOME_VAR}` - it will set the literal string
- For production deployments, set environment variables via:
  - **Vercel**: Environment Variables in project settings
  - **Netlify**: Build environment variables
  - **AWS Amplify**: Environment variables in app settings
  - **Docker**: Pass via `-e` flag or docker-compose environment section
  - **Build script**: Use a pre-build script to substitute values before build

**Verification**:
- [ ] All three .env files created
- [ ] REACT_APP_USE_MOCK_API=true in development (keeps existing behavior)
- [ ] Production URLs are correct

---

### Task 1.5: Create Configuration Module

**File**: `encore_frontend/src/config/api.config.ts`
```typescript
export const API_CONFIG = {
  // Base URLs
  ENCORE_BACKEND_URL: process.env.REACT_APP_API_BASE_URL || 'http://localhost:8000',
  BRAINIUM_API_URL: process.env.REACT_APP_BRAINIUM_API_URL || 'https://www.encore-ai.com/report/api',

  // API Keys
  BRAINIUM_API_KEY: process.env.REACT_APP_BRAINIUM_API_KEY || '',

  // Timeouts
  API_TIMEOUT: parseInt(process.env.REACT_APP_API_TIMEOUT || '30000', 10),

  // Feature Flags
  ENABLE_ANALYTICS: process.env.REACT_APP_ENABLE_ANALYTICS === 'true',
  ENABLE_DEBUG_LOGGING: process.env.REACT_APP_ENABLE_DEBUG_LOGGING === 'true',
  USE_MOCK_API: process.env.REACT_APP_USE_MOCK_API === 'true',

  // App Info
  VERSION: process.env.REACT_APP_VERSION || '1.0.0',
  ENVIRONMENT: process.env.REACT_APP_ENVIRONMENT || 'development',

  // Auth
  AUTH_TOKEN_KEY: 'encore_auth_token',
  AUTH_USER_KEY: 'encore_user',
  TOKEN_REFRESH_THRESHOLD: 5 * 60 * 1000, // 5 minutes before expiry
};

export const isDevelopment = () => API_CONFIG.ENVIRONMENT === 'development';
export const isProduction = () => API_CONFIG.ENVIRONMENT === 'production';
```

**File**: `encore_frontend/src/config/index.ts`
```typescript
export * from './api.config';
```

**Purpose**:
- Centralized configuration management
- Type-safe access to environment variables
- Default values for safety
- Helper functions for environment checks

**Verification**:
- [ ] `src/config/` directory created
- [ ] Both files created with correct TypeScript
- [ ] No compilation errors

---

### Task 1.6: Create Utility Modules

#### Utility 1: Error Handler

**File**: `encore_frontend/src/utils/errorHandler.ts`
```typescript
import { AxiosError } from 'axios';

export interface ApiError {
  status: number;
  message: string;
  details?: any;
  code?: string;
}

export const handleApiError = (error: unknown): ApiError => {
  if ((error as any).isAxiosError) {
    const axiosError = error as AxiosError<any>;

    if (axiosError.response) {
      // Server responded with error status
      return {
        status: axiosError.response.status,
        message: axiosError.response.data?.message || axiosError.response.data?.detail || 'Server error occurred',
        details: axiosError.response.data,
        code: axiosError.response.data?.code
      };
    } else if (axiosError.request) {
      // Request made but no response
      return {
        status: 0,
        message: 'Network error - Unable to reach server. Please check your connection.',
        code: 'NETWORK_ERROR'
      };
    }
  }

  // Generic error
  return {
    status: 0,
    message: error instanceof Error ? error.message : 'An unexpected error occurred',
    code: 'UNKNOWN_ERROR'
  };
};

export const getErrorMessage = (error: unknown): string => {
  const apiError = handleApiError(error);
  return apiError.message;
};

export const isAuthError = (error: ApiError): boolean => {
  return error.status === 401 || error.status === 403;
};

export const isNetworkError = (error: ApiError): boolean => {
  return error.status === 0 && error.code === 'NETWORK_ERROR';
};
```

#### Utility 2: Logger

**File**: `encore_frontend/src/utils/logger.ts`
```typescript
import { API_CONFIG } from '../config';

export enum LogLevel {
  DEBUG = 'DEBUG',
  INFO = 'INFO',
  WARN = 'WARN',
  ERROR = 'ERROR'
}

class Logger {
  private enabled: boolean;

  constructor() {
    this.enabled = API_CONFIG.ENABLE_DEBUG_LOGGING;
  }

  private log(level: LogLevel, message: string, data?: any) {
    if (!this.enabled && level === LogLevel.DEBUG) return;

    const timestamp = new Date().toISOString();
    const prefix = `[${timestamp}] [${level}]`;

    switch (level) {
      case LogLevel.DEBUG:
        console.log(prefix, message, data || '');
        break;
      case LogLevel.INFO:
        console.info(prefix, message, data || '');
        break;
      case LogLevel.WARN:
        console.warn(prefix, message, data || '');
        break;
      case LogLevel.ERROR:
        console.error(prefix, message, data || '');
        break;
    }
  }

  debug(message: string, data?: any) {
    this.log(LogLevel.DEBUG, message, data);
  }

  info(message: string, data?: any) {
    this.log(LogLevel.INFO, message, data);
  }

  warn(message: string, data?: any) {
    this.log(LogLevel.WARN, message, data);
  }

  error(message: string, error?: any) {
    this.log(LogLevel.ERROR, message, error);
  }

  // API call logging
  logApiRequest(method: string, url: string, data?: any) {
    this.debug(`API Request: ${method} ${url}`, data);
  }

  logApiResponse(method: string, url: string, status: number, data?: any) {
    this.debug(`API Response: ${method} ${url} [${status}]`, data);
  }

  logApiError(method: string, url: string, error: any) {
    this.error(`API Error: ${method} ${url}`, error);
  }
}

export const logger = new Logger();
```

#### Utility 3: Storage Helper

**File**: `encore_frontend/src/utils/storage.ts`
```typescript
import { API_CONFIG } from '../config';

/**
 * Safe localStorage wrapper with error handling
 */
export const storage = {
  set(key: string, value: any): void {
    try {
      const serialized = JSON.stringify(value);
      localStorage.setItem(key, serialized);
    } catch (error) {
      console.error(`Error saving to localStorage: ${key}`, error);
    }
  },

  get<T>(key: string): T | null {
    try {
      const serialized = localStorage.getItem(key);
      if (serialized === null) return null;
      return JSON.parse(serialized) as T;
    } catch (error) {
      console.error(`Error reading from localStorage: ${key}`, error);
      return null;
    }
  },

  remove(key: string): void {
    try {
      localStorage.removeItem(key);
    } catch (error) {
      console.error(`Error removing from localStorage: ${key}`, error);
    }
  },

  clear(): void {
    try {
      localStorage.clear();
    } catch (error) {
      console.error('Error clearing localStorage', error);
    }
  },

  // Auth-specific helpers
  setAuthToken(token: string): void {
    this.set(API_CONFIG.AUTH_TOKEN_KEY, token);
  },

  getAuthToken(): string | null {
    return this.get<string>(API_CONFIG.AUTH_TOKEN_KEY);
  },

  removeAuthToken(): void {
    this.remove(API_CONFIG.AUTH_TOKEN_KEY);
  },

  setUser(user: any): void {
    this.set(API_CONFIG.AUTH_USER_KEY, user);
  },

  getUser<T>(): T | null {
    return this.get<T>(API_CONFIG.AUTH_USER_KEY);
  },

  removeUser(): void {
    this.remove(API_CONFIG.AUTH_USER_KEY);
  }
};
```

#### Utility 4: Index Barrel

**File**: `encore_frontend/src/utils/index.ts`
```typescript
export * from './errorHandler';
export * from './logger';
export * from './storage';
```

**Purpose**:
- `errorHandler`: Standardized API error handling
- `logger`: Conditional logging (dev vs production)
- `storage`: Safe localStorage wrapper with auth helpers
- `index`: Barrel export for clean imports

**Verification**:
- [ ] `src/utils/` directory created
- [ ] All 4 files created
- [ ] No TypeScript compilation errors
- [ ] Can import: `import { logger, storage, handleApiError } from '../utils'`

---

### Task 1.7: Update .gitignore

**File**: `encore_frontend/.gitignore`

Add these lines (or ensure they exist):
```gitignore
# dependencies
/node_modules
/.pnp
.pnp.js

# testing
/coverage

# production
/build

# misc
.DS_Store
.env.local
.env.development.local
.env.test.local
.env.production.local

npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Environment files (keep .env.example files)
.env

# IDE
.vscode/
.idea/
*.swp
*.swo

# Server logs
.server.log
.server.pid

# Backups
mockup-backup/
```

**Critical**: Ensure `.env.local` is ignored (don't commit API keys!)

**Verification**:
- [ ] `.gitignore` updated
- [ ] `.env.local` will not be tracked by git
- [ ] Backup directory ignored

---

## 🧪 TESTING & VERIFICATION

### Test 1: Application Still Runs

```bash
cd encore_frontend

# Install dependencies
npm install

# Start development server
npm start
```

**Expected**:
- ✅ App compiles without errors
- ✅ App opens at http://localhost:3000
- ✅ Login page appears
- ✅ Can login with mock credentials
- ✅ All features work exactly as before
- ✅ No console errors

**Mock credentials** (from mockApi.ts):
- Super Admin: `admin@encore.com` / `admin123`
- Global Admin: `global@encore.com` / `global123`
- Venue Manager: `manager@venue1.com` / `manager123`

### Test 2: Environment Variables Load

Add temporary console.log to verify:

**File**: `encore_frontend/src/App.tsx` (add at top of component)
```typescript
import { API_CONFIG } from './config';

console.log('Environment loaded:', {
  environment: API_CONFIG.ENVIRONMENT,
  apiUrl: API_CONFIG.ENCORE_BACKEND_URL,
  useMockApi: API_CONFIG.USE_MOCK_API,
  version: API_CONFIG.VERSION
});
```

**Expected console output**:
```
Environment loaded: {
  environment: "development",
  apiUrl: "http://localhost:8000",
  useMockApi: true,
  version: "1.0.0"
}
```

**Remove the console.log after verification.**

### Test 3: Utilities Import Correctly

Create temporary test file:

**File**: `encore_frontend/src/utils/test.ts`
```typescript
import { logger, storage, handleApiError } from './index';
import { API_CONFIG } from '../config';

// Test logger
logger.info('Logger test');
logger.debug('Debug test');

// Test storage
storage.set('test', { foo: 'bar' });
const retrieved = storage.get('test');
console.log('Storage test:', retrieved);

// Test error handler
const mockError = new Error('Test error');
const handled = handleApiError(mockError);
console.log('Error handler test:', handled);

// Test config
console.log('Config test:', API_CONFIG.VERSION);
```

Run: `npx ts-node src/utils/test.ts`

**Expected**: No errors, output shows logger and utilities work.

**Delete `test.ts` after verification.**

### Test 4: Git Status

```bash
git status
```

**Expected**:
- Renamed files show as renamed (not deleted + added)
- New files are untracked
- No unexpected changes

### Test 5: Build Production

```bash
npm run build
```

**Expected**:
- ✅ Build completes successfully
- ✅ No warnings or errors
- ✅ `build/` directory created
- ✅ All chunks generated

---

## ✅ PHASE 1 COMPLETION CRITERIA

Before marking this phase complete, verify ALL of these:

**Files Created**:
- [ ] `encore_frontend/` directory exists (renamed from mockup)
- [ ] `encore_frontend/.env.development` created
- [ ] `encore_frontend/.env.production` created
- [ ] `encore_frontend/.env.local.example` created
- [ ] `encore_frontend/src/config/api.config.ts` created
- [ ] `encore_frontend/src/config/index.ts` created
- [ ] `encore_frontend/src/utils/errorHandler.ts` created
- [ ] `encore_frontend/src/utils/logger.ts` created
- [ ] `encore_frontend/src/utils/storage.ts` created
- [ ] `encore_frontend/src/utils/index.ts` created
- [ ] `encore_frontend/.gitignore` updated

**Dependencies Installed**:
- [ ] axios@^1.6.0
- [ ] jwt-decode@^4.0.0
- [ ] react-query@^3.39.3 (optional)
- [ ] react-toastify@^9.1.3 (optional)

**Functionality**:
- [ ] App runs without errors: `npm start`
- [ ] Can login with mock credentials
- [ ] All existing features work
- [ ] Environment variables load correctly
- [ ] Utilities can be imported
- [ ] Production build succeeds: `npm run build`
- [ ] No TypeScript compilation errors
- [ ] No console errors (except intentional debug logs)

**Git**:
- [ ] Rename tracked properly in git
- [ ] `.env.local` in .gitignore
- [ ] Backup exists at `mockup-backup/`

**Feature Flag**:
- [ ] `REACT_APP_USE_MOCK_API=true` in development
- [ ] App uses mock data (existing behavior preserved)
- [ ] Ready to toggle to real API in Phase 2

---

## 📚 REFERENCE DOCUMENTS

**Essential Reading**:
1. `/docs/plan-mockup-to-frontend/01-DETAILED-IMPLEMENTATION-PLAN.md` - Complete conversion plan
2. `/docs/plan-mockup-to-frontend/04-BACKEND-REQUIREMENTS.md` - API endpoints we'll connect to
3. `/mockup/.context.md` - Complete mockup documentation
4. `/mockup/src/services/mockApi.ts` - Current mock implementation (will replace in Phase 3)

**For Questions**:
- Backend API base URL: Check `/encore_backend/README.md`
- Mockup features: Check `/docs/plan-mockup/FEATURE-SHOWCASE.md`
- Type definitions: Check `/mockup/src/types/index.ts`

---

## 🚨 IMPORTANT NOTES

### DO NOT Change These Yet
- ❌ Don't modify `mockApi.ts` (Phase 3)
- ❌ Don't modify `authService.ts` (Phase 2)
- ❌ Don't modify React contexts (Phase 5)
- ❌ Don't modify components (Phase 6)
- ❌ Don't touch UI/styling (stays exactly the same)

### Must Keep Working
- ✅ All existing functionality must work after this phase
- ✅ Mock data must still work (REACT_APP_USE_MOCK_API=true)
- ✅ No visual changes to UI
- ✅ No breaking changes to existing code

### Safety
- Backup created before rename
- Git tracks the rename properly
- Can rollback if needed
- .env.local not committed (API keys safe)

---

## 🎯 SUCCESS DEFINITION

**Phase 1 is complete when**:
1. Directory renamed to `encore_frontend`
2. All infrastructure files created
3. Dependencies installed
4. App runs identically to before
5. Environment configuration working
6. Utilities ready for Phase 2
7. No breaking changes
8. Ready to implement real API integration

**Estimated Time**: 2-3 hours

**Next Phase**: Phase 2 - Authentication Implementation (JWT, real login)

---

## 💬 REPORTING BACK

After completing this phase, report:

1. **What was completed**: List of all files created/modified
2. **Test results**: All 5 tests passed?
3. **Issues encountered**: Any problems or blockers?
4. **Time taken**: Actual time vs estimate
5. **Ready for Phase 2**: Yes/No and why

**Example Report**:
```
✅ PHASE 1 COMPLETE

Files created: 14 files (3 .env, 2 config, 4 utils, 1 .gitignore update)
Dependencies: axios, jwt-decode, react-query, react-toastify
Tests: All 5 tests passed
  - App runs: ✅
  - Environment loads: ✅
  - Utilities work: ✅
  - Git status clean: ✅
  - Production builds: ✅
Time: 2.5 hours
Issues: None
Ready for Phase 2: YES

Note: App runs identically to mockup. Feature flag set to use mock API.
Ready to implement real authentication in Phase 2.
```

---

**READY TO START?** Copy this entire prompt into your Cursor chat and begin Phase 1!

Good luck! 🚀
