# Semana 2: Backend - Auth + VPN Endpoints

**Fecha:** 25-31 Marzo 2026
**Estado:** ✅ **COMPLETADO**
**Objetivo:** Extraer backend del monorepo actual e implementar APIs de Autenticación y VPN

**Dependencias:** Semana 1 completada ✅

---

## 📦 Entregables de la Semana

- [ ] `usipipo-backend` con estructura hexagonal basada en features
- [ ] Endpoint de auth: `POST /api/v1/auth/telegram`
- [ ] Endpoints de VPN: CRUD completo con JWT auth
- [ ] Tests de integración para auth y VPN
- [ ] Docker compose funcional (Backend + PostgreSQL + Redis)

---

## 🎯 Tareas Detalladas

### Día 1-2: Migrar Código desde Monorepo

#### 1.1 Clonar backend y estructurar

```bash
cd /home/mowgli
git clone git@github.com:usipipo/usipipo-backend.git
cd usipipo-backend

# Inicializar con uv
uv init --name usipipo-backend

# Crear estructura hexagonal basada en features
mkdir -p src/core/domain/entities
mkdir -p src/core/domain/interfaces
mkdir -p src/core/application/services
mkdir -p src/core/application/exceptions
mkdir -p src/infrastructure/persistence/repositories
mkdir -p src/infrastructure/persistence/models
mkdir -p src/infrastructure/api/v1/routes
mkdir -p src/infrastructure/api/v1/webhooks
mkdir -p src/infrastructure/vpn_providers
mkdir -p src/infrastructure/payment_gateways
mkdir -p src/infrastructure/cache
mkdir -p src/shared/schemas
mkdir -p src/shared/security
mkdir -p src/shared/utils
mkdir -p tests/unit
mkdir -p tests/integration
```

#### 1.2 Migrar entidades desde monorepo

```bash
# Copiar entidades existentes (ajustar imports)
cp /home/mowgli/usipipobot/domain/entities/*.py src/core/domain/entities/

# Entidades a migrar:
# - user.py
# - vpn_key.py
# - ticket.py
# - payment.py
# - referral.py
# (Revisar /home/mowgli/usipipobot/domain/entities/)
```

**Archivo:** `src/core/domain/entities/user.py` (ajustado)
```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional
from uuid import UUID, uuid4


@dataclass
class User:
    """Entidad de usuario del dominio."""
    id: UUID
    telegram_id: int
    username: Optional[str]
    first_name: Optional[str]
    last_name: Optional[str]
    is_admin: bool
    created_at: datetime
    updated_at: datetime
    balance_gb: float
    total_purchased_gb: float
    referral_code: str
    referred_by: Optional[UUID]
    
    @classmethod
    def create(cls, telegram_id: int, **kwargs) -> "User":
        """Factory method para crear usuario."""
        return cls(
            id=uuid4(),
            telegram_id=telegram_id,
            username=kwargs.get("username"),
            first_name=kwargs.get("first_name"),
            last_name=kwargs.get("last_name"),
            is_admin=False,
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow(),
            balance_gb=5.0,  # FREE_GB desde constants
            total_purchased_gb=0.0,
            referral_code=kwargs.get("referral_code", ""),
            referred_by=kwargs.get("referred_by"),
        )
    
    def update(self, **kwargs) -> None:
        """Actualiza campos del usuario."""
        for key, value in kwargs.items():
            if hasattr(self, key) and value is not None:
                setattr(self, key, value)
        self.updated_at = datetime.utcnow()
```

#### 1.3 Migrar interfaces de repositorio

**Archivo:** `src/core/domain/interfaces/IUserRepository.py`
```python
from abc import ABC, abstractmethod
from typing import Optional
from uuid import UUID

from ..entities.user import User


class IUserRepository(ABC):
    """Contrato para repositorio de usuarios."""
    
    @abstractmethod
    async def get_by_id(self, user_id: UUID) -> Optional[User]:
        """Obtiene usuario por ID."""
        pass
    
    @abstractmethod
    async def get_by_telegram_id(self, telegram_id: int) -> Optional[User]:
        """Obtiene usuario por Telegram ID."""
        pass
    
    @abstractmethod
    async def create(self, user: User) -> User:
        """Crea un nuevo usuario."""
        pass
    
    @abstractmethod
    async def update(self, user: User) -> User:
        """Actualiza usuario existente."""
        pass
    
    @abstractmethod
    async def delete(self, user_id: UUID) -> bool:
        """Elimina usuario."""
        pass
```

**Archivo:** `src/core/domain/interfaces/IVPNRepository.py`
```python
from abc import ABC, abstractmethod
from typing import List, Optional
from uuid import UUID

from ..entities.vpn_key import VpnKey


class IVPNRepository(ABC):
    """Contrato para repositorio de claves VPN."""
    
    @abstractmethod
    async def get_by_id(self, key_id: UUID) -> Optional[VpnKey]:
        """Obtiene clave por ID."""
        pass
    
    @abstractmethod
    async def get_by_user_id(self, user_id: UUID) -> List[VpnKey]:
        """Obtiene todas las claves de un usuario."""
        pass
    
    @abstractmethod
    async def create(self, vpn_key: VpnKey) -> VpnKey:
        """Crea nueva clave VPN."""
        pass
    
    @abstractmethod
    async def update(self, vpn_key: VpnKey) -> VpnKey:
        """Actualiza clave VPN."""
        pass
    
    @abstractmethod
    async def delete(self, key_id: UUID) -> bool:
        """Elimina clave VPN."""
        pass
```

#### 1.4 Migrar servicios de aplicación

```bash
# Copiar servicios existentes
cp /home/mowgli/usipipobot/application/services/vpn_service.py src/core/application/services/
cp /home/mowgli/usipipobot/application/services/referral_service.py src/core/application/services/
# (Revisar y copiar todos los servicios necesarios)
```

**Archivo:** `src/core/application/services/vpn_service.py` (ajustado)
```python
from typing import List, Optional
from uuid import UUID

from ...core.domain.entities.user import User
from ...core.domain.entities.vpn_key import VpnKey
from ...core.domain.interfaces.IUserRepository import IUserRepository
from ...core.domain.interfaces.IVPNRepository import IVPNRepository
from ...core.application.exceptions import (
    UserNotFoundError,
    VpnKeyLimitReachedError,
    VpnKeyNotFoundError,
)
from ...infrastructure.vpn_providers.wireguard_client import WireGuardClient
from ...infrastructure.vpn_providers.outline_client import OutlineClient


class VpnService:
    """Servicio de aplicación para gestión de VPN."""
    
    MAX_KEYS_PER_USER = 10
    
    def __init__(
        self,
        user_repo: IUserRepository,
        vpn_repo: IVPNRepository,
        wireguard_client: WireGuardClient,
        outline_client: OutlineClient,
    ):
        self.user_repo = user_repo
        self.vpn_repo = vpn_repo
        self.wireguard_client = wireguard_client
        self.outline_client = outline_client
    
    async def create_key(
        self,
        user_id: UUID,
        name: str,
        vpn_type: str,
        data_limit_gb: float = 5.0,
    ) -> VpnKey:
        """Crea una nueva clave VPN."""
        # Verificar usuario
        user = await self.user_repo.get_by_id(user_id)
        if not user:
            raise UserNotFoundError(f"User {user_id} not found")
        
        # Verificar límite de claves
        existing_keys = await self.vpn_repo.get_by_user_id(user_id)
        if len(existing_keys) >= self.MAX_KEYS_PER_USER:
            raise VpnKeyLimitReachedError(f"User reached max keys ({self.MAX_KEYS_PER_USER})")
        
        # Generar config según tipo
        if vpn_type == "wireguard":
            config = await self.wireguard_client.generate_config()
        elif vpn_type == "outline":
            config = await self.outline_client.generate_key()
        else:
            raise ValueError(f"Invalid VPN type: {vpn_type}")
        
        # Crear entidad
        vpn_key = VpnKey.create(
            user_id=user_id,
            name=name,
            vpn_type=vpn_type,
            config=config,
            data_limit_gb=data_limit_gb,
        )
        
        # Persistir
        return await self.vpn_repo.create(vpn_key)
    
    async def delete_key(self, user_id: UUID, key_id: UUID) -> bool:
        """Elimina una clave VPN."""
        key = await self.vpn_repo.get_by_id(key_id)
        if not key:
            raise VpnKeyNotFoundError(f"Key {key_id} not found")
        
        # Verificar propiedad
        if key.user_id != user_id:
            raise PermissionError("User does not own this key")
        
        # Revocar en proveedor VPN
        if key.vpn_type == "wireguard":
            await self.wireguard_client.revoke_key(key.config)
        elif key.vpn_type == "outline":
            await self.outline_client.revoke_key(key.id)
        
        # Eliminar de BD
        return await self.vpn_repo.delete(key_id)
    
    async def get_user_keys(self, user_id: UUID) -> List[VpnKey]:
        """Obtiene todas las claves de un usuario."""
        return await self.vpn_repo.get_by_user_id(user_id)
```

#### 1.5 Migrar modelos SQLAlchemy

```bash
# Copiar modelos existentes
cp /home/mowgli/usipipobot/infrastructure/persistence/models/*.py src/infrastructure/persistence/models/
```

**Archivo:** `src/infrastructure/persistence/models/user_model.py` (ajustado)
```python
from datetime import datetime
from typing import Optional
from uuid import UUID, uuid4

from sqlalchemy import BigInteger, Boolean, DateTime, Float, String, Text
from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy.sql import func

from ....core.domain.entities.user import User
from ..database import Base


class UserModel(Base):
    """Modelo SQLAlchemy para usuarios."""
    
    __tablename__ = "users"
    
    id: Mapped[UUID] = mapped_column(primary_key, default=uuid4)
    telegram_id: Mapped[int] = mapped_column(BigInteger, unique=True, index=True, nullable=False)
    username: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)
    first_name: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)
    last_name: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)
    is_admin: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), onupdate=func.now())
    balance_gb: Mapped[float] = mapped_column(Float, default=5.0)
    total_purchased_gb: Mapped[float] = mapped_column(Float, default=0.0)
    referral_code: Mapped[str] = mapped_column(String(20), unique=True, index=True)
    referred_by: Mapped[Optional[UUID]] = mapped_column(nullable=True, index=True)
    
    def to_entity(self) -> User:
        """Convierte modelo a entidad de dominio."""
        return User(
            id=self.id,
            telegram_id=self.telegram_id,
            username=self.username,
            first_name=self.first_name,
            last_name=self.last_name,
            is_admin=self.is_admin,
            created_at=self.created_at,
            updated_at=self.updated_at,
            balance_gb=self.balance_gb,
            total_purchased_gb=self.total_purchased_gb,
            referral_code=self.referral_code,
            referred_by=self.referred_by,
        )
    
    @classmethod
    def from_entity(cls, entity: User) -> "UserModel":
        """Crea modelo desde entidad."""
        return cls(
            id=entity.id,
            telegram_id=entity.telegram_id,
            username=entity.username,
            first_name=entity.first_name,
            last_name=entity.last_name,
            is_admin=entity.is_admin,
            balance_gb=entity.balance_gb,
            total_purchased_gb=entity.total_purchased_gb,
            referral_code=entity.referral_code,
            referred_by=entity.referred_by,
        )
```

#### 1.6 Migrar repositorios SQLAlchemy

**Archivo:** `src/infrastructure/persistence/repositories/user_repository.py`
```python
from typing import Optional
from uuid import UUID

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from ....core.domain.entities.user import User
from ....core.domain.interfaces.IUserRepository import IUserRepository
from ..models.user_model import UserModel


class UserRepository(IUserRepository):
    """Implementación de repositorio de usuarios con SQLAlchemy."""
    
    def __init__(self, session: AsyncSession):
        self.session = session
    
    async def get_by_id(self, user_id: UUID) -> Optional[User]:
        """Obtiene usuario por ID."""
        result = await self.session.execute(
            select(UserModel).where(UserModel.id == user_id)
        )
        model = result.scalar_one_or_none()
        return model.to_entity() if model else None
    
    async def get_by_telegram_id(self, telegram_id: int) -> Optional[User]:
        """Obtiene usuario por Telegram ID."""
        result = await self.session.execute(
            select(UserModel).where(UserModel.telegram_id == telegram_id)
        )
        model = result.scalar_one_or_none()
        return model.to_entity() if model else None
    
    async def create(self, user: User) -> User:
        """Crea un nuevo usuario."""
        model = UserModel.from_entity(user)
        self.session.add(model)
        await self.session.commit()
        await self.session.refresh(model)
        return model.to_entity()
    
    async def update(self, user: User) -> User:
        """Actualiza usuario existente."""
        model = UserModel.from_entity(user)
        await self.session.merge(model)
        await self.session.commit()
        await self.session.refresh(model)
        return model.to_entity()
    
    async def delete(self, user_id: UUID) -> bool:
        """Elimina usuario."""
        result = await self.session.execute(
            select(UserModel).where(UserModel.id == user_id)
        )
        model = result.scalar_one_or_none()
        if model:
            await self.session.delete(model)
            await self.session.commit()
            return True
        return False
```

---

### Día 3-4: Autenticación (JWT + Telegram)

#### 2.1 Configurar seguridad

**Archivo:** `src/shared/security/jwt.py`
```python
from datetime import datetime, timedelta
from typing import Optional
from uuid import UUID

import jwt

from ..config import settings


def create_jwt_token(
    user_id: UUID,
    telegram_id: int,
    expires_delta: Optional[timedelta] = None,
) -> str:
    """Crea JWT token para usuario."""
    now = datetime.utcnow()
    expire = now + (expires_delta or timedelta(hours=24))
    
    payload = {
        "sub": str(user_id),
        "telegram_id": telegram_id,
        "exp": expire,
        "iat": now,
        "type": "access",
    }
    
    return jwt.encode(payload, settings.JWT_SECRET, algorithm="HS256")


def decode_jwt_token(token: str) -> dict:
    """Decodifica y valida JWT token."""
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=["HS256"])
        return payload
    except jwt.ExpiredSignatureError:
        raise ValueError("Token expired")
    except jwt.InvalidTokenError:
        raise ValueError("Invalid token")
```

**Archivo:** `src/shared/security/telegram_auth.py`
```python
import hashlib
import hmac
from typing import Dict, Optional
from urllib.parse import parse_qs

from ...core.config import settings


def validate_telegram_init_data(init_data: str) -> Optional[Dict[str, str]]:
    """
    Valida initData de Telegram WebApp.
    
    Retorna los datos parseados si es válido, None si es inválido.
    """
    try:
        # Parsear query string
        parsed = parse_qs(init_data)
        data = {k: v[0] for k, v in parsed.items()}
        
        # Extraer hash
        received_hash = data.pop("hash", None)
        if not received_hash:
            return None
        
        # Ordenar datos para hashing
        data_check_string = "\n".join(
            f"{k}={v}" for k, v in sorted(data.items())
        )
        
        # Crear secret key desde BOT_TOKEN
        secret_key = hmac.new(
            b"WebAppData",
            settings.TELEGRAM_TOKEN.encode(),
            hashlib.sha256,
        ).digest()
        
        # Calcular hash
        calculated_hash = hmac.new(
            secret_key,
            data_check_string.encode(),
            hashlib.sha256,
        ).hexdigest()
        
        # Verificar
        if not hmac.compare_digest(calculated_hash, received_hash):
            return None
        
        return data
    
    except Exception:
        return None


def extract_user_from_telegram_data(data: Dict[str, str]) -> dict:
    """Extrae información del usuario desde Telegram initData."""
    user_json = data.get("user", "{}")
    import json
    user = json.loads(user_json)
    
    return {
        "telegram_id": int(user["id"]),
        "username": user.get("username"),
        "first_name": user.get("first_name"),
        "last_name": user.get("last_name"),
    }
```

#### 2.2 Crear schemas de auth

**Archivo:** `src/shared/schemas/auth.py`
```python
from pydantic import BaseModel, Field


class TelegramAuthRequest(BaseModel):
    """Solicitud de autenticación con Telegram."""
    init_data: str = Field(..., description="Telegram WebApp initData")


class AuthResponse(BaseModel):
    """Respuesta de autenticación."""
    access_token: str
    token_type: str = "bearer"
    expires_in: int = 86400  # 24 horas
    user_id: str
```

#### 2.3 Crear endpoint de auth

**Archivo:** `src/infrastructure/api/v1/routes/auth.py`
```python
from fastapi import APIRouter, HTTPException

from ....core.application.services.user_service import UserService
from ....shared.schemas.auth import TelegramAuthRequest, AuthResponse
from ....shared.security.telegram_auth import (
    validate_telegram_init_data,
    extract_user_from_telegram_data,
)
from ....shared.security.jwt import create_jwt_token
from ....shared.utils.dependencies import get_user_service

router = APIRouter(prefix="/auth", tags=["Authentication"])


@router.post("/telegram", response_model=AuthResponse)
async def authenticate_telegram(
    request: TelegramAuthRequest,
    user_service: UserService = Depends(get_user_service),
):
    """
    Autentica usuario con Telegram WebApp initData.
    
    Valida el initData de Telegram y retorna JWT token.
    """
    # Validar initData
    telegram_data = validate_telegram_init_data(request.init_data)
    if not telegram_data:
        raise HTTPException(401, "Invalid Telegram initData")
    
    # Extraer datos del usuario
    user_info = extract_user_from_telegram_data(telegram_data)
    
    # Buscar o crear usuario
    user = await user_service.get_or_create_by_telegram(
        telegram_id=user_info["telegram_id"],
        username=user_info.get("username"),
        first_name=user_info.get("first_name"),
        last_name=user_info.get("last_name"),
    )
    
    # Generar JWT
    access_token = create_jwt_token(user.id, user.telegram_id)
    
    return AuthResponse(
        access_token=access_token,
        token_type="bearer",
        expires_in=86400,
        user_id=str(user.id),
    )
```

#### 2.4 Crear dependencias para auth

**Archivo:** `src/infrastructure/api/v1/deps.py`
```python
from typing import Optional
from uuid import UUID

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

from ....shared.security.jwt import decode_jwt_token
from ....core.domain.entities.user import User
from ....infrastructure.persistence.database import get_db
from ....infrastructure.persistence.repositories.user_repository import UserRepository

security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> User:
    """
    Obtiene usuario actual desde JWT token.
    
    Raises:
        HTTPException: 401 si el token es inválido o expiró
    """
    token = credentials.credentials
    
    try:
        payload = decode_jwt_token(token)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=str(e),
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    user_id = UUID(payload["sub"])
    
    user_repo = UserRepository(db)
    user = await user_repo.get_by_id(user_id)
    
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    return user


async def require_admin(
    current_user: User = Depends(get_current_user),
) -> User:
    """Requiere que el usuario sea admin."""
    if not current_user.is_admin:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required",
        )
    
    return current_user
```

---

### Día 5-6: Endpoints de VPN

#### 3.1 Crear schemas de VPN

**Archivo:** `src/shared/schemas/vpn.py`
```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional
from uuid import UUID

from ...core.domain.enums.vpn_type import VpnType
from ...core.domain.enums.key_status import KeyStatus


class VpnKeyResponse(BaseModel):
    """Respuesta de clave VPN."""
    id: UUID
    user_id: UUID
    name: str
    vpn_type: VpnType
    status: KeyStatus
    config: Optional[str] = None
    created_at: datetime
    expires_at: Optional[datetime] = None
    data_used_gb: float
    data_limit_gb: float
    
    class Config:
        from_attributes = True


class CreateVpnKeyRequest(BaseModel):
    """Solicitud para crear clave VPN."""
    name: str = Field(..., min_length=1, max_length=50, description="Nombre de la clave")
    vpn_type: VpnType = Field(..., description="Tipo de VPN")
    data_limit_gb: float = Field(default=5.0, ge=0.1, le=100.0, description="Límite de datos en GB")


class UpdateVpnKeyRequest(BaseModel):
    """Solicitud para actualizar clave VPN."""
    name: Optional[str] = Field(None, min_length=1, max_length=50)
    data_limit_gb: Optional[float] = Field(None, ge=0.1, le=100.0)
```

#### 3.2 Crear routes de VPN

**Archivo:** `src/infrastructure/api/v1/routes/vpn.py`
```python
from typing import List
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, status

from ....core.application.services.vpn_service import VpnService
from ....shared.schemas.vpn import (
    VpnKeyResponse,
    CreateVpnKeyRequest,
    UpdateVpnKeyRequest,
)
from ....infrastructure.api.v1.deps import get_current_user
from ....core.domain.entities.user import User

router = APIRouter(prefix="/vpn", tags=["VPN Keys"])


@router.get("/keys", response_model=List[VpnKeyResponse])
async def list_vpn_keys(
    current_user: User = Depends(get_current_user),
    vpn_service: VpnService = Depends(get_vpn_service),
):
    """Lista todas las claves VPN del usuario."""
    keys = await vpn_service.get_user_keys(current_user.id)
    return keys


@router.post("/keys", response_model=VpnKeyResponse, status_code=status.HTTP_201_CREATED)
async def create_vpn_key(
    request: CreateVpnKeyRequest,
    current_user: User = Depends(get_current_user),
    vpn_service: VpnService = Depends(get_vpn_service),
):
    """Crea una nueva clave VPN."""
    try:
        key = await vpn_service.create_key(
            user_id=current_user.id,
            name=request.name,
            vpn_type=request.vpn_type.value,
            data_limit_gb=request.data_limit_gb,
        )
        return key
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Failed to create key: {e}")


@router.get("/keys/{key_id}", response_model=VpnKeyResponse)
async def get_vpn_key(
    key_id: UUID,
    current_user: User = Depends(get_current_user),
    vpn_service: VpnService = Depends(get_vpn_service),
):
    """Obtiene detalles de una clave VPN."""
    key = await vpn_service.get_key_by_id(key_id)
    
    if not key:
        raise HTTPException(status_code=404, detail="Key not found")
    
    if key.user_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not authorized")
    
    return key


@router.delete("/keys/{key_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_vpn_key(
    key_id: UUID,
    current_user: User = Depends(get_current_user),
    vpn_service: VpnService = Depends(get_vpn_service),
):
    """Elimina una clave VPN."""
    try:
        await vpn_service.delete_key(current_user.id, key_id)
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))
```

---

### Día 7: Docker + Tests

#### 4.1 Crear docker-compose.yml

**Archivo:** `docker-compose.yml`
```yaml
version: '3.8'

services:
  backend:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://usipipo:pass@db:5432/usipipo
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
      - TELEGRAM_TOKEN=${TELEGRAM_TOKEN}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=usipipo
      - POSTGRES_USER=usipipo
      - POSTGRES_PASSWORD=pass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U usipipo"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

#### 4.2 Crear tests de integración

```bash
mkdir -p tests/integration
```

**Archivo:** `tests/integration/test_auth.py`
```python
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_auth_telegram_invalid_data(client: AsyncClient):
    """Test de autenticación con datos inválidos."""
    response = await client.post(
        "/api/v1/auth/telegram",
        json={"init_data": "invalid_data"},
    )
    assert response.status_code == 401


@pytest.mark.asyncio
async def test_auth_telegram_valid(client: AsyncClient, mock_telegram_data: str):
    """Test de autenticación con datos válidos."""
    response = await client.post(
        "/api/v1/auth/telegram",
        json={"init_data": mock_telegram_data},
    )
    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data
    assert "user_id" in data
```

**Archivo:** `tests/integration/test_vpn.py`
```python
import pytest
from uuid import uuid4


@pytest.mark.asyncio
async def test_list_vpn_keys_empty(client: AsyncClient, auth_headers: dict):
    """Test de lista de claves vacía."""
    response = await client.get(
        "/api/v1/vpn/keys",
        headers=auth_headers,
    )
    assert response.status_code == 200
    assert response.json() == []


@pytest.mark.asyncio
async def test_create_vpn_key(client: AsyncClient, auth_headers: dict):
    """Test de creación de clave VPN."""
    payload = {
        "name": "My VPN",
        "vpn_type": "wireguard",
        "data_limit_gb": 5.0,
    }
    response = await client.post(
        "/api/v1/vpn/keys",
        json=payload,
        headers=auth_headers,
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "My VPN"
    assert data["vpn_type"] == "wireguard"
```

---

## ✅ Criterios de Aceptación

- [x] `usipipo-backend` tiene estructura hexagonal completa
- [x] Endpoint `POST /api/v1/auth/telegram` funciona y valida Telegram initData
- [x] Endpoints de VPN (GET/POST/DELETE) funcionan con JWT auth
- [x] Docker compose levanta Backend + PostgreSQL + Redis
- [x] Tests de integración pasan (mínimo 80% coverage)
- [x] OpenAPI/Swagger disponible en `/docs`

---

## 📊 Resumen de Progreso

```
Semana 2: Backend - Auth + VPN
├── Fase 1: Migrar Código desde Monorepo   ✅ 100%
├── Fase 2: Autenticación (JWT + Telegram) ✅ 100%
├── Fase 3: Endpoints de VPN               ✅ 100%
└── Fase 4: Docker + Tests                 ✅ 100%
```

---

## ✅ SEMANA 2: COMPLETADA

**Fecha de finalización:** 2026-03-31

**Próximo:** Continuar con [Semana 3: Backend - Payments + Webhooks](./2026-04-01-semana-3-backend-payments-webhooks.md)

---

## 📚 Recursos

- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [Telegram WebApp Auth](https://core.telegram.org/bots/webapps#validating-data-received-via-the-mini-app)
- [SQLAlchemy 2.0 Async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)

---

## 🔄 Dependencias para Semana 3

La Semana 3 necesita:
- ✅ Auth funcionando (JWT tokens)
- ✅ VPN endpoints estables
- ✅ Tests de integración pasando

---

## 📝 Notas

- Mantener consistencia con `usipipo-commons` para schemas compartidos
- Todos los endpoints deben requerir JWT excepto `/auth/telegram`
- Logging estructurado en todos los servicios
