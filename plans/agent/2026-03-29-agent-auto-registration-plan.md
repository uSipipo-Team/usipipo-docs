# VPN Agent Auto-Registration Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement automatic server registration flow where VPN agents register themselves with the backend on first metrics send using pre-generated API keys.

**Architecture:** Backend exposes registration endpoint that validates API keys and creates server records. Agent collects system metadata, sends registration request on first run, saves returned UUID to .env for subsequent runs.

**Tech Stack:** 
- Backend: Python 3.13, FastAPI, SQLAlchemy, asyncpg, PostgreSQL
- Agent: Go 1.21+, resty HTTP client
- Database: PostgreSQL with asyncpg
- Security: bcrypt for API key hashing, JWT for admin auth

---

## Phase 1: Backend Database Migration

### Task 1.1: Create Alembic Migration for agent_api_keys Table

**Files:**
- Create: `usipipo-backend/alembic/versions/YYYYMMDD_HHMMSS_add_agent_api_keys_table.py`

**Step 1: Generate migration file**

```bash
cd /home/mowgli/usipipo/usipipo-backend
alembic revision -m "add agent_api_keys table and server metadata columns"
```

Expected: Creates new migration file in `alembic/versions/`

**Step 2: Edit migration file with upgrade/downgrade**

```python
"""add agent_api_keys table and server metadata columns

Revision ID: abc123def456
Revises: previous_revision_id
Create Date: 2026-03-29 20:00:00.000000

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# revision identifiers
revision = 'abc123def456'
down_revision = 'previous_revision_id'
branch_labels = None
depends_on = None


def upgrade():
    # Create agent_api_keys table
    op.create_table(
        'agent_api_keys',
        sa.Column('id', sa.UUID(), nullable=False),
        sa.Column('api_key_hash', sa.String(length=255), nullable=False),
        sa.Column('status', sa.String(length=20), nullable=False, default='active'),
        sa.Column('server_id', sa.UUID(), nullable=True),
        sa.Column('created_at', sa.TIMESTAMP(timezone=True), nullable=True, server_default=sa.func.now()),
        sa.Column('used_at', sa.TIMESTAMP(timezone=True), nullable=True),
        sa.Column('expires_at', sa.TIMESTAMP(timezone=True), nullable=True),
        sa.Column('description', sa.String(length=255), nullable=True),
        sa.Column('created_by', sa.UUID(), nullable=True),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('api_key_hash'),
        sa.ForeignKeyConstraint(['server_id'], ['vpn_servers.id']),
        sa.ForeignKeyConstraint(['created_by'], ['admin_users.id']),
        sa.CheckConstraint("status IN ('active', 'used', 'revoked', 'expired')", name='chk_status')
    )
    
    # Create indexes
    op.create_index('idx_agent_api_keys_hash', 'agent_api_keys', ['api_key_hash'])
    op.create_index('idx_agent_api_keys_status', 'agent_api_keys', ['status'])
    
    # Add metadata columns to vpn_servers
    op.add_column('vpn_servers', sa.Column('agent_version', sa.String(length=20), nullable=True))
    op.add_column('vpn_servers', sa.Column('os_type', sa.String(length=50), nullable=True))
    op.add_column('vpn_servers', sa.Column('os_arch', sa.String(length=20), nullable=True))
    op.add_column('vpn_servers', sa.Column('last_registration_ip', postgresql.INET(), nullable=True))


def downgrade():
    # Remove columns from vpn_servers
    op.drop_column('vpn_servers', 'last_registration_ip')
    op.drop_column('vpn_servers', 'os_arch')
    op.drop_column('vpn_servers', 'os_type')
    op.drop_column('vpn_servers', 'agent_version')
    
    # Drop indexes
    op.drop_index('idx_agent_api_keys_status', table_name='agent_api_keys')
    op.drop_index('idx_agent_api_keys_hash', table_name='agent_api_keys')
    
    # Drop table
    op.drop_table('agent_api_keys')
```

**Step 3: Run migration**

```bash
cd /home/mowgli/usipipo/usipipo-backend
alembic upgrade head
```

Expected: `INFO  [alembic.runtime.migration] Context impl PostgresqlImpl. Will assume non-transactional DDL.`

**Step 4: Verify tables created**

```bash
PGPASSWORD=c319c4c2605479e7a3b7c8d4dfa5110c psql -U usipipo_backend_user -d usipipo_backend_db -h localhost -c "\d agent_api_keys"
```

Expected: Table structure with all columns

**Step 5: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add alembic/versions/YYYYMMDD_HHMMSS_*.py
git commit -m "feat(db): add agent_api_keys table for auto-registration"
```

---

### Task 1.2: Create Agent API Key Model

**Files:**
- Create: `usipipo-backend/src/infrastructure/persistence/models/agent_api_key_model.py`

**Step 1: Create model file**

```python
"""SQLAlchemy model for agent API keys."""

import uuid
from datetime import datetime
from typing import TYPE_CHECKING, Optional

from sqlalchemy import CheckConstraint, ForeignKey, Index, String, Text
from sqlalchemy.dialects.postgresql import INET, UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship

from src.infrastructure.persistence.database import Base


class AgentApiKeyModel(Base):
    """Model for agent API keys.
    
    Stores hashed API keys for agent authentication and registration.
    Each key can only be used once for registration.
    """
    
    __tablename__ = "agent_api_keys"
    
    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4
    )
    
    api_key_hash: Mapped[str] = mapped_column(
        String(255),
        unique=True,
        nullable=False,
        index=True
    )
    
    status: Mapped[str] = mapped_column(
        String(20),
        nullable=False,
        default="active",
        index=True
    )
    # Values: active, used, revoked, expired
    
    server_id: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("vpn_servers.id", ondelete="SET NULL"),
        nullable=True
    )
    
    created_at: Mapped[datetime] = mapped_column(
        default=datetime.utcnow
    )
    
    used_at: Mapped[Optional[datetime]] = mapped_column(
        nullable=True
    )
    
    expires_at: Mapped[Optional[datetime]] = mapped_column(
        nullable=True
    )
    
    description: Mapped[Optional[str]] = mapped_column(
        String(255),
        nullable=True
    )
    
    created_by: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("admin_users.id", ondelete="SET NULL"),
        nullable=True
    )
    
    # Relationships
    server = relationship("VpnServerModel", back_populates="agent_api_key")
    creator = relationship("AdminUserModel", back_populates="agent_api_keys")
    
    # Constraints
    __table_args__ = (
        CheckConstraint(
            "status IN ('active', 'used', 'revoked', 'expired')",
            name="chk_status"
        ),
        Index("idx_agent_api_keys_hash", "api_key_hash"),
        Index("idx_agent_api_keys_status", "status"),
    )
    
    def __repr__(self) -> str:
        return f"<AgentApiKey {self.id} - {self.status}>"
```

**Step 2: Update VpnServerModel to add relationship**

**Modify:** `usipipo-backend/src/infrastructure/persistence/models/vpn_server_model.py`

```python
# Add to VpnServerModel class:

# New columns for agent metadata
agent_version: Mapped[Optional[str]] = mapped_column(String(20), nullable=True)
os_type: Mapped[Optional[str]] = mapped_column(String(50), nullable=True)
os_arch: Mapped[Optional[str]] = mapped_column(String(20), nullable=True)
last_registration_ip: Mapped[Optional[str]] = mapped_column(INET, nullable=True)

# Relationship to agent_api_keys
agent_api_key = relationship("AgentApiKeyModel", back_populates="server", uselist=False)
```

**Step 3: Update AdminUserModel to add relationship**

**Modify:** `usipipo-backend/src/infrastructure/persistence/models/admin_user_model.py`

```python
# Add relationship
agent_api_keys = relationship("AgentApiKeyModel", back_populates="creator")
```

**Step 4: Run mypy to verify types**

```bash
cd /home/mowgli/usipipo/usipipo-backend
mypy src/infrastructure/persistence/models/
```

Expected: No type errors

**Step 5: Commit**

```bash
git add src/infrastructure/persistence/models/agent_api_key_model.py
git add src/infrastructure/persistence/models/vpn_server_model.py
git add src/infrastructure/persistence/models/admin_user_model.py
git commit -m "feat(models): add AgentApiKeyModel and update relationships"
```

---

### Task 1.3: Create Agent API Key Repository

**Files:**
- Create: `usipipo-backend/src/infrastructure/persistence/repositories/agent_api_key_repository.py`

**Step 1: Create repository file**

```python
"""Repository for agent API key operations."""

import uuid
from datetime import datetime
from typing import Optional

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from src.infrastructure.persistence.models.agent_api_key_model import AgentApiKeyModel


class AgentApiKeyRepository:
    """Repository for agent API key database operations."""
    
    def __init__(self, session: AsyncSession):
        self.session = session
    
    async def create(
        self,
        api_key_hash: str,
        description: Optional[str] = None,
        expires_at: Optional[datetime] = None,
        created_by: Optional[uuid.UUID] = None,
    ) -> AgentApiKeyModel:
        """Create a new agent API key."""
        api_key = AgentApiKeyModel(
            api_key_hash=api_key_hash,
            description=description,
            expires_at=expires_at,
            created_by=created_by,
        )
        self.session.add(api_key)
        await self.session.commit()
        await self.session.refresh(api_key)
        return api_key
    
    async def get_by_hash(self, api_key_hash: str) -> Optional[AgentApiKeyModel]:
        """Get API key by hash."""
        result = await self.session.execute(
            select(AgentApiKeyModel).where(
                AgentApiKeyModel.api_key_hash == api_key_hash
            )
        )
        return result.scalar_one_or_none()
    
    async def mark_as_used(
        self,
        api_key: AgentApiKeyModel,
        server_id: uuid.UUID,
    ) -> AgentApiKeyModel:
        """Mark API key as used and link to server."""
        api_key.status = "used"
        api_key.used_at = datetime.utcnow()
        api_key.server_id = server_id
        await self.session.commit()
        await self.session.refresh(api_key)
        return api_key
    
    async def revoke(self, api_key: AgentApiKeyModel) -> AgentApiKeyModel:
        """Revoke an API key."""
        api_key.status = "revoked"
        await self.session.commit()
        await self.session.refresh(api_key)
        return api_key
    
    async def list_keys(
        self,
        status: Optional[str] = None,
        limit: int = 50,
        offset: int = 0,
    ) -> list[AgentApiKeyModel]:
        """List API keys with optional filtering."""
        query = select(AgentApiKeyModel)
        
        if status:
            query = query.where(AgentApiKeyModel.status == status)
        
        query = query.order_by(AgentApiKeyModel.created_at.desc())
        query = query.offset(offset).limit(limit)
        
        result = await self.session.execute(query)
        return list(result.scalars().all())
```

**Step 2: Create test**

**Create:** `usipipo-backend/tests/unit/repositories/test_agent_api_key_repository.py`

```python
"""Tests for AgentApiKeyRepository."""

import pytest
from sqlalchemy.ext.asyncio import AsyncSession

from src.infrastructure.persistence.models.agent_api_key_model import AgentApiKeyModel
from src.infrastructure.persistence.repositories.agent_api_key_repository import (
    AgentApiKeyRepository,
)


@pytest.fixture
def repository(db_session: AsyncSession) -> AgentApiKeyRepository:
    """Create repository instance."""
    return AgentApiKeyRepository(db_session)


@pytest.mark.asyncio
async def test_create_api_key(repository: AgentApiKeyRepository):
    """Test creating a new API key."""
    api_key = await repository.create(
        api_key_hash="test_hash_123",
        description="Test key",
    )
    
    assert api_key.api_key_hash == "test_hash_123"
    assert api_key.status == "active"
    assert api_key.description == "Test key"


@pytest.mark.asyncio
async def test_get_by_hash(repository: AgentApiKeyRepository):
    """Test retrieving API key by hash."""
    # Create key
    created = await repository.create(api_key_hash="test_hash_456")
    
    # Retrieve
    retrieved = await repository.get_by_hash("test_hash_456")
    
    assert retrieved is not None
    assert retrieved.id == created.id


@pytest.mark.asyncio
async def test_mark_as_used(repository: AgentApiKeyRepository):
    """Test marking API key as used."""
    import uuid
    
    # Create key
    api_key = await repository.create(api_key_hash="test_hash_789")
    assert api_key.status == "active"
    
    # Mark as used
    server_id = uuid.uuid4()
    updated = await repository.mark_as_used(api_key, server_id)
    
    assert updated.status == "used"
    assert updated.server_id == server_id
    assert updated.used_at is not None
```

**Step 3: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-backend
pytest tests/unit/repositories/test_agent_api_key_repository.py -v
```

Expected: 3 tests passing

**Step 4: Commit**

```bash
git add src/infrastructure/persistence/repositories/agent_api_key_repository.py
git add tests/unit/repositories/test_agent_api_key_repository.py
git commit -m "feat(repository): add AgentApiKeyRepository with CRUD operations"
```

---

## Phase 2: Backend Services

### Task 2.1: Create Agent Registration Service

**Files:**
- Create: `usipipo-backend/src/core/application/services/agent_registration_service.py`

**Step 1: Create service file**

```python
"""Service for handling agent registration."""

import hashlib
import uuid
from datetime import datetime, timedelta
from typing import Optional

from loguru import logger
from sqlalchemy.ext.asyncio import AsyncSession

from src.infrastructure.persistence.models.agent_api_key_model import AgentApiKeyModel
from src.infrastructure.persistence.models.vpn_server_model import VpnServerModel
from src.infrastructure.persistence.repositories.agent_api_key_repository import (
    AgentApiKeyRepository,
)
from src.infrastructure.persistence.repositories.vpn_repository import VpnRepository


class AgentRegistrationService:
    """Service for handling agent registration and API key management."""
    
    def __init__(
        self,
        session: AsyncSession,
        api_key_repository: AgentApiKeyRepository,
        vpn_repository: VpnRepository,
    ):
        self.session = session
        self.api_key_repository = api_key_repository
        self.vpn_repository = vpn_repository
    
    @staticmethod
    def generate_api_key() -> str:
        """Generate a new agent API key.
        
        Format: agent_<32 hex characters>
        Example: agent_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
        """
        import secrets
        return f"agent_{secrets.token_hex(16)}"
    
    @staticmethod
    def hash_api_key(api_key: str) -> str:
        """Hash API key using SHA-256."""
        return hashlib.sha256(api_key.encode()).hexdigest()
    
    async def create_api_key(
        self,
        description: Optional[str] = None,
        expires_in_days: Optional[int] = None,
        created_by: Optional[uuid.UUID] = None,
    ) -> tuple[str, AgentApiKeyModel]:
        """Generate and store a new API key.
        
        Returns:
            Tuple of (plain_text_key, api_key_model)
            Store plain_text_key securely - it cannot be recovered!
        """
        # Generate key
        plain_key = self.generate_api_key()
        key_hash = self.hash_api_key(plain_key)
        
        # Calculate expiration
        expires_at = None
        if expires_in_days:
            expires_at = datetime.utcnow() + timedelta(days=expires_in_days)
        
        # Store in database
        api_key = await self.api_key_repository.create(
            api_key_hash=key_hash,
            description=description,
            expires_at=expires_at,
            created_by=created_by,
        )
        
        logger.info(f"Created new agent API key: {api_key.id}")
        
        return plain_key, api_key
    
    async def validate_api_key(self, api_key: str) -> Optional[AgentApiKeyModel]:
        """Validate an API key.
        
        Returns:
            AgentApiKeyModel if valid, None if invalid/revoked/expired
        """
        key_hash = self.hash_api_key(api_key)
        api_key_model = await self.api_key_repository.get_by_hash(key_hash)
        
        if api_key_model is None:
            logger.warning(f"Invalid API key attempt: {key_hash[:8]}...")
            return None
        
        # Check status
        if api_key_model.status == "revoked":
            logger.warning(f"Revoked API key used: {api_key_model.id}")
            return None
        
        # Check expiration
        if api_key_model.expires_at and datetime.utcnow() > api_key_model.expires_at:
            logger.warning(f"Expired API key used: {api_key_model.id}")
            # Mark as expired
            api_key_model.status = "expired"
            await self.session.commit()
            return None
        
        return api_key_model
    
    async def register_agent(
        self,
        api_key: str,
        hostname: str,
        ip_address: str,
        country_code: str,
        country_name: str,
        agent_version: str,
        os_type: str,
        os_arch: str,
        agent_url: str,
        supports_outline: bool = True,
        supports_wireguard: bool = True,
        region: Optional[str] = None,
        city: Optional[str] = None,
    ) -> VpnServerModel:
        """Register a new agent and create server record.
        
        Raises:
            ValueError: If API key is invalid or already used
        """
        # Validate API key
        api_key_model = await self.validate_api_key(api_key)
        if api_key_model is None:
            raise ValueError("Invalid or expired API key")
        
        # Check if already used
        if api_key_model.status == "used":
            raise ValueError("API key already used for registration")
        
        # Check if server already exists for this key
        if api_key_model.server_id:
            # Return existing server
            server = await self.vpn_repository.get_by_id(api_key_model.server_id)
            if server:
                return server
        
        # Create server record
        import uuid as uuid_lib
        server_id = uuid_lib.uuid4()
        
        server = VpnServerModel(
            id=server_id,
            name=f"{country_name} - {hostname}",
            country_code=country_code,
            country_name=country_name,
            city=city,
            region=region,
            agent_url=agent_url,
            agent_api_key=api_key,  # Store for backward compatibility
            supports_outline=supports_outline,
            supports_wireguard=supports_wireguard,
            status="online",
            agent_version=agent_version,
            os_type=os_type,
            os_arch=os_arch,
            last_registration_ip=ip_address,
        )
        
        self.session.add(server)
        await self.session.flush()  # Get server ID
        
        # Mark API key as used
        await self.api_key_repository.mark_as_used(api_key_model, server_id)
        
        logger.info(
            f"Registered new agent: {server_id} ({hostname}) from {ip_address}"
        )
        
        return server
```

**Step 2: Commit**

```bash
git add src/core/application/services/agent_registration_service.py
git commit -m "feat(service): add AgentRegistrationService for auto-registration"
```

---

## Phase 3: Backend API Routes

### Task 3.1: Create Agent Registration Routes

**Files:**
- Create: `usipipo-backend/src/infrastructure/api/v1/routes/agent_registration.py`
- Create: `usipipo-backend/src/shared/schemas/agent_registration.py`

**Step 1: Create Pydantic schemas**

```python
"""Pydantic schemas for agent registration."""

from typing import Optional

from pydantic import BaseModel, Field, HttpUrl


class AgentRegistrationRequest(BaseModel):
    """Request schema for agent registration."""
    
    hostname: str = Field(..., description="Server hostname", max_length=255)
    ip_address: str = Field(..., description="Public IP address", max_length=45)
    country_code: str = Field(..., description="ISO 3166-1 alpha-2 country code", min_length=2, max_length=2)
    country_name: str = Field(..., description="Country name", max_length=100)
    region: Optional[str] = Field(None, description="Region/state", max_length=100)
    city: Optional[str] = Field(None, description="City", max_length=100)
    agent_version: str = Field(..., description="Agent version", max_length=20)
    os_type: str = Field(..., description="Operating system", max_length=50)
    os_arch: str = Field(..., description="Architecture", max_length=20)
    agent_url: HttpUrl = Field(..., description="Agent API URL")
    supports_outline: bool = Field(True, description="Supports Outline VPN")
    supports_wireguard: bool = Field(True, description="Supports WireGuard")
    agent_api_key: str = Field(..., description="Agent API key", max_length=255)


class AgentRegistrationResponse(BaseModel):
    """Response schema for agent registration."""
    
    server_id: str
    status: str
    message: Optional[str] = None


class GenerateApiKeyRequest(BaseModel):
    """Request schema for generating API keys."""
    
    description: Optional[str] = Field(None, max_length=255)
    expires_in_days: Optional[int] = Field(None, ge=1, le=3650)


class GenerateApiKeyResponse(BaseModel):
    """Response schema for generated API key."""
    
    id: str
    api_key: str
    status: str
    created_at: str
    expires_at: Optional[str] = None


class ApiKeyListResponse(BaseModel):
    """Response schema for API key list."""
    
    keys: list[dict]
    total: int
```

**Step 2: Create API routes**

```python
"""API routes for agent registration."""

import uuid
from datetime import datetime

from fastapi import APIRouter, Depends, HTTPException, Query, status
from sqlalchemy.ext.asyncio import AsyncSession

from src.core.application.services.agent_registration_service import (
    AgentRegistrationService,
)
from src.infrastructure.persistence.database import get_db
from src.infrastructure.persistence.repositories.agent_api_key_repository import (
    AgentApiKeyRepository,
)
from src.infrastructure.persistence.repositories.vpn_repository import VpnRepository
from src.shared.schemas.agent_registration import (
    AgentRegistrationRequest,
    AgentRegistrationResponse,
    ApiKeyListResponse,
    GenerateApiKeyRequest,
    GenerateApiKeyResponse,
)

router = APIRouter(prefix="/servers", tags=["Agent Registration"])


def get_registration_service(db: AsyncSession = Depends(get_db)) -> AgentRegistrationService:
    """Get agent registration service instance."""
    return AgentRegistrationService(
        session=db,
        api_key_repository=AgentApiKeyRepository(db),
        vpn_repository=VpnRepository(db),
    )


@router.post(
    "/register-agent",
    response_model=AgentRegistrationResponse,
    status_code=status.HTTP_201_CREATED,
)
async def register_agent(
    request: AgentRegistrationRequest,
    service: AgentRegistrationService = Depends(get_registration_service),
):
    """Register a new VPN agent.
    
    **Authentication:** Requires X-API-Key header (agent API key)
    
    **Process:**
    1. Validate API key
    2. Create server record with metadata
    3. Mark API key as used
    4. Return server UUID
    
    **Idempotency:** Same API key returns existing server (409 if different metadata)
    """
    try:
        server = await service.register_agent(
            api_key=request.agent_api_key,
            hostname=request.hostname,
            ip_address=request.ip_address,
            country_code=request.country_code,
            country_name=request.country_name,
            agent_version=request.agent_version,
            os_type=request.os_type,
            os_arch=request.os_arch,
            agent_url=str(request.agent_url),
            supports_outline=request.supports_outline,
            supports_wireguard=request.supports_wireguard,
            region=request.region,
            city=request.city,
        )
        
        return AgentRegistrationResponse(
            server_id=str(server.id),
            status="registered",
            message="Server registered successfully",
        )
    
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e),
        )


@router.get(
    "/register-agent",
    response_model=AgentRegistrationResponse,
)
async def check_agent_registration(
    api_key: str,
    service: AgentRegistrationService = Depends(get_registration_service),
):
    """Check if an API key has been used for registration.
    
    Returns existing server_id if registered, or error if not yet registered.
    """
    api_key_model = await service.validate_api_key(api_key)
    
    if api_key_model is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid API key",
        )
    
    if api_key_model.status == "active":
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="API key not yet used for registration",
        )
    
    if api_key_model.server_id:
        return AgentRegistrationResponse(
            server_id=str(api_key_model.server_id),
            status="registered",
        )
    
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail="Server ID not found",
    )
```

**Step 3: Create admin routes for API key management**

**Create:** `usipipo-backend/src/infrastructure/api/v1/routes/admin_agent_keys.py`

```python
"""Admin routes for managing agent API keys."""

from typing import Optional

from fastapi import APIRouter, Depends, HTTPException, Query, status

from src.core.application.services.agent_registration_service import (
    AgentRegistrationService,
)
from src.infrastructure.persistence.database import get_db
from src.infrastructure.persistence.repositories.agent_api_key_repository import (
    AgentApiKeyRepository,
)
from src.shared.schemas.agent_registration import (
    ApiKeyListResponse,
    GenerateApiKeyRequest,
    GenerateApiKeyResponse,
)

router = APIRouter(prefix="/admin/agent-api-keys", tags=["Admin - Agent Keys"])


def get_service(db: AsyncSession = Depends(get_db)) -> AgentRegistrationService:
    """Get registration service."""
    return AgentRegistrationService(
        session=db,
        api_key_repository=AgentApiKeyRepository(db),
        vpn_repository=None,  # Not needed for key generation
    )


@router.post("", response_model=GenerateApiKeyResponse, status_code=status.HTTP_201_CREATED)
async def generate_api_key(
    request: GenerateApiKeyRequest,
    service: AgentRegistrationService = Depends(get_service),
    # TODO: Add admin authentication dependency
):
    """Generate a new agent API key.
    
    **Authentication:** Requires admin JWT token
    
    **Security:** Generated key is only shown once - store it securely!
    """
    plain_key, api_key_model = await service.create_api_key(
        description=request.description,
        expires_in_days=request.expires_in_days,
        # TODO: Pass created_by from admin auth
    )
    
    return GenerateApiKeyResponse(
        id=str(api_key_model.id),
        api_key=plain_key,
        status=api_key_model.status,
        created_at=api_key_model.created_at.isoformat(),
        expires_at=api_key_model.expires_at.isoformat() if api_key_model.expires_at else None,
    )


@router.get("", response_model=ApiKeyListResponse)
async def list_api_keys(
    status: Optional[str] = Query(None, pattern="^(active|used|revoked|expired)$"),
    limit: int = Query(50, ge=1, le=100),
    offset: int = Query(0, ge=0),
    service: AgentRegistrationService = Depends(get_service),
):
    """List agent API keys with optional filtering."""
    # TODO: Implement list method in service
    return ApiKeyListResponse(keys=[], total=0)
```

**Step 4: Update main.py to include new routers**

**Modify:** `usipipo-backend/src/main.py`

```python
# Add imports at top
from .infrastructure.api.v1.routes.agent_registration import (
    router as agent_registration_router,
)
from .infrastructure.api.v1.routes.admin_agent_keys import (
    router as admin_agent_keys_router,
)

# Add router includes (after existing includes)
app.include_router(agent_registration_router, prefix=api_prefix)
app.include_router(admin_agent_keys_router, prefix=api_prefix)
```

**Step 5: Commit**

```bash
git add src/infrastructure/api/v1/routes/agent_registration.py
git add src/infrastructure/api/v1/routes/admin_agent_keys.py
git add src/shared/schemas/agent_registration.py
git add src/main.py
git commit -m "feat(api): add agent registration and admin API key endpoints"
```

---

## Phase 4: Backend Tests

### Task 4.1: Create Integration Tests

**Files:**
- Create: `usipipo-backend/tests/integration/test_agent_registration.py`

**Step 1: Create test file**

```python
"""Integration tests for agent registration flow."""

import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_register_agent_with_valid_key(client: AsyncClient):
    """Test successful agent registration."""
    # Generate API key first
    gen_response = await client.post(
        "/api/v1/admin/agent-api-keys",
        json={"description": "Test key"},
    )
    api_key = gen_response.json()["api_key"]
    
    # Register agent
    response = await client.post(
        "/api/v1/servers/register-agent",
        headers={"X-API-Key": api_key},
        json={
            "hostname": "test-vps-1",
            "ip_address": "192.168.1.1",
            "country_code": "US",
            "country_name": "United States",
            "region": "Virginia",
            "city": "Ashburn",
            "agent_version": "0.2.0",
            "os_type": "linux",
            "os_arch": "amd64",
            "agent_url": "http://test-agent.duckdns.org:8080",
            "supports_outline": True,
            "supports_wireguard": True,
            "agent_api_key": api_key,
        },
    )
    
    assert response.status_code == 201
    data = response.json()
    assert data["status"] == "registered"
    assert "server_id" in data


@pytest.mark.asyncio
async def test_register_agent_with_invalid_key(client: AsyncClient):
    """Test registration with invalid API key."""
    response = await client.post(
        "/api/v1/servers/register-agent",
        headers={"X-API-Key": "agent_invalid_key"},
        json={
            "hostname": "test-vps-2",
            "ip_address": "192.168.1.2",
            "country_code": "US",
            "country_name": "United States",
            "agent_version": "0.2.0",
            "os_type": "linux",
            "os_arch": "amd64",
            "agent_url": "http://test-agent-2.duckdns.org:8080",
            "agent_api_key": "agent_invalid_key",
        },
    )
    
    assert response.status_code == 400
    assert "Invalid" in response.json()["detail"]


@pytest.mark.asyncio
async def test_register_agent_duplicate_key(client: AsyncClient):
    """Test that same API key cannot be used twice."""
    # Generate and use key first time
    gen_response = await client.post(
        "/api/v1/admin/agent-api-keys",
        json={"description": "Test key 2"},
    )
    api_key = gen_response.json()["api_key"]
    
    # First registration
    await client.post(
        "/api/v1/servers/register-agent",
        headers={"X-API-Key": api_key},
        json={
            "hostname": "test-vps-3",
            "ip_address": "192.168.1.3",
            "country_code": "US",
            "country_name": "United States",
            "agent_version": "0.2.0",
            "os_type": "linux",
            "os_arch": "amd64",
            "agent_url": "http://test-agent-3.duckdns.org:8080",
            "agent_api_key": api_key,
        },
    )
    
    # Second registration should fail
    response = await client.post(
        "/api/v1/servers/register-agent",
        headers={"X-API-Key": api_key},
        json={
            "hostname": "test-vps-4",
            "ip_address": "192.168.1.4",
            "country_code": "DE",
            "country_name": "Germany",
            "agent_version": "0.2.0",
            "os_type": "linux",
            "os_arch": "amd64",
            "agent_url": "http://test-agent-4.duckdns.org:8080",
            "agent_api_key": api_key,
        },
    )
    
    assert response.status_code == 400
    assert "already used" in response.json()["detail"]
```

**Step 2: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-backend
pytest tests/integration/test_agent_registration.py -v
```

Expected: 3 tests passing

**Step 3: Commit**

```bash
git add tests/integration/test_agent_registration.py
git commit -m "test(integration): add agent registration integration tests"
```

---

## Phase 5: Agent Implementation

### Task 5.1: Create Agent Registrar

**Files:**
- Create: `usipipo-agent/internal/registrar/registrar.go`
- Create: `usipipo-agent/internal/utils/geoip/geoip.go`

**Step 1: Create GeoIP utility**

```go
package geoip

import (
	"encoding/json"
	"fmt"
	
	"github.com/go-resty/resty/v2"
)

// GeoIPResponse represents the response from ip-api.com
type GeoIPResponse struct {
	Query       string `json:"query"`
	CountryCode string `json:"countryCode"`
	CountryName string `json:"countryName"`
	RegionName  string `json:"regionName"`
	City        string `json:"city"`
}

// GetLocation fetches public IP and geo location
func GetLocation(client *resty.Client) (*GeoIPResponse, error) {
	resp, err := client.R().
		Get("http://ip-api.com/json/")
	
	if err != nil {
		return nil, fmt.Errorf("failed to fetch geo location: %w", err)
	}
	
	if resp.StatusCode() != 200 {
		return nil, fmt.Errorf("geo API returned status %d", resp.StatusCode())
	}
	
	var geo GeoIPResponse
	if err := json.Unmarshal(resp.Body(), &geo); err != nil {
		return nil, fmt.Errorf("failed to parse geo response: %w", err)
	}
	
	return &geo, nil
}
```

**Step 2: Create registrar**

```go
package registrar

import (
	"encoding/json"
	"fmt"
	"os"
	"runtime"
	
	"github.com/go-resty/resty/v2"
	"github.com/uSipipo-Team/usipipo-agent/internal/config"
	"github.com/uSipipo-Team/usipipo-agent/internal/utils/geoip"
)

// Registrar handles agent registration with backend
type Registrar struct {
	backendURL string
	apiKey     string
	serverID   string
	client     *resty.Client
}

// RegistrationResponse represents backend response
type RegistrationResponse struct {
	ServerID string `json:"server_id"`
	Status   string `json:"status"`
	Message  string `json:"message,omitempty"`
}

// RegistrationRequest represents registration payload
type RegistrationRequest struct {
	Hostname          string `json:"hostname"`
	IPAddress         string `json:"ip_address"`
	CountryCode       string `json:"country_code"`
	CountryName       string `json:"country_name"`
	Region            string `json:"region,omitempty"`
	City              string `json:"city,omitempty"`
	AgentVersion      string `json:"agent_version"`
	OSType            string `json:"os_type"`
	OSArch            string `json:"os_arch"`
	AgentURL          string `json:"agent_url"`
	SupportsOutline   bool   `json:"supports_outline"`
	SupportsWireGuard bool   `json:"supports_wireguard"`
	AgentAPIKey       string `json:"agent_api_key"`
}

// NewRegistrar creates a new registrar instance
func NewRegistrar(cfg *config.Config) *Registrar {
	return &Registrar{
		backendURL: cfg.BackendURL,
		apiKey:     cfg.APIKey,
		serverID:   cfg.ServerID,
		client:     resty.New(),
	}
}

// RegisterOrGetServerID registers agent or returns existing server ID
func (r *Registrar) RegisterOrGetServerID() (string, error) {
	// If server ID already set and valid UUID, use it
	if r.serverID != "" && isValidUUID(r.serverID) {
		return r.serverID, nil
	}
	
	// Collect metadata
	metadata, err := r.collectMetadata()
	if err != nil {
		return "", fmt.Errorf("failed to collect metadata: %w", err)
	}
	
	// Send registration request
	endpoint := fmt.Sprintf("%s/api/v1/servers/register-agent", r.backendURL)
	
	resp, err := r.client.R().
		SetHeader("X-API-Key", r.apiKey).
		SetHeader("Content-Type", "application/json").
		SetBody(metadata).
		Post(endpoint)
	
	if err != nil {
		return "", fmt.Errorf("registration request failed: %w", err)
	}
	
	if resp.StatusCode() == 409 {
		// Already registered, get existing server ID
		return r.getExistingServerID()
	}
	
	if resp.StatusCode() != 201 {
		return "", fmt.Errorf("registration failed with status: %d - %s", resp.StatusCode(), resp.String())
	}
	
	// Parse response
	var result RegistrationResponse
	if err := json.Unmarshal(resp.Body(), &result); err != nil {
		return "", fmt.Errorf("failed to parse response: %w", err)
	}
	
	// Save server ID to .env
	if err := saveServerIDToEnv(result.ServerID); err != nil {
		// Log but don't fail - agent can still function
		fmt.Printf("Warning: Could not save SERVER_ID to .env: %v\n", err)
	}
	
	return result.ServerID, nil
}

func (r *Registrar) collectMetadata() (*RegistrationRequest, error) {
	hostname, err := os.Hostname()
	if err != nil {
		hostname = "unknown"
	}
	
	// Get geo location
	geo, err := geoip.GetLocation(r.client)
	if err != nil {
		// Use defaults if geo lookup fails
		geo = &geoip.GeoIPResponse{
			Query:       "unknown",
			CountryCode: "XX",
			CountryName: "Unknown",
			RegionName:  "Unknown",
			City:        "Unknown",
		}
	}
	
	cfg := config.Load()
	
	return &RegistrationRequest{
		Hostname:          hostname,
		IPAddress:         geo.Query,
		CountryCode:       geo.CountryCode,
		CountryName:       geo.CountryName,
		Region:            geo.RegionName,
		City:              geo.City,
		AgentVersion:      getVersion(),
		OSType:            runtime.GOOS,
		OSArch:            runtime.GOARCH,
		AgentURL:          cfg.AgentURL,
		SupportsOutline:   cfg.OutlineAPIURL != "",
		SupportsWireGuard: cfg.WireGuardInterface != "",
		AgentAPIKey:       r.apiKey,
	}, nil
}

func (r *Registrar) getExistingServerID() (string, error) {
	endpoint := fmt.Sprintf("%s/api/v1/servers/register-agent?api_key=%s", r.backendURL, r.apiKey)
	
	resp, err := r.client.R().Get(endpoint)
	if err != nil {
		return "", err
	}
	
	if resp.StatusCode() != 200 {
		return "", fmt.Errorf("failed to get existing registration: %d", resp.StatusCode())
	}
	
	var result RegistrationResponse
	if err := json.Unmarshal(resp.Body(), &result); err != nil {
		return "", err
	}
	
	return result.ServerID, nil
}

// Helper functions
func isValidUUID(uuid string) bool {
	// Simple UUID format check
	if len(uuid) != 36 {
		return false
	}
	for i, r := range uuid {
		if i == 8 || i == 13 || i == 18 || i == 23 {
			if r != '-' {
				return false
			}
			continue
		}
		if !((r >= '0' && r <= '9') || (r >= 'a' && r <= 'f') || (r >= 'A' && r <= 'F')) {
			return false
		}
	}
	return true
}

func getVersion() string {
	// Set via ldflags during build: -X main.Version=0.2.0
	return Version
}

var Version = "0.2.0-dev"

func saveServerIDToEnv(serverID string) error {
	envPath := "/opt/usipipo-agent/.env"
	
	// Read existing .env
	content, err := os.ReadFile(envPath)
	if err != nil {
		return err
	}
	
	// Update SERVER_ID line
	lines := string(content)
	// Simple replacement - can be improved with proper env parsing
	lines = replaceEnvVar(lines, "SERVER_ID", serverID)
	
	// Write back
	return os.WriteFile(envPath, []byte(lines), 0600)
}

func replaceEnvVar(content, key, value string) string {
	// Replace or add SERVER_ID line
	lines := strings.Split(content, "\n")
	found := false
	
	for i, line := range lines {
		if strings.HasPrefix(line, key+"=") {
			lines[i] = key + "=" + value
			found = true
			break
		}
	}
	
	if !found {
		lines = append(lines, key+"="+value)
	}
	
	return strings.Join(lines, "\n")
}
```

**Step 3: Add missing import**

```go
import (
	"strings"
	// ... other imports
)
```

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-agent
git add internal/registrar/registrar.go
git add internal/utils/geoip/geoip.go
git commit -m "feat(agent): add registrar for auto-registration with backend"
```

---

## Phase 6: Update Agent Reporter

### Task 6.1: Integrate Registration into Reporter

**Files:**
- Modify: `usipipo-agent/internal/reporter/reporter.go`

**Step 1: Update sendMetrics to call registration**

```go
// Modify sendMetrics function in reporter.go

func (r *Reporter) sendMetrics() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	
	// Ensure we have a valid server_id
	if r.serverID == "" || !isValidUUID(r.serverID) {
		log.Println("Server ID not set or invalid, attempting registration...")
		
		registrar := registrar.NewRegistrar(r.collector.GetConfig())
		serverID, err := registrar.RegisterOrGetServerID()
		if err != nil {
			log.Printf("Failed to register: %v", err)
			return
		}
		
		r.serverID = serverID
		log.Printf("Registered with server_id: %s", serverID)
	}
	
	// Collect metrics
	m, err := r.collector.GetMetrics(ctx)
	if err != nil {
		log.Printf("Failed to collect metrics: %v", err)
		return
	}
	
	// Send metrics
	endpoint := fmt.Sprintf("%s/api/v1/metrics/agents/%s", r.backendURL, r.serverID)
	
	resp, err := r.client.R().
		SetContext(ctx).
		SetHeader("X-API-Key", r.apiKey).
		SetBody(m).
		Post(endpoint)
	
	if err != nil {
		log.Printf("Failed to send metrics: %v", err)
		return
	}
	
	if resp.StatusCode() != 200 {
		log.Printf("Unexpected status from backend: %d", resp.StatusCode())
		return
	}
	
	log.Printf("Metrics sent successfully to backend")
}
```

**Step 2: Add UUID validation helper**

```go
func isValidUUID(uuid string) bool {
	if len(uuid) != 36 {
		return false
	}
	for i, r := range uuid {
		if i == 8 || i == 13 || i == 18 || i == 23 {
			if r != '-' {
				return false
			}
			continue
		}
		if !((r >= '0' && r <= '9') || (r >= 'a' && r <= 'f') || (r >= 'A' && r <= 'F')) {
			return false
		}
	}
	return true
}
```

**Step 3: Commit**

```bash
git add internal/reporter/reporter.go
git commit -m "feat(reporter): integrate auto-registration before sending metrics"
```

---

## Phase 7: Configuration and Deployment

### Task 7.1: Update Agent Configuration

**Files:**
- Modify: `usipipo-agent/internal/config/config.go`
- Modify: `usipipo-agent/.env.example`

**Step 1: Update config struct**

```go
// Add to Config struct in config.go
type Config struct {
	// ... existing fields ...
	
	// Agent metadata
	AgentURL          string
	SupportsOutline   bool
	SupportsWireGuard bool
}

// Update Load function to read new env vars
func Load() *Config {
	return &Config{
		// ... existing ...
		
		AgentURL:          getEnv("AGENT_URL", "http://localhost:8080"),
		SupportsOutline:   getEnv("SUPPORTS_OUTLINE", "true") == "true",
		SupportsWireGuard: getEnv("SUPPORTS_WIREGUARD", "true") == "true",
	}
}
```

**Step 2: Update .env.example**

```bash
# uSipipo VPN Agent Configuration

# Agent API Key (pre-generated from backend admin)
AGENT_API_KEY=agent_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6

# Server ID (auto-filled after registration, leave empty initially)
SERVER_ID=

# Backend URL
BACKEND_URL=http://usipipo.duckdns.org

# Agent metadata
AGENT_URL=http://localhost:8080
SUPPORTS_OUTLINE=true
SUPPORTS_WIREGUARD=true

# VPN Configuration
OUTLINE_API_URL=...
WG_INTERFACE=wg0
```

**Step 3: Commit**

```bash
git add internal/config/config.go
git add .env.example
git commit -m "feat(config): add agent metadata configuration"
```

---

## Phase 8: Testing and Documentation

### Task 8.1: Create Deployment Guide

**Files:**
- Create: `usipipo-docs/agent/AUTO-REGISTRATION-GUIDE.md`

**Step 1: Create guide**

```markdown
# Agent Auto-Registration Guide

## Overview

Agents now automatically register with the backend on first startup. This eliminates manual database entries.

## Admin Setup

### 1. Generate API Key

```bash
curl -X POST https://usipipo.duckdns.org/api/v1/admin/agent-api-keys \
  -H "Authorization: Bearer <admin_jwt>" \
  -H "Content-Type: application/json" \
  -d '{"description": "USA East VPS #1", "expires_in_days": 365}'
```

Response:
```json
{
  "api_key": "agent_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "status": "active"
}
```

**⚠️ Save the `api_key` - it's only shown once!**

### 2. Install Agent

On VPS:
```bash
# Download and install agent
/opt/usipipo-agent/install.sh

# Configure .env
sudo nano /opt/usipipo-agent/.env
```

Set:
```env
AGENT_API_KEY=agent_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
SERVER_ID=  # Leave empty - will be auto-filled
BACKEND_URL=http://usipipo.duckdns.org
AGENT_URL=http://usipipousa.duckdns.org:8080
```

### 3. Start Agent

```bash
sudo systemctl start usipipo-agent
sudo journalctl -u usipipo-agent -f
```

Expected logs:
```
Server ID not set or invalid, attempting registration...
Registered with server_id: 1bc5c426-29de-4440-9ec6-ada7866e2c08
Metrics sent successfully to backend
```

### 4. Verify Registration

```bash
# Check .env was updated
sudo cat /opt/usipipo-agent/.env | grep SERVER_ID
# Expected: SERVER_ID=1bc5c426-...

# Check backend
curl https://usipipo.duckdns.org/api/v1/admin/servers \
  -H "Authorization: Bearer <admin_jwt>"
```

## Troubleshooting

### Registration fails with "Invalid API key"
- Check API key is correct in .env
- Verify key not already used
- Check key not expired

### Registration fails with "Already used"
- API key can only register once
- Generate new key for this VPS
- Or reuse existing server_id from .env

### SERVER_ID not saved to .env
- Check file permissions: `ls -la /opt/usipipo-agent/.env`
- Agent runs as `usipipo` user - needs write access
- Metrics still sent, but re-registers on each restart
```

**Step 2: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-docs
git add agent/AUTO-REGISTRATION-GUIDE.md
git commit -m "docs: add agent auto-registration guide"
```

---

## ✅ Implementation Complete

### Summary of Changes

**Backend:**
- ✅ Database migration (agent_api_keys table)
- ✅ Models (AgentApiKeyModel, VpnServerModel updates)
- ✅ Repository (AgentApiKeyRepository)
- ✅ Service (AgentRegistrationService)
- ✅ API Routes (register-agent, admin API keys)
- ✅ Integration tests

**Agent:**
- ✅ Registrar (registration logic)
- ✅ GeoIP utility (auto-detect location)
- ✅ Reporter integration (auto-register before metrics)
- ✅ Config updates (metadata env vars)
- ✅ Env file writer (persist server_id)

**Documentation:**
- ✅ Auto-registration guide
- ✅ API documentation (via OpenAPI)

---

## 🚀 Next Steps

**After implementation:**

1. **Deploy backend:**
   ```bash
   cd /home/mowgli/usipipo/usipipo-backend
   alembic upgrade head
   sudo systemctl restart usipipo-backend
   ```

2. **Generate test API key:**
   ```bash
   curl -X POST https://usipipo.duckdns.org/api/v1/admin/agent-api-keys \
     -H "Authorization: Bearer <admin_jwt>" \
     -d '{"description": "Test VPS"}'
   ```

3. **Update agent .env:**
   ```bash
   sudo nano /opt/usipipo-agent/.env
   # Set AGENT_API_KEY=<generated_key>
   # Clear SERVER_ID=
   ```

4. **Restart agent and verify:**
   ```bash
   sudo systemctl restart usipipo-agent
   sudo journalctl -u usipipo-agent -f
   # Should see: "Registered with server_id: ..."
   ```

5. **Check database:**
   ```bash
   PGPASSWORD=... psql -U ... -c "SELECT * FROM agent_api_keys ORDER BY created_at DESC LIMIT 1;"
   ```

---

**Plan complete! Ready for execution.**
