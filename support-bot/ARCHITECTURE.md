# uSipipo Support Bot - Architecture

**Date:** 2026-03-28  
**Version:** 1.0  
**Bot Handle:** `@uSipipoSupport_Bot`  

---

## 🏗️ High-Level Architecture

### **System Context**

```
┌─────────────────┐
│   Telegram      │
│ @uSipipoSupport_Bot │
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

---

## 📦 Component Architecture

### **Directory Structure**

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
└── ...
```

---

## 🔌 Component Details

### **1. Handlers Layer**

**Responsibility:** Process user inputs and generate responses.

#### **TicketsHandler**
- `list_tickets()` - List all user's tickets
- `create_ticket()` - Create new ticket with category selection
- `view_ticket_callback()` - View ticket details
- `close_ticket_callback()` - Close ticket

#### **MessagesHandler**
- `send_message()` - Send message to ticket
- `view_messages()` - View ticket message history

---

### **2. Keyboards Layer**

**Responsibility:** Generate inline keyboards for user interactions.

#### **TicketsKeyboard**
- `tickets_list()` - Keyboard for ticket list
- `ticket_detail()` - Keyboard for ticket actions
- `categories()` - Category selection keyboard
- `back_to_tickets()` - Back navigation

#### **TicketsMessages**
- Message templates for all ticket operations
- Localized strings (Spanish)

---

### **3. Middlewares Layer**

**Responsibility:** Cross-cutting concerns (authentication, logging).

#### **AuthMiddleware**
- Verify JWT token exists
- Auto-refresh tokens before expiry
- Block unauthorized access

---

### **4. Infrastructure Layer**

**Responsibility:** External system integration.

#### **APIClient**
- HTTP client for backend API
- JWT authentication headers
- Error handling and retries

#### **Config**
- Pydantic settings
- Environment variable validation
- Type-safe configuration

#### **RedisPool**
- Redis connection pooling
- Singleton pattern
- Connection health checks

#### **TokenStorage**
- JWT token management
- Redis storage with TTL
- Auto-refresh logic

#### **Logger**
- Structured logging (JSON)
- Context-aware logging
- Log level configuration

#### **ErrorHandler**
- Global exception handling
- User-friendly error messages
- Error logging and alerting

---

## 🔄 Data Flow

### **Ticket Creation Flow**

```
1. User: /nuevoticket
2. Telegram → Bot API
3. AuthMiddleware: Verify JWT
4. TicketsHandler.create_ticket()
5. TicketsKeyboard.categories() → Show category selection
6. User: Select category
7. TicketsHandler.select_category_callback()
8. APIClient.post("/api/v1/tickets")
9. Backend: Create ticket in DB
10. Backend: Return ticket data
11. TicketsMessages.TICKET_CREATED → Show success
12. Bot → Telegram → User
```

### **Authentication Flow**

```
1. User: /start
2. Bot calls POST /auth/telegram/auto-register
3. Backend returns JWT (access + refresh)
4. TokenStorage.save(telegram_id, tokens)
5. Redis: SET support_bot:tokens:{telegram_id} EX 2592000
6. All subsequent calls: Authorization: Bearer {access_token}
7. Before expiry (5 min): Auto-refresh via POST /auth/refresh
```

---

## 🔐 Security Architecture

### **Authentication Layers**

```
┌─────────────────────────────────┐
│  1. Telegram Bot API Token      │ ← Bot identity
├─────────────────────────────────┤
│  2. JWT User Authentication     │ ← User identity
├─────────────────────────────────┤
│  3. Backend Role Verification   │ ← Permission check
└─────────────────────────────────┘
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
TTL: 30 days (2592000 seconds)
```

---

## 🧪 Testing Architecture

### **Test Pyramid**

```
        /\
       /  \      E2E Tests (~5)
      /____\     Integration Tests (~10)
     /      \    Unit Tests (~50)
    /________\
```

### **Test Categories**

| Layer | Tests | Coverage Goal |
|-------|-------|---------------|
| Handlers | ~30 | 90%+ |
| Keyboards | ~20 | 100% |
| Middleware | ~10 | 100% |
| Infrastructure | ~15 | 80%+ |
| Integration | ~10 | Critical paths |

---

## 📊 Monitoring Architecture

### **Logging Pipeline**

```
Bot Application → Structured Logs (JSON) → Log File → Log Aggregator
```

### **Log Format**

```json
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
| Avg Response Time | Bot response time | < 1 second |
| Error Rate | Failed operations | < 0.1% |
| Auth Failures | Failed auth attempts | Alert if > 10/day |

---

## 🔗 Related Documentation

- **Design Document:** `/plans/support-bot/2026-03-28-support-bot-design.md`
- **Deployment Guide:** `/support-bot/DEPLOYMENT.md`
- **User Guide:** `/support-bot/USER-GUIDE.md`
- **Backend API:** `/apis/backend-api-reference.md`

---

**Last Updated:** 2026-03-28  
**Version:** 1.0
