# uSipipo Support Bot - Design Document

**Date:** 2026-03-28  
**Version:** 1.0  
**Status:** Proposed  
**Bot Handle:** `@uSipipoSupport_Bot`  
**Related Backend:** usipipo-backend v0.10.0+  

---

## 📋 Overview

### **Purpose**
The uSipipo Support Bot is a dedicated Telegram bot for customer support ticket management. It provides users with a seamless interface to create, track, and communicate about their support tickets, separate from the main feature bot (`@usipipobot`).

### **Business Goals**
- ✅ **Separation of Concerns** - Support interactions isolated from main bot features
- ✅ **Improved Customer Experience** - Dedicated channel for support communications
- ✅ **Scalability** - Independent deployment and scaling of support infrastructure
- ✅ **Better Support Metrics** - Focused analytics on support performance
- ✅ **Staff Efficiency** - Clear separation between user features and support workflows

### **Scope**
This bot handles **user-facing support operations only**. Administrative staff operations are handled by a separate bot (`@uSipipoStaff_Bot`).

---

## 🎯 Features

### **User Commands**

| Command | Description | Example |
|---------|-------------|---------|
| `/start` | Initialize bot and show welcome message | `/start` |
| `/help` | Display help information | `/help` |
| `/tickets` | List all user's tickets | `/tickets` |
| `/nuevoticket` | Create a new support ticket | `/nuevoticket` |

### **Ticket Lifecycle**

```
1. User creates ticket → /nuevoticket
2. Selects category → Technical/Billing/Services/General
3. Ticket created with unique ID
4. User can view ticket status → /tickets
5. User can send messages to ticket
6. Staff responds (via @uSipipoStaff_Bot)
7. User receives notification
8. Ticket resolved → User closes ticket
```

### **Supported Ticket Categories**

| Category | Code | Use Case |
|----------|------|----------|
| 🖥️ Técnico | `technical` | VPN connection issues, technical problems |
| 💳 Pagos | `billing` | Payment issues, billing questions, refunds |
| 📦 Servicios | `services` | Plans, data packages, subscriptions |
| ❓ General | `general` | Other inquiries, general questions |

---

## 🏗️ Architecture

### **High-Level Architecture**

```
┌─────────────────┐
│   Telegram      │
│   @uSipipoSupport_Bot │
└────────┬────────┘
         │
         │ Telegram Bot API
         │
┌────────▼────────┐
│  Support Bot    │
│  Application    │
│  (Python 3.13)  │
└────────┬────────┘
         │
         │ HTTP/REST
         │
┌────────▼────────┐
│  usipipo-backend│
│  API v0.10.0+   │
└────────┬────────┘
         │
         │ SQLAlchemy Async
         │
┌────────▼────────┐
│  PostgreSQL     │
│  + Redis        │
└─────────────────┘
```

### **Project Structure**

```
usipipo-support-bot/
├── src/
│   ├── bot/
│   │   ├── handlers/
│   │   │   ├── tickets.py           # Ticket CRUD operations
│   │   │   └── messages.py          # Ticket message handling
│   │   ├── keyboards/
│   │   │   ├── tickets.py           # Ticket inline keyboards
│   │   │   └── messages_tickets.py  # Ticket messages
│   │   └── middlewares/
│   │       └── auth.py              # JWT authentication middleware
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
│   │   ├── test_tickets_handlers.py
│   │   └── test_messages_handlers.py
│   ├── integration/
│   │   └── test_backend_integration.py
│   └── conftest.py
├── docs/
│   ├── ARCHITECTURE.md
│   ├── DEPLOYMENT.md
│   └── USER-GUIDE.md
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
├── usipipo-support-bot.service
└── uv.lock
```

---

## 🔌 Backend Integration

### **API Endpoints**

#### **Ticket Management**

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| `GET` | `/api/v1/tickets` | List user's tickets | ✅ User JWT |
| `POST` | `/api/v1/tickets` | Create new ticket | ✅ User JWT |
| `GET` | `/api/v1/tickets/{id}` | Get ticket details | ✅ User JWT |
| `PATCH` | `/api/v1/tickets/{id}/close` | Close ticket | ✅ User JWT |
| `POST` | `/api/v1/tickets/{id}/messages` | Send message to ticket | ✅ User JWT |

### **Request/Response Examples**

#### **Create Ticket**
```http
POST /api/v1/tickets
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "category": "technical",
  "subject": "VPN connection issue",
  "message": "I cannot connect to the VPN server"
}
```

**Response:**
```json
{
  "id": "uuid-123",
  "ticket_number": "#TKT-12345",
  "category": "technical",
  "subject": "VPN connection issue",
  "status": "OPEN",
  "created_at": "2026-03-28T10:00:00Z"
}
```

#### **List Tickets**
```http
GET /api/v1/tickets
Authorization: Bearer <access_token>
```

**Response:**
```json
{
  "tickets": [
    {
      "id": "uuid-123",
      "ticket_number": "#TKT-12345",
      "subject": "VPN connection issue",
      "status": "OPEN",
      "created_at": "2026-03-28T10:00:00Z"
    }
  ],
  "total": 1
}
```

---

## 🔐 Authentication Flow

### **Invisible Authentication**

The Support Bot uses the same invisible authentication pattern as the main bot:

```
1. User sends /start
2. Bot calls POST /auth/telegram/auto-register
3. Backend returns JWT tokens (access + refresh)
4. Bot stores tokens in Redis (30-day expiry)
5. Bot auto-refreshes tokens 5 minutes before expiry
6. All API calls include access_token in Authorization header
```

### **Token Storage**

```python
# Redis Key Structure
Key: "support_bot:tokens:{telegram_id}"
Value: {
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "expires_at": 1711627200,
  "user_id": "uuid-123"
}
TTL: 30 days
```

---

## 💻 Implementation Details

### **Dependencies**

```toml
[project]
name = "usipipo-support-bot"
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
]
```

### **Key Components**

#### **1. TicketsHandler**
```python
class TicketsHandler:
    """Handler for ticket system."""
    
    async def list_tickets(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """List all user's tickets."""
        pass
    
    async def create_ticket(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Start ticket creation flow."""
        pass
    
    async def view_ticket_callback(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Handle view ticket callback."""
        pass
    
    async def close_ticket_callback(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Handle close ticket callback."""
        pass
```

#### **2. MessagesHandler**
```python
class MessagesHandler:
    """Handler for ticket messages."""
    
    async def send_message(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Send message to ticket."""
        pass
    
    async def view_messages(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """View ticket message history."""
        pass
```

#### **3. AuthMiddleware**
```python
class AuthMiddleware:
    """JWT authentication middleware."""
    
    async def __call__(self, update: Update, dispatcher: Dispatcher):
        """Check if user is authenticated."""
        telegram_id = update.effective_user.id
        tokens = await self.token_storage.get(telegram_id)
        
        if not tokens:
            await update.message.reply_text("❌ Por favor, iniciá sesión primero")
            return
        
        await dispatcher.process_update(update)
```

---

## 🧪 Testing Strategy

### **Test Categories**

| Type | Count | Description |
|------|-------|-------------|
| Unit Tests | ~50 | Handler logic, keyboards, messages |
| Integration Tests | ~10 | Backend API integration |
| End-to-End Tests | ~5 | Complete user workflows |
| **Total** | **~65** | **All automated** |

### **Test Coverage Goals**

- ✅ **Handlers:** 90%+ coverage
- ✅ **Keyboards:** 100% coverage
- ✅ **Integration:** Critical paths covered
- ✅ **E2E:** Happy paths + error scenarios

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
      - run: uv run ruff check src tests
      - run: uv run mypy src
      - run: uv run pytest --cov=src --cov-report=xml
      - run: uv run bandit -r src
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
| **Services** | Redis (shared with main bot) |

### **Docker Configuration**

```dockerfile
# Dockerfile
FROM python:3.13-slim

WORKDIR /app

RUN pip install uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY src/ ./src/

CMD ["python", "-m", "src"]
```

### **systemd Service**

```ini
# usipipo-support-bot.service
[Unit]
Description=uSipipo Support Telegram Bot
After=network.target redis.service

[Service]
Type=simple
User=usipipo
WorkingDirectory=/opt/usipipo-support-bot
Environment=PATH=/opt/usipipo-support-bot/.venv/bin
ExecStart=/opt/usipipo-support-bot/.venv/bin/python -m src
Restart=always

[Install]
WantedBy=multi-user.target
```

### **Environment Variables**

```bash
# .env
BOT_TOKEN=telegram_bot_token_support
BACKEND_URL=https://api.usipipo.com
API_PREFIX=/api/v1
REDIS_URL=redis://localhost:6379/1
LOG_LEVEL=INFO
```

---

## 📊 Monitoring & Observability

### **Logging**

```python
# Structured logging format
{
  "timestamp": "2026-03-28T10:00:00Z",
  "level": "INFO",
  "logger": "tickets_handler",
  "message": "User created ticket",
  "telegram_id": 123456789,
  "ticket_id": "uuid-123",
  "ticket_number": "#TKT-12345"
}
```

### **Metrics to Track**

| Metric | Description | Target |
|--------|-------------|--------|
| Tickets Created/Day | New tickets per day | Track trend |
| Avg Response Time | Time to first staff response | < 2 hours |
| Ticket Resolution Time | Time to close ticket | < 24 hours |
| User Satisfaction | Post-resolution rating | > 4.5/5 |

---

## 🔒 Security Considerations

### **Data Protection**

- ✅ **JWT Tokens** - Stored in Redis with TTL
- ✅ **HTTPS Only** - All API calls over TLS
- ✅ **No Sensitive Data in Logs** - PII redacted
- ✅ **Input Validation** - All user input sanitized
- ✅ **Rate Limiting** - Backend enforces limits

### **Access Control**

- ✅ **User Isolation** - Users can only see their own tickets
- ✅ **Token Auto-Refresh** - Seamless authentication
- ✅ **Session Management** - Tokens revoked on `/unlink`

---

## 📝 Migration from Main Bot

### **Files to Extract**

From `usipipo-telegram-bot`:

| File | Action | New Location |
|------|--------|--------------|
| `src/bot/handlers/tickets.py` | Move | `src/bot/handlers/tickets.py` |
| `src/bot/keyboards/tickets.py` | Move | `src/bot/keyboards/tickets.py` |
| `src/bot/keyboards/messages_tickets.py` | Move | `src/bot/keyboards/messages_tickets.py` |
| `tests/bot/test_tickets_handlers.py` | Move | `tests/bot/test_tickets_handlers.py` |
| `tests/integration/test_tickets_integration.py` | Move | `tests/integration/test_tickets_integration.py` |

### **Files to Create**

| File | Purpose |
|------|---------|
| `src/bot/handlers/messages.py` | Ticket message handling |
| `src/bot/middlewares/auth.py` | Authentication middleware |
| `src/infrastructure/*` | Infrastructure components |
| `docs/*` | Documentation |
| `.github/workflows/ci.yml` | CI/CD pipeline |

### **Removal from Main Bot**

After migration:
1. Delete ticket files from `usipipo-telegram-bot`
2. Remove handler registrations from `main.py`
3. Update tests
4. Update documentation
5. Release new version (v0.9.0)

---

## 📅 Timeline

| Phase | Duration | Tasks |
|-------|----------|-------|
| **Phase 1: Setup** | 1 day | Repo creation, structure, CI/CD |
| **Phase 2: Migration** | 2 days | Extract ticket module, adapt imports |
| **Phase 3: Testing** | 1 day | Unit + integration tests |
| **Phase 4: Deploy** | 1 day | Docker, systemd, monitoring |
| **Phase 5: Cleanup** | 0.5 day | Remove from main bot |
| **Total** | **5.5 days** | **~1 week** |

---

## 🎯 Success Criteria

### **Functional**
- ✅ All ticket commands working (`/tickets`, `/nuevoticket`)
- ✅ Users can create, view, and close tickets
- ✅ Authentication seamless and invisible
- ✅ All tests passing (90%+ coverage)

### **Non-Functional**
- ✅ Bot responds in < 1 second
- ✅ Zero downtime deployment
- ✅ Error rate < 0.1%
- ✅ Redis connection pooled and efficient

### **Business**
- ✅ Support interactions separated from main bot
- ✅ Clear audit trail for all tickets
- ✅ Improved customer experience
- ✅ Foundation for advanced support features

---

## 🔗 Related Documentation

- **Backend API:** `/home/mowgli/usipipo/usipipo-backend/docs/`
- **Main Bot:** `/home/mowgli/usipipo/usipipo-telegram-bot/`
- **Staff Bot Design:** `/home/mowgli/usipipo/usipipo-docs/plans/staff-bot/`
- **Ecosystem Context:** `/home/mowgli/usipipo/usipipo-docs/plans/shared/ECOSYSTEM-CONTEXT.md`

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
