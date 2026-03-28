# Legacy Bot Migration Summary

**Date:** 2026-03-28
**Source:** `/home/mowgli/usipipobot/telegram_bot/` (Legacy Monorepo)
**Target:** `/home/mowgli/usipipo/usipipo-telegram-bot/` (New Dedicated Repo)
**Status:** Phase 1 Complete → Phase 2 Complete → Phase 3 Complete → Phase 4 Complete → Phase 5 Complete → Phase 6 Complete

---

## 📊 Migration Overview

### **Legacy Bot Structure**
```
/home/mowgli/usipipobot/telegram_bot/
├── common/                      ← Shared utilities
│   ├── base_handler.py
│   ├── decorators.py
│   ├── keyboards.py
│   └── messages.py
├── features/                    ← Feature modules
│   ├── admin/                   ← Admin panel (14 files)
│   ├── admin_vpn/               ← VPN admin (10 files)
│   ├── basic_commands/          ← Basic commands ✅ MIGRATED
│   ├── buy_gb/                  ← Data packages (10 files)
│   ├── consumption/             ← Consumption billing (10 files)
│   ├── key_management/          ← User VPN keys (8 files)
│   ├── operations/              ← Operations menu
│   ├── payments/                ← Payments (crypto + stars)
│   ├── profile/                 ← User profile
│   ├── referrals/               ← Referral system
│   ├── subscriptions/           ← Subscription management
│   └── tickets/                 ← Support tickets
├── handlers/                    ← Main handlers
├── keyboards/                   ← Main keyboards
└── main.py                      ← Entry point
```

### **New Bot Structure**
```
/home/mowgli/usipipo/usipipo-telegram-bot/
├── src/
│   ├── bot/
│   │   ├── handlers/
│   │   │   ├── basic.py         ✅ MIGRATED (from basic_commands)
│   │   │   ├── auth.py          ✅ NEW (invisible auth)
│   │   │   ├── keys.py          ✅ NEW (VPN key management)
│   │   │   ├── operations.py    ✅ NEW (operations menu)
│   │   │   ├── consumption.py   ✅ NEW (consumption billing)
│   │   │   └── packages.py      ✅ NEW (data packages)
│   │   └── keyboards/
│   │       ├── main.py          ✅ MIGRATED
│   │       ├── auth.py          ✅ NEW
│   │       ├── keys.py          ✅ NEW
│   │       ├── messages_keys.py ✅ NEW
│   │       ├── operations.py    ✅ NEW
│   │       ├── messages_operations.py ✅ NEW
│   │       ├── consumption.py   ✅ NEW
│   │       ├── messages_consumption.py ✅ NEW
│   │       ├── packages.py      ✅ NEW
│   │       └── messages_packages.py ✅ NEW
│   └── infrastructure/
│       ├── api_client.py        ✅ MIGRATED
│       ├── config.py            ✅ NEW (pydantic-settings)
│       ├── redis.py             ✅ NEW (RedisPool)
│       ├── token_storage.py     ✅ NEW (TokenStorage)
│       ├── error_handler.py     ✅ MIGRATED
│       └── logger.py            ✅ MIGRATED
├── tests/                       ✅ 160 tests (160 passed)
├── .github/workflows/ci.yml     ✅ NEW (CI/CD)
└── .pre-commit-config.yaml      ✅ NEW
```

---

## ✅ Completed Migration (Phase 1 - Auth)

### **Migrated Components**

| Component | Legacy File | New File | Status |
|-----------|-------------|----------|--------|
| **Basic Commands** | `features/basic_commands/handlers_basic.py` | `src/bot/handlers/basic.py` | ✅ Complete |
| **Basic Messages** | `features/basic_commands/messages_basic.py` | `src/bot/keyboards/main.py` | ✅ Complete |
| **API Client** | N/A (new) | `src/infrastructure/api_client.py` | ✅ Complete |
| **Logger** | N/A (new) | `src/infrastructure/logger.py` | ✅ Complete |
| **Error Handler** | N/A (new) | `src/infrastructure/error_handler.py` | ✅ Complete |

### **New Components (Not in Legacy)**

| Component | File | Purpose |
|-----------|------|---------|
| **Auth Handler** | `src/bot/handlers/auth.py` | Invisible authentication |
| **Auth Messages** | `src/bot/keyboards/auth.py` | Auth constants |
| **Config** | `src/infrastructure/config.py` | pydantic-settings |
| **Redis Pool** | `src/infrastructure/redis.py` | Connection pooling |
| **Token Storage** | `src/infrastructure/token_storage.py` | JWT management |
| **CI/CD** | `.github/workflows/ci.yml` | GitHub Actions |
| **Pre-commit** | `.pre-commit-config.yaml` | Git hooks |

### **Test Coverage**

| Test Type | Count | Status |
|-----------|-------|--------|
| Unit Tests | 154 | ✅ 154 passed |
| Integration Tests | 6 | ✅ 6 passed |
| **Total** | **160** | ✅ **160 passed** |

---

## ✅ Completed Migration (Phase 3 - Operations + Profile)

### **Migrated Components**

| Component | Legacy File | New File | Status |
|-----------|-------------|----------|--------|
| **Operations Handlers** | `features/operations/handlers_operations.py` | `src/bot/handlers/operations.py` | ✅ Complete |
| **Operations Keyboards** | `features/operations/keyboards_operations.py` | `src/bot/keyboards/operations.py` | ✅ Complete |
| **Operations Messages** | N/A (new) | `src/bot/keyboards/messages_operations.py` | ✅ Complete |

**Commands Implemented:**
- `/operaciones` - Operations menu

**Backend Integration:**
- `GET /api/v1/referrals/me` - Get referral stats
- `GET /api/v1/transactions` - Get transactions history

**Test Coverage:** 21 new tests (all passing)

---

## ✅ Completed Migration (Phase 2 - VPN Key Management)

### **Migrated Components**

| Component | Legacy File | New File | Status |
|-----------|-------------|----------|--------|
| **Key Management Handlers** | `features/key_management/handlers_key_management.py` | `src/bot/handlers/keys.py` | ✅ Complete |
| **Key Info Handlers** | `features/key_management/handlers_key_info.py` | `src/bot/handlers/keys.py` | ✅ Complete |
| **Key Actions Handlers** | `features/key_management/handlers_key_actions.py` | `src/bot/handlers/keys.py` | ✅ Complete |
| **Key Latency Handlers** | `features/key_management/handlers_key_latency.py` | `src/bot/handlers/keys.py` | ✅ Complete |
| **Key Keyboards** | `features/key_management/keyboards_key_management.py` | `src/bot/keyboards/keys.py` | ✅ Complete |
| **Key Messages** | `features/key_management/messages_key_management.py` | `src/bot/keyboards/messages_keys.py` | ✅ Complete |

**Commands Implemented:**
- `/keys` - List VPN keys
- `/newkey` - Create new key
- `/delkey` - Delete key
- `/qr` - Show QR code

**Backend Integration:**
- `GET /api/v1/vpn/keys` - List keys
- `POST /api/v1/vpn/keys` - Create key
- `DELETE /api/v1/vpn/keys/{id}` - Delete key
- `GET /api/v1/vpn/keys/{id}/config` - Get config (QR/link)

**Test Coverage:** 25 new tests (all passing)

---

## 🟡 Pending Migration (Phase 4+) - Prioritized by Complexity

### **Priority 1: Consumption Billing (MEDIUM)** ⭐⭐⭐
**Complexity:** ⭐⭐⭐ (Medium)
**Files to Migrate:** 10
**Status:** 🟡 Next

| Legacy File | New File | Priority |
|-------------|----------|----------|
| `features/consumption/handlers_consumption.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/handlers_activation.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/handlers_cancellation.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/handlers_invoice.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/handlers_menu.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/handlers_status.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/keyboards_consumption.py` | `src/bot/keyboards/consumption.py` | P2 |
| `features/consumption/messages_consumption.py` | `src/bot/keyboards/consumption.py` | P2 |

**Commands:**
- `/consumo` - Consumption menu
- `/activar` - Activate consumption mode
- `/cancelar` - Cancel consumption mode
- `/factura` - View invoices

**Backend Integration:**
- `GET /api/v1/consumption/status` - Get consumption status
- `POST /api/v1/consumption/activate` - Activate consumption
- `POST /api/v1/consumption/cancel` - Cancel consumption
- `GET /api/v1/consumption/invoices` - Get invoices

**Estimated Effort:** 6-8 hours

---

### **Priority 2: Data Packages / Buy GB (MEDIUM)** ⭐⭐⭐
**Complexity:** ⭐⭐⭐ (Medium)
**Files to Migrate:** 10
**Status:** ⏳ Planned

---

### **Priority 3: Payments (MEDIUM-HARD)** ⭐⭐⭐⭐
**Complexity:** ⭐⭐⭐ (Medium)  
**Files to Migrate:** 10  
**Backend Endpoints:** Already available  

| Legacy File | New File | Priority |
|-------------|----------|----------|
| `features/consumption/handlers_consumption.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/handlers_activation.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/handlers_cancellation.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/handlers_invoice.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/handlers_menu.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/handlers_status.py` | `src/bot/handlers/consumption.py` | P2 |
| `features/consumption/keyboards_consumption.py` | `src/bot/keyboards/consumption.py` | P2 |
| `features/consumption/messages_consumption.py` | `src/bot/keyboards/consumption.py` | P2 |

**Commands:**
- `/consumo` - Consumption menu
- `/activar` - Activate consumption mode
- `/cancelar` - Cancel consumption mode
- `/factura` - View invoices

**Backend Integration:**
- `GET /api/v1/consumption/status` - Get consumption status
- `POST /api/v1/consumption/activate` - Activate consumption
- `POST /api/v1/consumption/cancel` - Cancel consumption
- `GET /api/v1/consumption/invoices` - Get invoices

**Estimated Effort:** 6-8 hours

---

### **Priority 5: Data Packages / Buy GB (MEDIUM)** ⭐⭐⭐
**Complexity:** ⭐⭐⭐ (Medium)  
**Files to Migrate:** 10  

| Legacy File | New File | Priority |
|-------------|----------|----------|
| `features/buy_gb/handlers_buy_gb.py` | `src/bot/handlers/packages.py` | P3 |
| `features/buy_gb/handlers_packages.py` | `src/bot/handlers/packages.py` | P3 |
| `features/buy_gb/handlers_payment_crypto.py` | `src/bot/handlers/packages.py` | P3 |
| `features/buy_gb/handlers_payment_stars.py` | `src/bot/handlers/packages.py` | P3 |
| `features/buy_gb/keyboards_buy_gb.py` | `src/bot/keyboards/packages.py` | P3 |
| `features/buy_gb/messages_buy_gb.py` | `src/bot/keyboards/packages.py` | P3 |

**Commands:**
- `/comprar` - Buy data packages
- `/paquetes` - View available packages
- `/pago/crypto` - Pay with crypto
- `/pago/stars` - Pay with Telegram Stars

**Backend Integration:**
- `GET /api/v1/data-packages` - List packages
- `POST /api/v1/payments/crypto` - Create crypto payment
- `POST /api/v1/payments/stars` - Create Stars payment

**Estimated Effort:** 6-8 hours

---

### **Priority 6: Payments (MEDIUM-HARD)** ⭐⭐⭐⭐
**Complexity:** ⭐⭐⭐⭐ (Medium-Hard)  
**Files to Migrate:** 8  
**Backend Integration:** TronDealer webhook already migrated  

| Legacy File | New File | Priority |
|-------------|----------|----------|
| `features/payments/handlers_payments.py` | `src/bot/handlers/payments.py` | P4 |
| `features/payments/handlers_crypto.py` | `src/bot/handlers/payments.py` | P4 |
| `features/payments/handlers_stars.py` | `src/bot/handlers/payments.py` | P4 |
| `features/payments/keyboards_payments.py` | `src/bot/keyboards/payments.py` | P4 |
| `features/payments/messages_payments.py` | `src/bot/keyboards/payments.py` | P4 |

**Backend Integration:**
- TronDealer webhook: ✅ Already migrated
- Telegram Stars: ✅ Already implemented in backend

**Estimated Effort:** 8-10 hours

---

### **Priority 7: Subscriptions (MEDIUM-HARD)** ⭐⭐⭐⭐
**Complexity:** ⭐⭐⭐⭐ (Medium-Hard)  
**Files to Migrate:** 6  

| Legacy File | New File | Priority |
|-------------|----------|----------|
| `features/subscriptions/handlers_subscriptions.py` | `src/bot/handlers/subscriptions.py` | P5 |
| `features/subscriptions/handlers_plans.py` | `src/bot/handlers/subscriptions.py` | P5 |
| `features/subscriptions/keyboards_subscriptions.py` | `src/bot/keyboards/subscriptions.py` | P5 |

**Commands:**
- `/suscripcion` - View subscription
- `/planes` - View available plans
- `/renovar` - Renew subscription

**Backend Integration:**
- `GET /api/v1/subscriptions/me` - Get user subscription
- `GET /api/v1/subscriptions/plans` - List plans
- `POST /api/v1/subscriptions/activate` - Activate subscription

**Estimated Effort:** 6-8 hours

---

### **Priority 8: Referrals (MEDIUM)** ⭐⭐⭐
**Complexity:** ⭐⭐⭐ (Medium)  
**Files to Migrate:** 4  

| Legacy File | New File | Priority |
|-------------|----------|----------|
| `features/referrals/handlers_referrals.py` | `src/bot/handlers/referrals.py` | P6 |
| `features/referrals/keyboards_referrals.py` | `src/bot/keyboards/referrals.py` | P6 |

**Commands:**
- `/referidos` - View referrals
- `/invitar` - Get referral link

**Backend Integration:**
- `GET /api/v1/referrals/me` - Get referral stats
- `GET /api/v1/referrals/link` - Get referral link

**Estimated Effort:** 4-6 hours

---

### **Priority 9: Tickets/Support (MEDIUM)** ⭐⭐⭐
**Complexity:** ⭐⭐⭐ (Medium)  
**Files to Migrate:** 6  

| Legacy File | New File | Priority |
|-------------|----------|----------|
| `features/tickets/handlers_tickets.py` | `src/bot/handlers/tickets.py` | P7 |
| `features/tickets/handlers_create.py` | `src/bot/handlers/tickets.py` | P7 |
| `features/tickets/keyboards_tickets.py` | `src/bot/keyboards/tickets.py` | P7 |

**Commands:**
- `/tickets` - View tickets
- `/nuevoticket` - Create new ticket
- `/mistickets` - View my tickets

**Backend Integration:**
- `GET /api/v1/tickets` - List tickets
- `POST /api/v1/tickets` - Create ticket

**Estimated Effort:** 4-6 hours

---

### **Priority 10: Admin Panel (HARD)** ⭐⭐⭐⭐⭐
**Complexity:** ⭐⭐⭐⭐⭐ (Hard)  
**Files to Migrate:** 24  
**Access Control:** Admin-only commands  

| Legacy File | New File | Priority |
|-------------|----------|----------|
| `features/admin/handlers_*.py` (14 files) | `src/bot/handlers/admin/` | P8 |
| `features/admin_vpn/handlers_*.py` (10 files) | `src/bot/handlers/admin/` | P8 |

**Admin Commands:**
- `/admin` - Admin dashboard
- `/users` - User management
- `/keys` - Key management (admin view)
- `/tickets` - Ticket management (admin view)
- `/servers` - Server monitoring

**Access Control:**
- Requires admin authentication
- `ADMIN_ID` from .env
- Middleware for admin-only commands

**Estimated Effort:** 16-20 hours

---

## 📋 Migration Roadmap

### **Phase 6: Payments + Subscriptions** ✅
- [x] Migrate payments handlers (crypto + stars)
- [x] Migrate payments keyboards
- [x] Migrate payments messages
- [x] Migrate subscriptions handlers
- [x] Migrate subscriptions keyboards
- [x] Migrate subscriptions messages
- [x] Integration with backend payments/subscriptions endpoints
- [x] Tests (103 tests)
- **Status:** COMPLETE

### **Phase 7: Referrals + Tickets** (Next)
- [ ] Migrate referrals
- [ ] Migrate tickets
- [ ] End-to-end testing
- **Estimated:** 8-12 hours

### **Phase 5: Payments + Subscriptions** (Medium Term)
- [ ] Migrate payments (crypto + stars)
- [ ] Migrate subscriptions
- [ ] Webhook integration testing
- **Estimated:** 14-18 hours

### **Phase 6: Referrals + Tickets** (Long Term)
- [ ] Migrate referrals
- [ ] Migrate tickets
- [ ] End-to-end testing
- **Estimated:** 8-12 hours

### **Phase 7: Admin Panel** (Final)
- [ ] Migrate admin handlers
- [ ] Admin access control middleware
- [ ] Admin dashboard
- **Estimated:** 16-20 hours

---

## 📊 Migration Progress

| Phase | Feature | Files | Status | Progress |
|-------|---------|-------|--------|----------|
| **Phase 1** | **Auth + Infrastructure** | 12 | ✅ Complete | 100% |
| **Phase 2** | **VPN Key Management** | 8 | ✅ Complete | 100% |
| **Phase 3** | **Operations + Profile** | 8 | ✅ Complete | 100% |
| **Phase 4** | **Consumption Billing** | 10 | ✅ Complete | 100% |
| **Phase 5** | **Data Packages** | 10 | ✅ Complete | 100% |
| **Phase 6** | **Payments + Subscriptions** | 14 | ✅ Complete | 100% |
| **Phase 7** | **Referrals + Tickets** | 10 | ⏳ Planned | 0% |
| **Phase 8** | **Admin Panel** | 24 | ⏳ Planned | 0% |
| **TOTAL** | **All Features** | **92** | 🟡 In Progress | **~65%** |

---

## 🎯 Recommendations

### **Start With (Easiest):**
1. ✅ **Phase 1: Auth** - COMPLETE
2. ✅ **Phase 2: VPN Key Management** - COMPLETE
3. ✅ **Phase 3: Operations + Profile** - COMPLETE
4. 🟡 **Phase 4: Consumption Billing** - Next (6-8 hours)

### **Why This Order:**
- **Low complexity** - Simple CRUD operations
- **High value** - Core user functionality
- **Backend ready** - All endpoints available
- **Builds momentum** - Quick wins

### **Leave for Last (Hardest):**
- **Admin Panel** - Requires access control, complex UI
- **Payments** - Requires thorough testing, security critical
- **Subscriptions** - Complex state management

**Estimated Total Effort Remaining:** 44-58 hours

---

## 🔗 Related Documentation

- **Ecosystem Context:** `/plans/ECOSYSTEM-CONTEXT.md`
- **Migration Progress:** `/plans/MIGRATION-PROGRESS.md`
- **Auth Implementation:** `/plans/TELEGRAM-AUTH-IMPLEMENTATION.md`
- **Integration Tests:** `usipipo-telegram-bot/INTEGRATION-TEST-SUMMARY.md`
- **Backend API Docs:** http://localhost:8001/docs

---

**Last Updated:** 2026-03-28
**Next Phase:** Referrals + Tickets (Phase 7)
**Estimated Total Effort Remaining:** 24-36 hours
