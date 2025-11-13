# Settings API - Quick Reference Guide

## 🚀 Quick Start

### 1. Start Backend
```bash
cd /home/ec2-user/encore-loyalty-new-system/encore_backend
source ../venv/bin/activate
python main.py
```

Backend will start on: http://localhost:8000  
API Documentation: http://localhost:8000/docs

### 2. Get Authentication Token
```bash
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@encore.com", "password": "admin123"}'
```

Response:
```json
{
  "token": "eyJ0eXAiOiJKV1QiLC...",
  "refresh_token": "eyJ0eXAiOiJKV1QiLC...",
  "user": { ... }
}
```

Save the `token` value for subsequent requests.

---

## 📋 API Endpoints

### Get Venue Settings
```bash
GET /api/settings/{venue_id}
```

**Example:**
```bash
curl -X GET http://localhost:8000/api/settings/test_venue_001 \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

**Response:**
```json
{
  "venue_id": "test_venue_001",
  "venue_name": "Unknown Venue",
  "ai_settings": {
    "enabled": false,
    "model_version": "claude-3-5-sonnet-20241022",
    "temperature": 0.7,
    "max_tokens": 500,
    "system_prompt": null,
    "auto_respond": false
  },
  "email_settings": {
    "enabled": false,
    "from_email": null,
    "reply_to": null,
    "signature": null,
    "auto_send": false
  },
  "notification_settings": {
    "email_on_feedback": true,
    "email_on_response": false,
    "slack_webhook": null
  },
  "custom_fields": {},
  "created_at": "2025-11-05T12:00:00",
  "updated_at": "2025-11-05T12:00:00",
  "updated_by": null
}
```

---

### Update Venue Settings
```bash
PUT /api/settings/{venue_id}
```

**Example: Enable AI**
```bash
curl -X PUT http://localhost:8000/api/settings/test_venue_001 \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "ai_settings": {
      "enabled": true,
      "temperature": 0.8,
      "max_tokens": 750,
      "auto_respond": true
    }
  }'
```

**Example: Update Email Settings**
```bash
curl -X PUT http://localhost:8000/api/settings/test_venue_001 \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "email_settings": {
      "enabled": true,
      "from_email": "noreply@encore.com",
      "signature": "Thanks,\nThe Encore Team"
    }
  }'
```

**Response:**
```json
{
  "success": true,
  "message": "Settings updated successfully",
  "settings": { ... }
}
```

---

### Create Venue Settings (Admin Only)
```bash
POST /api/settings/{venue_id}?venue_name={name}
```

**Example:**
```bash
curl -X POST "http://localhost:8000/api/settings/venue_123?venue_name=Awesome%20Restaurant" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

**Response:**
```json
{
  "success": true,
  "message": "Settings created successfully",
  "settings": { ... }
}
```

---

### Delete Venue Settings (Admin Only)
```bash
DELETE /api/settings/{venue_id}
```

**Example:**
```bash
curl -X DELETE http://localhost:8000/api/settings/test_venue_001 \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

**Response:**
```json
{
  "success": true,
  "message": "Settings deleted successfully"
}
```

---

## 🔑 Authentication

All endpoints require JWT authentication.

**Header Format:**
```
Authorization: Bearer YOUR_TOKEN_HERE
```

**Getting a Token:**
1. POST to `/api/v1/auth/login` with email/password
2. Extract `token` from response
3. Include in Authorization header for all requests

**Token Expiration:**
- Access token: 15 minutes
- Refresh token: 7 days
- Use `/api/v1/auth/refresh` to get new access token

---

## 🔒 Access Control

| Role | Get Settings | Update Settings | Create Settings | Delete Settings |
|------|--------------|-----------------|-----------------|-----------------|
| **super_admin** | All venues | All venues | ✅ | ✅ |
| **global_admin** | All venues | All venues | ✅ | ✅ |
| **venue_manager** | Own venue only | Own venue only | ❌ | ❌ |

---

## 📊 Settings Structure

### AI Settings
```json
{
  "enabled": false,
  "model_version": "claude-3-5-sonnet-20241022",
  "temperature": 0.7,          // 0.0 to 1.0
  "max_tokens": 500,           // 100 to 2000
  "system_prompt": null,       // Optional custom prompt
  "auto_respond": false        // Auto-generate responses
}
```

### Email Settings
```json
{
  "enabled": false,
  "from_email": null,          // Must be valid email
  "reply_to": null,            // Must be valid email
  "signature": null,           // Email signature text
  "auto_send": false           // Auto-send emails
}
```

### Notification Settings
```json
{
  "email_on_feedback": true,   // Email when feedback received
  "email_on_response": false,  // Email when response sent
  "slack_webhook": null        // Slack webhook URL
}
```

### Custom Fields
```json
{
  "custom_fields": {
    "key1": "value1",
    "key2": 123,
    "key3": {"nested": "object"}
  }
}
```

---

## ⚠️ Validation Rules

- **temperature:** Must be between 0.0 and 1.0
- **max_tokens:** Must be between 100 and 2000
- **from_email / reply_to:** Must be valid email format
- **venue_id:** Required in URL path
- **Partial Updates:** Only send fields you want to change

---

## 🐛 Common Errors

### 401 Unauthorized
**Cause:** Missing or invalid JWT token  
**Fix:** Ensure Authorization header is present with valid token

### 403 Forbidden
**Cause:** Insufficient permissions (not admin or wrong venue)  
**Fix:** Check user role and venue_id access

### 404 Not Found
**Cause:** Settings don't exist and trying to update directly  
**Fix:** Settings are auto-created on GET, or use POST to create explicitly

### 409 Conflict
**Cause:** Trying to create settings that already exist  
**Fix:** Use PUT to update existing settings instead

### 422 Validation Error
**Cause:** Invalid data (temperature out of range, invalid email, etc.)  
**Fix:** Check validation rules and correct request data

---

## 🧪 Testing

### Run Manual Test
```bash
cd /home/ec2-user/encore-loyalty-new-system
source venv/bin/activate
python encore_backend/tests/test_settings_api.py
```

### Test Requirements
- Backend running on port 8000
- MySQL database accessible
- Valid admin user in database

---

## 💡 Tips

1. **Default Settings:** GET always returns something (defaults if none exist)
2. **Partial Updates:** Only send the settings you want to change
3. **Custom Fields:** Use for venue-specific configuration
4. **Temperature:** Lower = more deterministic, Higher = more creative
5. **Max Tokens:** Higher = longer responses, but more expensive
6. **Audit Trail:** Check updated_by and updated_at for change tracking

---

## 📚 API Documentation

Interactive API documentation available at:
- **Swagger UI:** http://localhost:8000/docs
- **ReDoc:** http://localhost:8000/redoc

---

## 🔗 Related Documentation

- Phase 3A Completion Report: `PHASE-3A-COMPLETION-REPORT.md`
- Phase 2A Authentication: See CHANGELOG.md
- DynamoDB Table Script: `scripts/create_settings_table.py`
- Test Script: `tests/test_settings_api.py`

---

**Last Updated:** November 5, 2025  
**Backend Version:** Phase 3A Complete  
**API Base URL:** http://localhost:8000

