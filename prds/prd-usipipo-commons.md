# 📋 PRD - uSipipo Commons

> Product Requirements Document de la Librería Compartida

---

## 📊 Información del Producto

| Campo | Valor |
|-------|-------|
| **Nombre** | uSipipo Commons |
| **Repositorio** | https://github.com/uSipipo-Team/usipipo-commons |
| **PyPI** | https://pypi.org/project/usipipo-commons/ |
| **Estado** | ✅ Producción |
| **Versión Actual** | v0.12.0 |
| **Tech Stack** | Python 3.13+, Pydantic 2.x |

---

## 🎯 Propósito

Librería compartida que centraliza entidades de dominio, enums, constantes y utilidades usadas en todos los proyectos del ecosistema uSipipo.

---

## 📦 Estructura

```
usipipo_commons/
├── domain/
│   ├── entities/        # User, VpnKey, Payment, etc.
│   └── enums/           # KeyType, PaymentStatus, etc.
├── schemas/             # Request/Response schemas
├── constants/           # FREE_GB, PRICE_PER_GB, etc.
└── utils/              # validate_telegram_id, format_bytes
```

---

## 🔑 Entidades Principales

### **User**
- telegram_id, username, balance_gb, referral_code
- referral_credits, purchase_count, loyalty_bonus_percent

### **VpnKey**
- key_type (wireguard/outline), status, key_data
- data_limit_bytes, used_bytes, expires_at

### **Payment**
- amount_usd, method, status, transaction_hash
- crypto_address, telegram_star_invoice_id

### **SubscriptionPlan**
- plan_type, stars_paid, expires_at, is_active

### **ConsumptionBilling**
- mb_consumed, total_cost_usd, status, price_per_mb

### **DataPackage**
- package_type, data_limit_bytes, stars_paid, expires_at

### **Ticket**
- category, priority, status, admin_notes

---

## 🔢 Enums

| Enum | Valores |
|------|---------|
| **KeyType** | OUTLINE, WIREGUARD |
| **KeyStatus** | ACTIVE, EXPIRED, REVOKED, PENDING |
| **PaymentMethod** | CRYPTO_USDT, CRYPTO_USDC, TELEGRAM_STARS |
| **PaymentStatus** | PENDING, COMPLETED, FAILED, EXPIRED, REFUNDED |
| **PlanType** | FREE, ONE_MONTH, THREE_MONTHS, SIX_MONTHS, CONSUMPTION |
| **PackageType** | BASIC, ESTANDAR, AVANZADO, PREMIUM, UNLIMITED |
| **TicketCategory** | VPN_FAIL, PAYMENT, ACCOUNT, OTHER |
| **TicketPriority** | HIGH, MEDIUM, LOW |
| **TicketStatus** | OPEN, RESPONDED, RESOLVED, CLOSED |

---

## 🔢 Constantes

```python
FREE_GB = 5.0
REFERRAL_BONUS_GB = 5.0
PRICE_PER_GB = 0.50  # USD

PLAN_PRICES = {
    "one_month": 7.20,      # 360 Stars
    "three_months": 19.20,  # 960 Stars
    "six_months": 31.20,    # 1560 Stars
}

PACKAGE_PRICES = {
    "basic": 2.50,      # 5 GB
    "estandar": 7.50,   # 15 GB
    "avanzado": 15.00,  # 30 GB
    "premium": 25.00,   # 50 GB
    "unlimited": 50.00, # ∞
}

MAX_VPN_KEYS_PER_USER = 5
VPN_KEY_DATA_LIMIT_DEFAULT_GB = 5
CONSUMPTION_BILLING_CYCLE_DAYS = 30
```

---

## 🛠️ Utilidades

```python
# Validación
validate_telegram_id(telegram_id: int) -> bool
validate_wallet_address(address: str, token: str) -> bool

# Formateo
format_bytes(bytes_value: int) -> str  # "5.50 GB"
format_datetime(dt: datetime) -> str   # "2026-03-27 10:30:00 UTC"
format_currency(amount: float, currency: str) -> str  # "$5.00 USD"

# Conversión
gb_to_bytes(gb: float) -> int
mb_to_bytes(mb: float) -> int
bytes_to_gb(bytes_value: int) -> float
```

---

## 📈 Versiones

| Versión | Fecha | Cambios |
|---------|-------|---------|
| v0.12.0 | 2026-03-22 | Consumption billing entities |
| v0.11.0 | 2026-03-20 | Data package entities |
| v0.10.0 | 2026-03-18 | Subscription entities |
| v0.9.0 | 2026-03-15 | Payment entities completas |
| v0.1.0 | 2026-03-01 | Release inicial |

---

## 🔗 Proyectos que lo Usan

- usipipo-backend
- usipipo-telegram-bot
- usipipo-miniapp-web
- usipipo-landing

---

**Última actualización:** 2026-03-27
