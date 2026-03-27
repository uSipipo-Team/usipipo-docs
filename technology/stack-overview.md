# 🛠️ Technology Stack Overview - Visión General de Tecnologías

> Documento completo de todas las tecnologías utilizadas en el ecosistema uSipipo

---

## 📊 Tabla de Contenidos

1. [Arquitectura General](#arquitectura-general)
2. [Backend Stack](#backend-stack)
3. [Frontend Stack](#frontend-stack)
4. [Mobile Stack](#mobile-stack)
5. [Infrastructure Stack](#infrastructure-stack)
6. [DevOps & CI/CD](#devops--cicd)
7. [Monitoring & Observability](#monitoring--observability)
8. [Security Stack](#security-stack)
9. [Base de Datos](#base-de-datos)
10. [VPN Technologies](#vpn-technologies)

---

## 🏗️ Arquitectura General

### **Patrón Arquitectónico: Clean Architecture / Hexagonal**

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTACIÓN                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Landing  │  │   Bot    │  │ MiniApp  │  │  Mobile  │   │
│  │  Flask   │  │  PTB     │  │  Flask   │  │ Flutter  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    APLICACIÓN                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Backend API (FastAPI)                    │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │  │
│  │  │  Domain  │  │Application│  │  Infrastructure  │   │  │
│  │  │ Entities │  │ Services │  │  Repositories    │   │  │
│  │  └──────────┘  └──────────┘  └──────────────────┘   │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    DATOS                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │PostgreSQL│  │  Redis   │  │WireGuard │  │ Outline  │   │
│  │   15+    │  │   7+     │  │          │  │          │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 💻 Backend Stack

### **Framework Principal: FastAPI**

| Característica | Detalle |
|----------------|---------|
| **Framework** | FastAPI 0.109+ |
| **Lenguaje** | Python 3.13+ |
| **Tipo** | Asíncrono (async/await) |
| **Validación** | Pydantic 2.x |
| **Servidor** | Uvicorn + Gunicorn |

**¿Por qué FastAPI?**
- ✅ Alto rendimiento (comparable a Node.js y Go)
- ✅ Type hints nativos con Python
- ✅ Documentación automática (OpenAPI/Swagger)
- ✅ Soporte asíncrono completo
- ✅ Validación de datos integrada

---

### **ORM: SQLAlchemy 2.0**

| Característica | Detalle |
|----------------|---------|
| **ORM** | SQLAlchemy 2.0+ |
| **Modo** | Asíncrono (asyncpg) |
| **Migraciones** | Alembic |
| **Pool** | QueuePool con size 10-50 |

**Configuración:**
```python
# Database URL
postgresql+asyncpg://user:pass@localhost:5432/usipipo

# Pool settings
pool_size=20
max_overflow=30
pool_pre_ping=True
pool_recycle=3600
```

---

### **Caché: Redis**

| Característica | Detalle |
|----------------|---------|
| **Versión** | Redis 7.0+ |
| **Cliente** | redis-py (async) |
| **Usos** | Caché, rate limiting, JWT blacklist, token storage |

**Casos de uso:**
1. **JWT Blacklist** - Tokens revocados (TTL: 30 min)
2. **Bot Tokens** - JWT tokens por telegram_id (TTL: 30 días)
3. **Rate Limiting** - Contadores por IP/user (TTL: 1 min)
4. **Caché de Queries** - Resultados frecuentes (TTL: 5-60 min)

---

### **Package Manager: uv**

| Característica | Detalle |
|----------------|---------|
| **Herramienta** | uv (Astral) |
| **Velocidad** | 10-100x más rápido que pip |
| **Lock file** | uv.lock |
| **Python** | Gestión de versiones incluida |

**Comandos principales:**
```bash
uv sync --dev          # Instalar dependencias
uv run pytest          # Ejecutar tests
uv run python -m src   # Ejecutar aplicación
uv build               # Build para PyPI
```

---

### **Testing Stack**

| Herramienta | Propósito | Versión |
|-------------|-----------|---------|
| **pytest** | Framework de tests | 8.0+ |
| **pytest-asyncio** | Tests asíncronos | 0.23+ |
| **pytest-cov** | Coverage | 4.1+ |
| **httpx** | HTTP client para tests | 0.26+ |
| **factory-boy** | Factories para datos | 3.3+ |

**Cobertura requerida:** 80% mínimo

---

### **Code Quality Tools**

| Herramienta | Propósito | Configuración |
|-------------|-----------|---------------|
| **ruff** | Linting + formato | ruff.toml |
| **mypy** | Type checking | mypy.ini (strict) |
| **bandit** | Security scan | .bandit.yml |
| **pre-commit** | Git hooks | .pre-commit-config.yaml |

---

## 🖥️ Frontend Stack

### **Landing Page: Flask**

| Característica | Detalle |
|----------------|---------|
| **Framework** | Flask 3.1+ |
| **Templates** | Jinja2 |
| **CSS** | Custom (Cyberpunk theme) |
| **Server** | Gunicorn |
| **Proxy** | Caddy reverse proxy |

**Estructura:**
```
src/
├── features/
│   ├── home/
│   ├── pricing/
│   ├── docs/
│   └── status/
├── infrastructure/
│   └── web/
│       ├── app.py
│       ├── templates/
│       └── static/
└── core/
    └── config/
```

---

### **Telegram Bot: python-telegram-bot**

| Característica | Detalle |
|----------------|---------|
| **Framework** | python-telegram-bot 22.7 |
| **HTTP Client** | httpx (async) |
| **Tipo** | Polling + Webhooks |
| **Storage** | Redis para tokens |

**Arquitectura:**
```
src/
├── bot/
│   ├── handlers/      # Command handlers
│   └── keyboards/     # Inline keyboards
├── application/
│   ├── use_cases/     # Business logic
│   └── ports/         # Interfaces
└── infrastructure/
    ├── api_client.py  # HTTP to backend
    └── redis.py       # Token storage
```

---

### **MiniApp Web: Flask**

| Característica | Detalle |
|----------------|---------|
| **Framework** | Flask 3.1+ |
| **Telegram API** | WebApp API |
| **Auth** | Telegram Init Data validation |
| **UI** | Telegram native components + custom |

---

## 📱 Mobile Stack

### **Android App: Flutter**

| Característica | Detalle |
|----------------|---------|
| **Framework** | Flutter 3.16+ |
| **Lenguaje** | Dart 3.11+ |
| **State Mgmt** | Riverpod 2.4+ |
| **Navegación** | Go Router 12.0+ |
| **HTTP** | Dio 5.3+ |

**Arquitectura: Clean Architecture + Riverpod**

```
lib/
├── core/
│   ├── constants/
│   └── theme/
├── domain/
│   ├── entities/
│   └── repositories/
├── data/
│   ├── datasources/
│   ├── repositories/
│   └── models/
└── presentation/
    ├── providers/
    ├── screens/
    └── widgets/
```

---

### **Dependencias Clave Flutter**

```yaml
dependencies:
  flutter_riverpod: ^2.4.0      # State management
  go_router: ^12.0.0            # Navigation
  dio: ^5.3.0                   # HTTP client
  flutter_secure_storage: ^9.0.0 # JWT storage
  firebase_messaging: ^14.7.0   # Push notifications
  flutter_local_notifications: ^16.0.0
  connectivity_plus: ^5.0.0     # Network status
  shared_preferences: ^2.2.0    # Local storage
  google_fonts: ^6.0.0          # Typography
  flutter_svg: ^2.0.0           # SVG support
```

---

### **VPN Platform Channels**

**Native Android (Kotlin):**
```kotlin
// WireGuard
val wgInterface = WgInterface(...)
VpnService().addTunInterface(wgInterface)

// Outline
val ssClient = ShadowsocksClient(config)
ssClient.connect()
```

**Dart Interface:**
```dart
class VpnPlatform {
  static const platform = MethodChannel('com.usipipo.vpn/platform');
  
  Future<bool> connect({
    required String protocol,
    required String config,
  }) async {
    return await platform.invokeMethod('connect', {
      'protocol': protocol,
      'config': config,
    });
  }
}
```

---

## 🏢 Infrastructure Stack

### **Cloud Providers**

| Proveedor | Servicios | Uso |
|-----------|-----------|-----|
| **AWS** | EC2, RDS, ElastiCache | Producción principal |
| **DigitalOcean** | Droplets | VPN servers |
| **Google Cloud** | Firebase (FCM) | Notificaciones push |

---

### **Contenerización**

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| **Docker** | 24.0+ | Contenedores |
| **Docker Compose** | 2.20+ | Desarrollo local |
| **Multi-stage builds** | - | Imágenes mínimas |

**Ejemplo Dockerfile (Backend):**
```dockerfile
# Build stage
FROM python:3.13-slim as builder

WORKDIR /app
RUN pip install uv
COPY pyproject.toml uv.lock ./
RUN uv export --frozen | pip install -r /dev/stdin

# Runtime stage
FROM python:3.13-slim

WORKDIR /app
COPY --from=builder /usr/local/lib/python3.13/site-packages /usr/local/lib/python3.13/site-packages
COPY src/ ./src/

USER nobody
CMD ["python", "-m", "src"]
```

---

### **Orquestación**

| Tecnología | Estado | Propósito |
|------------|--------|-----------|
| **systemd** | ✅ Producción | Servicios en VPS |
| **Kubernetes** | 🟡 Planificado | Escalamiento futuro |
| **Docker Swarm** | ⏳ Evaluando | Orquestación ligera |

---

### **Reverse Proxy: Caddy**

| Característica | Detalle |
|----------------|---------|
| **Proxy** | Caddy 2.7+ |
| **TLS** | Automático (Let's Encrypt) |
| **Config** | Caddyfile |

**Caddyfile:**
```
usipipo.duckdns.org {
    # Landing Page
    @landing { not path /miniapp/* }
    reverse_proxy @landing localhost:5000
    
    # MiniApp
    @miniapp path /miniapp/*
    reverse_proxy @miniapp localhost:5001
    
    # Backend API
    @api path /api/*
    reverse_proxy @api localhost:8000
    
    tls dev@usipipo.com
}
```

---

## 🔧 DevOps & CI/CD

### **GitHub Actions**

**Workflows principales:**

| Workflow | Trigger | Acciones |
|----------|---------|----------|
| **CI** | push, PR | lint, test, type-check, security |
| **CD** | release | build, push Docker, deploy |
| **Dependabot** | schedule | auto-update dependencies |

**Ejemplo CI:**
```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv run ruff check .
      
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
      redis:
        image: redis:7
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv run pytest --cov=src
      
  type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv run mypy .
```

---

### **Pre-commit Hooks**

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
      
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
      
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.7
    hooks:
      - id: bandit
        args: ["-r", "src/"]
```

---

### **Git Workflow**

**Estrategia: Git Flow**

```
main (producción)
  │
  ├── develop (integración)
  │     │
  │     ├── feature/auth
  │     ├── feature/payments
  │     └── feature/vpn
  │
  └── hotfix/urgent-fix
```

**Ramas:**
- `main` - Producción (protegida)
- `develop` - Integración (protegida)
- `feature/*` - Features nuevas
- `hotfix/*` - Fixes urgentes
- `release/*` - Preparación de release

---

## 📊 Monitoring & Observability

### **Stack de Monitoring**

| Herramienta | Propósito | Estado |
|-------------|-----------|--------|
| **Prometheus** | Métricas | 🟡 Planificado |
| **Grafana** | Dashboards | 🟡 Planificado |
| **Jaeger** | Distributed tracing | 🟡 Planificado |
| **Loki** | Logs | 🟡 Planificado |

---

### **Métricas Clave (SLOs)**

| Métrica | Objetivo | Cómo medir |
|---------|----------|------------|
| **Uptime** | 99.9% | Prometheus |
| **Latencia p95** | < 200ms | Jaeger |
| **Error rate** | < 0.1% | Prometheus |
| **VPN success rate** | > 99% | Custom metrics |
| **Payment success rate** | > 95% | Custom metrics |

---

### **Logging**

| Nivel | Formato | Destino |
|-------|---------|---------|
| **Estructurado** | JSON | stdout → Loki |
| **Niveles** | DEBUG, INFO, WARNING, ERROR, CRITICAL | |
| **Correlación** | request_id, user_id, trace_id | |

**Configuración Python:**
```python
import logging
import json

logging.basicConfig(
    level=logging.INFO,
    format='{"timestamp": "%(asctime)s", "level": "%(levelname)s", "message": "%(message)s", "request_id": "%(request_id)s"}'
)
```

---

## 🔐 Security Stack

### **Autenticación**

| Tecnología | Propósito |
|------------|-----------|
| **JWT** | Access + Refresh tokens |
| **PyJWT** | Generación/validación |
| **python-jose** | Alternativa JWT |
| **bcrypt** | Password hashing |

---

### **Seguridad de API**

| Herramienta | Propósito |
|-------------|-----------|
| **SlowAPI** | Rate limiting |
| **FastAPI Security** | OAuth2, JWT bearer |
| **CORS Middleware** | Cross-origin restrictions |
| **HTTPS** | TLS 1.3 (Caddy) |

---

### **Seguridad de Código**

| Herramienta | Propósito |
|-------------|-----------|
| **Bandit** | Security scanning |
| **Safety** | Dependency vulnerabilities |
| **Secretlint** | Secrets detection |
| **GitLeaks** | Git history scanning |

---

## 💾 Base de Datos

### **PostgreSQL**

| Característica | Detalle |
|----------------|---------|
| **Versión** | 15+ |
| **Driver** | asyncpg |
| **ORM** | SQLAlchemy 2.0 |
| **Migraciones** | Alembic |

**Configuración de Producción:**
```
max_connections = 200
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 64MB
maintenance_work_mem = 512MB
```

---

### **Esquema de Base de Datos**

```
┌─────────────────────────────────────────┐
│              Tablas (15)                │
├─────────────────────────────────────────┤
│ users                                   │
│ vpn_keys                                │
│ payments                                │
│ data_packages                           │
│ subscription_plans                      │
│ subscription_transactions               │
│ consumption_billings                    │
│ consumption_invoices                    │
│ tickets                                 │
│ ticket_messages                         │
│ referrals                               │
│ wallet_pools                            │
│ wallets                                 │
│ devices                                 │
│ auth_providers                          │
└─────────────────────────────────────────┘
```

---

### **Índices Principales**

```sql
-- Users
CREATE INDEX idx_users_telegram_id ON users(telegram_id);
CREATE INDEX idx_users_referral_code ON users(referral_code);

-- VPN Keys
CREATE INDEX idx_vpn_keys_user_id ON vpn_keys(user_id);
CREATE INDEX idx_vpn_keys_status ON vpn_keys(status);

-- Payments
CREATE INDEX idx_payments_user_id ON payments(user_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_created_at ON payments(created_at);

-- Tickets
CREATE INDEX idx_tickets_user_id ON tickets(user_id);
CREATE INDEX idx_tickets_status ON tickets(status);
```

---

## 🔌 VPN Technologies

### **WireGuard**

| Característica | Detalle |
|----------------|---------|
| **Tipo** | VPN moderno de alto rendimiento |
| **Protocolo** | UDP |
| **Cifrado** | ChaCha20, Poly1305, Curve25519 |
| **Puerto** | 51820/udp |
| **Comandos** | wg, wg-quick |

**Configuración del Servidor:**
```ini
[Interface]
PrivateKey = <server_private_key>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.0.0.2/32
```

---

### **Outline (Shadowsocks)**

| Característica | Detalle |
|----------------|---------|
| **Tipo** | Proxy SOCKS5 cifrado |
| **Cifrado** | AES-256-GCM |
| **API** | REST (mTLS) |
| **Desarrollador** | Jigsaw (Google) |

**API Endpoints:**
```
POST /access-keys          # Crear key
GET  /access-keys          # Listar keys
DELETE /access-keys/:id    # Eliminar key
GET  /metrics/transfer     # Métricas
```

**URL de acceso:**
```
ss://YWVzLTI1Ni1nY206cGFzc3dvcmQ@hostname:port/#name
```

---

## 📈 Comparativa de Tecnologías

### **Backend Frameworks**

| Framework | Rendimiento | Curva | Ecosistema | Decisión |
|-----------|-------------|-------|------------|----------|
| **FastAPI** | ⭐⭐⭐⭐⭐ | Baja | Grande | ✅ Elegido |
| Django | ⭐⭐⭐ | Media | Muy grande | ❌ Muy pesado |
| Flask | ⭐⭐⭐ | Baja | Grande | ❌ Menos features |
| Sanic | ⭐⭐⭐⭐ | Media | Pequeño | ❌ Menos maduro |

---

### **State Management (Flutter)**

| Librería | Complejidad | Performance | Decisión |
|----------|-------------|-------------|----------|
| **Riverpod** | Media | ⭐⭐⭐⭐⭐ | ✅ Elegido |
| Provider | Baja | ⭐⭐⭐ | ❌ Limitado |
| Bloc | Alta | ⭐⭐⭐⭐ | ❌ Verboso |
| GetX | Baja | ⭐⭐⭐ | ❌ Controversial |

---

## 🚀 Roadmap Tecnológico

### **Q2 2026**
- [ ] Implementar Prometheus + Grafana
- [ ] Implementar Jaeger tracing
- [ ] Migrar a Kubernetes (evaluación)

### **Q3 2026**
- [ ] iOS app (Swift/SwiftUI)
- [ ] Multi-region deployment
- [ ] Auto-scaling

### **Q4 2026**
- [ ] SOC 2 Type I compliance
- [ ] Advanced threat detection
- [ ] Machine learning para fraud detection

---

## 📚 Recursos Relacionados

- [Backend Stack](backend-stack.md)
- [Frontend Stack](frontend-stack.md)
- [Infrastructure Stack](infrastructure-stack.md)

---

**Última actualización:** 2026-03-27
