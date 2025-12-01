# ISSUE-002: Browser Automation Cannot Type Into React Controlled Inputs

## Status: 🔴 BLOCKING

**Created**: 2025-11-26
**Priority**: CRITICAL (Blocks all browser-based testing)
**Component**: Frontend Testing / Browser Automation

---

## Problem Summary

The Cursor browser automation tool (MCP browser tools) cannot properly interact with React controlled input components. When typing into input fields, the text appears visually but **React's state is not updated**, causing form submissions to send empty values.

---

## Symptoms

1. **Login form fails with 422 error** (Pydantic validation: "email must have an @-sign")
2. Text appears in input fields visually after typing
3. React state remains empty (`""`) when form is submitted
4. API receives empty `email` and `password` fields
5. Demo account buttons (which use `setState` directly) work correctly - get 401 instead of 422

---

## Root Cause Analysis

React controlled inputs work by:
1. User types → triggers `onChange` event
2. `onChange` handler calls `setState(newValue)`
3. React re-renders with new value
4. Input displays the state value

**The browser automation tool bypasses step 1** - it manipulates the DOM directly without triggering React's synthetic event system. The `onChange` handlers never fire, so React state remains at initial value (`""`).

### Evidence

| Test | Result | Conclusion |
|------|--------|------------|
| Type with `slowly: false` | 422 error | State not updated |
| Type with `slowly: true` | 422 error | Still doesn't trigger onChange |
| Click demo button + submit | 401 error | setState works, API receives valid email |
| Direct curl to API | 200 success | Backend works correctly |

---

## Affected Code

### Login Component (`encore_frontend/src/pages/Login.tsx`)

```typescript
const [email, setEmail] = useState('');  // State stays empty
const [password, setPassword] = useState('');

// These onChange handlers never fire during automation
<input
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}  // Never called
  ...
/>
```

### Console Evidence

```
[INFO] AuthService: Logging in [object Object]
[INFO] Attempting login [object Object]
[ERROR] AuthService: Login failed [object Object]
```

The `[object Object]` indicates the logger is receiving the metadata object `{ email }` but the actual email value is empty.

### Network Evidence

```json
// Request to /api/v1/auth/login returns 422
{
  "detail": [{
    "type": "value_error",
    "loc": ["body", "email"],
    "msg": "value is not a valid email address: An email address must have an @-sign.",
    "input": ""  // Empty string!
  }]
}
```

---

## Impact

- ❌ Cannot test login flow via browser automation
- ❌ Cannot test any authenticated features via browser
- ❌ Cannot test impersonation feature
- ❌ Cannot do end-to-end testing
- ✅ API testing via curl works
- ✅ Manual browser testing works

---

## Potential Solutions

### Option A: Modify Login Component (Recommended)

Add uncontrolled input support alongside controlled:

```typescript
// Use refs to read DOM values directly as fallback
const emailRef = useRef<HTMLInputElement>(null);
const passwordRef = useRef<HTMLInputElement>(null);

const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  
  // Try controlled state first, fall back to DOM value
  const emailValue = email || emailRef.current?.value || '';
  const passwordValue = password || passwordRef.current?.value || '';
  
  await login(emailValue, passwordValue);
};

<input
  ref={emailRef}
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  ...
/>
```

### Option B: Create Test User with Demo Credentials

Add a real user account matching the demo button credentials:

```bash
# Add user to DynamoDB with email: admin@encore.com, password: password123
python scripts/add_demo_user.py
```

Then demo buttons would work for authentication.

### Option C: Use JavaScript Injection

Inject JavaScript to set React state directly:

```javascript
// Would need to expose state setter or use React DevTools protocol
window.__setLoginEmail('test@example.com');
```

### Option D: Direct localStorage Authentication

Skip the login form entirely for testing:

```bash
# Get token via curl
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"tjrottet@encoreloyalty.net","password":"password222"}' | jq -r '.token')

# Inject into browser localStorage
# Navigate to app with pre-set authentication
```

---

## Recommended Fix: Option A + Option B

1. **Implement Option A** - Makes the login form more robust
2. **Implement Option B** - Provides working test credentials for demo buttons

This gives two paths to successful login during automated testing.

---

## Files to Modify

1. `encore_frontend/src/pages/Login.tsx` - Add ref-based fallback
2. `encore_backend/scripts/add_demo_user.py` - New script to add demo users
3. `encore_backend/services/auth_service.py` - Ensure DynamoDB user lookup works

---

## Verification Steps

After fix:
1. Navigate to login page
2. Type credentials (or click demo button)
3. Submit form
4. Verify redirect to dashboard
5. Verify user is authenticated

---

## Related Issues

- None

## Related Documentation

- `/docs/IMPLEMENTATION/VENUE-TESTING-STRATEGY.md`
- `/docs/IMPLEMENTATION/DYNAMODB-VENUE-USERS-PLAN.md`

