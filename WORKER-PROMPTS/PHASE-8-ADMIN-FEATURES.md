# Phase 8: Admin Features - System Health & User Management

## 🎯 TASK OVERVIEW

Implement real data for the Admin System Monitoring and User Management pages, replacing mock API calls with real backend endpoints.

**Duration**: 4-6 hours
**Priority**: P2 - Admin features
**Dependencies**: Phase 7 complete (Analytics working)

---

## 📚 REQUIRED READING

1. `/CLAUDE.md` - Project overview
2. `/docs/SUPERVISOR/CURRENT-STATUS.md` - Current state
3. Existing files to understand:
   - `encore_backend/routes/health.py` - Existing health endpoints
   - `encore_backend/routes/admin.py` - Existing admin endpoints
   - `encore_frontend/src/pages/admin/SystemMonitoring.tsx` - UI to update
   - `encore_frontend/src/pages/admin/UserManagement.tsx` - UI to update
   - `encore_frontend/src/contexts/AdminContext.tsx` - Context using mock data

---

## 📋 DELIVERABLES

### Phase 8A: System Health Real Data

**Goal**: Replace `mockApi.getSystemHealth()` with real `/health` endpoint data

#### Backend Changes

The `/health/detailed` endpoint already exists. Create a new endpoint that formats the response for the frontend:

**Create**: `encore_backend/routes/admin.py` - Add these endpoints:

```python
# Add to existing admin.py

class ServiceHealth(BaseModel):
    name: str
    status: Literal["healthy", "degraded", "down"]
    details: str
    latency: Optional[int] = None  # milliseconds

class SystemHealthResponse(BaseModel):
    overall: Literal["healthy", "degraded", "down"]
    services: List[ServiceHealth]
    lastChecked: str  # ISO datetime

@router.get("/system-health", response_model=SystemHealthResponse)
async def get_system_health(
    current_user: User = Depends(get_current_user),
    mysql_service: MySQLService = Depends(get_mysql_service),
    dynamo_service: RestaurantDynamoDBService = Depends(get_dynamodb_service)
):
    """
    Get system health status for admin monitoring.
    
    Checks:
    - MySQL database connectivity
    - DynamoDB connectivity  
    - AWS Bedrock availability
    - AWS SES availability
    """
    check_admin_permission(current_user)
    
    services = []
    overall = "healthy"
    
    # Check MySQL
    try:
        start = time.time()
        mysql_service.get_all_venues()  # Simple query test
        latency = int((time.time() - start) * 1000)
        services.append(ServiceHealth(
            name="MySQL Database",
            status="healthy",
            details="Connected via SSH tunnel",
            latency=latency
        ))
    except Exception as e:
        services.append(ServiceHealth(
            name="MySQL Database",
            status="down",
            details=str(e)
        ))
        overall = "degraded"
    
    # Check DynamoDB
    try:
        start = time.time()
        # Simple table describe to check connectivity
        dynamo_service.table.table_status
        latency = int((time.time() - start) * 1000)
        services.append(ServiceHealth(
            name="DynamoDB",
            status="healthy",
            details=f"Table: {dynamo_service.table.name}",
            latency=latency
        ))
    except Exception as e:
        services.append(ServiceHealth(
            name="DynamoDB",
            status="down",
            details=str(e)
        ))
        overall = "degraded"
    
    # Check Bedrock (just config, not actual call)
    if settings.bedrock_enabled:
        services.append(ServiceHealth(
            name="AWS Bedrock",
            status="healthy",
            details=f"Model: {settings.bedrock_model_id}"
        ))
    else:
        services.append(ServiceHealth(
            name="AWS Bedrock",
            status="degraded",
            details="Disabled in configuration"
        ))
    
    # Check SES
    if settings.ses_enabled:
        services.append(ServiceHealth(
            name="AWS SES",
            status="healthy",
            details=f"Sender: {settings.ses_sender_email}"
        ))
    else:
        services.append(ServiceHealth(
            name="AWS SES",
            status="degraded",
            details="Disabled in configuration"
        ))
    
    return SystemHealthResponse(
        overall=overall,
        services=services,
        lastChecked=datetime.utcnow().isoformat()
    )
```

#### Frontend Changes

**Create**: `encore_frontend/src/services/api/adminSystemApi.ts`

```typescript
/**
 * Admin System API Service
 * 
 * Real API calls for system health and admin monitoring
 */

import { apiClient } from '../apiClient';
import { SystemHealth, HealthStatus } from '../../types';

interface ServiceHealthResponse {
    name: string;
    status: 'healthy' | 'degraded' | 'down';
    details: string;
    latency?: number;
}

interface SystemHealthApiResponse {
    overall: 'healthy' | 'degraded' | 'down';
    services: ServiceHealthResponse[];
    lastChecked: string;
}

/**
 * Get real system health status
 */
export const getSystemHealth = async (): Promise<SystemHealth> => {
    const response = await apiClient.get<SystemHealthApiResponse>('/api/v1/admin/system-health');
    const data = response.data;
    
    // Transform to match frontend SystemHealth type
    return {
        overall: data.overall as HealthStatus,
        services: data.services.map(s => ({
            name: s.name,
            status: s.status as HealthStatus,
            details: s.details,
            latency: s.latency
        })),
        lastChecked: data.lastChecked
    };
};

export const adminSystemApi = {
    getSystemHealth,
};

export default adminSystemApi;
```

**Update**: `encore_frontend/src/contexts/AdminContext.tsx`

Change the `refreshSystemHealth` function:

```typescript
// Add import at top
import adminSystemApi from '../services/api/adminSystemApi';

// Update refreshSystemHealth function
const refreshSystemHealth = async () => {
    try {
        const healthData = await adminSystemApi.getSystemHealth();
        setSystemHealth(healthData);
    } catch (error) {
        console.error('Failed to fetch system health:', error);
        // Fallback to mock if real API fails
        const healthData = await mockApi.getSystemHealth();
        setSystemHealth(healthData);
    }
};
```

---

### Phase 8B: User List API (Read-Only)

**Goal**: Replace `mockApi.getAdminUsers()` with real data from MySQL `user` table

#### Backend Changes

**Add to**: `encore_backend/routes/admin.py`

```python
# Add these models
class AdminUserResponse(BaseModel):
    id: int
    email: str
    name: str
    role: str
    venue_id: Optional[int] = None
    is_active: bool
    last_active: Optional[str] = None  # We may not have this data
    permissions: List[str] = []

class AdminUsersListResponse(BaseModel):
    users: List[AdminUserResponse]
    total: int

# Role to permissions mapping
ROLE_PERMISSIONS = {
    "sysadmin": ["manage_venues", "global_settings", "view_analytics", "manage_users", "system_monitoring", "view_feedback", "provision_ai", "manage_billing"],
    "super_admin": ["manage_venues", "global_settings", "view_analytics", "manage_users", "system_monitoring", "view_feedback", "provision_ai", "manage_billing"],
    "global_admin": ["manage_venues", "global_settings", "view_analytics", "system_monitoring", "view_feedback", "provision_ai"],
    "owner": ["manage_venues", "view_analytics", "view_feedback", "provision_ai"],
    "manager": ["view_analytics", "view_feedback"],
    "gm": ["view_analytics", "view_feedback"],
    "regional": ["view_analytics", "view_feedback"],
    "venue_manager": ["view_analytics", "view_feedback"],
}

@router.get("/users", response_model=AdminUsersListResponse)
async def list_admin_users(
    current_user: User = Depends(get_current_user),
    mysql_service: MySQLService = Depends(get_mysql_service)
):
    """
    List all admin users from MySQL database.
    
    Only accessible to super_admin/sysadmin roles.
    Returns users with their roles and computed permissions.
    """
    # Only super admins can list users
    if current_user.role not in ["sysadmin", "super_admin"]:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Only super admins can list users"
        )
    
    try:
        query = """
            SELECT 
                iduser as id,
                emailaddress as email,
                CONCAT(firstname, ' ', lastname) as name,
                usertype as role,
                idlocation as venue_id,
                status as is_active,
                lastlogin as last_active
            FROM user
            WHERE status = 1
            ORDER BY usertype, name
        """
        
        results = mysql_service._execute_query(query)
        
        users = []
        for row in results:
            role = row.get('role', 'venue_manager')
            permissions = ROLE_PERMISSIONS.get(role, [])
            
            users.append(AdminUserResponse(
                id=row['id'],
                email=row['email'],
                name=row['name'] or 'Unknown',
                role=role,
                venue_id=row.get('venue_id'),
                is_active=bool(row.get('is_active', True)),
                last_active=row.get('last_active').isoformat() if row.get('last_active') else None,
                permissions=permissions
            ))
        
        return AdminUsersListResponse(users=users, total=len(users))
        
    except Exception as e:
        logger.error(f"Failed to list users: {e}")
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"Failed to fetch users: {str(e)}"
        )


@router.get("/users/{user_id}", response_model=AdminUserResponse)
async def get_admin_user(
    user_id: int,
    current_user: User = Depends(get_current_user),
    mysql_service: MySQLService = Depends(get_mysql_service)
):
    """
    Get single admin user by ID.
    """
    if current_user.role not in ["sysadmin", "super_admin"]:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Only super admins can view user details"
        )
    
    try:
        query = """
            SELECT 
                iduser as id,
                emailaddress as email,
                CONCAT(firstname, ' ', lastname) as name,
                usertype as role,
                idlocation as venue_id,
                status as is_active,
                lastlogin as last_active
            FROM user
            WHERE iduser = %s
        """
        
        results = mysql_service._execute_query(query, (user_id,))
        
        if not results:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"User {user_id} not found"
            )
        
        row = results[0]
        role = row.get('role', 'venue_manager')
        permissions = ROLE_PERMISSIONS.get(role, [])
        
        return AdminUserResponse(
            id=row['id'],
            email=row['email'],
            name=row['name'] or 'Unknown',
            role=role,
            venue_id=row.get('venue_id'),
            is_active=bool(row.get('is_active', True)),
            last_active=row.get('last_active').isoformat() if row.get('last_active') else None,
            permissions=permissions
        )
        
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Failed to get user {user_id}: {e}")
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"Failed to fetch user: {str(e)}"
        )
```

#### Frontend Changes

**Update**: `encore_frontend/src/services/api/adminSystemApi.ts` - Add user functions:

```typescript
// Add to existing adminSystemApi.ts

import { AdminUser } from '../../types';

interface AdminUserApiResponse {
    id: number;
    email: string;
    name: string;
    role: string;
    venue_id?: number;
    is_active: boolean;
    last_active?: string;
    permissions: string[];
}

interface AdminUsersListApiResponse {
    users: AdminUserApiResponse[];
    total: number;
}

/**
 * Get list of admin users
 */
export const getAdminUsers = async (): Promise<AdminUser[]> => {
    const response = await apiClient.get<AdminUsersListApiResponse>('/api/v1/admin/users');
    
    // Transform to match frontend AdminUser type
    return response.data.users.map(u => ({
        id: String(u.id),
        email: u.email,
        name: u.name,
        role: u.role as 'super_admin' | 'admin' | 'support',
        permissions: u.permissions,
        lastActive: u.last_active || new Date().toISOString(),
    }));
};

/**
 * Get single admin user
 */
export const getAdminUser = async (userId: string): Promise<AdminUser> => {
    const response = await apiClient.get<AdminUserApiResponse>(`/api/v1/admin/users/${userId}`);
    const u = response.data;
    
    return {
        id: String(u.id),
        email: u.email,
        name: u.name,
        role: u.role as 'super_admin' | 'admin' | 'support',
        permissions: u.permissions,
        lastActive: u.last_active || new Date().toISOString(),
    };
};

// Update export
export const adminSystemApi = {
    getSystemHealth,
    getAdminUsers,
    getAdminUser,
};
```

**Update**: `encore_frontend/src/contexts/AdminContext.tsx`

In the initial data load, update the users call:

```typescript
// In loadInitialData function, change:
// usersData: mockApi.getAdminUsers(),
// To:
usersData: await adminSystemApi.getAdminUsers().catch(() => mockApi.getAdminUsers()),
```

---

## 🧪 TESTING

### Backend Tests

```bash
# Test system health endpoint
curl -s http://localhost:8000/api/v1/admin/system-health \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# Expected response:
{
    "overall": "healthy",
    "services": [
        {"name": "MySQL Database", "status": "healthy", "details": "Connected via SSH tunnel", "latency": 45},
        {"name": "DynamoDB", "status": "healthy", "details": "Table: restaurant_reviews_development", "latency": 23},
        {"name": "AWS Bedrock", "status": "healthy", "details": "Model: us.anthropic.claude-3-5-sonnet-20241022-v2:0"},
        {"name": "AWS SES", "status": "healthy", "details": "Sender: marian.dumitrascu@predictifsolutions.com"}
    ],
    "lastChecked": "2025-11-28T20:30:00.000000"
}

# Test users list endpoint
curl -s http://localhost:8000/api/v1/admin/users \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# Expected response:
{
    "users": [
        {
            "id": 1,
            "email": "tjrottet@encoreloyalty.net",
            "name": "TJ Rottet",
            "role": "sysadmin",
            "venue_id": null,
            "is_active": true,
            "last_active": "2025-11-28T15:00:00",
            "permissions": ["manage_venues", "global_settings", ...]
        },
        ...
    ],
    "total": 5
}
```

### Frontend Tests

1. Navigate to **System Monitoring** (`/admin/monitoring`)
   - Verify real service status shown (MySQL, DynamoDB, Bedrock, SES)
   - Verify latency values are realistic (not mock 45ms for everything)
   - Verify refresh button works

2. Navigate to **User Management** (`/admin/users`)
   - Verify real users from MySQL are shown
   - Verify user count matches database
   - Verify permissions are computed correctly

---

## ⚠️ IMPORTANT NOTES

### MySQL Service Method

The `mysql_service._execute_query()` method should be used. Check the actual method name in `DirectMySQLService`:
- It might be `_execute_query()` or `execute_query()` 
- Check `encore_backend/db/direct_mysql_service.py` for exact method name

### Error Handling

- If real API fails, fall back to mock data gracefully
- Don't break the UI if backend is temporarily unavailable

### Role Mapping

Frontend uses: `super_admin`, `admin`, `support`
Backend has: `sysadmin`, `super_admin`, `global_admin`, `owner`, `manager`, `gm`, `regional`, `venue_manager`

Map appropriately:
- `sysadmin`, `super_admin` → `super_admin`
- `global_admin`, `owner` → `admin`
- Others → `support`

### Not In Scope

The following are **NOT** part of Phase 8:
- User create/update/delete (keep using legacy system)
- Error logs real data (use CloudWatch)
- Activity log real data (future enhancement)

---

## ✅ ACCEPTANCE CRITERIA

### Phase 8A: System Health
- [ ] `/api/v1/admin/system-health` endpoint returns real service status
- [ ] MySQL latency is measured and displayed
- [ ] DynamoDB status is checked
- [ ] Bedrock/SES config status shown
- [ ] System Monitoring page shows real data
- [ ] Refresh button triggers real API call

### Phase 8B: User List
- [ ] `/api/v1/admin/users` returns real users from MySQL
- [ ] `/api/v1/admin/users/{id}` returns single user
- [ ] User Management page shows real user data
- [ ] Permissions computed from role
- [ ] Only super_admin/sysadmin can access

### General
- [ ] No TypeScript errors
- [ ] No linter errors
- [ ] Both services still work (backend + frontend)
- [ ] Graceful fallback to mock if API fails

---

## 📝 AFTER COMPLETION

1. Update `/CHANGELOG.md` with Phase 8 changes
2. Update `/docs/SUPERVISOR/CURRENT-STATUS.md` 
3. Test both pages in browser
4. Report completion with test results

---

## 🚀 GETTING STARTED

```bash
# Get auth token
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"tjrottet@encoreloyalty.net","password":"password222"}' | jq -r '.token')

echo "Token: ${TOKEN:0:50}..."

# Test existing health endpoint first
curl -s http://localhost:8000/health/detailed | python3 -m json.tool
```

**Services Running:**
- Backend: http://localhost:8000
- Frontend: http://localhost:3000
- Login: tjrottet@encoreloyalty.net / password222

---

*Created: 2025-11-28*
*Phase: 8 - Admin Features*
*Estimated Time: 4-6 hours*

