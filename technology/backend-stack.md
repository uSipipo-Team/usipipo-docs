# 💻 Backend Stack - Tecnología del Backend

> Documento detallado de la arquitectura y tecnologías del backend uSipipo

---

## 📊 Información General

| Campo | Valor |
|-------|-------|
| **Framework** | FastAPI 0.109+ |
| **Lenguaje** | Python 3.13+ |
| **Base de Datos** | PostgreSQL 15+ |
| **Caché** | Redis 7.0+ |
| **ORM** | SQLAlchemy 2.0 (async) |
| **Migraciones** | Alembic |
| **Package Manager** | uv |

---

## 🏗️ Arquitectura del Backend

### **Clean Architecture / Hexagonal**

```
src/
├── core/
│   ├── domain/
│   │   ├── entities/       # Entidades de negocio puras
│   │   ├── enums/          # Enumeraciones del dominio
│   │   ├── exceptions/     # Excepciones de dominio
│   │   └── interfaces/     # Interfaces del dominio
│   │
│   ├── application/
│   │   ├── services/       # Casos de uso (29 servicios)
│   │   ├── ports/          # Puertos (interfaces)
│   │   │   ├── repositories/
│   │   │   ├── vpn_providers/
│   │   │   └── payment_gateways/
│   │   └── dtos/           # Data Transfer Objects
│   │
│   └── ports/
│       └── interfaces/     # Interfaces compartidas
│
├── infrastructure/
│   ├── api/
│   │   ├── routes/         # Endpoints FastAPI
│   │   ├── deps/           # Dependencias (inyección)
│   │   └── webhooks/       # Handlers de webhooks
│   │
│   ├── api_clients/
│   │   ├── trondealer.py   # Cliente HTTP TronDealer
│   │   ├── telegram.py     # Cliente Telegram Bot API
│   │   └── outline.py      # Cliente Outline API
│   │
│   ├── jobs/
│   │   ├── billing_cycle.py    # Cierre de ciclos
│   │   ├── payment_checker.py  # Verificación de pagos
│   │   └── key_expirer.py      # Expiración de keys
│   │
│   ├── payment_gateways/
│   │   ├── trondealer_gateway.py
│   │   └── telegram_stars_gateway.py
│   │
│   ├── persistence/
│   │   ├── models/         # Modelos SQLAlchemy
│   │   ├── repositories/   # Implementación de repositorios
│   │   └── database.py     # Conexión DB
│   │
│   ├── vpn_providers/
│   │   ├── wireguard_provider.py
│   │   └── outline_provider.py
│   │
│   └── config/
│       ├── settings.py     # Configuración (pydantic-settings)
│       └── constants.py    # Constantes
│
└── shared/
    ├── schemas/            # Pydantic schemas
    ├── security/           # JWT, hashing, crypto
    ├── middleware/         # Middleware (CORS, rate limit)
    └── logger.py           # Logging configurado
```

---

## 🔌 FastAPI - Framework Principal

### **Configuración de la Aplicación**

```python
# src/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from slowapi import SlowAPI
from contextlib import asynccontextmanager

from src.core.config import settings
from src.infrastructure.api.routes import api_router
from src.infrastructure.database import init_db

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await init_db()
    yield
    # Shutdown
    await app.state.redis.close()

app = FastAPI(
    title="uSipipo API",
    description="VPN Key Management API",
    version="0.10.0",
    docs_url="/docs" if settings.DEBUG else None,
    redoc_url="/redoc" if settings.DEBUG else None,
    lifespan=lifespan,
)

# Rate limiting
app.state.limiter = SlowAPI()
app.add_exception_handler(RateLimitExceeded, rate_limit_exceeded_handler)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Router
app.include_router(api_router, prefix="/api/v1")
```

---

### **Estructura de Rutas**

```python
# src/infrastructure/api/routes.py
from fastapi import APIRouter
from .routes import auth, users, vpn_keys, payments, subscriptions, billing, tickets, admin

api_router = APIRouter()

api_router.include_router(auth.router, prefix="/auth", tags=["Authentication"])
api_router.include_router(users.router, prefix="/users", tags=["Users"])
api_router.include_router(vpn_keys.router, prefix="/vpn/keys", tags=["VPN Keys"])
api_router.include_router(payments.router, prefix="/payments", tags=["Payments"])
api_router.include_router(subscriptions.router, prefix="/subscriptions", tags=["Subscriptions"])
api_router.include_router(billing.router, prefix="/billing", tags=["Billing"])
api_router.include_router(tickets.router, prefix="/tickets", tags=["Support"])
api_router.include_router(admin.router, prefix="/admin", tags=["Admin"])
```

---

### **Endpoints por Módulo**

| Módulo | Endpoints | Descripción |
|--------|-----------|-------------|
| **auth** | 6 | Autenticación JWT, Telegram, refresh |
| **users** | 4 | CRUD usuarios, perfil, estadísticas |
| **vpn_keys** | 7 | Generación, gestión, métricas de keys |
| **payments** | 6 | Pagos crypto, Stars, historial, reembolsos |
| **subscriptions** | 5 | Planes, activación, estado |
| **billing** | 5 | Ciclos de consumo, invoices |
| **tickets** | 6 | Soporte, mensajes, resolución |
| **admin** | 8 | Panel administrativo, estadísticas |

---

## 💾 SQLAlchemy 2.0 - ORM

### **Configuración Asíncrona**

```python
# src/infrastructure/persistence/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class Database:
    def __init__(self):
        self.engine = None
        self.async_session_maker = None
    
    async def connect(self, url: str):
        self.engine = create_async_engine(
            url,
            echo=settings.DEBUG,
            pool_size=20,
            max_overflow=30,
            pool_pre_ping=True,
            pool_recycle=3600,
        )
        self.async_session_maker = async_sessionmaker(
            self.engine,
            class_=AsyncSession,
            expire_on_commit=False,
        )
    
    async def disconnect(self):
        if self.engine:
            await self.engine.dispose()
    
    async def get_session(self) -> AsyncSession:
        async with self.async_session_maker() as session:
            try:
                yield session
                await session.commit()
            except Exception:
                await session.rollback()
                raise
            finally:
                await session.close()

database = Database()
```

---

### **Modelo de Usuario**

```python
# src/infrastructure/persistence/models/user.py
from sqlalchemy import Column, String, BigInteger, Boolean, Numeric, DateTime, ForeignKey
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from uuid import uuid4

from src.infrastructure.persistence.database import Base

class User(Base):
    __tablename__ = "users"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    telegram_id = Column(BigInteger, unique=True, nullable=False, index=True)
    username = Column(String(255), nullable=True)
    first_name = Column(String(255), nullable=True)
    last_name = Column(String(255), nullable=True)
    is_admin = Column(Boolean, default=False)
    
    # Balance y datos
    balance_gb = Column(Numeric(10, 2), default=5.0)
    total_purchased_gb = Column(Numeric(10, 2), default=0.0)
    
    # Referidos
    referral_code = Column(String(50), unique=True, nullable=False, index=True)
    referred_by = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=True)
    referral_credits = Column(BigInteger, default=0)
    
    # Estadísticas
    purchase_count = Column(BigInteger, default=0)
    loyalty_bonus_percent = Column(BigInteger, default=0)
    welcome_bonus_used = Column(Boolean, default=False)
    
    # Timestamps
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
    
    # Relaciones
    vpn_keys = relationship("VpnKey", back_populates="user", cascade="all, delete-orphan")
    payments = relationship("Payment", back_populates="user")
    subscriptions = relationship("SubscriptionPlan", back_populates="user")
    tickets = relationship("Ticket", back_populates="user")
    
    def __repr__(self) -> str:
        return f"<User(id={self.id}, telegram_id={self.telegram_id})>"
```

---

### **Modelo de VPN Key**

```python
# src/infrastructure/persistence/models/vpn_key.py
from sqlalchemy import Column, String, BigInteger, DateTime, ForeignKey, Enum as SQLEnum
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from uuid import uuid4
import enum

from src.infrastructure.persistence.database import Base

class KeyType(str, enum.Enum):
    WIREGUARD = "wireguard"
    OUTLINE = "outline"

class KeyStatus(str, enum.Enum):
    ACTIVE = "active"
    EXPIRED = "expired"
    REVOKED = "revoked"
    PENDING = "pending"

class VpnKey(Base):
    __tablename__ = "vpn_keys"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False, index=True)
    
    # Tipo y estado
    key_type = Column(SQLEnum(KeyType), nullable=False)
    status = Column(SQLEnum(KeyStatus), default=KeyStatus.ACTIVE, index=True)
    name = Column(String(255), nullable=False)
    
    # Datos de conexión
    key_data = Column(String, nullable=False)  # WireGuard config o ss:// URL
    external_id = Column(String(255), nullable=True)  # ID en proveedor VPN
    
    # Límites y uso
    data_limit_bytes = Column(BigInteger, nullable=False, default=5 * 1024**3)
    used_bytes = Column(BigInteger, default=0)
    last_seen_at = Column(DateTime(timezone=True), nullable=True)
    
    # Timestamps
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    billing_reset_at = Column(DateTime(timezone=True), nullable=False)
    expires_at = Column(DateTime(timezone=True), nullable=True)
    
    # Relaciones
    user = relationship("User", back_populates="vpn_keys")
    
    # Propiedades computadas
    @property
    def is_active(self) -> bool:
        return self.status == KeyStatus.ACTIVE
    
    @property
    def used_gb(self) -> float:
        return self.used_bytes / (1024 ** 3)
    
    @property
    def data_limit_gb(self) -> float:
        return self.data_limit_bytes / (1024 ** 3)
    
    @property
    def remaining_bytes(self) -> int:
        return max(0, self.data_limit_bytes - self.used_bytes)
    
    @property
    def is_over_limit(self) -> bool:
        return self.used_bytes >= self.data_limit_bytes
    
    def add_usage(self, bytes_used: int):
        self.used_bytes += bytes_used
    
    def needs_reset(self) -> bool:
        """Verifica si han pasado 30 días desde el último reset"""
        from datetime import datetime, timezone, timedelta
        return datetime.now(timezone.utc) > self.billing_reset_at + timedelta(days=30)
    
    def reset_billing_cycle(self):
        """Resetea el ciclo de billing"""
        from datetime import datetime, timezone, timedelta
        self.used_bytes = 0
        self.billing_reset_at = datetime.now(timezone.utc) + timedelta(days=30)
```

---

## 🔀 Alembic - Migraciones

### **Configuración**

```python
# alembic.ini
[alembic]
script_location = migrations
sqlalchemy.url = postgresql+asyncpg://user:pass@localhost:5432/usipipo

[post_write_hooks]
hooks = ruff
ruff.type = exec
ruff.command = ruff format REVISION_SCRIPT_FILENAME
```

---

### **Ejemplo de Migración**

```python
# migrations/versions/001_create_users_table.py
"""Create users table

Revision ID: 001
Revises: 
Create Date: 2026-03-01 10:00:00.000000

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision = '001'
down_revision = None
branch_labels = None
depends_on = None

def upgrade() -> None:
    # Create enum types
    sa.Enum('WIREGUARD', 'OUTLINE', name='keytype').create(op.get_bind())
    sa.Enum('ACTIVE', 'EXPIRED', 'REVOKED', 'PENDING', name='keystatus').create(op.get_bind())
    
    # Create users table
    op.create_table(
        'users',
        sa.Column('id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('telegram_id', sa.BigInteger(), nullable=False),
        sa.Column('username', sa.String(255), nullable=True),
        sa.Column('first_name', sa.String(255), nullable=True),
        sa.Column('last_name', sa.String(255), nullable=True),
        sa.Column('is_admin', sa.Boolean(), default=False),
        sa.Column('balance_gb', sa.Numeric(10, 2), default=5.0),
        sa.Column('total_purchased_gb', sa.Numeric(10, 2), default=0.0),
        sa.Column('referral_code', sa.String(50), unique=True, nullable=False),
        sa.Column('referred_by', postgresql.UUID(as_uuid=True), nullable=True),
        sa.Column('referral_credits', sa.BigInteger(), default=0),
        sa.Column('purchase_count', sa.BigInteger(), default=0),
        sa.Column('loyalty_bonus_percent', sa.BigInteger(), default=0),
        sa.Column('welcome_bonus_used', sa.Boolean(), default=False),
        sa.Column('created_at', sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column('updated_at', sa.DateTime(timezone=True), onupdate=sa.func.now()),
        sa.PrimaryKeyConstraint('id'),
        sa.ForeignKeyConstraint(['referred_by'], ['users.id']),
    )
    
    # Create indexes
    op.create_index('ix_users_telegram_id', 'users', ['telegram_id'])
    op.create_index('ix_users_referral_code', 'users', ['referral_code'])

def downgrade() -> None:
    op.drop_index('ix_users_referral_code', 'users')
    op.drop_index('ix_users_telegram_id', 'users')
    op.drop_table('users')
    
    sa.Enum('ACTIVE', 'EXPIRED', 'REVOKED', 'PENDING', name='keystatus').drop(op.get_bind())
    sa.Enum('WIREGUARD', 'OUTLINE', name='keytype').drop(op.get_bind())
```

---

## 🔌 Redis - Caché y Rate Limiting

### **Configuración del Cliente**

```python
# src/infrastructure/redis.py
import redis.asyncio as redis
from typing import Optional

class RedisClient:
    def __init__(self):
        self.client: Optional[redis.Redis] = None
    
    async def connect(self, url: str):
        self.client = redis.from_url(
            url,
            encoding="utf-8",
            decode_responses=True,
            max_connections=50,
        )
    
    async def disconnect(self):
        if self.client:
            await self.client.close()
    
    # JWT Blacklist
    async def blacklist_token(self, token: str, expiry_seconds: int):
        key = f"usipipo:blacklist:{token}"
        await self.client.setex(key, expiry_seconds, "1")
    
    async def is_token_blacklisted(self, token: str) -> bool:
        key = f"usipipo:blacklist:{token}"
        return await self.client.exists(key)
    
    # Bot Token Storage
    async def store_bot_tokens(self, telegram_id: int, access_token: str, refresh_token: str, ttl_days: int = 30):
        key = f"usipipo:bot:tokens:{telegram_id}"
        data = {
            "access_token": access_token,
            "refresh_token": refresh_token,
        }
        await self.client.hset(key, mapping=data)
        await self.client.expire(key, ttl_days * 24 * 60 * 60)
    
    async def get_bot_tokens(self, telegram_id: int) -> Optional[dict]:
        key = f"usipipo:bot:tokens:{telegram_id}"
        data = await self.client.hgetall(key)
        return data if data else None
    
    # Rate Limiting
    async def is_rate_limited(self, key: str, max_requests: int, window_seconds: int) -> bool:
        current = await self.client.incr(key)
        if current == 1:
            await self.client.expire(key, window_seconds)
        return current > max_requests

redis_client = RedisClient()
```

---

## 🔐 Seguridad

### **JWT Authentication**

```python
# src/shared/security/jwt.py
from datetime import datetime, timedelta, timezone
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext

from src.core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
REFRESH_TOKEN_EXPIRE_DAYS = 30

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    
    to_encode.update({"exp": expire, "type": "access"})
    return jwt.encode(to_encode, settings.SECRET_KEY, algorithm=ALGORITHM)

def create_refresh_token(data: dict) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    to_encode.update({"exp": expire, "type": "refresh"})
    return jwt.encode(to_encode, settings.SECRET_KEY, algorithm=ALGORITHM)

def decode_token(token: str) -> Optional[dict]:
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except JWTError:
        return None

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)
```

---

### **Rate Limiting (SlowAPI)**

```python
# src/shared/middleware/rate_limiter.py
from slowapi import Limiter
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import Request
from fastapi.responses import JSONResponse

limiter = Limiter(key_func=get_remote_address)

async def rate_limit_exceeded_handler(request: Request, exc: RateLimitExceeded) -> JSONResponse:
    return JSONResponse(
        status_code=429,
        content={
            "error": "rate_limit_exceeded",
            "message": "Demasiadas solicitudes. Por favor intenta de nuevo más tarde.",
            "retry_after": str(exc.detail),
        }
    )

# Decoradores para endpoints
@limiter.limit("5/minute")  # Auth endpoints
@limiter.limit("30/minute")  # Admin endpoints
@limiter.limit("60/minute")  # Default
@limiter.limit("100/minute")  # Webhooks
```

---

## 🧪 Testing

### **Configuración de Pytest**

```python
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
asyncio_mode = auto
addopts = 
    -v
    --strict-markers
    --cov=src
    --cov-report=html
    --cov-report=term-missing
    --cov-fail-under=80
```

---

### **Fixtures Principales**

```python
# tests/conftest.py
import pytest
import pytest_asyncio
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

from src.main import app
from src.infrastructure.persistence.database import Base, Database

@pytest_asyncio.fixture
async def db_session():
    engine = create_async_engine(
        "postgresql+asyncpg://user:pass@localhost:5432/usipipo_test",
        echo=True,
    )
    
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    
    async_session_maker = async_sessionmaker(engine, expire_on_commit=False)
    
    async with async_session_maker() as session:
        yield session
    
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    
    await engine.dispose()

@pytest_asyncio.fixture
async def client(db_session: AsyncSession):
    async with AsyncClient(
        app=app,
        base_url="http://test",
    ) as ac:
        ac.app.state.db = db_session
        yield ac
```

---

### **Ejemplo de Test**

```python
# tests/unit/test_auth.py
import pytest
from httpx import AsyncClient
from datetime import datetime, timezone

@pytest.mark.asyncio
async def test_telegram_auto_register(client: AsyncClient):
    """Test auto-registro vía Telegram"""
    # Arrange
    telegram_init_data = "query_id=...&user=%7B%22id%22%3A123456789%7D&hash=abc123"
    
    # Act
    response = await client.post(
        "/api/v1/auth/telegram/auto-register",
        headers={"Telegram-Init-Data": telegram_init_data},
    )
    
    # Assert
    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data
    assert "refresh_token" in data
    assert data["user"]["telegram_id"] == 123456789
    assert data["user"]["balance_gb"] == 5.0

@pytest.mark.asyncio
async def test_token_refresh(client: AsyncClient, authenticated_client: AsyncClient):
    """Test refresh de token"""
    # Act
    response = await authenticated_client.post("/api/v1/auth/refresh")
    
    # Assert
    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data
```

---

## 📦 Dependencias

### **Runtime Dependencies**

```toml
# pyproject.toml
[project]
dependencies = [
    "fastapi>=0.109.0",
    "uvicorn[standard]>=0.27.0",
    "gunicorn>=21.2.0",
    "sqlalchemy[asyncio]>=2.0.25",
    "asyncpg>=0.29.0",
    "alembic>=1.13.0",
    "pydantic>=2.5.0",
    "pydantic-settings>=2.1.0",
    "python-dotenv>=1.0.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "redis>=5.0.0",
    "httpx>=0.26.0",
    "slowapi>=0.1.9",
    "usipipo-commons>=0.12.0",
]
```

---

### **Development Dependencies**

```toml
[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.1.0",
    "httpx>=0.26.0",
    "factory-boy>=3.3.0",
    "ruff>=0.3.0",
    "mypy>=1.8.0",
    "bandit>=1.7.7",
    "pre-commit>=3.6.0",
]
```

---

## 🚀 Deployment

### **systemd Service**

```ini
# /etc/systemd/system/usipipo-backend.service
[Unit]
Description=uSipipo Backend API
After=network.target postgresql.service redis.service

[Service]
Type=notify
User=usipipo
Group=usipipo
WorkingDirectory=/opt/usipipo/usipipo-backend
EnvironmentFile=/opt/usipipo/.env
ExecStart=/opt/usipipo/.venv/bin/python -m src
ExecReload=/bin/kill -s HUP $MAINPID
Restart=always
RestartSec=10
TimeoutStopSec=30

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/usipipo/usipipo-backend/logs

# Resources
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```

---

### **Variables de Entorno**

```bash
# .env
APP_ENV=production
DEBUG=false
LOG_LEVEL=INFO

# Database
DATABASE_URL=postgresql+asyncpg://usipipo:password@localhost:5432/usipipo_prod

# Redis
REDIS_URL=redis://localhost:6379

# Security
SECRET_KEY=<generar con secrets.token_urlsafe(32)>
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=30

# Telegram
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrsTUVwxyz

# TronDealer
TRONDEALER_API_KEY=td_api_key_xxx
TRONDEALER_WEBHOOK_SECRET=wh_secret_xxx

# VPN
WIREGUARD_INTERFACE=wg0
WIREGUARD_SERVER_PUBLIC_KEY=xxx
WIREGUARD_SERVER_PRIVATE_KEY=xxx
OUTLINE_API_URL=https://outline-server:8080
OUTLINE_CERT_PATH=/etc/outline/cert.pem

# CORS
ALLOWED_ORIGINS=https://usipipo.com,https://usipipo.duckdns.org
```

---

## 📈 Métricas y Health Checks

### **Endpoints de Health**

```python
# src/infrastructure/api/routes/health.py
from fastapi import APIRouter, Depends
from sqlalchemy import text
from src.infrastructure.persistence.database import Database

router = APIRouter()

@router.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "version": "0.10.0",
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }

@router.get("/health/ready")
async def readiness_check(db: Database = Depends()):
    try:
        await db.execute(text("SELECT 1"))
        db_status = "ok"
    except Exception as e:
        db_status = f"error: {str(e)}"
    
    return {
        "status": "ready" if db_status == "ok" else "not_ready",
        "checks": {
            "database": db_status,
        }
    }

@router.get("/health/live")
async def liveness_check():
    return {"status": "alive"}
```

---

## 📚 Recursos Relacionados

- [Stack Overview](stack-overview.md)
- [Backend API Reference](../apis/backend-api-reference.md)
- [PRD Backend](../prds/prd-usipipo-backend.md)

---

**Última actualización:** 2026-03-27
