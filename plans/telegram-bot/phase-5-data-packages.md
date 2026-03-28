# 📦 Phase 5: Data Packages / Buy GB - Implementation Plan

**Date:** 2026-03-28  
**Priority:** P3 (Medium)  
**Estimated Effort:** 6-8 hours  
**Files to Migrate:** 10 legacy files → 4 new files  
**Migration Progress:** 45% → 55% (after completion)

---

## 🎯 Overview

Phase 5 implements the **Data Packages** purchase system, allowing users to buy GB data packages using:
1. **Telegram Stars** (in-app purchases via Telegram's payment system)
2. **Crypto** (USDT via TronDealer integration)

This is a core feature for the uSipipo VPN ecosystem, providing users with flexible data top-up options.

---

## 📊 Legacy Code Analysis

### Source Files (Legacy Monorepo)
```
/home/mowgli/usipipobot/telegram_bot/features/buy_gb/
├── handlers_buy_gb.py              (2910 bytes) - Main handler router
├── handlers_packages.py            (7069 bytes) - Package display
├── handlers_payment_crypto.py      (10054 bytes) - Crypto payment flow
├── handlers_payment_stars.py       (6367 bytes) - Stars payment flow
├── handlers_confirmation.py        (20875 bytes) - Payment confirmation
├── handlers_slots.py               (3631 bytes) - Data slots management
├── keyboards_buy_gb.py             (4266 bytes) - Inline keyboards
├── messages_buy_gb.py              (10742 bytes) - UI messages
├── __init__.py                     (327 bytes)
└── __pycache__/
```

**Total:** 10 files (~66 KB)

### Target Structure (New Multi-Repo)
```
/home/mowgli/usipipo/usipipo-telegram-bot/src/
├── bot/
│   ├── handlers/
│   │   └── packages.py             (NEW - All package handlers)
│   └── keyboards/
│       ├── packages.py             (NEW - Package keyboards)
│       └── messages_packages.py    (NEW - Package messages)
└── infrastructure/ (no changes)

tests/
└── bot/
    └── test_packages_handlers.py   (NEW - Unit tests)
```

**Total:** 4 new files

---

## 🎹 Commands to Implement

| Command | Callback | Description |
|---------|----------|-------------|
| `/comprar` | - | Buy data packages (alias: `/packages`) |
| `/paquetes` | - | View available packages |
| - | `buy_gb_menu` | Show buy GB menu |
| - | `select_payment_*` | Select payment method for package |
| - | `pay_stars_*` | Pay with Telegram Stars |
| - | `pay_crypto_*` | Pay with USDT/crypto |
| - | `view_data_summary` | View user's data summary |
| - | `buy_slots_menu` | View data slots menu |

---

## 🔌 Backend Integration

### API Endpoints Required

| Endpoint | Method | Purpose | Status |
|----------|--------|---------|--------|
| `GET /api/v1/data-packages` | GET | List available packages | ✅ Available |
| `POST /api/v1/payments/crypto` | POST | Create crypto payment | ✅ Available |
| `POST /api/v1/payments/stars` | POST | Create Stars payment | ✅ Available |
| `GET /api/v1/users/me/data-summary` | GET | Get user data summary | ✅ Available |
| `GET /api/v1/users/me/slots` | GET | Get user's data slots | ✅ Available |

### Backend Routes Location
- **Data Packages:** `/home/mowgli/usipipo/usipipo-backend/src/infrastructure/api/v1/routes/data_packages.py`
- **Payments:** `/home/mowgli/usipipo/usipipo-backend/src/infrastructure/api/v1/routes/payments.py`

---

## 📋 Implementation Tasks

### Task 1: Create Package Handlers (`packages.py`)
**Estimated:** 2-3 hours  
**Lines:** ~400-500

**Components to migrate:**
- `PackagesMixin` → `PackagesHandler` class
- `PaymentCryptoMixin` → Integrated in handler
- `PaymentStarsMixin` → Integrated in handler
- `ConfirmationMixin` → Integrated in handler
- `SlotsMixin` → Integrated in handler

**Key methods:**
```python
class PackagesHandler:
    def __init__(self, api_client: APIClient, token_storage: TokenStorage)
    
    async def show_packages(update, context)
    async def select_payment_method(update, context)
    async def pay_with_stars(update, context)
    async def pay_with_crypto(update, context)
    async def pre_checkout_callback(update, context)
    async def successful_payment(update, context)
    async def view_data_summary(update, context)
    async def show_slots_menu(update, context)
```

**Backend calls:**
- `GET /data-packages` - List packages
- `POST /payments/crypto` - Create crypto payment
- `POST /payments/stars` - Create Stars payment
- `GET /users/me/data-summary` - Get data summary

---

### Task 2: Create Package Keyboards (`packages.py`)
**Estimated:** 1 hour  
**Lines:** ~200-250

**Keyboards to create:**
```python
class PackagesKeyboard:
    @staticmethod
    def packages_menu() -> InlineKeyboardMarkup
    @staticmethod
    def payment_method_selection(package_type: str) -> InlineKeyboardMarkup
    @staticmethod
    def stars_payment(amount: int) -> InlineKeyboardMarkup
    @staticmethod
    def crypto_payment(amount_usd: float) -> InlineKeyboardMarkup
    @staticmethod
    def confirmation(payment_id: str) -> InlineKeyboardMarkup
    @staticmethod
    def back_to_packages() -> InlineKeyboardMarkup
    @staticmethod
    def slots_menu() -> InlineKeyboardMarkup
```

---

### Task 3: Create Package Messages (`messages_packages.py`)
**Estimated:** 1 hour  
**Lines:** ~300-350

**Message categories:**
```python
class PackagesMessages:
    class Menu:
        PACKAGES_LIST = "..."  # List of available packages
        PACKAGE_DETAILS = "..."  # Package details
    
    class Payment:
        SELECT_METHOD = "..."  # Choose Stars or Crypto
        STARS_PAYMENT = "..."  # Stars payment instructions
        CRYPTO_PAYMENT = "..."  # Crypto payment instructions
        CONFIRMATION = "..."  # Payment confirmation
    
    class Slots:
        SLOTS_MENU = "..."  # Data slots management
        SLOT_DETAILS = "..."  # Slot information
    
    class Summary:
        DATA_SUMMARY = "..."  # User's data usage summary
    
    class Error:
        SYSTEM_ERROR = "..."
        INVALID_PACKAGE = "..."
        PAYMENT_FAILED = "..."
```

---

### Task 4: Register Handlers in `main.py`
**Estimated:** 30 minutes  
**Lines:** ~20-30

**Changes:**
```python
# Import
from src.bot.handlers.packages import (
    PackagesHandler,
    get_packages_handlers,
    get_packages_callback_handlers,
    get_packages_payment_handlers,
)

# Global variable
_packages_handler: PackagesHandler | None = None

# Initialization
_packages_handler = PackagesHandler(_api_client, _token_storage)

# Command handlers
app.add_handler(CommandHandler("comprar", packages_handler.show_packages))
app.add_handler(CommandHandler("packages", packages_handler.show_packages))
app.add_handler(CommandHandler("paquetes", packages_handler.show_packages))

# Callback handlers
for handler in get_packages_callback_handlers(_api_client, _token_storage):
    app.add_handler(handler)

# Payment handlers (special)
for handler in get_packages_payment_handlers(_api_client, _token_storage):
    app.add_handler(handler)
```

---

### Task 5: Write Unit Tests
**Estimated:** 1.5-2 hours  
**Lines:** ~400-500  
**Target:** 35-40 tests

**Test categories:**
```python
class TestPackagesHandler:
    # Initialization
    test_packages_handler_initialization
    
    # Messages
    test_packages_messages_constants_exist
    test_packages_messages_has_menu_messages
    test_packages_messages_has_payment_messages
    test_packages_messages_has_error_messages
    
    # Keyboards
    test_packages_keyboard_main_menu_exists
    test_packages_keyboard_payment_selection_exists
    test_packages_keyboard_stars_payment_exists
    test_packages_keyboard_crypto_payment_exists
    
    # Handlers
    test_show_packages_command
    test_select_payment_method_callback
    test_pay_with_stars_flow
    test_pay_with_crypto_flow
    test_pre_checkout_callback
    test_successful_payment_callback
    test_view_data_summary
    
    # Integration
    test_packages_handler_get_auth_headers
    test_get_packages_handlers_returns_list
```

---

### Task 6: Verification & Code Review
**Estimated:** 30 minutes

**Quality gates:**
- ✅ Ruff linting (passed)
- ✅ Mypy type checking (clean)
- ✅ Pytest (35-40 tests passing)
- ✅ Code review (no critical issues)

---

## 📦 Package Configuration

### Expected Package Structure (from backend)
```python
PACKAGE_OPTIONS = [
    {"type": "small", "gb": 5, "stars": 600, "usd": 5.00},
    {"type": "medium", "gb": 10, "stars": 1200, "usd": 10.00},
    {"type": "large", "gb": 25, "stars": 3000, "usd": 25.00},
    {"type": "xl", "gb": 50, "stars": 6000, "usd": 50.00},
]
```

### Pricing Logic
- **1 USDT ≈ 120 Stars** (Telegram's conversion rate)
- **Price per GB:** ~$1.00 USD (varies by package size)
- **Bulk discount:** Larger packages have better per-GB pricing

---

## 🔐 Payment Flow

### Stars Payment Flow
```
1. User selects package
2. Bot shows payment method selection
3. User chooses "Pay with Stars"
4. Bot sends invoice via Telegram Bot API
5. User completes payment in Telegram
6. Bot receives SUCCESSFUL_PAYMENT update
7. Bot calls backend to activate package
8. Bot confirms activation to user
```

### Crypto Payment Flow
```
1. User selects package
2. Bot shows payment method selection
3. User chooses "Pay with Crypto"
4. Bot calls backend POST /payments/crypto
5. Backend returns payment address & amount
6. Bot shows payment instructions
7. User sends USDT to address
8. TronDealer webhook notifies backend
9. Backend activates package
10. Bot confirms activation to user
```

---

## 📊 Migration Progress

| Phase | Feature | Status | Files | Tests | Lines |
|-------|---------|--------|-------|-------|-------|
| Phase 1 | Auth + Infrastructure | ✅ 100% | 12 | 25 | 1,200 |
| Phase 2 | VPN Key Management | ✅ 100% | 8 | 39 | 2,100 |
| Phase 3 | Operations + Profile | ✅ 100% | 8 | 21 | 1,400 |
| Phase 4 | Consumption Billing | ✅ 100% | 10 | 45 | 1,874 |
| **Phase 5** | **Data Packages** | ⏳ 0% | **0/10** | **0/40** | **0/1,500** |
| Phase 6 | Payments + Subscriptions | ⏳ 0% | 0/14 | 0/30 | 0/1,800 |
| Phase 7 | Referrals + Tickets | ⏳ 0% | 0/10 | 0/25 | 0/1,200 |
| Phase 8 | Admin Panel | ⏳ 0% | 0/24 | 0/50 | 0/3,000 |

**Current:** 45% complete (38/92 files)  
**After Phase 5:** 55% complete (50/92 files)

---

## 🚀 Release Plan

### Version: v0.6.0 - Data Packages Complete

**Changelog entries:**
```markdown
## [0.6.0] - 2026-03-28

### Added
- **Data Packages System** - Buy GB data packages
- **Payment Methods** - Telegram Stars and Crypto (USDT)
- **New Commands:** `/comprar`, `/paquetes`, `/packages`
- **Data Slots** - Manage multiple data packages
- **Data Summary** - View usage statistics

### Files Created
- `src/bot/handlers/packages.py` (~500 lines)
- `src/bot/keyboards/packages.py` (~250 lines)
- `src/bot/keyboards/messages_packages.py` (~350 lines)
- `tests/bot/test_packages_handlers.py` (~450 lines, 40 tests)

### Backend Integration
- GET /api/v1/data-packages - List packages
- POST /api/v1/payments/stars - Create Stars payment
- POST /api/v1/payments/crypto - Create crypto payment
- GET /api/v1/users/me/data-summary - Get usage stats

### Quality
- 40 new unit tests (190 total)
- Ruff clean
- Mypy clean
- Code review approved
```

**Tag:** `v0.6.0`  
**Release:** https://github.com/uSipipo-Team/usipipo-telegram-bot/releases/tag/v0.6.0

---

## ⚠️ Important Considerations

### 1. Telegram Stars Configuration
- Requires Telegram Bot API 7.0+
- Stars are purchased by users from Telegram
- Bot receives Stars and can cash out after 14 days
- Test in production only (no test environment for Stars)

### 2. Crypto Payment Security
- TronDealer webhook already migrated (Phase 3)
- Verify payment signatures (HMAC-SHA256)
- Handle payment timeouts gracefully
- Implement retry logic for failed webhooks

### 3. Data Slots Management
- Users can have multiple active data packages
- Slots are consumed in order (FIFO)
- Expired slots are automatically deactivated
- Users can manually activate/deactivate slots

### 4. Error Handling
- Payment failures (insufficient funds, timeout)
- Backend API errors (5xx responses)
- Network issues during payment
- User cancels payment mid-flow

---

## 📝 Next Steps After Phase 5

### Phase 6: Payments + Subscriptions
- **Payments:** General payment management (not package-specific)
- **Subscriptions:** Monthly VPN plans activation/renewal
- **Estimated:** 8-10 hours
- **Files:** 14 files

### Phase 7: Referrals + Tickets
- **Referrals:** Invite friends, earn credits
- **Tickets:** Support system
- **Estimated:** 6-8 hours
- **Files:** 10 files

### Phase 8: Admin Panel
- **Admin commands:** User management, monitoring
- **Access control:** Admin-only features
- **Estimated:** 12-16 hours
- **Files:** 24 files

---

## 🎯 Success Criteria

Phase 5 is complete when:
- ✅ All 4 commands working (`/comprar`, `/packages`, `/paquetes`, data summary)
- ✅ Stars payment flow tested and working
- ✅ Crypto payment flow tested and working
- ✅ Data slots management functional
- ✅ 35-40 unit tests passing
- ✅ Ruff and mypy clean
- ✅ Code review approved
- ✅ Release v0.6.0 published

---

**Ready to start Phase 5?** 🚀

**Generated:** 2026-03-28  
**Author:** uSipipo Migration Team  
**Status:** Ready for implementation
