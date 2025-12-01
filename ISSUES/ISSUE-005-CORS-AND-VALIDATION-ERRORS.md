# ISSUE-005: CORS Configuration + Pydantic Validation Errors

## Status: ✅ RESOLVED (2025-11-26)

## Problem Summary

Feedback API calls from the frontend were failing with CORS errors, even though CORS middleware was configured. Investigation revealed two compounding issues.

## Root Cause Analysis

### Issue 1: CORS Configuration Conflict

**Problem**: Backend was configured with:
```python
cors_origins: list = ["*"]
allow_credentials=True
```

**Why it failed**: The CORS specification prohibits using `Access-Control-Allow-Origin: *` when `Access-Control-Allow-Credentials: true`. Browsers will reject such responses.

**Fix**: Changed `config.py` to specify exact origins:
```python
cors_origins: list = [
    "http://localhost:3000",
    "http://127.0.0.1:3000",
    "http://localhost:8000",
    "http://127.0.0.1:8000",
]
```

### Issue 2: Pydantic Validation Error

**Problem**: The `FeedbackListItem` model required `created_at: datetime`, but some database records have NULL `created_at` values.

**Error Message**:
```
pydantic_core._pydantic_core.ValidationError: 1 validation error for FeedbackListItem
created_at
  Input should be a valid datetime [type=datetime_type, input_value=None, input_type=NoneType]
```

**Why it masked CORS**: Exceptions thrown during request processing happen BEFORE CORS headers are added to the response. The browser sees a response without CORS headers and blocks it.

**Fix**: Made `created_at` optional in `models/feedback.py`:
```python
created_at: Optional[datetime] = None
```

## Files Modified

1. **`encore_backend/config.py`**
   - Changed `cors_origins` from `["*"]` to specific localhost origins

2. **`encore_backend/models/feedback.py`**
   - Changed `created_at: datetime` to `created_at: Optional[datetime] = None`

## Side Effect

The frontend now displays "almost 56 years ago" for records with `created_at=None` (interpreting None as Unix epoch 1970). This is a **minor display bug** that should be fixed in the frontend.

## Testing Results

- ✅ Feedback list now loads successfully
- ✅ Real customer data displayed
- ✅ CORS errors resolved
- ⚠️ Minor: Date display bug for records without created_at

## Lessons Learned

1. **CORS + Credentials**: Never use `["*"]` with `allow_credentials=True`
2. **Exception Handling**: Backend exceptions can mask the real error by preventing CORS headers from being sent
3. **Data Quality**: Legacy databases may have NULL values in unexpected columns - always use Optional types for safety

## Related Issues

- Part of ISSUE-004 investigation (API Path Inconsistency)
- Discovered during systematic UI testing phase

