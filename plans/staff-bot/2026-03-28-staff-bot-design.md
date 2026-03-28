# uSipipo Staff Bot - Design Document

**Date:** 2026-03-28  
**Version:** 1.0  
**Status:** Proposed  
**Bot Handle:** `@uSipipoStaff_Bot`  
**Related Backend:** usipipo-backend v0.10.0+  

---

## 📋 Overview

### **Purpose**
The uSipipo Staff Bot is a dedicated Telegram bot for administrative operations. It provides authorized staff members with complete control over user management, VPN keys, support tickets, and server monitoring through a secure, role-based interface.

### **Business Goals**
- ✅ **Complete Administrative Control** - Full access to all backend admin endpoints
- ✅ **Role-Based Access** - Staff permissions via Telegram ID whitelist
- ✅ **Operational Efficiency** - Quick actions for common admin tasks
- ✅ **Security First** - Strict access control, audit logging
- ✅ **Separation of Duties** - Clear separation from user-facing bots

### **Scope**
This bot handles **all administrative operations** for the uSipipo ecosystem. It is exclusively for authorized staff members and provides access to:
- User management (view, block, delete, assign roles)
- VPN key management (view, toggle, delete)
- Support ticket management (view all, assign, respond)
- Server monitoring (status, statistics)
- Dashboard and analytics

---

## 🎯 Features

### **Staff Commands**

#### **Dashboard**
| Command | Description | Example |
|---------|-------------|---------|
| `/start` | Initialize bot and show welcome | `/start` |
| `/admin` | Main admin dashboard | `/admin` |
| `/stats` | Platform statistics | `/stats` |

#### **User Management**
| Command | Description | Example |
|---------|-------------|---------|
| `/users` | List all users (paginated) | `/users` |
| `/user/{id}` | View user details | `/user/123456789` |
| `/block/{id}` | Block a user | `/block/123456789` |
| `/unblock/{id}` | Unblock a user | `/unblock/123456789` |
| `/assignrole/{id}` | Assign role to user | `/assignrole/123456789` |
| `/deleteuser/{id}` | Delete user permanently | `/deleteuser/123456789` |

#### **VPN Key Management**
| Command | Description | Example |
|---------|-------------|---------|
| `/keys` | List all VPN keys | `/keys` |
| `/userkeys/{id}` | List keys for specific user | `/userkeys/123456789` |
| `/togglekey/{id}` | Activate/deactivate key | `/togglekey/uuid-123` |
| `/deletekey/{id}` | Delete VPN key | `/deletekey/uuid-123` |

#### **Ticket Management**
| Command | Description | Example |
|---------|-------------|---------|
| `/tickets` | List all support tickets | `/tickets` |
| `/ticket/{id}` | View ticket details | `/ticket/uuid-123` |
| `/assignticket/{id}` | Assign ticket to staff | `/assignticket/uuid-123` |
| `/closeticket/{id}` | Close ticket | `/closeticket/uuid-123` |

#### **Server Monitoring**
| Command | Description | Example |
|---------|-------------|---------|
| `/servers` | View server status | `/servers` |
| `/serverstats` | View server statistics | `/serverstats` |

### **Inline Keyboard Actions**

All commands support interactive inline keyboards for:
- ✅ Pagination (previous/next pages)
- ✅ Quick actions (block, unblock, delete)
- ✅ Filtering (by status, type, date)
- ✅ Navigation (back to dashboard)

---

## 🏗️ Architecture

### **High-Level Architecture**

```
┌─────────────────┐
│   Telegram      │
│   @uSipipoStaff_Bot   │
└────────┬────────┘
         │
         │ Telegram Bot API
         │
┌────────▼────────┐
│  Staff Bot      │
│  Application    │
│  (Python 3.13)  │
│  + Admin Auth   │
└────────┬────────┘
         │
         │ HTTP/REST
         │
┌────────▼────────┐
│  usipipo-backend│
│  API v0.10.0+   │
│  (Admin Routes) │
└────────┬────────┘
         │
         │ SQLAlchemy Async
         │
┌────────▼────────┐
│  PostgreSQL     │
│  + Redis        │
└─────────────────┘
```

### **Security Architecture**

```
┌──────────────────────────────────────────┐
│  Staff Bot Security Layers               │
├──────────────────────────────────────────┤
│  1. Telegram ID Whitelist                │
│  2. JWT Token Authentication             │
│  3. Backend Admin Role Verification      │
│  4. Rate Limiting                        │
│  5. Audit Logging                        │
└──────────────────────────────────────────┘
```

### **Project Structure**

```
usipipo-staff-bot/
├── src/
│   ├── bot/
│   │   ├── handlers/
│   │   │   ├── admin.py             # Dashboard & main menu
│   │   │   ├── users.py             # User management
│   │   │   ├── keys.py              # VPN key management
│   │   │   ├── tickets.py           # Ticket management
│   │   │   ├── servers.py           # Server monitoring
│   │   │   └── stats.py             # Statistics & analytics
│   │   ├── keyboards/
│   │   │   ├── admin.py             # Dashboard keyboards
│   │   │   ├── users.py             # User action keyboards
│   │   │   ├── keys.py              # Key action keyboards
│   │   │   ├── tickets.py           # Ticket action keyboards
│   │   │   ├── servers.py           # Server status keyboards
│   │   │   └── stats.py             # Stats filter keyboards
│   │   └── middlewares/
│   │       └── admin_auth.py        # Staff authorization middleware
│   ├── infrastructure/
│   │   ├── api_client.py            # HTTP client for backend API
│   │   ├── config.py                # Pydantic settings
│   │   ├── redis.py                 # Redis connection pool
│   │   ├── token_storage.py         # JWT token management
│   │   ├── logger.py                # Structured logging
│   │   └── error_handler.py         # Global error handling
│   └── main.py                      # Application entry point
├── tests/
│   ├── bot/
│   │   ├── test_admin_handlers.py
│   │   ├── test_users_handlers.py
│   │   ├── test_keys_handlers.py
│   │   ├── test_tickets_handlers.py
│   │   └── test_servers_handlers.py
│   ├── integration/
│   │   └── test_backend_integration.py
│   └── conftest.py
├── docs/
│   ├── ARCHITECTURE.md
│   ├── DEPLOYMENT.md
│   └── STAFF-GUIDE.md
├── .github/
│   └── workflows/
│       └── ci.yml
├── .env
├── .gitignore
├── .pre-commit-config.yaml
├── CHANGELOG.md
├── Dockerfile
├── example.env
├── LICENSE
├── pyproject.toml
├── README.md
├── requirements.txt
├── usipipo-staff-bot.service
└── uv.lock
```

---

## 🔌 Backend Integration

### **Admin API Endpoints**

#### **Dashboard**
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/admin/dashboard/stats` | Get dashboard statistics |

#### **User Management**
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/admin/users` | List all users |
| `GET` | `/api/v1/admin/users/paginated` | Paginated user list |
| `POST` | `/api/v1/admin/users/{id}/status` | Update user status |
| `POST` | `/api/v1/admin/users/{id}/role` | Assign role to user |
| `POST` | `/api/v1/admin/users/{id}/block` | Block user |
| `POST` | `/api/v1/admin/users/{id}/unblock` | Unblock user |
| `DELETE` | `/api/v1/admin/users/{id}` | Delete user |

#### **VPN Key Management**
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/admin/keys` | List all VPN keys |
| `GET` | `/api/v1/admin/users/{id}/keys` | List user's keys |
| `POST` | `/api/v1/admin/keys/{id}/toggle` | Toggle key status |
| `DELETE` | `/api/v1/admin/keys/{id}` | Delete VPN key |

#### **Ticket Management**
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/admin/tickets` | List all tickets |
| `GET` | `/api/v1/admin/tickets/{id}` | Get ticket details |
| `PATCH` | `/api/v1/admin/tickets/{id}/assign` | Assign ticket to staff |
| `PATCH` | `/api/v1/admin/tickets/{id}/status` | Update ticket status |
| `POST` | `/api/v1/admin/tickets/{id}/messages` | Send message to ticket |

#### **Server Management**
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/admin/servers/status` | Get server status |
| `GET` | `/api/v1/admin/servers/stats` | Get server statistics |

### **Request/Response Examples**

#### **Get Dashboard Stats**
```http
GET /api/v1/admin/dashboard/stats
Authorization: Bearer <admin_access_token>
```

**Response:**
```json
{
  "total_users": 1250,
  "active_users": 980,
  "new_users_today": 15,
  "total_keys": 3500,
  "active_keys": 2800,
  "total_revenue_usd": 12500.00,
  "revenue_today_usd": 350.00
}
```

#### **Block User**
```http
POST /api/v1/admin/users/123456789/block
Authorization: Bearer <admin_access_token>
```

**Response:**
```json
{
  "success": true,
  "operation": "block_user",
  "target_id": 123456789,
  "message": "User blocked successfully",
  "timestamp": "2026-03-28T10:00:00Z"
}
```

---

## 🔐 Security & Access Control

### **Staff Authorization**

```python
# .env configuration
SUPPORT_STAFF_IDS=123456789,987654321,111222333
ADMIN_SECRET_KEY=your_super_secret_admin_key
```

### **Admin Auth Middleware**

```python
class AdminAuthMiddleware:
    """Middleware for staff authorization."""
    
    def __init__(self, allowed_staff_ids: list[int]):
        self.allowed_ids = set(allowed_staff_ids)
    
    async def __call__(
        self,
        update: Update,
        dispatcher: Dispatcher
    ):
        """Verify user is authorized staff member."""
        if update.effective_user is None:
            return
        
        user_id = update.effective_user.id
        
        # Check if user is in whitelist
        if user_id not in self.allowed_ids:
            logger.warning(f"Unauthorized access attempt by {user_id}")
            await update.message.reply_text(
                "❌ Acceso Denegado: No tenés permisos de staff."
            )
            return
        
        # Verify backend admin role
        tokens = await self.token_storage.get(user_id)
        if not tokens:
            await update.message.reply_text(
                "❌ Error de autenticación. Por favor reiniciá el bot."
            )
            return
        
        await dispatcher.process_update(update)
```

### **Security Checklist**

- ✅ **Telegram ID Whitelist** - Only authorized staff can access
- ✅ **JWT Authentication** - All API calls authenticated
- ✅ **Backend Role Verification** - Admin role verified on backend
- ✅ **Rate Limiting** - Backend enforces admin rate limits
- ✅ **Audit Logging** - All admin actions logged
- ✅ **Secure Token Storage** - Tokens in Redis with TTL
- ✅ **HTTPS Only** - All communications encrypted
- ✅ **Input Validation** - All inputs sanitized

---

## 💻 Implementation Details

### **Dependencies**

```toml
[project]
name = "usipipo-staff-bot"
version = "0.1.0"
requires-python = ">=3.13"

dependencies = [
    "usipipo-commons>=0.12.0",
    "python-telegram-bot>=21.0",
    "pydantic-settings>=2.0.0",
    "redis>=7.3.0",
    "httpx>=0.27.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.0.0",
    "mypy>=1.0.0",
    "ruff>=0.1.0",
    "bandit>=1.7.0",
]
```

### **Key Components**

#### **1. AdminHandler (Dashboard)**
```python
class AdminHandler:
    """Handler for admin dashboard."""
    
    async def show_dashboard(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Show main admin dashboard with stats."""
        pass
    
    async def refresh_stats(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Refresh dashboard statistics."""
        pass
```

#### **2. UsersHandler**
```python
class UsersHandler:
    """Handler for user management."""
    
    async def list_users(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """List all users with pagination."""
        pass
    
    async def view_user_detail(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """View detailed user information."""
        pass
    
    async def block_user(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Block a user."""
        pass
    
    async def unblock_user(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Unblock a user."""
        pass
    
    async def delete_user(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Delete user permanently."""
        pass
```

#### **3. KeysHandler**
```python
class KeysHandler:
    """Handler for VPN key management."""
    
    async def list_all_keys(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """List all VPN keys from all users."""
        pass
    
    async def list_user_keys(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """List all keys for specific user."""
        pass
    
    async def toggle_key(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Activate or deactivate a key."""
        pass
    
    async def delete_key(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Delete VPN key completely."""
        pass
```

#### **4. TicketsHandler**
```python
class TicketsHandler:
    """Handler for ticket management (staff view)."""
    
    async def list_all_tickets(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """List all support tickets."""
        pass
    
    async def view_ticket_detail(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """View ticket with all messages."""
        pass
    
    async def assign_ticket(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Assign ticket to staff member."""
        pass
    
    async def respond_to_ticket(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Send response to ticket."""
        pass
    
    async def close_ticket(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Close support ticket."""
        pass
```

#### **5. ServersHandler**
```python
class ServersHandler:
    """Handler for server monitoring."""
    
    async def show_server_status(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Show status of all VPN servers."""
        pass
    
    async def show_server_stats(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Show comprehensive server statistics."""
        pass
```

---

## 🧪 Testing Strategy

### **Test Categories**

| Type | Count | Description |
|------|-------|-------------|
| Unit Tests | ~100 | Handler logic, keyboards, messages |
| Integration Tests | ~20 | Backend API integration |
| Security Tests | ~10 | Auth, authorization, access control |
| End-to-End Tests | ~10 | Complete admin workflows |
| **Total** | **~140** | **All automated** |

### **Test Coverage Goals**

- ✅ **Handlers:** 95%+ coverage
- ✅ **Middlewares:** 100% coverage (security critical)
- ✅ **Keyboards:** 100% coverage
- ✅ **Integration:** All admin endpoints covered
- ✅ **Security:** Auth flows fully tested

### **Security Testing**

```python
class TestAdminAuthMiddleware:
    """Test admin authorization middleware."""
    
    async def test_unauthorized_user_denied(self):
        """Test that unauthorized users are denied access."""
        pass
    
    async def test_authorized_user_allowed(self):
        """Test that authorized staff can access."""
        pass
    
    async def test_missing_token_denied(self):
        """Test that missing tokens are denied."""
        pass
```

### **CI/CD Pipeline**

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
      - run: pip install uv && uv sync --dev
      
      # Code Quality
      - run: uv run ruff check src tests
      - run: uv run mypy src
      
      # Security Scan
      - run: uv run bandit -r src
      
      # Tests
      - run: uv run pytest --cov=src --cov-report=xml
      
      # Upload coverage
      - uses: codecov/codecov-action@v3
```

---

## 🚀 Deployment

### **Infrastructure Requirements**

| Resource | Specification |
|----------|--------------|
| **CPU** | 1 core |
| **RAM** | 512 MB |
| **Storage** | 2 GB |
| **Network** | Outbound HTTPS (443) |
| **Services** | Redis (shared), PostgreSQL |

### **Docker Configuration**

```dockerfile
# Dockerfile
FROM python:3.13-slim

WORKDIR /app

# Install dependencies
RUN pip install uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# Copy source
COPY src/ ./src/

# Run bot
CMD ["python", "-m", "src"]
```

### **systemd Service**

```ini
# usipipo-staff-bot.service
[Unit]
Description=uSipipo Staff Admin Telegram Bot
After=network.target redis.service

[Service]
Type=simple
User=usipipo
WorkingDirectory=/opt/usipipo-staff-bot
Environment=PATH=/opt/usipipo-staff-bot/.venv/bin
EnvironmentFile=/opt/usipipo-staff-bot/.env
ExecStart=/opt/usipipo-staff-bot/.venv/bin/python -m src
Restart=always
RestartSec=5

# Security
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

### **Environment Variables**

```bash
# .env
# Bot Configuration
BOT_TOKEN=telegram_bot_token_staff
BOT_USERNAME=uSipipoStaff_Bot

# Backend API
BACKEND_URL=https://api.usipipo.com
API_PREFIX=/api/v1

# Redis
REDIS_URL=redis://localhost:6379/2

# Security
SUPPORT_STAFF_IDS=123456789,987654321,111222333
ADMIN_SECRET_KEY=your_super_secret_admin_key

# Logging
LOG_LEVEL=INFO
LOG_FILE=/var/log/usipipo/staff-bot.log
```

---

## 📊 Monitoring & Observability

### **Logging**

```python
# Structured logging format (JSON)
{
  "timestamp": "2026-03-28T10:00:00Z",
  "level": "INFO",
  "logger": "users_handler",
  "message": "Admin blocked user",
  "admin_telegram_id": 123456789,
  "admin_username": "admin_john",
  "target_user_id": 987654321,
  "operation": "block_user",
  "success": true
}
```

### **Audit Log Events**

| Event | Description | Logged Data |
|-------|-------------|-------------|
| `user_blocked` | Admin blocked a user | admin_id, target_user_id, reason |
| `user_unblocked` | Admin unblocked a user | admin_id, target_user_id |
| `user_deleted` | Admin deleted a user | admin_id, target_user_id |
| `key_toggled` | Admin toggled a key | admin_id, key_id, new_status |
| `key_deleted` | Admin deleted a key | admin_id, key_id |
| `ticket_assigned` | Admin assigned ticket | admin_id, ticket_id, assignee |
| `ticket_responded` | Admin responded to ticket | admin_id, ticket_id |
| `unauthorized_access` | Unauthorized access attempt | telegram_id, username, command |

### **Metrics to Track**

| Metric | Description | Target |
|--------|-------------|--------|
| Admin Actions/Day | Total admin operations | Track trend |
| Avg Response Time | Bot response time | < 500ms |
| Error Rate | Failed operations | < 0.1% |
| Unauthorized Attempts | Failed auth attempts | Alert if > 5/day |

---

## 📝 Admin Workflows

### **User Management Workflow**

```
1. Admin: /users
2. Bot: Shows paginated list of users
3. Admin: Selects user (clicks inline button)
4. Bot: Shows user detail with actions
5. Admin: Clicks "Block User"
6. Bot: Shows confirmation dialog
7. Admin: Confirms
8. Bot: Calls POST /admin/users/{id}/block
9. Bot: Shows success message
10. Bot: Logs audit event
```

### **Ticket Response Workflow**

```
1. Admin: /tickets
2. Bot: Shows list of open tickets
3. Admin: Selects ticket
4. Bot: Shows ticket detail + messages
5. Admin: Clicks "Responder"
6. Bot: Prompts for message text
7. Admin: Types response
8. Bot: Calls POST /admin/tickets/{id}/messages
9. Bot: Shows success message
10. User: Receives notification via @uSipipoSupport_Bot
```

### **VPN Key Troubleshooting**

```
1. User reports VPN issue (via support ticket)
2. Staff: /userkeys/{user_id}
3. Bot: Shows all keys for user
4. Staff: Identifies problematic key
5. Staff: Clicks "Toggle Key" → "Deactivate"
6. Bot: Confirms key deactivated
7. Staff: Clicks "Toggle Key" → "Activate"
8. Bot: Confirms key reactivated
9. Staff: Responds to ticket with status
```

---

## 🔒 Security Considerations

### **Access Control Matrix**

| Role | View Users | Manage Users | View Keys | Manage Keys | View Tickets | Respond Tickets |
|------|-----------|--------------|-----------|-------------|--------------|-----------------|
| **Super Admin** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Support Staff** | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ |
| **Viewer** | ✅ | ❌ | ✅ | ❌ | ✅ | ❌ |

*Note: Initial implementation uses simple whitelist. Role-based access can be added in future versions.*

### **Security Best Practices**

1. **Least Privilege** - Only grant necessary permissions
2. **Defense in Depth** - Multiple security layers
3. **Audit Everything** - All actions logged
4. **Rate Limiting** - Prevent abuse
5. **Token Rotation** - Regular token refresh
6. **Session Timeout** - Auto-logout after inactivity
7. **IP Whitelisting** - Optional: restrict by IP
8. **2FA Support** - Future enhancement

---

## 📅 Timeline

| Phase | Duration | Tasks |
|-------|----------|-------|
| **Phase 1: Setup** | 1 day | Repo creation, structure, CI/CD |
| **Phase 2: Core Handlers** | 3 days | Admin, Users, Keys handlers |
| **Phase 3: Tickets + Servers** | 2 days | Tickets, Servers handlers |
| **Phase 4: Security** | 1 day | Auth middleware, audit logging |
| **Phase 5: Testing** | 1.5 days | Unit + integration + security tests |
| **Phase 6: Deploy** | 1 day | Docker, systemd, monitoring |
| **Total** | **9.5 days** | **~2 weeks** |

---

## 🎯 Success Criteria

### **Functional**
- ✅ All admin commands working
- ✅ User management complete (CRUD operations)
- ✅ VPN key management complete
- ✅ Ticket management complete (view, assign, respond)
- ✅ Server monitoring functional
- ✅ All tests passing (95%+ coverage)

### **Security**
- ✅ Zero unauthorized access
- ✅ All admin actions logged
- ✅ Rate limiting enforced
- ✅ Token management secure
- ✅ Audit trail complete

### **Non-Functional**
- ✅ Bot responds in < 500ms
- ✅ Zero downtime deployment
- ✅ Error rate < 0.1%
- ✅ Redis connection pooled

### **Business**
- ✅ Staff can manage platform efficiently
- ✅ Clear audit trail for compliance
- ✅ Faster response to issues
- ✅ Improved operational efficiency

---

## 🔗 Related Documentation

- **Backend Admin API:** `/home/mowgli/usipipo/usipipo-backend/src/infrastructure/api/v1/routes/admin.py`
- **Support Bot Design:** `/home/mowgli/usipipo/usipipo-docs/plans/support-bot/`
- **Main Bot:** `/home/mowgli/usipipo/usipipo-telegram-bot/`
- **Ecosystem Context:** `/home/mowgli/usipipo/usipipo-docs/plans/shared/ECOSYSTEM-CONTEXT.md`
- **Backend API Docs:** http://localhost:8001/docs

---

## 📋 Approval

**Design Author:** uSipipo Development Team  
**Review Status:** ⏳ Pending Review  
**Approved By:** _TBD_  
**Approval Date:** _TBD_

---

**Last Updated:** 2026-03-28  
**Version:** 1.0  
**Status:** Proposed
