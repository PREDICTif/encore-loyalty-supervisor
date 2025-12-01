# Phase 11: Unify Feedback Pages - Remove Mock Version

## Worker Prompt for AI Assistant

**Task**: Unify admin and venue feedback pages, remove mock version
**Priority**: HIGH (Feature parity issue)
**Estimated Effort**: 3-4 hours
**Dependencies**: Phase 9 & 10 complete

---

## 🎯 OBJECTIVE

Unify the feedback management experience so that:
1. Both admin and venue users see the **same UI and features**
2. Remove the old mock-based `FeedbackList.tsx` page
3. Admin sees all venues with selector
4. Venue manager (or impersonated) sees only their venue's feedback
5. All users can: Analyze, Generate Response, Send Email

---

## 📋 PROBLEM STATEMENT

### Current State (Broken)

```
ADMIN (/admin/feedback):
├── Uses AIFeedback.tsx ✅
├── Real API calls ✅
├── AI Analysis ✅
├── Generate Response ✅
└── Full features ✅

VENUE (/venue/feedback):
├── Uses FeedbackList.tsx ❌ (OLD)
├── Mock data via FeedbackContext ❌
├── No AI Analysis ❌
├── Different UI ❌
└── Missing features ❌
```

### Desired State

```
ADMIN (/admin/feedback):
├── Uses AIFeedback.tsx
├── Shows venue selector dropdown
├── Can switch between all venues
└── Full AI features

VENUE (/venue/feedback):
├── Uses AIFeedback.tsx (SAME PAGE)
├── Auto-filtered to user's venue
├── No venue selector (fixed to their venue)
└── Full AI features (SAME as admin)

IMPERSONATION:
├── Uses AIFeedback.tsx
├── Auto-filtered to impersonated venue
├── Full AI features
└── Shows impersonation banner
```

---

## 🏗️ IMPLEMENTATION PLAN

### Part 1: Update AIFeedback.tsx to Support Both Roles

**File**: `encore_frontend/src/pages/admin/AIFeedback.tsx`

Add role-based behavior:

```tsx
import { useAuth } from '../../contexts/AuthContext';

const AIFeedback: React.FC = () => {
    const { user, isImpersonating } = useAuth();
    
    // Determine if user is admin or venue manager
    const adminRoles = ['global_admin', 'super_admin', 'sysadmin'];
    const isAdmin = user?.role && adminRoles.includes(user.role);
    
    // For venue managers or impersonation, get their venue ID
    const userVenueId = user?.venue_id || user?.venue_ids?.[0];
    
    // State for venues
    const [clients, setClients] = useState<Client[]>([]);
    const [selectedClient, setSelectedClient] = useState<Client | null>(null);
    
    // Load clients based on role
    useEffect(() => {
        loadClients();
    }, []);
    
    const loadClients = async () => {
        try {
            setLoadingClients(true);
            const response = await clientsApi.getClients();
            const onboardedClients = response.clients.filter(c => c.is_migrated);
            
            if (isAdmin && !isImpersonating) {
                // Admin: Show all venues
                setClients(onboardedClients);
                if (onboardedClients.length > 0 && !selectedClient) {
                    setSelectedClient(onboardedClients[0]);
                }
            } else {
                // Venue manager or impersonating: Filter to their venue
                const userClients = onboardedClients.filter(
                    c => c.client_id === userVenueId || 
                         c.client_id === Number(userVenueId)
                );
                setClients(userClients);
                if (userClients.length > 0) {
                    setSelectedClient(userClients[0]);
                }
            }
        } catch (error) {
            console.error('Failed to load clients:', error);
        } finally {
            setLoadingClients(false);
        }
    };
    
    // Conditionally render venue selector
    const renderVenueSelector = () => {
        // Only show selector for admins who are NOT impersonating
        if (!isAdmin || isImpersonating) {
            // For venue managers: show venue name as header instead
            return (
                <div className="mb-6">
                    <h2 className="text-xl font-semibold text-gray-800">
                        {selectedClient?.display_name || 'Your Venue'}
                    </h2>
                    <p className="text-sm text-gray-500">
                        Feedback for your venue
                    </p>
                </div>
            );
        }
        
        // Admin: show dropdown selector
        return (
            <div className="mb-6">
                <label className="block text-sm font-medium text-gray-700 mb-2">
                    Select Venue
                </label>
                <select
                    value={selectedClient?.client_id || ''}
                    onChange={(e) => {
                        const client = clients.find(c => c.client_id === Number(e.target.value));
                        setSelectedClient(client || null);
                    }}
                    className="w-full max-w-md px-3 py-2 border border-gray-300 rounded-lg"
                >
                    {clients.map(client => (
                        <option key={client.client_id} value={client.client_id}>
                            {client.display_name} ({client.total_reviews} reviews)
                        </option>
                    ))}
                </select>
            </div>
        );
    };
    
    // ... rest of component
};
```

### Part 2: Update App.tsx Routing

**File**: `encore_frontend/src/App.tsx`

Route venue feedback to AIFeedback:

```tsx
// BEFORE (two different pages):
<Route path="/admin/feedback" element={<AIFeedback />} />
<Route path="/venue/feedback" element={<FeedbackList />} />

// AFTER (same page for both):
// Admin route
<Route
    path="/admin/feedback"
    element={
        <ProtectedRoute allowedRoles={['global_admin', 'super_admin', 'sysadmin']}>
            <AdminProvider>
                <AIFeedback />
            </AdminProvider>
        </ProtectedRoute>
    }
/>

// Venue route - NOW USES AIFeedback
<Route
    path="/venue/feedback"
    element={
        <ProtectedRoute allowedRoles={['venue_manager', 'manager', 'owner', 'gm', 'regional']}>
            <AIFeedback />  {/* Same component! */}
        </ProtectedRoute>
    }
/>
```

### Part 3: Update Sidebar Navigation

**File**: `encore_frontend/src/components/layout/Sidebar.tsx`

Update venue link to match admin naming:

```tsx
const managerLinks = [
    { to: '/venue', icon: HomeIcon, label: 'Dashboard' },
    { to: '/venue/feedback', icon: SparklesIcon, label: 'AI Feedback' },  // Changed icon and label
    { to: '/venue/settings', icon: Cog6ToothIcon, label: 'AI Settings' },
    { to: '/venue/analytics', icon: ChartBarIcon, label: 'Analytics' },
];
```

### Part 4: Update AuthContext for Impersonation Check

**File**: `encore_frontend/src/contexts/AuthContext.tsx`

Ensure `isImpersonating` is exported:

```tsx
// Make sure this is in the context value
const value = {
    user,
    isAuthenticated,
    isLoading,
    isImpersonating,  // Should already be there
    login,
    logout,
    refreshToken,
    impersonateVenue,
    endImpersonation,
};
```

### Part 5: Remove Old Mock Files

**Delete these files** (or move to archive):

```
encore_frontend/src/pages/venue/FeedbackList.tsx      → DELETE
encore_frontend/src/pages/venue/FeedbackDetail.tsx    → DELETE (if mock-based)
encore_frontend/src/contexts/FeedbackContext.tsx      → DELETE (if only used by FeedbackList)
```

**Check if FeedbackContext is used elsewhere before deleting!**

```bash
# Run this to check usage
grep -r "FeedbackContext\|useFeedback" encore_frontend/src --include="*.tsx" --include="*.ts"
```

### Part 6: Handle Missing Venue ID Edge Case

In `AIFeedback.tsx`, handle case where venue user has no venue assigned:

```tsx
// After loading clients
if (!isAdmin && clients.length === 0) {
    return (
        <Layout>
            <div className="flex flex-col items-center justify-center h-64">
                <ExclamationTriangleIcon className="h-12 w-12 text-yellow-500 mb-4" />
                <h2 className="text-xl font-semibold text-gray-800">No Venue Assigned</h2>
                <p className="text-gray-600 mt-2">
                    Your account is not associated with an onboarded venue.
                </p>
                <p className="text-gray-500 text-sm mt-1">
                    Please contact your administrator.
                </p>
            </div>
        </Layout>
    );
}
```

---

## 📁 FILES TO MODIFY

| File | Action | Changes |
|------|--------|---------|
| `encore_frontend/src/pages/admin/AIFeedback.tsx` | MODIFY | Add role-based filtering, venue header for non-admins |
| `encore_frontend/src/App.tsx` | MODIFY | Route `/venue/feedback` to `AIFeedback` |
| `encore_frontend/src/components/layout/Sidebar.tsx` | MODIFY | Update venue link icon/label |
| `encore_frontend/src/pages/venue/FeedbackList.tsx` | DELETE | Remove old mock page |
| `encore_frontend/src/pages/venue/FeedbackDetail.tsx` | DELETE | Remove old mock page (if applicable) |
| `encore_frontend/src/contexts/FeedbackContext.tsx` | DELETE | Remove if no longer needed |
| `CHANGELOG.md` | UPDATE | Document changes |

---

## ✅ ACCEPTANCE CRITERIA

### Functional Requirements

- [ ] Admin at `/admin/feedback` sees all venues with dropdown selector
- [ ] Venue manager at `/venue/feedback` sees only their venue's feedback
- [ ] Impersonated user at `/venue/feedback` sees only impersonated venue's feedback
- [ ] Both roles can: Analyze feedback
- [ ] Both roles can: Generate AI Response
- [ ] Both roles can: Send Customer Email
- [ ] Sidebar shows correct navigation for both roles
- [ ] Old mock-based FeedbackList.tsx is removed

### UI Requirements

- [ ] Admin: Shows venue dropdown selector
- [ ] Venue: Shows venue name as header (no dropdown)
- [ ] Same UI components for both roles
- [ ] Time filters work for both roles
- [ ] Analysis results display the same way

### Edge Cases

- [ ] Venue user with no assigned venue sees helpful error message
- [ ] Venue user with non-onboarded venue sees appropriate message
- [ ] Impersonation banner still shows when impersonating

---

## 🧪 TESTING SCENARIOS

### Scenario 1: Admin View
```
1. Login as admin (tjrottet@encoreloyalty.net)
2. Navigate to /admin/feedback
3. ✅ Should see venue dropdown with all onboarded venues
4. ✅ Can switch between venues
5. ✅ Can analyze, generate response, send email
```

### Scenario 2: Venue Manager View
```
1. Login as venue manager (or impersonate)
2. Navigate to /venue/feedback
3. ✅ Should see venue name as header (no dropdown)
4. ✅ Only sees their venue's feedback
5. ✅ Can analyze, generate response, send email (same features!)
```

### Scenario 3: Impersonation
```
1. Login as admin
2. Impersonate a venue
3. Navigate to /venue/feedback
4. ✅ Should see impersonated venue's feedback only
5. ✅ Impersonation banner visible
6. ✅ All AI features available
```

### Scenario 4: No Venue Assigned
```
1. Login as venue user with no venue_id
2. Navigate to /venue/feedback
3. ✅ Should see "No Venue Assigned" message
4. ✅ No error/crash
```

---

## 🚀 GETTING STARTED

### Step 1: Understand Current State
```bash
# Check what uses FeedbackContext
grep -r "FeedbackContext\|useFeedback" encore_frontend/src --include="*.tsx" --include="*.ts"

# Check current routing
grep -n "venue/feedback\|FeedbackList" encore_frontend/src/App.tsx
```

### Step 2: Modify AIFeedback.tsx
- Add `useAuth` import
- Add role detection logic
- Add conditional venue selector

### Step 3: Update Routing
- Change `/venue/feedback` to use `AIFeedback`
- Remove `FeedbackProvider` wrapper for venue route

### Step 4: Update Sidebar
- Change icon from `ChatBubbleLeftRightIcon` to `SparklesIcon`
- Change label from "Feedback Management" to "AI Feedback"

### Step 5: Clean Up Old Files
- Delete `FeedbackList.tsx`
- Delete `FeedbackDetail.tsx` (if mock-based)
- Delete `FeedbackContext.tsx` (if no longer used)

### Step 6: Test All Scenarios
- Test as admin
- Test as venue manager
- Test with impersonation
- Test edge cases

---

## ⚠️ IMPORTANT NOTES

1. **Check FeedbackContext usage** before deleting - may be used elsewhere
2. **Keep FeedbackDetail.tsx** if it's used and functional (may need separate update)
3. **Test impersonation flow** - ensure venue_id is correctly passed
4. **AdminProvider may not be needed** for venue route - remove if not required
5. **Preserve all AI features** - don't accidentally remove functionality

---

## 📝 HANDOVER CHECKLIST

After implementation:
- [ ] All scenarios tested
- [ ] No TypeScript errors
- [ ] No console errors
- [ ] Old files removed or archived
- [ ] CHANGELOG.md updated
- [ ] Build succeeds

---

**Created**: 2025-11-28
**Author**: AI Supervisor
**Status**: Ready for implementation

