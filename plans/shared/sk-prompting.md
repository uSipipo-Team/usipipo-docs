# Prompt para Continuar la Migración del Ecosistema

**Objetivo:** Backend 100% completo + Infraestructura lista + Multi-Bot Architecture implementada + Support Bot en producción + VPN Agent multi-país operativo + Auto-Registration implementado.

**Fecha de Actualización:** 2026-03-30

---

## 🎯 Contexto: Migración usipipobot (monorepo) → Multi-Repo → Multi-Bot → Multi-País

### 📊 Estado Actual (2026-03-30)

#### ✅ Backend Completado (100%):
1. **Payments** - 100% ✅ (+ TronDealer webhook migrado)
2. **VPN Management** - 100% ✅ (+ ServerRegistry para multi-país)
3. **Subscriptions** - 100% ✅
4. **Consumption Billing** - 100% ✅
5. **User Management** - 100% ✅
6. **Tickets/Support System** - 100% ✅ (Migrado a @uSipipoSupport_Bot)
7. **Admin Panel** - 100% ✅
8. **Data Packages** - 100% ✅
9. **Referrals** - 100% ✅
10. **Wallet Management** - 100% ✅
11. **Device Registration** - 100% ✅ (push notifications)
12. **Telegram Auth (Invisible)** - 100% ✅ (auto-register + refresh)
13. **ServerRegistry** - 100% ✅ (multi-país orchestration)
14. **Agent Auto-Registration** - 100% ✅ (NUEVO! v0.12.0)

#### ✅ Multi-Client Architecture Completada (4 semanas):
- **Week 1:** user_id int → UUID ✅
- **Week 2:** VpnKey + Enum Unification ✅
- **Week 3:** Multi-Client Auth (email/password + Telegram) ✅
- **Week 4:** Device Registration (push notifications) ✅

#### ✅ Multi-Bot Architecture Completada:
- **@usipipobot** (Main Bot) - v1.2.0 ✅
  - VPN Keys, Payments, Subscriptions, Consumption, Data Packages, Referrals
  - MainMenuKeyboard con botones inline
  - Botón "💬 Soporte Técnico" → @uSipipoSupport_Bot
  - Tickets migrados a @uSipipoSupport_Bot
- **@uSipipoSupport_Bot** (Support Bot) - v0.2.0 ✅
  - Ticket management system
  - Welcome menu con botones inline
  - Deep link handling (?start=help_from_main)
  - 58 tests (100% passing)
  - CI/CD configurado
  - Production ready (systemd service)

#### ✅ VPN Agent Architecture + Auto-Registration (NUEVO!):
- **usipipo-agent** - v0.2.2 ✅ (Auto-Registration + Backend v0.12.0)
  - Go 1.21+ con Gin framework
  - WireGuard con wgctrl library oficial
  - Outline Manager API integration
  - Rate limiting (10 RPS, burst 20)
  - **Auto-Registration:** Se registra automáticamente con backend al iniciar
  - **Metadata auto-colectada:** hostname, IP, país, OS, versión
  - Auto-report metrics a backend (cada 1 min)
  - Multi-platform builds (linux, windows, darwin × amd64, arm64)
  - GitHub Actions CI/CD
  - Systemd service ready
  - Install script con auto-update (--update flag)
  - **FIX v0.2.2:** Registrar constructor helper (NewRegistrarFromValues)
  - **FIX v0.2.1:** UUID visibility (isValidUUID → IsValidUUID)
  - **FIX v0.2.0:** Auto-Registration initial implementation
  - **FIX v0.1.20:** Missing assets en GitHub Releases (timing issue)
  - **FIX v0.1.19:** Build errors (unused imports, type conversions)

#### ✅ Infraestructura Completada:
- **Backend API** v0.12.0 - Running on port 8001 ✅ (Auto-Registration endpoints)
- **Landing Page** - Running on port 5000 ✅
- **Caddy Proxy** - Path prefix routing configured ✅
- **GitHub Wiki** - 4 pages published ✅
- **usipipo-commons** v0.13.0 - Published on PyPI ✅ (Server entity + ServerStatus)
- **TronDealer Webhook** - Migrado y testeado ✅
- **Telegram Bot CI/CD** - Configurado ✅
- **Support Bot CI/CD** - Configurado ✅
- **Agent CI/CD** - Configurado ✅
- **Branch Protection** - All repos ✅
- **Support Bot** - Deployed & Released (systemd service) ✅
- **Main Menu Keyboard** - Implemented ✅
- **Deep Link Handling** - Implemented ✅
- **Rate Limiting** - Agent production-ready ✅
- **Auto-Registration** - Tested & Verified ✅

---

## 🎉 LATEST RELEASES (2026-03-30)

### **Backend v0.12.0** - Auto-Registration API (NEW!)

**What's New:**
- ✅ **Database:** New `agent_api_keys` table for secure API key management
- ✅ **Service:** `AgentRegistrationService` for registration logic
- ✅ **API Endpoints:**
  - `POST /api/v1/servers/register-agent` - Register new agent
  - `GET /api/v1/servers/register-agent` - Check registration status
  - `POST /api/v1/admin/agent-api-keys` - Generate API keys (admin)
  - `GET /api/v1/admin/agent-api-keys` - List API keys (admin)
- ✅ **Security:** API keys hashed with SHA-256, single-use, optional expiration
- ✅ **Tests:** Integration tests for registration flow
- ✅ **Docs:** Auto-Registration Guide
- ✅ **FIX:** `agent_api_key_rel` back_populates mismatch

**Release:** https://github.com/uSipipo-Team/usipipo-backend/releases/tag/v0.12.0

---

### **VPN Agent v0.2.2** - Registrar Constructor Fix

**What's New:**
- ✅ **FIX:** Add `NewRegistrarFromValues()` helper for callers without Config object
- ✅ **FIX:** Update `reporter.go` to use `NewRegistrarFromValues()`
- ✅ **FIX:** Remove unused `config` import from `reporter.go`
- ✅ **CI/CD:** All 6 platform builds passing
- ✅ **Assets:** 8 artifacts generated successfully

**Release:** https://github.com/uSipipo-Team/usipipo-agent/releases/tag/v0.2.2

---

### **VPN Agent v0.2.1** - UUID Visibility Fix

**What's New:**
- ✅ **FIX:** `isValidUUID` → `IsValidUUID` (public function visibility)
- ⚠️ **NOTE:** Intermediate fix, use v0.2.2 instead

**Release:** Superseded by v0.2.2

---

### **VPN Agent v0.2.0** - Auto-Registration Initial

**What's New:**
- ✅ **registrar package** - Handles registration with backend
- ✅ **geoip utility** - GeoIP location lookup (ip-api.com)
- ✅ **reporter.go** - Integrated auto-registration before sending metrics
- ✅ **config.go** - New config variables (AgentURL, SupportsOutline, SupportsWireGuard)
- ✅ **.env.example** - Updated with AGENT_API_KEY, SERVER_ID (auto-filled)
- ✅ **AUTO-REGISTRATION-GUIDE.md** - Complete setup guide

**Release:** Superseded by v0.2.2

---

## 📁 Estructura de Directorios (Actualizada)

### **Backend (v0.12.0)**
```
/home/mowgli/usipipo/usipipo-backend/
├── src/core/application/services/
│   ├── agent_registration_service.py  ← NEW: Auto-registration logic
│   ├── server_registry_service.py
│   └── ...
├── src/infrastructure/api/v1/routes/
│   ├── agent_registration.py  ← NEW: Agent registration endpoints
│   ├── admin_agent_keys.py    ← NEW: Admin API key management
│   └── ...
├── src/shared/schemas/
│   └── agent_registration.py  ← NEW: Pydantic schemas
├── src/infrastructure/persistence/models/
│   ├── agent_api_key_model.py  ← NEW: API key model
│   └── vpn_server_model.py     ← Updated: metadata columns
├── CHANGELOG.md ← Updated to v0.12.0
└── pyproject.toml ← Version 0.12.0
```

### **VPN Agent (v0.2.2)**
```
/home/mowgli/usipipo/usipipo-agent/
├── internal/
│   ├── registrar/
│   │   └── registrar.go  ← NEW: Registration logic
│   │       - NewRegistrar(cfg) - From Config object
│   │       - NewRegistrarFromValues(...) - Helper for simple callers
│   │       - IsValidUUID() - Public UUID validation
│   ├── utils/geoip/
│   │   └── geoip.go  ← NEW: GeoIP lookup
│   └── reporter/
│       └── reporter.go  ← Updated: Auto-registration integration
├── AUTO-REGISTRATION-GUIDE.md  ← NEW: Complete guide
├── CHANGELOG.md ← Updated to v0.2.2
└── .env.example ← Updated with new variables
```

---

## 🔐 Auto-Registration Flow

### **Registration Process**
```
1. Admin generates API key via backend /admin/agent-api-keys
2. Admin copies AGENT_API_KEY to agent .env (SERVER_ID= leave empty)
3. Agent starts → reads AGENT_API_KEY
4. Agent collects metadata:
   - hostname, IP (GeoIP), country, region, city
   - agent_version, os_type, os_arch
   - agent_url, supports_outline, supports_wireguard
5. Agent POST /api/v1/servers/register-agent → Backend
6. Backend validates API key (hash match, not used/expired)
7. Backend creates vpn_servers record with metadata
8. Backend marks API key as "used"
9. Backend returns server_id (UUID)
10. Agent saves UUID to .env (SERVER_ID=...)
11. Agent sends metrics every 1 minute using UUID
```

### **API Key Security**
- ✅ SHA-256 hashed before storage
- ✅ Shown only once during generation
- ✅ Single-use (status: active → used)
- ✅ Optional expiration dates
- ✅ Admin-only generation
- ✅ Revocable at any time

---

## 📊 Timeline Actualizada

| Week | Dates | Focus | Status |
|------|-------|-------|--------|
| Week 1-6 | Mar 18 - Apr 28 | Core Backend Features | ✅ Complete |
| Week 7 | Apr 29-May 5 | Admin Panel | ✅ Complete |
| Week 8 | May 6-12 | Data Packages + Referrals | ✅ Complete |
| Week 9 | May 13-19 | Wallet Management | ✅ Complete |
| Week 10 | May 20-26 | Infrastructure + Wiki | ✅ Complete |
| **Week 11-14** | **Mar 22-Apr 18** | **Multi-Client Architecture** | ✅ **Complete** |
| **Week 15-19** | **Apr 19-May 20** | **Telegram Bot Auth** | ✅ **Complete (100%)** |
| **Week 20-22** | **May 21-Jun 10** | **VPN Key Management (Bot)** | ✅ **Complete (100%)** |
| **Week 23-24** | **Jun 11-24** | **Operations + Profile (Bot)** | ✅ **Complete (100%)** |
| **Week 25-27** | **Jun 25-Jul 15** | **Consumption + Packages (Bot)** | ✅ **Complete** |
| **Week 28-29** | **Jul 16-29** | **Payments + Subscriptions (Bot)** | ✅ **Complete** |
| **Week 30-31** | **Jul 30-Aug 12** | **Referrals + Tickets (Bot)** | ✅ **Complete** |
| **Week 32** | **Aug 13-20** | **Support Bot Migration** | ✅ **Complete** |
| **Week 33-36** | **Aug 21-Sep 15** | **VPN Agent Development** | ✅ **Complete** |
| **Week 37** | **Sep 16-22** | **Auto-Registration Implementation** | ✅ **Complete** |
| Week 38-41 | Sep 23-Oct 14 | Admin Panel (Bot) | 📋 Planned |
| Week 42-45 | Oct 15-Nov 5 | Android App (Go + Kotlin) | 📋 Planned |

---

## 📚 Documentation

### **Backend Documentation**
- **GitHub Wiki:** https://github.com/uSipipo-Team/usipipo-backend/wiki
- **API Docs:** http://localhost:8001/docs (Swagger)
- **Releases:** https://github.com/uSipipo-Team/usipipo-backend/releases

### **VPN Agent Documentation (UPDATED!)**
- **Repo:** https://github.com/uSipipo-Team/usipipo-agent
- **Release v0.2.2:** https://github.com/uSipipo-Team/usipipo-agent/releases/tag/v0.2.2
- **AUTO-REGISTRATION-GUIDE.md** - Complete setup and troubleshooting guide
- **Design Doc:** `usipipo-docs/plans/agent/2026-03-29-agent-auto-registration-design.md`
- **Implementation Plan:** `usipipo-docs/plans/agent/2026-03-29-agent-auto-registration-plan.md`

---

## ✅ Completed Infrastructure Tasks (UPDATED)

- [x] Backend systemd service configured
- [x] Landing page service running
- [x] Caddy path prefix routing
- [x] GitHub Wiki published (4 pages)
- [x] Telegram token updated in .env
- [x] usipipo-commons v0.13.0 on PyPI
- [x] Backend v0.12.0 released (Auto-Registration API)
- [x] TronDealer webhook migrated and tested
- [x] TronDealer documentation added
- [x] Main Bot CI/CD configured
- [x] Support Bot CI/CD configured
- [x] Agent CI/CD configured
- [x] Branch protection enabled (all repos)
- [x] Integration tests with production backend
- [x] **Support Bot created & released (v0.2.0)**
- [x] **Support Bot documentation complete**
- [x] **Tickets migrated from main bot**
- [x] **Main bot updated to v1.2.0**
- [x] **Multi-bot documentation published**
- [x] **MainMenuKeyboard implemented**
- [x] **Deep link handling implemented**
- [x] **ConversationHandler para creación de claves**
- [x] **APIClient.delete() method**
- [x] **GET /users/me endpoint**
- [x] **Support Bot systemd service** habilitado
- [x] **VPN Agent created & released (v0.2.2)**
- [x] **VPN Agent documentation complete**
- [x] **Install script with auto-update (--update flag)**
- [x] **Rate limiting for production security**
- [x] **wgctrl library for WireGuard** (no shell commands)
- [x] **Generic usipipo user creation**
- [x] **Sudoers configuration for WireGuard**
- [x] **Build errors fixed** (unused imports, type conversions)
- [x] **CI Debug Workflow skill** created
- [x] **GitHub Release assets fix** (v0.1.20 rebuild con 8 assets subidos)
- [x] **Install script tested** (download, update functionality working)
- [x] **Auto-Registration implemented** (backend v0.12.0 + agent v0.2.2)
- [x] **Auto-Registration tested & verified** (metrics flowing correctly)

---

## 🚀 Next Steps

### **1. Android App Refactoring (Go + Kotlin)**

**Architecture:**
```
usipipovpnapp/
├── go/
│   └── vpn-engine/
│       ├── wireguard.go (wgctrl library)
│       ├── outline.go (Outline API)
│       └── main.go (JNI bridge)
├── android/
│   └── app/
│       └── src/main/java/com/usipipo/vpn/
│           ├── MainActivity.kt
│           ├── VpnService.kt
│           └── JNIBridge.kt
└── build.gradle.kts
```

**Implementation:**
- Go engine for VPN operations (wgctrl, Outline API)
- Kotlin UI with Jetpack Compose
- JNI bridge for Go ↔ Kotlin communication
- Background VPN service (VpnService)
- Production-ready for Play Store

### **2. Admin Panel Bot** (Next Phase)
```bash
# Commands to implement:
# /admin - Admin dashboard
# /users - User management
# /keys - Key management (admin view)
# /tickets - Ticket management (admin view)
# /servers - Server monitoring
```

### **3. Documentation Portal**
- usipipo-docs repository ready
- Deploy on port 4000

### **4. Monitoring & Observability**
- Implement logging aggregation
- Set up alerts
- Dashboard creation

---

**Last Updated:** 2026-03-30
**Backend Status:** 100% COMPLETE ✅ (v0.12.0 - Auto-Registration API)
**Multi-Client Status:** 100% COMPLETE ✅
**Multi-Bot Status:** 100% COMPLETE ✅
**VPN Agent Status:** 100% COMPLETE ✅ (v0.2.2 - Auto-Registration + Fixes)
**Auto-Registration:** TESTED & VERIFIED ✅
**Main Bot:** v1.2.0 (MainMenuKeyboard + Soporte) ✅
**Support Bot:** v0.2.0 (Welcome Menu + Deep Link) ✅
**Tests:** 348 total (348 passed) ✅
**Documentation:** Complete ✅
**Next:** Android App Refactoring (Go + Kotlin)
