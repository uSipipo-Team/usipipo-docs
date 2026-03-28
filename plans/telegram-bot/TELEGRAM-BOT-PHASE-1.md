# Telegram Bot Refactor - Phase 1 Summary

**Date:** 2026-03-21
**Status:** Ready to Start
**Repository:** `usipipo-telegram-bot`
**Backend Dependency:** `usipipo-backend` v0.2.1+ (100% complete)

---

## 🎯 Phase 1 Objective

Migrate basic Telegram bot commands from monorepo to the new `usipipo-telegram-bot` repository with backend API integration.

**Estimated Effort:** 3-4 days

---

## 📁 Current Monorepo Structure

```
/home/mowgli/usipipobot/
└── telegram_bot/
    ├── common/
    │   ├── __init__.py
    │   ├── base_handler.py        # Base handler class
    │   ├── decorators.py          # Custom decorators
    │   ├── keyboards.py           # Reusable keyboards
    │   └── messages.py            # Base message templates
    │
    ├── features/
    │   ├── basic_commands/
    │   │   ├── __init__.py
    │   │   ├── handlers_basic.py  # /start, /help handlers
    │   │   └── messages_basic.py  # Message templates
    │   │
    │   ├── operations/            # Phase 2: VPN operations
    │   ├── admin_vpn/             # Phase 2: Admin features
    │   ├── consumption/           # Phase 4: Consumption billing
    │   └── ...                    # Other features
    │
    ├── keyboards/
    │   ├── __init__.py
    │   └── main_menu.py           # Main menu keyboard
    │
    └── handlers/
        └── handler_initializer.py # Handler registration
```

---

## 🏗️ Target Repository Structure

```
/home/mowgli/usipipo/usipipo-telegram-bot/
├── src/
│   ├── __init__.py
│   ├── __main__.py               # Entry point
│   ├── main.py                   # Bot initialization
│   │
│   ├── core/
│   │   ├── config.py             # Settings (pydantic-settings)
│   │   ├── logger.py             # Logging configuration
│   │   └── errors.py             # Custom exceptions
│   │
│   ├── infrastructure/
│   │   ├── api/
│   │   │   ├── client.py         # Backend API client
│   │   │   └── endpoints.py      # API endpoint definitions
│   │   └── database/             # Future: local DB if needed
│   │
│   ├── features/
│   │   ├── basic_commands/
│   │   │   ├── __init__.py
│   │   │   ├── handlers.py       # /start, /help, /menu
│   │   │   ├── messages.py       # Message templates
│   │   │   └── keyboards.py      # Basic keyboards
│   │   │
│   │   └── ...                   # Future phases
│   │
│   └── shared/
│       ├── decorators.py         # Reusable decorators
│       └── utils.py              # Utility functions
│
├── tests/
│   ├── unit/
│   │   ├── test_basic_handlers.py
│   │   └── test_messages.py
│   └── integration/
│       └── test_api_client.py
│
├── docs/
│   ├── ARCHITECTURE.md
│   ├── COMMANDS.md
│   └── DEPLOYMENT.md
│
├── pyproject.toml
├── example.env
└── README.md
```

---

## 📋 Phase 1 Tasks

### **Task 1: Repository Setup** ✅
- [x] Repository already created: `usipipo-telegram-bot`
- [ ] Update `pyproject.toml` dependencies:
  ```python
  dependencies = [
      "usipipo-commons>=0.9.0",
      "python-telegram-bot>=21.0",
      "pydantic-settings>=2.0.0",
      "httpx>=0.27.0",  # For API client
  ]
  ```
- [ ] Update `example.env`:
  ```bash
  TELEGRAM_BOT_TOKEN=your-bot-token
  BACKEND_API_URL=https://usipipo.duckdns.org/api/v1
  LOG_LEVEL=INFO
  ```

### **Task 2: Core Infrastructure**
- [ ] Create `src/core/config.py` with pydantic-settings
- [ ] Create `src/core/logger.py` with structured logging
- [ ] Create `src/core/errors.py` with custom exceptions

### **Task 3: Backend API Client**
- [ ] Create `src/infrastructure/api/client.py`:
  ```python
  class BackendAPIClient:
      async def auth_telegram(self, telegram_id: int) -> str: ...
      async def get_user(self, user_id: UUID) -> User: ...
      async def get_user_wallet(self, user_id: UUID) -> Wallet: ...
  ```
- [ ] Create `src/infrastructure/api/endpoints.py` with endpoint constants
- [ ] Add error handling and retry logic

### **Task 4: Basic Commands Migration**
- [ ] Migrate `/start` command:
  - Welcome message
  - User registration via backend API
  - Main menu display
- [ ] Migrate `/help` command:
  - Command list
  - FAQ section
- [ ] Migrate `/menu` command:
  - Main menu keyboard
  - Navigation options

### **Task 5: Keyboards & Messages**
- [ ] Create `src/features/basic_commands/keyboards.py`:
  - Main menu keyboard (inline buttons)
  - Help keyboard
- [ ] Create `src/features/basic_commands/messages.py`:
  - Welcome message template
  - Help message template
  - Error messages

### **Task 6: Bot Initialization**
- [ ] Create `src/main.py`:
  - Application builder setup
  - Handler registration
  - Error handlers
  - Graceful shutdown
- [ ] Create `src/__main__.py` entry point

### **Task 7: Testing**
- [ ] Write unit tests for handlers (10+ tests)
- [ ] Write unit tests for messages (5+ tests)
- [ ] Write integration tests for API client (5+ tests)
- [ ] Achieve >80% code coverage

### **Task 8: Documentation**
- [ ] Update `README.md` with setup instructions
- [ ] Create `docs/ARCHITECTURE.md`
- [ ] Create `docs/COMMANDS.md` with command reference
- [ ] Create `docs/DEPLOYMENT.md`

### **Task 9: PR & Merge**
- [ ] Create feature branch: `feature/phase-1-basic-commands`
- [ ] Run tests: `uv run pytest`
- [ ] Run linter: `uv run ruff check .`
- [ ] Run type checker: `uv run mypy .`
- [ ] Create PR with comprehensive description
- [ ] Code review and merge

---

## 🔧 Code to Migrate (from Monorepo)

### **Basic Commands (`handlers_basic.py`)**
```python
# Source: /home/mowgli/usipipobot/telegram_bot/features/basic_commands/handlers_basic.py
class BasicHandler:
    async def help_handler(self, update: Update, _context: ContextTypes.DEFAULT_TYPE):
        """Muestra la lista de comandos disponibles."""
        user = update.effective_user
        if user is None:
            logger.warning("No user in update for /help command")
            return

        user_id = user.id
        logger.info(f"User {user_id} executed /help command")

        try:
            if update.message:
                await update.message.reply_text(text=BasicMessages.HELP_TEXT, parse_mode="Markdown")
        except Exception as e:
            logger.error(f"❌ Error en /help para usuario {user_id}: {e}")
```

### **Messages (`messages_basic.py`)**
```python
# Source: /home/mowgli/usipipobot/telegram_bot/features/basic_commands/messages_basic.py
class BasicMessages:
    HELP_TEXT = """
👋 *Comandos Disponibles*

/start - Iniciar el bot
/help - Mostrar esta ayuda
/menu - Volver al menú principal
"""
```

### **Keyboards (`main_menu.py`)**
```python
# Source: /home/mowgli/usipipobot/telegram_bot/keyboards/main_menu.py
from telegram import InlineKeyboardButton, InlineKeyboardMarkup

def get_main_menu_keyboard() -> InlineKeyboardMarkup:
    keyboard = [
        [InlineKeyboardButton("🔑 Mis Claves", callback_data="my_keys")],
        [InlineKeyboardButton("💳 Comprar", callback_data="buy")],
        [InlineKeyboardButton("👤 Perfil", callback_data="profile")],
        [InlineKeyboardButton("🎧 Soporte", callback_data="support")],
    ]
    return InlineKeyboardMarkup(keyboard)
```

---

## 🔗 Backend API Integration

### **Authentication Flow**
```python
# 1. User sends /start
# 2. Bot calls backend API to authenticate
POST /api/v1/auth/telegram
{
    "telegram_id": 123456,
    "username": "john_doe",
    "first_name": "John"
}

# 3. Backend returns JWT token
{
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "token_type": "bearer"
}

# 4. Bot stores token for subsequent requests
```

### **User Retrieval**
```python
# Get user info
GET /api/v1/users/me
Headers: Authorization: Bearer {token}

Response:
{
    "id": "uuid",
    "telegram_id": 123456,
    "username": "john_doe",
    "balance_gb": 5.0,
    "referral_code": "ref_abc123"
}
```

---

## ✅ Acceptance Criteria

- [ ] `/start` command works and registers user via backend API
- [ ] `/help` command displays help message
- [ ] `/menu` command shows main menu keyboard
- [ ] All commands log user actions
- [ ] Error handling for API failures
- [ ] 20+ unit tests passing
- [ ] 10+ integration tests passing
- [ ] `uv run ruff check .` passes
- [ ] `uv run mypy .` passes
- [ ] `uv run pytest --cov=src` shows >80% coverage
- [ ] PR merged to main

---

## 📅 Timeline

| Day | Tasks |
|-----|-------|
| Day 1 | Repository setup, core infrastructure, API client |
| Day 2 | Basic commands migration (/start, /help, /menu) |
| Day 3 | Keyboards, messages, bot initialization |
| Day 4 | Testing, documentation, PR creation |

---

## 🚀 Next Phases Overview

### **Phase 2: VPN Key Management** (Week 2)
- Create/list/delete VPN keys
- View key usage statistics
- Download key configuration files

### **Phase 3: Payments + Subscriptions** (Week 3)
- Crypto payment flow
- Telegram Stars integration
- Subscription activation

### **Phase 4: Advanced Features** (Week 4)
- Ticket system
- Referral program
- Data packages
- Consumption billing

### **Phase 5: Wallet + Final Testing** (Week 5)
- BSC wallet management
- Complete integration testing
- Production deployment

---

**Last Updated:** 2026-03-21
**Ready to Start:** Phase 1 - Basic Commands
