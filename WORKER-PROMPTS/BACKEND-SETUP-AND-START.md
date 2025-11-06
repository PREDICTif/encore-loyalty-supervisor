# Backend Setup and Start - Worker Prompt
## Make encore_backend Operational

**Assigned To**: Cursor Chat (Claude Sonnet 4.5)
**Task**: Configure and start FastAPI backend
**Duration**: 30-45 minutes
**Date**: November 5, 2025
**Priority**: P1 - Critical (Required for frontend integration)

---

## 🎯 MISSION

Get the FastAPI backend (`./encore_backend`) operational and running on `http://localhost:8000`.

**Current State**: Backend code complete, venv ready, SSH key exists
**Target State**: Backend running, health check passes, ready for API development

---

## 📊 CURRENT STATE ANALYSIS

### ✅ What's Already Working

1. **Python Virtual Environment**: Created at `/home/ec2-user/encore-loyalty-new-system/venv`
2. **Dependencies Installed**: All packages from requirements.txt are installed
   - fastapi==0.104.1 ✅
   - uvicorn[standard]==0.24.0 ✅
   - pydantic==2.5.0 ✅
   - boto3==1.34.0 ✅
   - mysql-connector-python==8.1.0 ✅
   - sshtunnel==0.4.0 ✅
   - And all others ✅

3. **Backend Code**: Complete FastAPI application
   - Routes: health, clients, onboard, comments ✅
   - Services: MySQL, DynamoDB, Bedrock, SES ✅
   - Middleware: CORS, auth ✅
   - Models and dependencies ✅

4. **SSH Key**: Exists at `encore_backend/key/Encoreepad_ppk.pem` ✅

### ❌ What's Missing

1. **`.env` File**: Doesn't exist (only .env.example)
2. **AWS Credentials**: Need to be configured
3. **Testing**: App hasn't been started yet

---

## 📁 PROJECT STRUCTURE

```
/home/ec2-user/encore-loyalty-new-system/
├── venv/                          ← Python virtual environment (EXISTS)
│   └── bin/activate               ← Activation script
├── encore_backend/                ← FastAPI backend
│   ├── .env.example               ← Template (EXISTS)
│   ├── .env                       ← NEED TO CREATE
│   ├── main.py                    ← Entry point
│   ├── config.py                  ← Settings management
│   ├── requirements.txt           ← Dependencies (already installed)
│   ├── key/
│   │   └── Encoreepad_ppk.pem    ← SSH key (EXISTS, chmod 600)
│   ├── routes/                    ← API endpoints
│   ├── services/                  ← Business logic
│   ├── middleware/                ← CORS, auth
│   ├── models/                    ← Data models
│   └── db/                        ← Database utilities
└── encore_frontend/               ← Frontend (Phase 1 in progress)
```

---

## 🛠️ DETAILED TASKS

### Task 1: Create .env File

**Objective**: Create environment configuration file with proper settings.

**Location**: `/home/ec2-user/encore-loyalty-new-system/encore_backend/.env`

**Action**: Create `.env` file with these contents:

```bash
# Database Settings (Customer-provided - DO NOT CHANGE)
SSH_HOST=52.14.57.81
SSH_PORT=22
SSH_USERNAME=ubuntu
SSH_KEY_FILE=key/Encoreepad_ppk.pem

MYSQL_HOST=127.0.0.1
MYSQL_USER=root
MYSQL_PASSWORD=bOuDCUvd8
MYSQL_DATABASE=encoree3_mobile

# AWS Settings - ALREADY CONFIGURED via 'aws configure'
# The EC2 instance already has AWS CLI configured with proper credentials
# These environment variables are optional (boto3 will use default credentials)
AWS_REGION=us-east-1
# AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY not needed - using EC2 default credentials

# DynamoDB Settings
DYNAMODB_TABLE_NAME=restaurant_reviews_development

# Bedrock AI Settings
BEDROCK_ENABLED=false
BEDROCK_MODEL_ID=anthropic.claude-3-5-sonnet-20241022-v2:0
S3_BUCKET_PROFILES=restaurant-customer-profiles-dev
BEDROCK_TEST_LIMIT=10
BEDROCK_CONCURRENT_LIMIT=5

# SES Email Settings
SES_ENABLED=false
SES_SENDER_EMAIL=noreply@predictif.com
SES_MANAGER_EMAIL=manager@predictif.com

# API Security
API_KEY=dev-test-key-12345
REQUIRE_AUTH=false

# App Settings
DEBUG=true
HOST=0.0.0.0
PORT=8000
```

**Important Notes**:
- **For MVP development, we're setting**:
  - `BEDROCK_ENABLED=false` (disable AI for now, will enable on Day 4)
  - `SES_ENABLED=false` (disable email for now, will enable on Day 5)
  - `REQUIRE_AUTH=false` (disable auth during initial setup)
  - `DEBUG=true` (enable debug mode for development)
- **AWS credentials**: If you don't have real AWS credentials yet, leave the placeholder values. The app will start but AWS features won't work.
- **MySQL settings**: These are customer-provided and correct.
- **SSH key**: Already exists and configured.

**Commands**:
```bash
cd /home/ec2-user/encore-loyalty-new-system/encore_backend

# Copy from example
cp .env.example .env

# Edit the file with the configuration above
nano .env  # or use your preferred editor
```

**Verification**:
- [ ] `.env` file exists in `encore_backend/` directory
- [ ] File contains all required environment variables
- [ ] MySQL credentials match .env.example
- [ ] AWS services disabled for initial setup (BEDROCK_ENABLED=false, SES_ENABLED=false)

---

### Task 2: Verify SSH Key Permissions

**Objective**: Ensure SSH key has correct permissions for security.

**Commands**:
```bash
cd /home/ec2-user/encore-loyalty-new-system/encore_backend

# Check current permissions
ls -l key/Encoreepad_ppk.pem

# Set correct permissions (owner read-only)
chmod 600 key/Encoreepad_ppk.pem

# Verify
ls -l key/Encoreepad_ppk.pem
# Should show: -rw------- (600)
```

**Verification**:
- [ ] SSH key permissions are `600` (owner read-write only)
- [ ] File owner is correct (ec2-user)

---

### Task 3: Test Backend Startup (Basic)

**Objective**: Start the backend and verify it runs without errors.

**Commands**:
```bash
# Navigate to project root
cd /home/ec2-user/encore-loyalty-new-system

# Activate virtual environment
source venv/bin/activate

# Navigate to backend
cd encore_backend

# Start the server
python main.py
```

**Expected Output**:
```
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

**If you see errors**:
- **Import errors**: Missing dependencies → Run `pip install -r requirements.txt`
- **AWS errors**: This is OK if AWS services are disabled
- **MySQL errors**: This is OK initially, we'll test separately

**Verification**:
- [ ] Server starts without critical errors
- [ ] Server is listening on port 8000
- [ ] No import/dependency errors

**Note**: Leave the server running for testing. Open a new terminal for next steps.

---

### Task 4: Test Health Endpoint

**Objective**: Verify the API is responding to requests.

**Commands** (in a NEW terminal):
```bash
# Test root endpoint
curl http://localhost:8000/

# Expected output:
# {"message":"Restaurant Reviews Onboarding API","version":"3.0","docs":"/docs"}

# Test health endpoint
curl http://localhost:8000/api/v1/health

# Expected output includes:
# {
#   "status": "healthy",
#   "version": "3.0",
#   "services": { ... },
#   ...
# }
```

**Interactive API Documentation**:
Open in browser: `http://localhost:8000/docs`

**Verification**:
- [ ] Root endpoint returns JSON with app name and version
- [ ] Health endpoint returns status information
- [ ] Can access Swagger UI at /docs
- [ ] Services show status (MySQL may show error, that's OK for now)

---

### Task 5: Test MySQL Connection (Optional)

**Objective**: Verify MySQL SSH tunnel connection works.

**Note**: This may take 30-60 seconds on first connection (SSH tunnel setup).

**Commands**:
```bash
# Test clients endpoint (requires MySQL)
curl http://localhost:8000/api/v1/clients/

# This should return a list of restaurant clients
# If it works: You'll see JSON array with client data
# If it fails: Check error message for details
```

**If MySQL connection fails**:
1. Check `.env` file has correct MySQL credentials
2. Verify SSH key permissions are 600
3. Verify SSH host (52.14.57.81) is reachable
4. Check firewall/security group settings

**If you can't fix MySQL now**: That's OK! You can continue and fix it later when needed.

**Verification**:
- [ ] Can connect to MySQL via SSH tunnel
- [ ] Clients endpoint returns data
- OR [ ] MySQL error is documented for later resolution

---

### Task 6: Understand What Endpoints Exist

**Objective**: Know what the backend currently provides.

**Current Endpoints** (from code analysis):

| Endpoint | Method | Description | Status |
|----------|--------|-------------|--------|
| `/` | GET | Root info | ✅ Working |
| `/api/v1/health` | GET | Health check | ✅ Working |
| `/api/v1/clients/` | GET | List restaurant clients | 🟡 Requires MySQL |
| `/api/v1/clients/{id}/reviews` | GET | Get client reviews | 🟡 Requires MySQL |
| `/api/v1/onboard/{id}` | POST | Onboard client to DynamoDB | 🟡 Requires MySQL + AWS |
| `/api/v1/onboard/{id}/status` | GET | Check onboard status | 🟡 Requires AWS |
| `/api/v1/comments/new-comment` | POST | Submit new comment | 🟡 Requires AWS + MySQL |
| `/api/v1/comments/restaurants` | GET | List restaurants | 🟡 Requires MySQL |

**What's Missing for MVP**:
- ❌ Authentication endpoints (login, refresh, logout)
- ❌ Settings endpoints (get/update venue settings)
- ❌ Feedback endpoints (list, get, update feedback)
- ❌ Analytics endpoints (metrics, charts)
- ❌ Venues endpoints (list, get venues)
- ❌ AI response generation endpoints

**These will be implemented in Phase 2+ as per MVP plan.**

**Verification**:
- [ ] Understand current endpoint capabilities
- [ ] Know which endpoints work without AWS
- [ ] Know what needs to be built next

---

### Task 7: Create Unified Management Scripts

**Objective**: Create management scripts at project root to control BOTH frontend and backend.

**Background**: Currently, `encore_frontend/` has START.sh, STOP.sh, and STATUS.sh that only manage the React frontend (port 3000). We need unified scripts that manage both services.

**Actions**:

#### 7.1: Move Scripts to Project Root

```bash
cd /home/ec2-user/encore-loyalty-new-system

# Move scripts from encore_frontend to project root
mv encore_frontend/START.sh ./START.sh
mv encore_frontend/STOP.sh ./STOP.sh
mv encore_frontend/STATUS.sh ./STATUS.sh

# Make them executable
chmod +x START.sh STOP.sh STATUS.sh
```

#### 7.2: Update START.sh (Manage Both Services)

Replace the entire content of `/home/ec2-user/encore-loyalty-new-system/START.sh`:

```bash
#!/bin/bash

# START.sh - Start Frontend and Backend for Encore Loyalty System
# Usage: ./START.sh [frontend|backend|all]

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
FRONTEND_DIR="$PROJECT_ROOT/encore_frontend"
BACKEND_DIR="$PROJECT_ROOT/encore_backend"
VENV_DIR="$PROJECT_ROOT/venv"

FRONTEND_PID_FILE="$FRONTEND_DIR/.server.pid"
BACKEND_PID_FILE="$BACKEND_DIR/.server.pid"
FRONTEND_LOG="$FRONTEND_DIR/.server.log"
BACKEND_LOG="$BACKEND_DIR/.server.log"

FRONTEND_PORT=3000
BACKEND_PORT=8000

# Parse arguments
MODE="${1:-all}"

echo "========================================"
echo "Encore Loyalty System - START"
echo "========================================"

start_frontend() {
    echo ""
    echo "🎨 FRONTEND (React @ port $FRONTEND_PORT)"
    echo "----------------------------------------"

    # Check if already running
    if [ -f "$FRONTEND_PID_FILE" ]; then
        PID=$(cat "$FRONTEND_PID_FILE")
        if ps -p "$PID" > /dev/null 2>&1; then
            echo "⚠️  Frontend is already running (PID: $PID)"
            return 0
        fi
    fi

    # Check port
    if lsof -Pi :$FRONTEND_PORT -sTCP:LISTEN -t >/dev/null 2>&1 ; then
        echo "❌ Port $FRONTEND_PORT is in use"
        return 1
    fi

    # Check dependencies
    if [ ! -d "$FRONTEND_DIR/node_modules" ]; then
        echo "📦 Installing frontend dependencies..."
        cd "$FRONTEND_DIR" && npm install
    fi

    # Start frontend
    echo "🚀 Starting frontend..."
    cd "$FRONTEND_DIR"
    nohup npm start > "$FRONTEND_LOG" 2>&1 &
    FRONTEND_PID=$!
    echo $FRONTEND_PID > "$FRONTEND_PID_FILE"

    sleep 3
    if ps -p "$FRONTEND_PID" > /dev/null 2>&1; then
        echo "✅ Frontend started (PID: $FRONTEND_PID)"
        echo "   URL: http://localhost:$FRONTEND_PORT"
    else
        echo "❌ Frontend failed to start"
        return 1
    fi
}

start_backend() {
    echo ""
    echo "⚙️  BACKEND (FastAPI @ port $BACKEND_PORT)"
    echo "----------------------------------------"

    # Check if already running
    if [ -f "$BACKEND_PID_FILE" ]; then
        PID=$(cat "$BACKEND_PID_FILE")
        if ps -p "$PID" > /dev/null 2>&1; then
            echo "⚠️  Backend is already running (PID: $PID)"
            return 0
        fi
    fi

    # Check port
    if lsof -Pi :$BACKEND_PORT -sTCP:LISTEN -t >/dev/null 2>&1 ; then
        echo "❌ Port $BACKEND_PORT is in use"
        return 1
    fi

    # Check venv
    if [ ! -d "$VENV_DIR" ]; then
        echo "❌ Virtual environment not found at $VENV_DIR"
        return 1
    fi

    # Check .env file
    if [ ! -f "$BACKEND_DIR/.env" ]; then
        echo "❌ Backend .env file not found"
        echo "   Create $BACKEND_DIR/.env first"
        return 1
    fi

    # Start backend
    echo "🚀 Starting backend..."
    cd "$BACKEND_DIR"
    source "$VENV_DIR/bin/activate"
    nohup python main.py > "$BACKEND_LOG" 2>&1 &
    BACKEND_PID=$!
    echo $BACKEND_PID > "$BACKEND_PID_FILE"

    sleep 3
    if ps -p "$BACKEND_PID" > /dev/null 2>&1; then
        echo "✅ Backend started (PID: $BACKEND_PID)"
        echo "   URL: http://localhost:$BACKEND_PORT"
        echo "   API Docs: http://localhost:$BACKEND_PORT/docs"
    else
        echo "❌ Backend failed to start"
        echo "   Check log: $BACKEND_LOG"
        return 1
    fi
}

# Execute based on mode
case "$MODE" in
    frontend)
        start_frontend
        ;;
    backend)
        start_backend
        ;;
    all)
        start_backend
        start_frontend
        ;;
    *)
        echo "❌ Invalid mode: $MODE"
        echo "Usage: ./START.sh [frontend|backend|all]"
        exit 1
        ;;
esac

echo ""
echo "========================================"
echo "✅ STARTUP COMPLETE"
echo "========================================"
echo ""
echo "📋 Commands:"
echo "   ./STATUS.sh  - Check status"
echo "   ./STOP.sh    - Stop services"
echo ""
```

#### 7.3: Update STOP.sh (Manage Both Services)

Replace the entire content of `/home/ec2-user/encore-loyalty-new-system/STOP.sh`:

```bash
#!/bin/bash

# STOP.sh - Stop Frontend and Backend for Encore Loyalty System
# Usage: ./STOP.sh [frontend|backend|all]

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
FRONTEND_DIR="$PROJECT_ROOT/encore_frontend"
BACKEND_DIR="$PROJECT_ROOT/encore_backend"

FRONTEND_PID_FILE="$FRONTEND_DIR/.server.pid"
BACKEND_PID_FILE="$BACKEND_DIR/.server.pid"

FRONTEND_PORT=3000
BACKEND_PORT=8000

# Parse arguments
MODE="${1:-all}"

echo "========================================"
echo "Encore Loyalty System - STOP"
echo "========================================"

stop_frontend() {
    echo ""
    echo "🎨 FRONTEND (port $FRONTEND_PORT)"
    echo "----------------------------------------"

    if [ -f "$FRONTEND_PID_FILE" ]; then
        PID=$(cat "$FRONTEND_PID_FILE")
        if ps -p "$PID" > /dev/null 2>&1; then
            echo "🛑 Stopping frontend (PID: $PID)..."
            kill -15 "$PID" 2>/dev/null
            sleep 2
            if ps -p "$PID" > /dev/null 2>&1; then
                kill -9 "$PID" 2>/dev/null
            fi
            rm -f "$FRONTEND_PID_FILE"
            echo "✅ Frontend stopped"
        else
            echo "⚠️  Frontend not running (stale PID file)"
            rm -f "$FRONTEND_PID_FILE"
        fi
    else
        # Check port
        PID=$(lsof -ti :$FRONTEND_PORT 2>/dev/null)
        if [ ! -z "$PID" ]; then
            echo "🛑 Stopping process on port $FRONTEND_PORT (PID: $PID)..."
            kill -9 "$PID" 2>/dev/null
            echo "✅ Frontend stopped"
        else
            echo "✅ Frontend already stopped"
        fi
    fi
}

stop_backend() {
    echo ""
    echo "⚙️  BACKEND (port $BACKEND_PORT)"
    echo "----------------------------------------"

    if [ -f "$BACKEND_PID_FILE" ]; then
        PID=$(cat "$BACKEND_PID_FILE")
        if ps -p "$PID" > /dev/null 2>&1; then
            echo "🛑 Stopping backend (PID: $PID)..."
            kill -15 "$PID" 2>/dev/null
            sleep 2
            if ps -p "$PID" > /dev/null 2>&1; then
                kill -9 "$PID" 2>/dev/null
            fi
            rm -f "$BACKEND_PID_FILE"
            echo "✅ Backend stopped"
        else
            echo "⚠️  Backend not running (stale PID file)"
            rm -f "$BACKEND_PID_FILE"
        fi
    else
        # Check port
        PID=$(lsof -ti :$BACKEND_PORT 2>/dev/null)
        if [ ! -z "$PID" ]; then
            echo "🛑 Stopping process on port $BACKEND_PORT (PID: $PID)..."
            kill -9 "$PID" 2>/dev/null
            echo "✅ Backend stopped"
        else
            echo "✅ Backend already stopped"
        fi
    fi
}

# Execute based on mode
case "$MODE" in
    frontend)
        stop_frontend
        ;;
    backend)
        stop_backend
        ;;
    all)
        stop_frontend
        stop_backend
        ;;
    *)
        echo "❌ Invalid mode: $MODE"
        echo "Usage: ./STOP.sh [frontend|backend|all]"
        exit 1
        ;;
esac

echo ""
echo "========================================"
echo "✅ SHUTDOWN COMPLETE"
echo "========================================"
echo ""
```

#### 7.4: Update STATUS.sh (Check Both Services)

Replace the entire content of `/home/ec2-user/encore-loyalty-new-system/STATUS.sh`:

```bash
#!/bin/bash

# STATUS.sh - Check status of Frontend and Backend for Encore Loyalty System
# Usage: ./STATUS.sh

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
FRONTEND_DIR="$PROJECT_ROOT/encore_frontend"
BACKEND_DIR="$PROJECT_ROOT/encore_backend"

FRONTEND_PID_FILE="$FRONTEND_DIR/.server.pid"
BACKEND_PID_FILE="$BACKEND_DIR/.server.pid"
FRONTEND_LOG="$FRONTEND_DIR/.server.log"
BACKEND_LOG="$BACKEND_DIR/.server.log"

FRONTEND_PORT=3000
BACKEND_PORT=8000

echo "========================================"
echo "Encore Loyalty System - STATUS"
echo "========================================"

# Backend Status
echo ""
echo "⚙️  BACKEND (FastAPI @ port $BACKEND_PORT)"
echo "----------------------------------------"

BACKEND_RUNNING=false
if [ -f "$BACKEND_PID_FILE" ]; then
    BACKEND_PID=$(cat "$BACKEND_PID_FILE")
    if ps -p "$BACKEND_PID" > /dev/null 2>&1; then
        echo "✅ Status: RUNNING (PID: $BACKEND_PID)"
        BACKEND_RUNNING=true
    else
        echo "❌ Status: NOT RUNNING (stale PID)"
    fi
else
    PORT_PID=$(lsof -ti :$BACKEND_PORT 2>/dev/null)
    if [ ! -z "$PORT_PID" ]; then
        echo "⚠️  Status: RUNNING on port (PID: $PORT_PID, no PID file)"
        BACKEND_RUNNING=true
    else
        echo "❌ Status: NOT RUNNING"
    fi
fi

# Backend URL check
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:$BACKEND_PORT/api/v1/health 2>/dev/null)
if [ "$HTTP_STATUS" = "200" ]; then
    echo "✅ Health: HEALTHY (HTTP $HTTP_STATUS)"
    echo "   📡 API: http://localhost:$BACKEND_PORT"
    echo "   📚 Docs: http://localhost:$BACKEND_PORT/docs"
else
    echo "❌ Health: NOT RESPONDING (HTTP ${HTTP_STATUS:-000})"
fi

# Backend log
if [ -f "$BACKEND_LOG" ]; then
    echo "📄 Log: $BACKEND_LOG ($(du -h "$BACKEND_LOG" | cut -f1))"
else
    echo "📄 Log: No log file"
fi

# Frontend Status
echo ""
echo "🎨 FRONTEND (React @ port $FRONTEND_PORT)"
echo "----------------------------------------"

FRONTEND_RUNNING=false
if [ -f "$FRONTEND_PID_FILE" ]; then
    FRONTEND_PID=$(cat "$FRONTEND_PID_FILE")
    if ps -p "$FRONTEND_PID" > /dev/null 2>&1; then
        echo "✅ Status: RUNNING (PID: $FRONTEND_PID)"
        FRONTEND_RUNNING=true
    else
        echo "❌ Status: NOT RUNNING (stale PID)"
    fi
else
    PORT_PID=$(lsof -ti :$FRONTEND_PORT 2>/dev/null)
    if [ ! -z "$PORT_PID" ]; then
        echo "⚠️  Status: RUNNING on port (PID: $PORT_PID, no PID file)"
        FRONTEND_RUNNING=true
    else
        echo "❌ Status: NOT RUNNING"
    fi
fi

# Frontend URL check
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:$FRONTEND_PORT 2>/dev/null)
if [ "$HTTP_STATUS" = "200" ]; then
    echo "✅ Health: HEALTHY (HTTP $HTTP_STATUS)"
    echo "   🌐 URL: http://localhost:$FRONTEND_PORT"
else
    echo "❌ Health: NOT RESPONDING (HTTP ${HTTP_STATUS:-000})"
fi

# Frontend log
if [ -f "$FRONTEND_LOG" ]; then
    echo "📄 Log: $FRONTEND_LOG ($(du -h "$FRONTEND_LOG" | cut -f1))"
else
    echo "📄 Log: No log file"
fi

# Public IP
echo ""
echo "🌍 PUBLIC ACCESS"
echo "----------------------------------------"
PUBLIC_IP=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" 2>/dev/null | xargs -I {} curl -s -H "X-aws-ec2-metadata-token: {}" http://169.254.169.254/latest/meta-data/public-ipv4 2>/dev/null)
if [ ! -z "$PUBLIC_IP" ]; then
    echo "📍 Public IP: $PUBLIC_IP"
    echo "   Backend:  http://$PUBLIC_IP:$BACKEND_PORT"
    echo "   Frontend: http://$PUBLIC_IP:$FRONTEND_PORT"
else
    echo "⚠️  Not on EC2 or metadata unavailable"
fi

# Summary
echo ""
echo "========================================"
echo "📊 SYSTEM SUMMARY"
echo "========================================"

if [ "$BACKEND_RUNNING" = true ] && [ "$FRONTEND_RUNNING" = true ]; then
    echo "✅ SYSTEM STATUS: FULLY OPERATIONAL"
    echo ""
    echo "🔗 Quick Links:"
    echo "   Backend API:  http://localhost:$BACKEND_PORT"
    echo "   Backend Docs: http://localhost:$BACKEND_PORT/docs"
    echo "   Frontend App: http://localhost:$FRONTEND_PORT"
elif [ "$BACKEND_RUNNING" = true ]; then
    echo "⚠️  SYSTEM STATUS: BACKEND ONLY"
    echo "   Frontend is not running. Start with: ./START.sh frontend"
elif [ "$FRONTEND_RUNNING" = true ]; then
    echo "⚠️  SYSTEM STATUS: FRONTEND ONLY"
    echo "   Backend is not running. Start with: ./START.sh backend"
else
    echo "❌ SYSTEM STATUS: ALL SERVICES DOWN"
    echo "   Start with: ./START.sh"
fi

echo ""
```

#### 7.5: Test the New Scripts

```bash
cd /home/ec2-user/encore-loyalty-new-system

# Test status (should show nothing running yet)
./STATUS.sh

# Start backend only
./START.sh backend

# Check status
./STATUS.sh

# Start frontend
./START.sh frontend

# Check full status
./STATUS.sh

# Test stop
./STOP.sh all
```

**Verification**:
- [ ] Scripts moved to project root
- [ ] Scripts are executable (chmod +x)
- [ ] START.sh can start backend and frontend separately or together
- [ ] STOP.sh can stop services individually or all
- [ ] STATUS.sh shows status of both services
- [ ] All scripts work correctly

---

## ✅ COMPLETION CRITERIA

**Phase Complete When**:

1. **Environment Configured**:
   - [ ] `.env` file created with proper settings
   - [ ] SSH key permissions set to 600
   - [ ] AWS noted as already configured (no credential setup needed)

2. **Server Running**:
   - [ ] Backend starts without critical errors
   - [ ] Server listening on port 8000
   - [ ] No import/dependency errors

3. **Basic Endpoints Working**:
   - [ ] Root endpoint (/) returns JSON
   - [ ] Health endpoint (/api/v1/health) returns status
   - [ ] Swagger UI accessible at /docs

4. **MySQL Connection** (Optional for now):
   - [ ] Clients endpoint works OR error documented
   - [ ] SSH tunnel functional OR issue noted for later

5. **Unified Management Scripts**:
   - [ ] START.sh, STOP.sh, STATUS.sh moved to project root
   - [ ] Scripts updated to manage both frontend and backend
   - [ ] All scripts tested and working
   - [ ] Can start/stop services individually or together

6. **Documentation**:
   - [ ] Current capabilities understood
   - [ ] Known issues documented
   - [ ] Ready for Phase 2 API development

---

## 🚨 TROUBLESHOOTING

### Issue: Server won't start

**Symptoms**: Python errors, import failures, module not found

**Solutions**:
```bash
# Reinstall dependencies
source venv/bin/activate
pip install -r requirements.txt

# Check Python version (needs 3.9+)
python --version

# Try running directly with uvicorn
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

---

### Issue: Port 8000 already in use

**Symptoms**: `Address already in use` error

**Solutions**:
```bash
# Find process using port 8000
lsof -i :8000

# Kill the process (replace PID)
kill -9 <PID>

# Or use a different port
PORT=8001 python main.py
```

---

### Issue: MySQL connection fails

**Symptoms**: Timeout, connection refused, SSH errors

**Solutions**:
```bash
# Test SSH connection manually
ssh -i key/Encoreepad_ppk.pem ubuntu@52.14.57.81

# Check SSH key permissions
chmod 600 key/Encoreepad_ppk.pem

# Verify .env file has correct settings
cat .env | grep -E "(SSH_|MYSQL_)"
```

**Temporary Workaround**: Disable endpoints requiring MySQL for now.

---

### Issue: AWS service errors

**Symptoms**: Boto3 errors, credential errors

**Solutions**:
- This is expected if AWS credentials aren't configured yet
- Verify `.env` has `BEDROCK_ENABLED=false` and `SES_ENABLED=false`
- Health endpoint will show these services as unavailable (that's OK)

**Note**: We'll configure AWS services in Phase 2 when needed.

---

### Issue: Cannot access Swagger UI

**Symptoms**: 404 error at /docs

**Solutions**:
```bash
# Check if server is running
curl http://localhost:8000/

# Check if FastAPI docs are enabled
# In config.py, verify debug=True

# Try accessing on different host
curl http://127.0.0.1:8000/docs
```

---

## 📋 TESTING CHECKLIST

After completing setup, verify:

**Server Start**:
- [ ] Activate venv: `source venv/bin/activate`
- [ ] Start server: `cd encore_backend && python main.py`
- [ ] No critical errors in console
- [ ] Server listening on port 8000

**Endpoints**:
- [ ] Root: `curl http://localhost:8000/` returns JSON
- [ ] Health: `curl http://localhost:8000/api/v1/health` returns status
- [ ] Docs: Open `http://localhost:8000/docs` in browser

**Database** (Optional):
- [ ] Clients: `curl http://localhost:8000/api/v1/clients/` returns data OR error noted

**Cleanup**:
- [ ] Stop server: Press Ctrl+C
- [ ] Document any errors or warnings
- [ ] Ready for Phase 2

---

## 📊 REPORTING BACK

After completing this setup, report:

### Success Report Template

```
✅ BACKEND SETUP COMPLETE

Environment:
- .env file created: ✅
- SSH key permissions: ✅ (600)
- AWS credentials: ✅ Already configured on EC2 (no setup needed)
- AWS services: Disabled for development (will enable in Phase 2+)
- MySQL credentials: Configured from customer data

Server Status:
- Backend starts: ✅
- Port 8000 listening: ✅
- No critical errors: ✅

Endpoints Tested:
- Root (/): ✅ Returns {"message": "...", "version": "3.0"}
- Health (/api/v1/health): ✅ Returns status
- Docs (/docs): ✅ Accessible
- Clients (/api/v1/clients/): [✅ Works / ⚠️ MySQL error - noted]

Unified Management Scripts:
- Scripts moved to project root: ✅
- START.sh: ✅ Can start frontend, backend, or both
- STOP.sh: ✅ Can stop services individually or all
- STATUS.sh: ✅ Shows status of both services
- Tested: ✅ All scripts working

Current Capabilities:
- Health monitoring: ✅
- MySQL via SSH tunnel: [✅ / ⚠️]
- AWS DynamoDB: Available (credentials configured, currently disabled)
- AWS Bedrock AI: Available (credentials configured, currently disabled)
- AWS SES Email: Available (credentials configured, currently disabled)

System Management:
- ./START.sh [frontend|backend|all] - Start services
- ./STOP.sh [frontend|backend|all] - Stop services
- ./STATUS.sh - Check system status

Issues Found: [None / List any issues]

Ready for Phase 2: YES

Notes:
- Backend runs successfully on http://localhost:8000
- Frontend runs on http://localhost:3000 (if started)
- Unified scripts manage both services
- Swagger UI available for API testing at /docs
- MySQL connection [working/needs debugging]
- AWS services disabled for initial development, will enable when needed
- Ready to implement authentication endpoints (Phase 2)
```

---

## 🎯 SUCCESS DEFINITION

**Backend is operational when**:
1. Server starts without critical errors
2. Health endpoint responds
3. Swagger UI accessible
4. Basic endpoints return data OR errors are documented
5. Ready for Phase 2 API development

**Time**: 30-45 minutes

**Next Phase**: Phase 2 - Authentication Implementation (Backend + Frontend)

---

## 🔗 REFERENCE DOCUMENTS

**Backend Documentation**:
- Backend README: `/encore_backend/README.md`
- Configuration: `/encore_backend/config.py`
- Main entry point: `/encore_backend/main.py`

**Related Plans**:
- MVP Execution Plan: `/docs/SUPERVISOR/MVP-EXECUTION-PLAN.md`
- Backend Requirements: `/docs/plan-mockup-to-frontend/04-BACKEND-REQUIREMENTS.md`
- Phase 1 (Frontend): `/docs/SUPERVISOR/WORKER-PROMPTS/PHASE-1-FOUNDATION-SETUP.md`

**Investigation Reports**:
- Archive Analysis: `/docs/investigation/ENCORE-ARCHIVE-ANALYSIS-REPORT.md`
- Database Schema: Referenced in archive analysis report

---

## 💡 IMPORTANT NOTES

### About AWS Services
- **For MVP Week 1**: We're disabling AWS services to focus on core functionality
- **AWS Bedrock** (AI): Will enable on Day 4 when implementing AI response generation
- **AWS SES** (Email): Will enable on Day 5 when implementing email sending
- **AWS DynamoDB**: Will configure when needed for data storage
- **AWS S3**: Will configure for customer profiles storage

### About Authentication
- **Initial Setup**: `REQUIRE_AUTH=false` (no API key required)
- **Phase 2**: Will implement JWT authentication
- **Production**: Will enable full authentication

### About MySQL Connection
- **SSH Tunnel**: May be slow on first connection (30-60 seconds)
- **Customer Data**: Credentials are from customer's legacy system
- **Read-Only**: We only read from MySQL, never write to it

### Development vs Production
- **Current Setup**: Development mode for MVP
- **Production Setup**: Will configure later with proper AWS credentials, authentication, and monitoring

---

**READY TO START?** Copy this prompt into Cursor and begin backend setup!

Good luck! 🚀
