# Telegram Bot - Migración a Arquitectura Hexagonal

**Fecha:** 2026-03-24
**Branch:** `feature/hexagonal-architecture`
**Estado:** En Progreso

---

## 🎯 Objetivo

Refactorizar la estructura del bot Telegram desde una arquitectura basada en features planos a **Arquitectura Hexagonal (Ports & Adapters)** para:

1. **Escalabilidad** - Crecimiento sostenible sin acoplamiento
2. **Testabilidad** - Tests unitarios aislados con mocks
3. **Mantenibilidad** - Separación clara de responsabilidades
4. **Producción Ready** - Preparado para escalado continuo

---

## 📦 Dependencia: usipipo-commons

**Versión requerida:** `>=0.12.0`

Las entidades del dominio **NO se duplican** en el bot. Se importan directamente desde `usipipo-commons`:

```python
from usipipo_commons.domain.entities import User, VpnKey, Payment, Referral
from usipipo_commons.domain.enums import KeyType, KeyStatus, PaymentStatus, PaymentMethod
from usipipo_commons.constants import FREE_GB, MAX_KEYS_PER_USER, FREE_KEYS_LIMIT
```

### Entidades Disponibles desde usipipo-commons:

| Entidad | Descripción | Campos Principales |
|---------|-------------|-------------------|
| `User` | Usuario del sistema | id, telegram_id, balance_gb, referral_code, is_admin |
| `VpnKey` | Clave VPN (WireGuard/Outline) | id, user_id, key_type, status, key_data, data_limit_gb |
| `Payment` | Pago realizado | id, user_id, amount_usd, method, status, crypto_address |
| `Referral` | Relación de referido | referrer_id, referred_id, is_active, bonus_applied |
| `ConsumptionBilling` | Billing por consumo | id, user_id, status, current_period |
| `SubscriptionPlan` | Plan de suscripción | id, name, price, data_limit, duration |
| `Wallet` | Billetera del usuario | id, user_id, balance, status |

### Enums Disponibles:

- `KeyType`: WIREGUARD | OUTLINE
- `KeyStatus`: ACTIVE | INACTIVE | EXPIRED
- `PaymentStatus`: PENDING | COMPLETED | FAILED | EXPIRED
- `PaymentMethod`: CRYPTO | TELEGRAM_STARS
- `PlanType`: FREE | BASIC | PRO | ENTERPRISE

---

## 📐 Arquitectura Hexagonal - Conceptos

```
┌─────────────────────────────────────────────────────────┐
│                     DOMINIO (Core)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  Entities   │  │   Value     │  │  Services   │     │
│  │             │  │   Objects   │  │  (Domain)   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
                            ↑ ↓
┌─────────────────────────────────────────────────────────┐
│                 APLICACIÓN (Use Cases)                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Ports     │  │   Use       │  │  Commands/  │     │
│  │ (Interfaces)│  │   Cases     │  │   Queries   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
                            ↑ ↓
┌─────────────────────────────────────────────────────────┐
│                 INFRAESTRUCTURA (Adapters)               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Primary   │  │  Secondary  │  │  Utils/     │     │
│  │  Adapters   │  │   Adapters  │  │  Helpers    │     │
│  │ (Driving)   │  │  (Driven)   │  │             │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### Capas:

1. **Dominio** - Reglas de negocio puras (sin dependencias externas)
   - Entities: Modelos de negocio (User, VpnKey, Payment, etc.)
   - Value Objects: Objetos inmutables con validación
   - Domain Services: Lógica de negocio transversal

2. **Aplicación** - Casos de uso (orquestación)
   - Ports: Interfaces que definen contratos
   - Use Cases: Implementación de casos de uso
   - DTOs: Objetos de transferencia de datos

3. **Infraestructura** - Adaptadores (implementaciones concretas)
   - Primary Adapters: Entradas (Telegram handlers, API routes)
   - Secondary Adapters: Salidas (Backend API, Redis, DB)
   - Utils: Logger, config, helpers

---

## 📁 Nueva Estructura de Directorios

### Estructura Actual (Pre-Refactor)
```
src/
├── bot/
│   ├── handlers/
│   │   ├── basic.py
│   │   └── auth.py
│   └── keyboards/
│       ├── main.py
│       └── auth.py
└── infrastructure/
    ├── api_client.py
    ├── config.py
    ├── error_handler.py
    ├── logger.py
    ├── redis.py
    └── token_storage.py
```

### Nueva Estructura Hexagonal (Post-Refactor)

**Nota:** Las entidades del dominio (`User`, `VpnKey`, `Payment`, etc.) **NO** se crean en este repositorio. Se importan desde `usipipo-commons`.

```
src/
├── __init__.py
├── __main__.py
├── main.py
│
├── application/                     ← APLICACIÓN (Casos de uso)
│   ├── __init__.py
│   ├── ports/                       ← Interfaces (contratos)
│   │   ├── __init__.py
│   │   ├── token_storage_port.py    ← Port para almacenamiento de tokens
│   │   ├── user_repository_port.py  ← Port para repositorio de usuarios
│   │   ├── vpn_key_repository_port.py
│   │   ├── payment_repository_port.py
│   │   └── backend_api_port.py      ← Port para API externa
│   ├── use_cases/                   ← Implementación de casos de uso
│   │   ├── __init__.py
│   │   ├── auth/
│   │   │   ├── __init__.py
│   │   │   ├── auto_register_user.py    ← Caso: Auto-registrar usuario
│   │   │   ├── refresh_token.py         ← Caso: Refresh token
│   │   │   └── get_user_profile.py      ← Caso: Obtener perfil
│   │   ├── vpn_keys/
│   │   │   ├── __init__.py
│   │   │   ├── create_vpn_key.py        ← Caso: Crear VPN key
│   │   │   ├── list_user_keys.py        ← Caso: Listar keys
│   │   │   ├── delete_vpn_key.py        ← Caso: Eliminar key
│   │   │   └── get_key_config.py        ← Caso: Obtener config
│   │   ├── payments/
│   │   │   ├── __init__.py
│   │   │   ├── create_crypto_payment.py
│   │   │   └── create_stars_payment.py
│   │   └── referrals/
│   │       ├── __init__.py
│   │       ├── get_referral_code.py
│   │       └── get_referral_stats.py
│   ├── commands/                  ← Command objects (write operations)
│   │   ├── __init__.py
│   │   └── create_vpn_key_command.py
│   ├── queries/                   ← Query objects (read operations)
│   │   ├── __init__.py
│   │   └── get_user_profile_query.py
│   └── dtos/                      ← DTOs para transferencia
│       ├── __init__.py
│       ├── user_dto.py
│       ├── vpn_key_dto.py
│       └── payment_dto.py
│
└── infrastructure/                ← INFRAESTRUCTURA (Adaptadores)
    ├── __init__.py
    ├── primary_adapters/          ← Adaptadores de entrada (driving)
    │   ├── __init__.py
    │   ├── telegram/              ← Telegram bot adapters
    │   │   ├── __init__.py
    │   │   ├── bot_adapter.py         ← Adapter principal del bot
    │   │   ├── handlers/              ← Handlers de Telegram
    │   │   │   ├── __init__.py
    │   │   │   ├── auth_handler.py      ← Handler de auth
    │   │   │   ├── basic_handler.py     ← Handler de comandos básicos
    │   │   │   ├── vpn_key_handler.py   ← Handler de VPN keys
    │   │   │   ├── payment_handler.py   ← Handler de pagos
    │   │   │   └── referral_handler.py  ← Handler de referidos
    │   │   └── keyboards/             ← Teclados inline/reply
    │   │       ├── __init__.py
    │   │       ├── auth_keyboard.py
    │   │       ├── main_menu_keyboard.py
    │   │       ├── vpn_key_keyboard.py
    │   │       └── payment_keyboard.py
    │   └── api/                   ← API REST adapters (si aplica)
    │       └── __init__.py
    ├── secondary_adapters/        ← Adaptadores de salida (driven)
    │   ├── __init__.py
    │   ├── backend_api/           ← Adaptador hacia Backend API
    │   │   ├── __init__.py
    │   │   ├── backend_api_adapter.py   ← Implementación del port
    │   │   ├── http_client.py           ← Cliente HTTP (httpx)
    │   │   └── endpoints/               ← Endpoints específicos
    │   │       ├── auth_endpoints.py
    │   │       ├── user_endpoints.py
    │   │       ├── vpn_endpoints.py
    │   │       └── payment_endpoints.py
    │   ├── redis/                 ← Adaptador Redis
    │   │   ├── __init__.py
    │   │   ├── redis_adapter.py         ← Implementación del port
    │   │   └── redis_pool.py            ← Connection pool
    │   └── persistence/           ← Persistencia (si aplica)
    │       └── __init__.py
    ├── config/                    ← Configuración
    │   ├── __init__.py
    │   └── settings.py            ← pydantic-settings
    ├── logging/                   ← Logging
    │   ├── __init__.py
    │   └── logger.py
    └── error_handling/            ← Manejo de errores
        ├── __init__.py
        ├── exceptions.py          ← Excepciones personalizadas
        └── error_handler.py       ← Error handler global
```

**Estructura de Dominio (en usipipo-commons):**
```
usipipo-commons/usipipo_commons/
├── domain/
│   ├── entities/          ← User, VpnKey, Payment, Referral, etc.
│   ├── enums/             ← KeyType, KeyStatus, PaymentStatus, etc.
│   └── interfaces/        ← IWalletRepository, IWalletPoolRepository
├── schemas/               ← Pydantic schemas (DTOs)
├── constants/             ← FREE_GB, MAX_KEYS_PER_USER, etc.
└── utils/                 ← Validators, formatters
```

---

## 🔄 Plan de Migración

### Fase 1: Cimientos y Setup (Día 1)
- [ ] Crear estructura de directorios `application/`, `infrastructure/`
- [ ] Actualizar `pyproject.toml` con `usipipo-commons>=0.12.0`
- [ ] Configurar imports desde usipipo-commons:
  ```python
  from usipipo_commons.domain.entities import User, VpnKey, Payment
  from usipipo_commons.domain.enums import KeyType, KeyStatus, PaymentStatus
  from usipipo_commons.constants import FREE_GB, MAX_KEYS_PER_USER
  ```
- [ ] Tests de configuración de imports (5+ tests)

### Fase 2: Ports y Use Cases de Auth (Día 2)
- [ ] Definir **Ports**:
  - [ ] `TokenStoragePort`
  - [ ] `UserRepositoryPort`
  - [ ] `BackendApiPort`
- [ ] Implementar **Use Cases** de Auth:
  - [ ] `AutoRegisterUser` (auto-registro con Telegram)
  - [ ] `RefreshToken` (refresh de JWT)
  - [ ] `GetUserProfile` (obtener perfil)
- [ ] Implementar **Commands/Queries**:
  - [ ] `AutoRegisterUserCommand`
  - [ ] `RefreshTokenCommand`
- [ ] Tests de casos de uso (15+ tests)

### Fase 3: Adaptadores de Infraestructura (Día 3)
- [ ] **Secondary Adapters**:
  - [ ] `BackendApiAdapter` (implementación de BackendApiPort)
  - [ ] `RedisAdapter` (implementación de TokenStoragePort)
  - [ ] `TokenStorage` (con Redis)
- [ ] **Primary Adapters**:
  - [ ] `TelegramBotAdapter` (orquestador principal)
  - [ ] `AuthHandler` (handler de /start, /me, /unlink)
  - [ ] `BasicHandler` (handler de /help, /menu)
- [ ] **Keyboards**:
  - [ ] `AuthKeyboard`
  - [ ] `MainMenuKeyboard`
- [ ] Tests de adaptadores (15+ tests)

### Fase 4: VPN Key Management (Día 4-5)
- [ ] **Domain**:
  - [ ] `VpnKeyService` (lógica de creación)
- [ ] **Application**:
  - [ ] `CreateVpnKey` use case
  - [ ] `ListUserKeys` use case
  - [ ] `DeleteVpnKey` use case
  - [ ] `GetKeyConfig` use case
- [ ] **Infrastructure**:
  - [ ] `VpnKeyHandler` (Telegram handler)
  - [ ] `VpnKeyKeyboard` (teclados)
  - [ ] `VpnEndpoints` (backend API calls)
- [ ] Tests (20+ tests)

### Fase 5: Payments y Referrals (Día 6-7)
- [ ] **Domain**:
  - [ ] `PaymentService` (lógica de pagos)
- [ ] **Application**:
  - [ ] `CreateCryptoPayment` use case
  - [ ] `CreateStarsPayment` use case
  - [ ] `GetReferralCode` use case
  - [ ] `GetReferralStats` use case
- [ ] **Infrastructure**:
  - [ ] `PaymentHandler`, `ReferralHandler`
  - [ ] `PaymentKeyboard`, `ReferralKeyboard`
  - [ ] `PaymentEndpoints`, `ReferralEndpoints`
- [ ] Tests (20+ tests)

### Fase 6: Integración y Cleanup (Día 8)
- [ ] Integrar todos los handlers en `main.py`
- [ ] Configurar Dependency Injection
- [ ] Tests de integración (10+ tests)
- [ ] Documentación de arquitectura
- [ ] Code review y refactor final
- [ ] Merge a main

---

## 📊 Métricas de Éxito

| Métrica | Objetivo | Actual |
|---------|----------|--------|
| **Tests Unitarios** | 100+ tests | 0 |
| **Cobertura** | > 80% | N/A |
| **Acoplamiento** | Bajo (dependencias hacia adentro) | - |
| **Tiempo de Build** | < 5 min | - |
| **Complejidad Ciclomática** | < 10 por función | - |

---

## 🔧 Dependencias Técnicas

```toml
[project]
dependencies = [
    "aiogram>=3.4.0",
    "httpx>=0.25.0",
    "python-dotenv>=1.0.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
    "redis>=5.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
    "pytest-cov>=4.1.0",
    "mypy>=1.5.0",
    "ruff>=0.1.0",
    "types-python-dotenv",
]
```

---

## 📚 Principios de Diseño

### 1. **Dependency Rule**
- Dependencias apuntan hacia adentro (dominio)
- Dominio no conoce infraestructura
- Infraestructura depende de ports (interfaces)

### 2. **Single Responsibility**
- Cada entidad tiene una responsabilidad clara
- Cada use case hace una sola cosa
- Cada adapter tiene un propósito definido

### 3. **Inversión de Dependencias**
- Alta nivel no depende de bajo nivel
- Ambos dependen de abstracciones (ports)

### 4. **Separación de Responsabilidades**
- Dominio: Reglas de negocio puras
- Aplicación: Orquestación de casos de uso
- Infraestructura: Implementaciones concretas

---

## 🎯 Beneficios Esperados

1. **Testabilidad**: Mocks fáciles de ports
2. **Mantenibilidad**: Cambios localizados
3. **Escalabilidad**: Nuevas features sin romper existing
4. **Flexibilidad**: Cambiar infraestructura sin tocar dominio
5. **Claridad**: Código auto-documentado por capas

---

## 📋 Checklist de Verificación

- [ ] Estructura de directorios creada
- [ ] Entities del dominio implementadas
- [ ] Value Objects con validación
- [ ] Ports definidos como interfaces
- [ ] Use Cases implementados
- [ ] Adapters primarios (Telegram)
- [ ] Adapters secundarios (Backend, Redis)
- [ ] Tests unitarios por capa
- [ ] Tests de integración
- [ ] Documentación actualizada
- [ ] CI/CD configurado
- [ ] Merge a main

---

**Próximo Paso:** Ejecutar Fase 1 - Cimientos del Dominio
