# 📋 PRD - uSipipo Backend

> Product Requirements Document del Backend API

---

## 📊 Información del Producto

| Campo | Valor |
|-------|-------|
| **Nombre** | uSipipo Backend API |
| **Repositorio** | https://github.com/uSipipo-Team/usipipo-backend |
| **Estado** | ✅ Producción |
| **Versión Actual** | v0.10.0 |
| **Tech Stack** | FastAPI, PostgreSQL, Redis, SQLAlchemy |
| **Team** | Backend Team |
| **Última Actualización** | 2026-03-24 |

---

## 🎯 Visión General

El backend de uSipipo es una API REST construida con FastAPI que sirve como el core del ecosistema VPN. Proporciona autenticación, gestión de VPN keys, procesamiento de pagos, facturación por consumo, y administración de usuarios.

---

## 🏗️ Arquitectura

### **Patrón: Clean Architecture / Hexagonal**

```
src/
├── core/
│   ├── domain/           # Entidades y reglas de negocio
│   ├── application/      # 29 casos de uso
│   └── ports/           # Interfaces (repositories, services)
│
├── infrastructure/
│   ├── api/             # FastAPI routes, webhooks
│   ├── api_clients/     # HTTP clients externos
│   ├── jobs/            # Background jobs
│   ├── payment_gateways/ # TronDealer, Telegram Stars
│   ├── persistence/     # SQLAlchemy models, repositorios
│   └── vpn_providers/   # WireGuard, Outline
│
└── shared/
    ├── config/          # Settings (pydantic-settings)
    ├── schemas/         # Pydantic schemas
    ├── security/        # JWT, hashing
    └── logger.py        # Logging estructurado
```

---

## 🔌 Funcionalidades Principales

### **1. Autenticación y Autorización**

**Features:**
- JWT tokens (access 30min + refresh 30d)
- Telegram WebApp signature validation
- Auto-registro de usuarios
- Token refresh automático
- JWT blacklist en Redis
- Rate limiting (5/min auth, 30/min admin)

**Endpoints:**
```
POST   /api/v1/auth/telegram/auto-register
POST   /api/v1/auth/telegram/validate
POST   /api/v1/auth/manual/request
POST   /api/v1/auth/manual/validate
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
```

---

### **2. Gestión de Usuarios**

**Features:**
- Perfil de usuario (CRUD)
- Balance de datos (GB)
- Códigos de referido
- Historial de compras
- Estadísticas de uso

**Endpoints:**
```
GET    /api/v1/users/me
PUT    /api/v1/users/me
GET    /api/v1/users/{id}
GET    /api/v1/users/{id}/stats
```

**Entidades:**
- User (telegram_id, username, balance_gb, referral_code)
- AuthProvider (telegram, email)

---

### **3. VPN Keys**

**Features:**
- Generación de keys WireGuard (wg-quick)
- Generación de keys Outline (Shadowsocks)
- Activación/desactivación
- Reset de ciclo de billing
- Métricas de uso (bytes, last_seen)
- Límite de 5 keys por usuario

**Endpoints:**
```
GET    /api/v1/vpn/keys
POST   /api/v1/vpn/keys
GET    /api/v1/vpn/keys/{id}
PUT    /api/v1/vpn/keys/{id}
DELETE /api/v1/vpn/keys/{id}
GET    /api/v1/vpn/keys/{id}/metrics
POST   /api/v1/vpn/keys/{id}/reset
```

**Entidades:**
- VpnKey (key_type, key_data, data_limit_bytes, expires_at)

---

### **4. Pagos**

**Features:**
- Pagos con crypto (USDT/USDC vía TronDealer)
- Pagos con Telegram Stars
- Historial de pagos
- Reembolsos (parcial)
- Webhooks de confirmación

**Endpoints:**
```
GET    /api/v1/payments
POST   /api/v1/payments/crypto
POST   /api/v1/payments/stars
GET    /api/v1/payments/{id}
POST   /api/v1/payments/{id}/refund
```

**Entidades:**
- Payment (amount_usd, method, status, transaction_hash)
- CryptoOrder (tron_dealer_order_id, wallet_address)
- CryptoTransaction (tx_hash, confirmations)

---

### **5. Suscripciones**

**Features:**
- Planes: 1 mes, 3 meses, 6 meses
- Activación automática
- Renovación recurrente
- Notificación de expiración
- Historial de transacciones

**Endpoints:**
```
GET    /api/v1/subscriptions/plans
GET    /api/v1/subscriptions/me
POST   /api/v1/subscriptions/activate
GET    /api/v1/subscriptions/{id}
```

**Entidades:**
- SubscriptionPlan (plan_type, stars_paid, expires_at)
- SubscriptionTransaction (transaction_id, status)

---

### **6. Facturación por Consumo**

**Features:**
- Tracking de consumo en tiempo real
- Ciclos de 30 días
- Invoices automáticas
- Pago con crypto o Stars
- Historial de facturación

**Endpoints:**
```
GET    /api/v1/billing/cycles
GET    /api/v1/billing/cycles/{id}
GET    /api/v1/billing/cycles/{id}/invoice
POST   /api/v1/billing/cycles/{id}/pay
```

**Entidades:**
- ConsumptionBilling (mb_consumed, total_cost_usd, status)
- ConsumptionInvoice (amount_usd, wallet_address, status)

---

### **7. Paquetes de Datos**

**Features:**
- Compra de data adicional
- Paquetes: 5, 15, 30, 50, unlimited GB
- Vencimiento a 30 días
- Acumulable con suscripción

**Endpoints:**
```
GET    /api/v1/data-packages/options
POST   /api/v1/data-packages/purchase
GET    /api/v1/data-packages/me
```

**Entidades:**
- DataPackage (package_type, data_limit_bytes, expires_at)

---

### **8. Wallets y Balance**

**Features:**
- Wallets múltiples por usuario
- Pool de saldo compartido
- Depósitos y retiros
- Transferencias internas

**Endpoints:**
```
GET    /api/v1/wallets
POST   /api/v1/wallets
GET    /api/v1/wallets/{id}
POST   /api/v1/wallets/{id}/deposit
POST   /api/v1/wallets/{id}/withdraw
GET    /api/v1/wallets/pool
```

**Entidades:**
- Wallet (user_id, balance_stars, type)
- WalletPool (shared_balance)

---

### **9. Tickets de Soporte**

**Features:**
- Creación de tickets
- Categorías: VPN_FAIL, PAYMENT, ACCOUNT, OTHER
- Prioridades: HIGH, MEDIUM, LOW
- Mensajes entre usuario y admin
- Resolución y cierre

**Endpoints:**
```
GET    /api/v1/tickets
POST   /api/v1/tickets
GET    /api/v1/tickets/{id}
POST   /api/v1/tickets/{id}/messages
PUT    /api/v1/tickets/{id}/status
```

**Entidades:**
- Ticket (category, priority, status, admin_notes)
- TicketMessage (from_user_id, from_admin, message)

---

### **10. Admin Panel**

**Features:**
- Listado de usuarios
- Búsqueda y filtros
- Gestión de VPN keys
- Estadísticas del sistema
- Logs de operaciones

**Endpoints:**
```
GET    /api/v1/admin/users
GET    /api/v1/admin/users/{id}
GET    /api/v1/admin/keys
GET    /api/v1/admin/stats
POST   /api/v1/admin/keys/{id}/revoke
```

**Entidades:**
- AdminUserInfo (total_keys, active_keys, stars_balance)
- AdminKeyInfo (key_type, access_url, is_active)
- ServerStatus (is_healthy, total_keys, version)

---

## 📦 Modelo de Datos

### **Tablas Principales (15)**

```sql
-- Usuarios
users (
  id UUID PRIMARY KEY,
  telegram_id BIGINT UNIQUE,
  username VARCHAR,
  balance_gb DECIMAL,
  referral_code VARCHAR UNIQUE,
  created_at TIMESTAMP
)

-- VPN Keys
vpn_keys (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  key_type VARCHAR, -- 'wireguard' | 'outline'
  status VARCHAR,   -- 'active' | 'expired' | 'revoked'
  key_data TEXT,
  data_limit_bytes BIGINT,
  used_bytes BIGINT,
  expires_at TIMESTAMP
)

-- Pagos
payments (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  amount_usd DECIMAL,
  method VARCHAR, -- 'crypto_usdt' | 'crypto_usdc' | 'telegram_stars'
  status VARCHAR, -- 'pending' | 'completed' | 'failed'
  transaction_hash VARCHAR
)

-- Suscripciones
subscription_plans (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  plan_type VARCHAR, -- 'one_month' | 'three_months' | 'six_months'
  stars_paid INTEGER,
  expires_at TIMESTAMP
)

-- Facturación por consumo
consumption_billings (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  mb_consumed DECIMAL,
  total_cost_usd DECIMAL,
  status VARCHAR,    -- 'active' | 'closed' | 'paid'
  started_at TIMESTAMP
)

-- Invoices de consumo
consumption_invoices (
  id UUID PRIMARY KEY,
  billing_id UUID REFERENCES consumption_billings(id),
  amount_usd DECIMAL,
  wallet_address VARCHAR,
  status VARCHAR,    -- 'pending' | 'paid' | 'expired'
  expires_at TIMESTAMP
)

-- Paquetes de datos
data_packages (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  package_type VARCHAR, -- 'basic' | 'estandar' | 'avanzado' | 'premium' | 'unlimited'
  data_limit_bytes BIGINT,
  expires_at TIMESTAMP
)

-- Tickets
tickets (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  category VARCHAR,  -- 'vpn_fail' | 'payment' | 'account' | 'other'
  priority VARCHAR,  -- 'high' | 'medium' | 'low'
  status VARCHAR,    -- 'open' | 'responded' | 'resolved' | 'closed'
  created_at TIMESTAMP
)

-- Mensajes de tickets
ticket_messages (
  id UUID PRIMARY KEY,
  ticket_id UUID REFERENCES tickets(id),
  from_user_id BIGINT,
  from_admin BOOLEAN,
  message TEXT,
  created_at TIMESTAMP
)

-- Wallets
wallets (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  balance_stars INTEGER,
  type VARCHAR       -- 'main' | 'savings'
)

-- Wallet pools
wallet_pools (
  id UUID PRIMARY KEY,
  name VARCHAR,
  total_balance INTEGER
)

-- Dispositivos (FCM)
devices (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  fcm_token VARCHAR,
  platform VARCHAR,  -- 'android' | 'ios' | 'telegram'
  created_at TIMESTAMP
)

-- Proveedores de auth
auth_providers (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  provider VARCHAR,  -- 'telegram' | 'email'
  provider_id VARCHAR
)

-- Referidos
referrals (
  id UUID PRIMARY KEY,
  referrer_id UUID REFERENCES users(id),
  referred_id UUID REFERENCES users(id),
  credits_earned INTEGER,
  created_at TIMESTAMP
)

-- Transacciones de suscripción
subscription_transactions (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  transaction_id VARCHAR UNIQUE,
  plan_type VARCHAR,
  amount_stars INTEGER,
  status VARCHAR,    -- 'pending' | 'completed' | 'failed'
  created_at TIMESTAMP
)
```

---

## 🔐 Seguridad

### **Autenticación**

**JWT Configuration:**
```python
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
REFRESH_TOKEN_EXPIRE_DAYS = 30
```

**Flujo:**
1. Usuario envía Telegram WebApp Init Data
2. Backend valida signature con bot token
3. Auto-registra usuario si no existe
4. Genera access token + refresh token
5. Client usa access token para requests
6. Auto-refresh 5 min antes de expiración

---

### **Autorización**

**Roles:**
- `user` - Permisos básicos
- `admin` - Acceso completo

**Decoradores:**
```python
@require_auth()              # Cualquier usuario autenticado
@require_role("admin")       # Solo admins
@require_ownership("user_id") # Solo dueño del recurso
```

---

### **Rate Limiting**

**Configuración (SlowAPI):**
```python
# Auth endpoints
5 requests / minute

# Admin endpoints
30 requests / minute

# Default
60 requests / minute

# Webhooks
100 requests / minute
```

---

### **Seguridad de Webhooks**

**TronDealer:**
- HMAC SHA-256 signature
- Token rotativo (WebhookToken entity)
- Validación de timestamp (< 5 min)

**Telegram:**
- Validación de Init Data
- Hash verification con bot token

---

## 🔌 Integraciones Externas

### **1. TronDealer (Pagos Crypto)**

**API:** `https://api.trondealer.com/v2`

**Endpoints usados:**
```
POST /order/create   - Crear orden de pago
GET  /order/status   - Verificar estado
GET  /wallet/balance - Verificar recepción
```

**Redes soportadas:**
- BSC (BEP20)
- Ethereum (ERC20)
- Polygon
- TRON (TRC20)

**Webhook:**
```
POST /webhook/trondealer
Headers: X-Webhook-Signature: <hmac-sha256>
Body: {
  "order_id": "...",
  "amount": 10.0,
  "token": "USDT",
  "network": "BSC",
  "tx_hash": "0x..."
}
```

---

### **2. Telegram**

**Bot API:**
- `sendMessage` - Notificaciones
- `sendInvoice` - Telegram Stars
- `answerCallbackQuery` - Interacciones

**WebApp:**
- Validación de Init Data
- Deep linking para auth

---

### **3. WireGuard**

**Comandos CLI:**
```bash
wg genkey                    # Generar private key
wg pubkey                    # Generar public key
wg-quick up wg0              # Levantar interfaz
wg show wg0                  # Ver métricas
```

**Configuración:**
```ini
[Interface]
PrivateKey = <server_private_key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.0.0.2/32
```

---

### **4. Outline (Shadowsocks)**

**API REST:**
```
POST /access-keys          # Crear key
GET  /access-keys          # Listar keys
DELETE /access-keys/:id    # Eliminar key
GET  /metrics/transfer     # Métricas de uso
```

**Autenticación:**
- mTLS (certificados cliente)
- SSL pinning

**Formato de URL:**
```
ss://YWVzLTI1Ni1nY206cGFzc3dvcmQ@hostname:port#name
```

---

## 📊 Métricas y Monitoring

### **Métricas Clave**

| Métrica | Objetivo | Cómo medir |
|---------|----------|------------|
| **Uptime** | 99.9% | Prometheus |
| **Latencia p95** | < 200ms | Jaeger |
| **Error rate** | < 0.1% | Logs |
| **Requests/seg** | 100-500 | Prometheus |
| **VPN success rate** | > 99% | Connection logs |
| **Payment success rate** | > 95% | Payment logs |

---

### **Endpoints de Health**

```
GET /health              # Health check básico
GET /health/ready        # Ready check (dependencias)
GET /health/live         # Liveness check
GET /metrics             # Prometheus metrics
```

**Response:**
```json
{
  "status": "healthy",
  "version": "0.10.0",
  "timestamp": "2026-03-27T10:30:00Z",
  "checks": {
    "database": "ok",
    "redis": "ok",
    "vpn_provider": "ok"
  }
}
```

---

## 🚀 Deployment

### **Requisitos**

**Hardware:**
- CPU: 2 cores mínimo, 4 cores recomendado
- RAM: 2 GB mínimo, 4 GB recomendado
- Disco: 20 GB SSD

**Software:**
- Python 3.13+
- PostgreSQL 15+
- Redis 7.0+
- Docker 24.0+ (opcional)

---

### **Variables de Entorno**

```bash
# App
APP_ENV=production
DEBUG=false
LOG_LEVEL=INFO

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/usipipo_prod

# Redis
REDIS_URL=redis://localhost:6379

# Security
SECRET_KEY=<generar con secrets.token_urlsafe(32)>
JWT_ALGORITHM=HS256

# Telegram
TELEGRAM_BOT_TOKEN=<bot_token>

# TronDealer
TRONDEALER_API_KEY=<api_key>
TRONDEALER_WEBHOOK_SECRET=<secret>

# VPN
WIREGUARD_INTERFACE=wg0
OUTLINE_API_URL=https://outline-server:8080
OUTLINE_CERT_PATH=/path/to/cert.pem
```

---

### **systemd Service**

```ini
[Unit]
Description=uSipipo Backend API
After=network.target postgresql.service redis.service

[Service]
Type=notify
User=usipipo
WorkingDirectory=/opt/usipipo/usipipo-backend
EnvironmentFile=/opt/usipipo/.env
ExecStart=/opt/usipipo/.venv/bin/python -m src
Restart=always
RestartSec=10

# Security
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

---

## 🧪 Testing

### **Estrategia de Tests**

**Cobertura requerida:** 80% mínimo

**Tipos de tests:**
```
tests/
├── unit/                 # Tests unitarios
│   ├── application/      # Use cases
│   ├── domain/           # Entidades
│   └── infrastructure/   # Adaptadores
│
├── integration/          # Tests de integración
│   ├── test_api.py       # Endpoints
│   ├── test_database.py  # Repositorios
│   └── test_webhooks.py  # Webhooks
│
└── functional/           # Tests funcionales
    ├── test_auth_flow.py
    ├── test_payment_flow.py
    └── test_vpn_key_lifecycle.py
```

---

### **Comandos de Test**

```bash
# Todos los tests
uv run pytest

# Con coverage
uv run pytest --cov=src --cov-report=html

# Tests específicos
uv run pytest tests/unit/test_auth.py -v

# Tests de integración
uv run pytest tests/integration/ -v -k "not slow"
```

---

## 📈 Roadmap

### **v0.11.0 (2026-04)**
- [ ] iOS push notifications
- [ ] Multi-language support (i18n)
- [ ] Advanced analytics

### **v0.12.0 (2026-05)**
- [ ] WireGuard iOS SDK integration
- [ ] Batch operations para admin
- [ ] Audit logging

### **v1.0.0 (2026-06)**
- [ ] SOC 2 compliance
- [ ] Multi-region deployment
- [ ] Auto-scaling

---

## 📚 Recursos Relacionados

- [API Reference](../apis/backend-api-reference.md)
- [Ecosystem Architecture](../context/ecosystem-architecture.md)
- [Technology Stack](../technology/backend-stack.md)

---

**Última actualización:** 2026-03-27
