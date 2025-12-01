# Worker Prompt: Fix Browser Automation Login Issue (ISSUE-002)

## 🎯 Objective

Fix the login form so that browser automation tools can successfully authenticate users. Currently, React controlled inputs don't update state when the browser automation types into them, causing login to fail with empty credentials.

---

## 📋 Context

### The Problem

The Cursor browser automation tool types text into input fields, but **React's `onChange` handlers don't fire**. This means:

1. User types `tjrottet@encoreloyalty.net` into email field
2. Text appears visually in the input
3. But React state `email` remains `""`
4. Form submits with empty email → 422 validation error

### Why This Happens

React controlled inputs (`value={state}` + `onChange={setState}`) rely on synthetic events. Browser automation tools manipulate the DOM directly without triggering these events, so React never knows the value changed.

### Evidence

- **Demo buttons work**: They call `setState` directly, proving the API and auth flow are correct
- **curl works**: `curl -X POST /api/v1/auth/login` with valid JSON succeeds
- **Typing fails**: Browser automation typing → 422 "email must have @-sign" (empty string sent)

---

## 🔧 Implementation Tasks

### Task 1: Add Ref-Based Fallback to Login Form

Modify `Login.tsx` to read DOM values as fallback when React state is empty.

**File**: `encore_frontend/src/pages/Login.tsx`

**Current Code** (lines 6-29):
```typescript
const Login: React.FC = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const { login, error, clearError, loading: authLoading } = useAuth();
  const navigate = useNavigate();

  // ... demo users ...

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    clearError();

    try {
      await login(email, password);
      navigate('/');
    } catch (err) {
      console.error('Login failed:', err);
    }
  };
```

**Updated Code**:
```typescript
import React, { useState, useRef } from 'react';
// ... other imports ...

const Login: React.FC = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const { login, error, clearError, loading: authLoading } = useAuth();
  const navigate = useNavigate();
  
  // Refs for fallback DOM value reading (supports browser automation)
  const emailRef = useRef<HTMLInputElement>(null);
  const passwordRef = useRef<HTMLInputElement>(null);

  // ... demo users ...

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    clearError();

    // Use React state if available, otherwise read from DOM directly
    // This fallback supports browser automation tools that don't trigger onChange
    const emailValue = email || emailRef.current?.value || '';
    const passwordValue = password || passwordRef.current?.value || '';

    if (!emailValue || !passwordValue) {
      console.error('Login failed: Empty credentials');
      return;
    }

    try {
      await login(emailValue, passwordValue);
      navigate('/');
    } catch (err) {
      console.error('Login failed:', err);
    }
  };
```

**Also update the input elements** (around lines 49-72):
```tsx
<input
  ref={emailRef}  // ADD THIS
  id="email"
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-primary focus:border-transparent transition-shadow"
  placeholder="Enter your email"
  required
/>

// ... and for password ...

<input
  ref={passwordRef}  // ADD THIS
  id="password"
  type="password"
  value={password}
  onChange={(e) => setPassword(e.target.value)}
  className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-primary focus:border-transparent transition-shadow"
  placeholder="Enter your password"
  required
/>
```

---

### Task 2: Create Demo User in DynamoDB

Create a script to add a demo user that matches the demo button credentials.

**File**: `encore_backend/scripts/add_demo_user.py` (NEW)

```python
"""
Add Demo User to DynamoDB

Creates a demo admin user in DynamoDB for testing purposes.
This user matches the demo button credentials in the frontend.

Run: python scripts/add_demo_user.py
"""

import boto3
import bcrypt
import sys
import os

# Add parent directory to path
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..'))

def add_demo_user():
    """Add demo admin user to DynamoDB"""
    
    dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
    table = dynamodb.Table('encore_venue_users')
    
    # Demo user credentials (matches frontend demo buttons)
    email = 'admin@encore.com'
    password = 'password123'
    
    # Hash the password
    password_hash = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt()).decode('utf-8')
    
    print(f"\n{'='*60}")
    print(f"Adding Demo User to DynamoDB")
    print(f"{'='*60}\n")
    print(f"Email: {email}")
    print(f"Password: {password}")
    print(f"Role: sysadmin (for admin dashboard access)")
    
    try:
        # Check if user already exists
        response = table.get_item(Key={'email': email})
        if 'Item' in response:
            print(f"\n⚠️  User '{email}' already exists. Updating...")
        
        # Add/update user
        table.put_item(Item={
            'email': email,
            'password_hash': password_hash,
            'name': 'Demo Admin',
            'role': 'sysadmin',
            'venue_id': 0,  # No specific venue (global admin)
            'is_active': True,
            'created_at': '2025-11-26T00:00:00Z',
            'updated_at': '2025-11-26T00:00:00Z'
        })
        
        print(f"\n✅ Demo user added successfully!")
        print(f"\nYou can now login with:")
        print(f"  Email: {email}")
        print(f"  Password: {password}")
        
        return True
        
    except Exception as e:
        print(f"\n❌ Error adding demo user: {e}")
        return False


if __name__ == "__main__":
    success = add_demo_user()
    sys.exit(0 if success else 1)
```

---

### Task 3: Update Auth Service for Demo User

Ensure the auth service can authenticate the demo user from DynamoDB.

**File**: `encore_backend/services/auth_service.py`

The dual authentication is already implemented. Verify that:
1. MySQL lookup happens first (for real admins)
2. DynamoDB lookup happens second (for demo/venue users)
3. Password comparison uses bcrypt

The current implementation should already support this. Test after adding the demo user.

---

## ✅ Verification Steps

### Step 1: Verify Ref-Based Login Works

```bash
# Start frontend if not running
cd encore_frontend && npm start

# In browser automation:
# 1. Navigate to http://localhost:3000/login
# 2. Type email slowly
# 3. Type password slowly  
# 4. Click Sign In
# 5. Should redirect to dashboard (not 422 error)
```

### Step 2: Verify Demo User Works

```bash
# Add demo user
cd encore_backend
source ../venv/bin/activate
python scripts/add_demo_user.py

# Test via curl
curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@encore.com","password":"password123"}' | jq .

# Should return token and user object
```

### Step 3: Verify Demo Button Works in Browser

1. Navigate to http://localhost:3000/login
2. Click "Global Admin" demo button
3. Click "Sign In"
4. Should redirect to admin dashboard

---

## 📁 Files Modified

| File | Change |
|------|--------|
| `encore_frontend/src/pages/Login.tsx` | Add refs and fallback DOM reading |
| `encore_backend/scripts/add_demo_user.py` | New script to add demo user |

---

## 🧪 Testing Checklist

- [ ] Login form compiles without errors
- [ ] Ref-based fallback reads DOM values correctly
- [ ] Demo user added to DynamoDB
- [ ] Demo user can authenticate via curl
- [ ] Demo button login works in browser
- [ ] Browser automation can login (primary goal)
- [ ] Real user login still works
- [ ] Impersonation feature accessible after login

---

## ⚠️ Important Notes

1. **Don't remove controlled input behavior** - The refs are a FALLBACK, not a replacement
2. **Keep onChange handlers** - Normal React behavior should still work
3. **Demo user is for testing only** - Consider removing in production or using env flag
4. **The underlying issue is browser automation limitation** - This is a workaround

---

## 📚 Related Files

- Issue documentation: `/docs/SUPERVISOR/ISSUES/ISSUE-002-BROWSER-AUTOMATION-REACT-INPUTS.md`
- Auth service: `/encore_backend/services/auth_service.py`
- DynamoDB user table setup: `/encore_backend/scripts/create_venue_users_table.py`
- Existing venue user seeding: `/encore_backend/scripts/seed_venue_users.py`

---

## 🎯 Success Criteria

1. Browser automation can type credentials and login successfully
2. Demo button login works
3. Real user login (tjrottet@encoreloyalty.net) still works
4. No TypeScript errors
5. No console errors during login flow

