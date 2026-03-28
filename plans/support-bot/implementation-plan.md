# uSipipo Support Bot - Implementation Plan

**Date:** 2026-03-28  
**Based on:** `/home/mowgli/usipipo/usipipo-docs/plans/support-bot/2026-03-28-support-bot-design.md`  
**Target Repository:** `usipipo-support-bot` (new)  
**Timeline:** ~1 week (5.5 days)  

---

## 📋 Overview

This plan outlines the implementation tasks for creating the uSipipo Support Bot from scratch, based on the approved design document.

---

## 🎯 Tasks

### **Task 1: Repository Setup & Infrastructure**

**Description:**
Create the new `usipipo-support-bot` repository with professional structure and all standard files.

**Requirements:**
- [ ] Create GitHub repository `usipipo-telegram-bot/usipipo-support-bot`
- [ ] Initialize with Python 3.13, uv package manager
- [ ] Create `pyproject.toml` with all dependencies
- [ ] Create `.gitignore`, `.pre-commit-config.yaml`
- [ ] Create `README.md` with badges and documentation
- [ ] Create `LICENSE` (MIT)
- [ ] Create `CHANGELOG.md` (v0.1.0)
- [ ] Create `CONTRIBUTING.md`
- [ ] Create `CODE_OF_CONDUCT.md`
- [ ] Create `SECURITY.md`
- [ ] Create `example.env` with all required variables
- [ ] Create `Dockerfile` (multi-stage, production-ready)
- [ ] Create `usipipo-support-bot.service` (systemd)
- [ ] Create `.github/workflows/ci.yml` (Ruff, Mypy, Pytest, Bandit)
- [ ] Create `docs/ARCHITECTURE.md`
- [ ] Create `docs/DEPLOYMENT.md`
- [ ] Create `docs/USER-GUIDE.md`

**Files to Create:**
```
usipipo-support-bot/
├── pyproject.toml
├── .gitignore
├── .pre-commit-config.yaml
├── README.md
├── LICENSE
├── CHANGELOG.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── SECURITY.md
├── example.env
├── Dockerfile
├── usipipo-support-bot.service
├── .github/workflows/ci.yml
├── docs/ARCHITECTURE.md
├── docs/DEPLOYMENT.md
└── docs/USER-GUIDE.md
```

**Acceptance Criteria:**
- ✅ All files created with professional content
- ✅ CI/CD workflow configured
- ✅ Pre-commit hooks configured
- ✅ Docker build successful
- ✅ README comprehensive with badges

---

### **Task 2: Core Infrastructure Components**

**Description:**
Implement the core infrastructure components for the Support Bot.

**Requirements:**
- [ ] Create `src/infrastructure/config.py` (pydantic-settings)
- [ ] Create `src/infrastructure/redis.py` (RedisPool singleton)
- [ ] Create `src/infrastructure/api_client.py` (HTTP client)
- [ ] Create `src/infrastructure/token_storage.py` (JWT management)
- [ ] Create `src/infrastructure/logger.py` (structured logging)
- [ ] Create `src/infrastructure/error_handler.py` (global error handling)
- [ ] Create `src/bot/middlewares/auth.py` (JWT authentication middleware)

**Files to Create:**
```
src/infrastructure/
├── config.py
├── redis.py
├── api_client.py
├── token_storage.py
├── logger.py
└── error_handler.py

src/bot/middlewares/
└── auth.py
```

**Acceptance Criteria:**
- ✅ All components implemented following hexagonal architecture
- ✅ Config uses pydantic-settings with validation
- ✅ Redis connection pooled and tested
- ✅ API client supports JWT auth and auto-refresh
- ✅ Token storage with Redis integration
- ✅ Structured logging (JSON format)
- ✅ Auth middleware verifies JWT before processing
- ✅ All components have unit tests

---

### **Task 3: Ticket Handlers Implementation**

**Description:**
Migrate and adapt the ticket handlers from the main bot to the support bot.

**Requirements:**
- [ ] Create `src/bot/handlers/tickets.py` (TicketsHandler)
- [ ] Create `src/bot/handlers/messages.py` (MessagesHandler)
- [ ] Create `src/bot/keyboards/tickets.py` (TicketsKeyboard)
- [ ] Create `src/bot/keyboards/messages_tickets.py` (TicketsMessages)
- [ ] Implement `list_tickets()` - List user's tickets
- [ ] Implement `create_ticket()` - Create new ticket with category selection
- [ ] Implement `view_ticket_callback()` - View ticket details
- [ ] Implement `close_ticket_callback()` - Close ticket
- [ ] Implement `send_message()` - Send message to ticket
- [ ] Implement `view_messages()` - View ticket message history
- [ ] Register handlers in `src/main.py`

**Files to Create:**
```
src/bot/
├── handlers/
│   ├── tickets.py
│   └── messages.py
├── keyboards/
│   ├── tickets.py
│   └── messages_tickets.py
└── middlewares/
    └── auth.py (from Task 2)

src/main.py
```

**Acceptance Criteria:**
- ✅ All handlers migrated from main bot
- ✅ Category selection works (technical, billing, services, general)
- ✅ Ticket CRUD operations functional
- ✅ Message sending/receiving works
- ✅ Inline keyboards for all actions
- ✅ All handlers have unit tests (90%+ coverage)
- ✅ Integration tests with backend passing

---

### **Task 4: Main Application & Entry Point**

**Description:**
Create the main application entry point and wire all components together.

**Requirements:**
- [ ] Create `src/main.py` with application factory
- [ ] Implement `create_application()` function
- [ ] Initialize dependencies (Redis, API client, token storage)
- [ ] Register command handlers (`/start`, `/help`, `/tickets`, `/nuevoticket`)
- [ ] Register callback handlers (ticket actions)
- [ ] Register auth middleware
- [ ] Register error handler
- [ ] Create `src/__init__.py`
- [ ] Create `src/bot/__init__.py`
- [ ] Create `src/infrastructure/__init__.py`

**Files to Create:**
```
src/
├── __init__.py
├── main.py
├── bot/__init__.py
└── infrastructure/__init__.py
```

**Acceptance Criteria:**
- ✅ Application starts successfully
- ✅ All command handlers registered
- ✅ All callback handlers registered
- ✅ Middleware chain working
- ✅ Error handling global
- ✅ Bot responds to `/start` and `/help`
- ✅ Bot responds to `/tickets` and `/nuevoticket`

---

### **Task 5: Testing Suite**

**Description:**
Implement comprehensive test suite for the Support Bot.

**Requirements:**
- [ ] Create `tests/conftest.py` (pytest fixtures)
- [ ] Create `tests/bot/test_tickets_handlers.py` (~30 tests)
- [ ] Create `tests/bot/test_messages_handlers.py` (~20 tests)
- [ ] Create `tests/bot/test_auth_middleware.py` (~10 tests)
- [ ] Create `tests/integration/test_backend_integration.py` (~10 tests)
- [ ] Create `tests/infrastructure/test_config.py` (~5 tests)
- [ ] Create `tests/infrastructure/test_redis.py` (~5 tests)
- [ ] Create `tests/infrastructure/test_api_client.py` (~10 tests)
- [ ] Ensure 90%+ code coverage
- [ ] Configure pytest-asyncio
- [ ] Configure coverage reporting

**Files to Create:**
```
tests/
├── conftest.py
├── bot/
│   ├── test_tickets_handlers.py
│   ├── test_messages_handlers.py
│   └── test_auth_middleware.py
├── integration/
│   └── test_backend_integration.py
└── infrastructure/
    ├── test_config.py
    ├── test_redis.py
    └── test_api_client.py
```

**Acceptance Criteria:**
- ✅ All tests passing (100%)
- ✅ Code coverage 90%+
- ✅ CI/CD pipeline runs tests successfully
- ✅ Integration tests pass with production backend
- ✅ Mock fixtures for external dependencies
- ✅ Async tests configured correctly

---

### **Task 6: Deployment Configuration**

**Description:**
Configure deployment infrastructure for production deployment.

**Requirements:**
- [ ] Finalize `Dockerfile` (multi-stage, optimized)
- [ ] Create `.dockerignore`
- [ ] Create `docker-compose.yml` (for local development)
- [ ] Finalize `usipipo-support-bot.service` (systemd)
- [ ] Create deployment script (`scripts/deploy.sh`)
- [ ] Create environment setup script (`scripts/setup-env.sh`)
- [ ] Document deployment in `docs/DEPLOYMENT.md`
- [ ] Test Docker build locally
- [ ] Test systemd service locally (if possible)

**Files to Create:**
```
├── Dockerfile
├── .dockerignore
├── docker-compose.yml
├── usipipo-support-bot.service
└── scripts/
    ├── deploy.sh
    └── setup-env.sh
```

**Acceptance Criteria:**
- ✅ Docker build successful
- ✅ Docker image runs bot correctly
- ✅ systemd service starts bot
- ✅ Deployment scripts tested
- ✅ Documentation complete

---

### **Task 7: Migration from Main Bot**

**Description:**
Extract ticket module from main bot and clean up.

**Requirements:**
- [ ] Copy `handlers/tickets.py` from main bot
- [ ] Copy `keyboards/tickets.py` from main bot
- [ ] Copy `keyboards/messages_tickets.py` from main bot
- [ ] Copy test files from main bot
- [ ] Adapt imports for new structure
- [ ] Delete ticket files from main bot
- [ ] Remove ticket handlers from main bot `main.py`
- [ ] Update main bot tests
- [ ] Update main bot documentation
- [ ] Prepare main bot v0.9.0 release notes

**Files to Modify in Main Bot:**
```
usipipo-telegram-bot/
├── src/bot/handlers/tickets.py (DELETE)
├── src/bot/keyboards/tickets.py (DELETE)
├── src/bot/keyboards/messages_tickets.py (DELETE)
├── src/main.py (REMOVE ticket handlers)
├── tests/bot/test_tickets_handlers.py (DELETE)
└── CHANGELOG.md (UPDATE for v0.9.0)
```

**Acceptance Criteria:**
- ✅ Support bot has all ticket functionality
- ✅ Main bot no longer has ticket functionality
- ✅ Main bot tests still passing
- ✅ Main bot v0.9.0 ready for release
- ✅ No breaking changes for main bot users

---

### **Task 8: Documentation & Release**

**Description:**
Finalize documentation and prepare for release.

**Requirements:**
- [ ] Update `README.md` with final features
- [ ] Update `CHANGELOG.md` (v0.1.0 release)
- [ ] Complete `docs/ARCHITECTURE.md`
- [ ] Complete `docs/DEPLOYMENT.md`
- [ ] Complete `docs/USER-GUIDE.md`
- [ ] Create `INTEGRATION-TEST-SUMMARY.md`
- [ ] Create GitHub release v0.1.0
- [ ] Update ecosystem documentation
- [ ] Announce release in team channels

**Acceptance Criteria:**
- ✅ All documentation complete and accurate
- ✅ GitHub release published
- ✅ Release notes comprehensive
- ✅ Team notified of new bot
- ✅ Bot ready for production deployment

---

## 🧪 Testing Strategy

### **Unit Tests**
- Handlers: 90%+ coverage
- Keyboards: 100% coverage
- Middlewares: 100% coverage
- Infrastructure: 80%+ coverage

### **Integration Tests**
- Backend API integration
- Redis integration
- Authentication flow
- Ticket CRUD operations

### **End-to-End Tests**
- Complete ticket creation flow
- Ticket viewing flow
- Message sending flow
- Error scenarios

---

## 📊 Success Metrics

| Metric | Target |
|--------|--------|
| Test Coverage | 90%+ |
| Tests Passing | 100% |
| CI/CD Pass Rate | 100% |
| Docker Build | Success |
| Bot Response Time | < 1 second |
| Error Rate | < 0.1% |

---

## 🔗 Related Documentation

- **Design Document:** `/home/mowgli/usipipo/usipipo-docs/plans/support-bot/2026-03-28-support-bot-design.md`
- **Main Bot:** `/home/mowgli/usipipo/usipipo-telegram-bot/`
- **Backend:** `/home/mowgli/usipipo/usipipo-backend/`
- **Ecosystem Context:** `/home/mowgli/usipipo/usipipo-docs/plans/shared/ECOSYSTEM-CONTEXT.md`

---

**Last Updated:** 2026-03-28  
**Version:** 1.0  
**Status:** Ready for Implementation
