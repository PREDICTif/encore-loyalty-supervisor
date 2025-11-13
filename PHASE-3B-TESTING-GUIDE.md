# Phase 3B Frontend Settings Integration - Testing Guide

## ✅ Implementation Complete

All Phase 3B tasks have been successfully completed:

1. ✅ Settings types created (`src/types/settings.ts`)
2. ✅ Settings API service created (`src/api/settingsApi.ts`)
3. ✅ Mock settings data created (`src/mockData/settingsData.ts`)
4. ✅ Settings custom hook created (`src/hooks/useSettings.ts`)
5. ✅ Settings page UI created (`src/pages/venue/Settings.tsx` + CSS)
6. ✅ API exports centralized (`src/api/index.ts`)
7. ✅ Frontend compiles without errors (`npm run build` successful)
8. ✅ No TypeScript or linter errors
9. ✅ CHANGELOG.md updated

## 🧪 Manual Testing Instructions

### Prerequisites

- Node.js and npm installed
- Backend running (for real API mode) or mock mode enabled (for standalone testing)

### Test 1: Mock API Mode (No Backend Required)

This tests the frontend settings integration without needing the backend running.

```bash
# 1. Navigate to frontend directory
cd /home/ec2-user/encore-loyalty-new-system/encore_frontend

# 2. Verify .env.local has mock mode enabled
cat .env.local
# Should show: REACT_APP_USE_MOCK_API=true

# 3. Install dependencies (if not already done)
npm install

# 4. Start development server
npm start

# 5. Open browser to http://localhost:3000
# 6. Login with test credentials
# 7. Navigate to Settings page
```

**Expected Behavior in Mock Mode:**

- ✅ Settings page loads with mock data for venue_001 (Downtown Bistro)
- ✅ Console shows `[MOCK] Getting settings for venue: venue_001`
- ✅ All settings sections display (AI, Email, Notifications)
- ✅ Initial values match mock data:
  - AI enabled: ✓
  - Temperature: 0.8
  - Max tokens: 750
  - Email enabled: ✓
  - From email: noreply@downtownbistro.com
- ✅ Changing any setting shows "Saving..." indicator
- ✅ Success message appears after save
- ✅ Console shows `[MOCK] Updating settings for venue: venue_001`
- ✅ Changes persist during the session
- ✅ Refresh page and settings load again from mock data

### Test 2: Real API Mode (Backend Required)

This tests the full integration with the backend settings API.

```bash
# 1. Start backend (in separate terminal)
cd /home/ec2-user/encore-loyalty-new-system/encore_backend
source ../venv/bin/activate
python main.py
# Backend should be running on http://localhost:8000

# 2. Update frontend to use real API
cd /home/ec2-user/encore-loyalty-new-system/encore_frontend
# Edit .env.local and set: REACT_APP_USE_MOCK_API=false

# 3. Restart frontend
npm start

# 4. Open browser to http://localhost:3000
# 5. Login with real credentials
# 6. Navigate to Settings page
```

**Expected Behavior in Real API Mode:**

- ✅ Settings page loads with settings from backend DynamoDB
- ✅ NO `[MOCK]` prefix in console logs
- ✅ Network tab shows requests to `/api/v1/settings/venue_001`
- ✅ JWT token automatically added to Authorization header
- ✅ Settings display correctly from database
- ✅ Changing any setting makes PUT request to backend
- ✅ Success message appears after successful save
- ✅ Backend responds with updated settings
- ✅ Changes persist across page refreshes (stored in DynamoDB)
- ✅ Last updated timestamp updates correctly

### Test 3: Error Handling

Test various error scenarios:

**A. Backend Unavailable (Real API Mode)**
```bash
# With REACT_APP_USE_MOCK_API=false
# Stop the backend server
# Refresh the settings page
```
**Expected:** Error message displayed with retry button

**B. Invalid Venue ID**
```bash
# Modify Settings.tsx temporarily to use invalid venue ID
const currentVenueId = 'invalid_venue_999';
```
**Expected:** Error message or default settings displayed

**C. Network Timeout**
```bash
# Simulate slow network in browser DevTools
# Try to load or update settings
```
**Expected:** Loading spinner shows, timeout error after 30 seconds

### Test 4: Form Validation & Disabled States

**A. Parent-Child Disabling**
```bash
# 1. Disable AI (uncheck "Enable AI Response Generation")
# Expected: All AI settings inputs become disabled (grayed out)

# 2. Enable AI again
# Expected: All AI settings inputs become enabled

# 3. Disable Email (uncheck "Enable Email Sending")
# Expected: All email settings inputs become disabled
```

**B. Range Validation**
```bash
# 1. Temperature slider: Move to 0.0, 0.5, 1.0
# Expected: Value updates, saves successfully

# 2. Max tokens: Try values 100, 500, 2000
# Expected: All valid values save successfully

# 3. Max tokens: Try value < 100 or > 2000
# Expected: Browser validation prevents invalid input
```

### Test 5: Responsive Design

```bash
# Open Settings page
# Open browser DevTools (F12)
# Toggle device toolbar (Ctrl+Shift+M)
# Test on different screen sizes:
# - Mobile: 375px width
# - Tablet: 768px width
# - Desktop: 1920px width
```

**Expected:**
- ✅ Layout adapts to screen size
- ✅ Form controls remain usable on mobile
- ✅ Text remains readable at all sizes
- ✅ "Saving..." indicator stays visible on all sizes

### Test 6: Authentication Integration

```bash
# 1. Access Settings page while logged out
# Expected: Redirect to /login

# 2. Login and access Settings page
# Expected: Settings load successfully

# 3. Check Network tab -> Settings API request
# Expected: Authorization: Bearer <token> header present

# 4. Wait for token to expire (15 minutes)
# Expected: Token auto-refreshes, settings continue to work
```

## 🔍 Verification Checklist

After all manual testing:

- [ ] Settings page loads in mock mode
- [ ] Settings page loads in real API mode
- [ ] All form controls work (checkboxes, inputs, sliders, textareas)
- [ ] Settings save successfully in both modes
- [ ] Loading states display correctly
- [ ] Error states display correctly
- [ ] Success messages appear and auto-dismiss
- [ ] Parent settings disable children when unchecked
- [ ] Validation works for number inputs
- [ ] Responsive design works on mobile/tablet/desktop
- [ ] JWT authentication headers are sent
- [ ] Token refresh works automatically
- [ ] Console has no errors
- [ ] Network tab shows correct API requests
- [ ] Changes persist in database (real API mode)
- [ ] Mock data updates during session (mock mode)

## 📝 Testing Notes

### Mock Mode vs Real API Mode

**When to use Mock Mode:**
- Frontend-only development
- Backend is not running
- Testing UI/UX without database changes
- Rapid prototyping

**When to use Real API Mode:**
- End-to-end integration testing
- Database persistence testing
- Authentication flow testing
- Production-like testing

### Console Debugging

In Mock Mode, you'll see:
```
[MOCK] Getting settings for venue: venue_001
[MOCK] Updating settings for venue: venue_001 {ai_settings: {...}}
```

In Real API Mode, you'll see:
```
[API] GET /api/v1/settings/venue_001
[API] PUT /api/v1/settings/venue_001
```

### Known Limitations

1. **Venue Selection**: Currently hardcoded to `venue_001`. Future enhancement needed to get venue from auth context.
2. **Toast Notifications**: Using console.log instead of react-toastify. Can be enhanced later.
3. **Optimistic Updates**: Not implemented. UI waits for backend confirmation before updating.
4. **Settings History**: Not tracked in UI (only in database).
5. **Bulk Operations**: Only single-setting updates supported.

## 🚀 Next Steps (Phase 4)

After Phase 3B is validated:

1. **Phase 4A**: Implement Feedback system backend
2. **Phase 4B**: Integrate Feedback system frontend
3. **Connect Settings to Feedback**: Use AI auto-respond and email auto-send settings

## 📞 Support

If you encounter issues:

1. Check browser console for errors
2. Check Network tab for API request/response
3. Verify .env.local configuration
4. Verify backend is running (if using real API mode)
5. Check CHANGELOG.md for recent changes
6. Review Phase 3B prompt document for specifications

## ✅ Sign-off

Phase 3B is ready for:
- [ ] QA testing
- [ ] Product owner review
- [ ] Integration with Phase 4
- [ ] Deployment to staging environment

**Implementation Date:** 2025-11-05
**Status:** ✅ COMPLETE



