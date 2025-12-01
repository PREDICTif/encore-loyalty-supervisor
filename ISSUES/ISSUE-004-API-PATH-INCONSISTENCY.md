# ISSUE-004: API Path Versioning Inconsistency

## Status: ⚠️ TECH DEBT (Mitigated)

## Problem

The backend API uses inconsistent path versioning:
- Auth & Admin routes: `/api/v1/...`
- Feedback, Settings, AI routes: `/api/...` (no versioning)

This caused frontend API calls to fail with 404 errors when the paths didn't match.

## Fixes Applied (2025-11-26)

### Frontend Files Updated

1. **`feedbackApi.ts`** - Changed all endpoints from `/api/v1/feedback/...` to `/api/feedback/...`
2. **`settingsApi.ts`** - Changed all endpoints from `/api/v1/settings/...` to `/api/settings/...`

### Current Path Mapping

| API | Backend Path | Frontend Path | Status |
|-----|-------------|---------------|--------|
| Auth | `/api/v1/auth/...` | `/api/v1/auth/...` | ✅ |
| Admin | `/api/v1/admin/...` | `/api/v1/admin/...` | ✅ |
| Settings | `/api/settings/...` | `/api/settings/...` | ✅ |
| Feedback | `/api/feedback/...` | `/api/feedback/...` | ✅ |
| AI | `/api/ai/...` | `/api/ai/...` | ✅ |

## Recommendation for Future

Standardize all backend routes to use consistent versioning (e.g., all `/api/v1/...`).

This would involve:
1. Update `encore_backend/routes/feedback_routes.py` - add `v1` prefix
2. Update `encore_backend/routes/settings_routes.py` - add `v1` prefix  
3. Update `encore_backend/routes/ai_routes.py` - add `v1` prefix
4. Update frontend API files to match

**Priority**: LOW (system works correctly now)
**Effort**: ~1 hour

## Impact

- **Before fix**: Feedback list returned 404, Settings API failed
- **After fix**: All API calls work correctly

## Related

- BUG discovered during systematic testing on 2025-11-26
- Found while testing feedback list as impersonated venue

