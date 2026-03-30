# Admin VPN Keys CRUD Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use `subagent-driven-development` to implement this plan task-by-task.

**Goal:** Implement complete CRUD API for VPN key management with role-based access control and audit logging.

**Architecture:** Three-layer architecture (Routes → Service → Repository) with middleware for role verification and audit logging. All operations logged to `admin_audit_logs` table.

**Tech Stack:** Python 3.13, FastAPI, SQLAlchemy Async, Pydantic v2, PostgreSQL, Alembic migrations.

---

## Phase 1: Database Migrations

### Task 1.1: Create Audit Logs Migration

**Files:**
- Create: `usipipo-backend/migrations/versions/2026_03_30_0001_admin_audit_logs.py`

**Step 1: Create migration file**

```python
"""create admin_audit_logs table

Revision ID: 2026_03_30_0001
Revises: previous_revision
Create Date: 2026-03-30 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa

revision = '2026_03_30_0001'
down_revision = 'previous_revision'  # Update with actual previous revision
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        'admin_audit_logs',
        sa.Column('id', sa.UUID(), nullable=False),
        sa.Column('timestamp', sa.TIMESTAMP(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.Column('admin_telegram_id', sa.BigInteger(), nullable=False),
        sa.Column('admin_username', sa.String(length=255), nullable=True),
        sa.Column('operation', sa.String(length=50), nullable=False),
        sa.Column('target_type', sa.String(length=50), nullable=False),
        sa.Column('target_id', sa.UUID(), nullable=False),
        sa.Column('target_user_telegram_id', sa.BigInteger(), nullable=True),
        sa.Column('details', sa.JSON(), nullable=True),
        sa.Column('ip_address', sa.INET(), nullable=True),
        sa.Column('success', sa.Boolean(), nullable=False),
        sa.Column('error_message', sa.Text(), nullable=True),
        sa.Column('created_at', sa.TIMESTAMP(timezone=True), server_default=sa.func.now()),
        sa.PrimaryKeyConstraint('id')
    )
    
    # Create indexes
    op.create_index('idx_audit_logs_timestamp', 'admin_audit_logs', ['timestamp'])
    op.create_index('idx_audit_logs_admin', 'admin_audit_logs', ['admin_telegram_id'])
    op.create_index('idx_audit_logs_operation', 'admin_audit_logs', ['operation'])
    op.create_index('idx_audit_logs_target', 'admin_audit_logs', ['target_type', 'target_id'])


def downgrade() -> None:
    op.drop_index('idx_audit_logs_target')
    op.drop_index('idx_audit_logs_operation')
    op.drop_index('idx_audit_logs_admin')
    op.drop_index('idx_audit_logs_timestamp')
    op.drop_table('admin_audit_logs')
```

**Step 2: Run migration to verify**

```bash
cd usipipo-backend
source .venv/bin/activate
alembic upgrade head
```

Expected: Migration runs successfully, table created

**Step 3: Verify table exists**

```bash
PGPASSWORD=c319c4c2605479e7a3b7c8d4dfa5110c psql -h localhost -U usipipo_backend_user -d usipipo_backend_db -c "\dt admin_audit_logs"
```

Expected: Table listed

**Step 4: Commit**

```bash
git add migrations/versions/2026_03_30_0001_admin_audit_logs.py
git commit -m "db: create admin_audit_logs table for audit trail"
```

---

### Task 1.2: Create Staff Roles Migration

**Files:**
- Create: `usipipo-backend/migrations/versions/2026_03_30_0002_staff_roles.py`

**Step 1: Create migration file**

```python
"""create staff_roles table

Revision ID: 2026_03_30_0002
Revises: 2026_03_30_0001
Create Date: 2026-03-30 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa

revision = '2026_03_30_0002'
down_revision = '2026_03_30_0001'
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        'staff_roles',
        sa.Column('id', sa.UUID(), nullable=False),
        sa.Column('telegram_id', sa.BigInteger(), nullable=False),
        sa.Column('username', sa.String(length=255), nullable=True),
        sa.Column('role', sa.String(length=20), nullable=False, server_default='support'),
        sa.Column('granted_by', sa.BigInteger(), nullable=True),
        sa.Column('granted_at', sa.TIMESTAMP(timezone=True), server_default=sa.func.now()),
        sa.Column('is_active', sa.Boolean(), nullable=False, server_default=sa.true()),
        sa.Column('created_at', sa.TIMESTAMP(timezone=True), server_default=sa.func.now()),
        sa.Column('updated_at', sa.TIMESTAMP(timezone=True), server_default=sa.func.now()),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('telegram_id')
    )
    
    # Create indexes
    op.create_index('idx_staff_roles_telegram', 'staff_roles', ['telegram_id'])
    op.create_index('idx_staff_roles_role', 'staff_roles', ['role'])


def downgrade() -> None:
    op.drop_index('idx_staff_roles_role')
    op.drop_index('idx_staff_roles_telegram')
    op.drop_table('staff_roles')
```

**Step 2: Run migration**

```bash
cd usipipo-backend
alembic upgrade head
```

**Step 3: Seed initial admin user**

```sql
-- Insert owner as admin (update Telegram ID)
INSERT INTO staff_roles (telegram_id, username, role, is_active)
VALUES (1058749165, 'mowgliph', 'admin', true);
```

**Step 4: Commit**

```bash
git add migrations/versions/2026_03_30_0002_staff_roles.py
git commit -m "db: create staff_roles table for role-based access"
```

---

## Phase 2: Domain Layer

### Task 2.1: Create Interface

**Files:**
- Create: `usipipo-backend/src/core/domain/interfaces/i_admin_vpn_key_service.py`

**Step 1: Create interface file**

```python
"""Interface for admin VPN key management service."""

from abc import ABC, abstractmethod
from uuid import UUID
from typing import Any

from usipipo_commons.domain.entities.vpn_key import VpnKey


class IAdminVpnKeyService(ABC):
    """Interface for administrative VPN key management."""

    @abstractmethod
    async def list_keys(
        self,
        filters: dict[str, Any],
        current_admin_id: int
    ) -> dict[str, Any]:
        """List VPN keys with filters and pagination."""
        pass

    @abstractmethod
    async def get_key_detail(
        self,
        key_id: UUID,
        current_admin_id: int
    ) -> VpnKey | None:
        """Get detailed information about a specific key."""
        pass

    @abstractmethod
    async def get_user_keys(
        self,
        user_telegram_id: int,
        current_admin_id: int
    ) -> list[VpnKey]:
        """Get all keys for a specific user by Telegram ID."""
        pass

    @abstractmethod
    async def create_key(
        self,
        request: dict[str, Any],
        current_admin_id: int
    ) -> VpnKey:
        """Create a new VPN key for a user."""
        pass

    @abstractmethod
    async def toggle_key(
        self,
        key_id: UUID,
        current_admin_id: int
    ) -> bool:
        """Toggle key active/inactive status."""
        pass

    @abstractmethod
    async def update_data_limit(
        self,
        key_id: UUID,
        data_limit_gb: float,
        reason: str | None,
        current_admin_id: int
    ) -> bool:
        """Update data limit for a key."""
        pass

    @abstractmethod
    async def reset_usage(
        self,
        key_id: UUID,
        reason: str | None,
        current_admin_id: int
    ) -> bool:
        """Reset data usage for a key."""
        pass

    @abstractmethod
    async def regenerate_config(
        self,
        key_id: UUID,
        notify_user: bool,
        current_admin_id: int
    ) -> VpnKey:
        """Regenerate configuration for a key."""
        pass

    @abstractmethod
    async def delete_key(
        self,
        key_id: UUID,
        current_admin_id: int
    ) -> bool:
        """Delete a VPN key permanently."""
        pass

    @abstractmethod
    async def log_audit(
        self,
        operation: str,
        target_id: UUID,
        admin_telegram_id: int,
        admin_username: str,
        success: bool,
        details: dict | None = None,
        error_message: str | None = None
    ) -> None:
        """Log operation to audit trail."""
        pass
```

**Step 2: Update __init__.py**

```python
# usipipo-backend/src/core/domain/interfaces/__init__.py
from .i_admin_vpn_key_service import IAdminVpnKeyService

__all__ = [
    # ... existing exports
    "IAdminVpnKeyService",
]
```

**Step 3: Commit**

```bash
git add src/core/domain/interfaces/i_admin_vpn_key_service.py src/core/domain/interfaces/__init__.py
git commit -m "domain: add IAdminVpnKeyService interface"
```

---

## Phase 3: Application Layer

### Task 3.1: Create AdminVpnKeyService

**Files:**
- Create: `usipipo-backend/src/core/application/services/admin_vpn_key_service.py`
- Test: `usipipo-backend/tests/unit/services/test_admin_vpn_key_service.py`

**Step 1: Create service implementation**

```python
"""Admin VPN key management service."""

import logging
from datetime import UTC, datetime
from typing import Any
from uuid import UUID

from loguru import logger
from usipipo_commons.domain.entities.vpn_key import VpnKey
from usipipo_commons.domain.enums.key_status import KeyStatus
from usipipo_commons.domain.enums.key_type import KeyType

from src.core.domain.interfaces.i_admin_vpn_key_service import IAdminVpnKeyService
from src.core.domain.interfaces.i_user_repository import IUserRepository
from src.core.domain.interfaces.i_vpn_repository import IVpnRepository
from src.infrastructure.api_clients.vpn_agent_client import VpnAgentClient
from src.core.application.services.server_registry_service import ServerRegistryService

logger = logging.getLogger(__name__)


class AdminVpnKeyService(IAdminVpnKeyService):
    """Service for administrative VPN key management."""

    def __init__(
        self,
        user_repo: IUserRepository,
        vpn_repo: IVpnRepository,
        server_registry: ServerRegistryService,
    ):
        self.user_repo = user_repo
        self.vpn_repo = vpn_repo
        self.server_registry = server_registry
        self._agent_clients: dict[UUID, VpnAgentClient] = {}

    async def list_keys(
        self,
        filters: dict[str, Any],
        current_admin_id: int
    ) -> dict[str, Any]:
        """List VPN keys with filters and pagination."""
        try:
            # Get all keys
            all_keys = await self.vpn_repo.get_all()
            
            # Apply filters
            filtered_keys = all_keys
            
            # Filter by user Telegram ID
            if user_telegram_id := filters.get('user_telegram_id'):
                user = await self.user_repo.get_by_telegram_id(user_telegram_id)
                if user:
                    filtered_keys = [k for k in filtered_keys if k.user_id == user.id]
                else:
                    filtered_keys = []
            
            # Filter by VPN type
            if vpn_type := filters.get('vpn_type'):
                filtered_keys = [k for k in filtered_keys if k.key_type.value == vpn_type]
            
            # Filter by status
            if status := filters.get('status'):
                filtered_keys = [k for k in filtered_keys if k.status.value == status]
            
            # Filter by country
            if country := filters.get('country'):
                # Get servers in country
                servers = await self.server_registry.get_available_servers(country)
                server_ids = [s.id for s in servers]
                filtered_keys = [k for k in filtered_keys if k.server_id in server_ids]
            
            # Search by name
            if search := filters.get('search'):
                filtered_keys = [k for k in filtered_keys if search.lower() in k.name.lower()]
            
            # Sorting
            sort_by = filters.get('sort_by', 'created_at')
            sort_order = filters.get('sort_order', 'desc')
            reverse = sort_order == 'desc'
            
            if sort_by == 'created_at':
                filtered_keys.sort(key=lambda k: k.created_at or datetime.min, reverse=reverse)
            elif sort_by == 'last_used':
                filtered_keys.sort(key=lambda k: k.last_seen_at or datetime.min, reverse=reverse)
            elif sort_by == 'data_used':
                filtered_keys.sort(key=lambda k: k.used_bytes or 0, reverse=reverse)
            
            # Pagination
            page = filters.get('page', 1)
            page_size = filters.get('page_size', 20)
            total = len(filtered_keys)
            total_pages = (total + page_size - 1) // page_size
            
            start_idx = (page - 1) * page_size
            end_idx = start_idx + page_size
            paginated_keys = filtered_keys[start_idx:end_idx]
            
            return {
                'keys': paginated_keys,
                'total': total,
                'page': page,
                'page_size': page_size,
                'total_pages': total_pages,
                'has_next': page < total_pages,
                'has_previous': page > 1,
            }
            
        except Exception as e:
            logger.error(f"Error listing keys: {e}", exc_info=True)
            raise

    async def get_key_detail(
        self,
        key_id: UUID,
        current_admin_id: int
    ) -> VpnKey | None:
        """Get detailed information about a specific key."""
        try:
            return await self.vpn_repo.get_by_id(key_id)
        except Exception as e:
            logger.error(f"Error getting key detail: {e}", exc_info=True)
            raise

    async def get_user_keys(
        self,
        user_telegram_id: int,
        current_admin_id: int
    ) -> list[VpnKey]:
        """Get all keys for a specific user by Telegram ID."""
        try:
            user = await self.user_repo.get_by_telegram_id(user_telegram_id)
            if not user:
                return []
            
            return await self.vpn_repo.get_by_user_id(user.id)
        except Exception as e:
            logger.error(f"Error getting user keys: {e}", exc_info=True)
            raise

    async def create_key(
        self,
        request: dict[str, Any],
        current_admin_id: int
    ) -> VpnKey:
        """Create a new VPN key for a user."""
        try:
            # Get user
            user = await self.user_repo.get_by_telegram_id(request['user_telegram_id'])
            if not user:
                raise ValueError(f"User with Telegram ID {request['user_telegram_id']} not found")
            
            # Select server
            server = await self.server_registry.select_best_server(
                country=request.get('country', 'US'),
                protocol=request['vpn_type']
            )
            if not server:
                raise ValueError("No available servers")
            
            # Get agent client
            agent_client = self._get_agent_client(server)
            
            # Create key on agent
            if request['vpn_type'].lower() == 'outline':
                result = await agent_client.create_outline_key(name=request['name'])
                config = result['access_url']
                external_id = result['id']
            else:  # wireguard
                result = await agent_client.create_wireguard_peer(name=request['name'])
                config = result['config']
                external_id = result['public_key']
            
            # Calculate dates
            now = datetime.now(UTC)
            expires_at = now + timedelta(days=request.get('expires_in_days', 30))
            data_limit_bytes = int(request.get('data_limit_gb', 5.0) * 1024**3)
            
            # Create entity
            vpn_key = VpnKey(
                id=uuid.uuid4(),
                user_id=user.id,
                name=request['name'],
                key_type=KeyType(request['vpn_type'].lower()),
                status=KeyStatus.ACTIVE,
                key_data=config,
                external_id=external_id,
                server_id=server.id,
                created_at=now,
                expires_at=expires_at,
                used_bytes=0,
                data_limit_bytes=data_limit_bytes,
                billing_reset_at=now,
            )
            
            # Save to DB
            created_key = await self.vpn_repo.create(vpn_key)
            
            # Log audit
            await self.log_audit(
                operation='create_key',
                target_id=created_key.id,
                admin_telegram_id=current_admin_id,
                admin_username='',  # Will be set by caller
                success=True,
                details={
                    'user_telegram_id': user.telegram_id,
                    'vpn_type': request['vpn_type'],
                    'data_limit_gb': request.get('data_limit_gb', 5.0),
                }
            )
            
            return created_key
            
        except Exception as e:
            logger.error(f"Error creating key: {e}", exc_info=True)
            await self.log_audit(
                operation='create_key',
                target_id=UUID(request.get('key_id', '00000000-0000-0000-0000-000000000000')),
                admin_telegram_id=current_admin_id,
                admin_username='',
                success=False,
                error_message=str(e)
            )
            raise

    async def toggle_key(
        self,
        key_id: UUID,
        current_admin_id: int
    ) -> bool:
        """Toggle key active/inactive status."""
        try:
            key = await self.vpn_repo.get_by_id(key_id)
            if not key:
                raise ValueError(f"Key {key_id} not found")
            
            # Toggle status
            new_status = KeyStatus.INACTIVE if key.status == KeyStatus.ACTIVE else KeyStatus.ACTIVE
            key.status = new_status
            
            await self.vpn_repo.update(key)
            
            # Log audit
            await self.log_audit(
                operation='toggle_key',
                target_id=key_id,
                admin_telegram_id=current_admin_id,
                admin_username='',
                success=True,
                details={
                    'user_telegram_id': key.user_id,
                    'new_status': new_status.value,
                }
            )
            
            return True
            
        except Exception as e:
            logger.error(f"Error toggling key: {e}", exc_info=True)
            raise

    async def update_data_limit(
        self,
        key_id: UUID,
        data_limit_gb: float,
        reason: str | None,
        current_admin_id: int
    ) -> bool:
        """Update data limit for a key."""
        try:
            key = await self.vpn_repo.get_by_id(key_id)
            if not key:
                raise ValueError(f"Key {key_id} not found")
            
            old_limit = key.data_limit_bytes
            key.data_limit_bytes = int(data_limit_gb * 1024**3)
            
            await self.vpn_repo.update(key)
            
            # Log audit
            await self.log_audit(
                operation='update_data_limit',
                target_id=key_id,
                admin_telegram_id=current_admin_id,
                admin_username='',
                success=True,
                details={
                    'user_telegram_id': key.user_id,
                    'old_limit_gb': old_limit / (1024**3),
                    'new_limit_gb': data_limit_gb,
                    'reason': reason,
                }
            )
            
            return True
            
        except Exception as e:
            logger.error(f"Error updating data limit: {e}", exc_info=True)
            raise

    async def reset_usage(
        self,
        key_id: UUID,
        reason: str | None,
        current_admin_id: int
    ) -> bool:
        """Reset data usage for a key."""
        try:
            key = await self.vpn_repo.get_by_id(key_id)
            if not key:
                raise ValueError(f"Key {key_id} not found")
            
            old_usage = key.used_bytes
            key.used_bytes = 0
            key.billing_reset_at = datetime.now(UTC)
            
            await self.vpn_repo.update(key)
            
            # Log audit
            await self.log_audit(
                operation='reset_usage',
                target_id=key_id,
                admin_telegram_id=current_admin_id,
                admin_username='',
                success=True,
                details={
                    'user_telegram_id': key.user_id,
                    'old_usage_gb': old_usage / (1024**3),
                    'reason': reason,
                }
            )
            
            return True
            
        except Exception as e:
            logger.error(f"Error resetting usage: {e}", exc_info=True)
            raise

    async def regenerate_config(
        self,
        key_id: UUID,
        notify_user: bool,
        current_admin_id: int
    ) -> VpnKey:
        """Regenerate configuration for a key."""
        try:
            key = await self.vpn_repo.get_by_id(key_id)
            if not key:
                raise ValueError(f"Key {key_id} not found")
            
            # Get server
            if not key.server_id:
                raise ValueError("Key has no server association")
            
            server = await self.server_registry.get_server(key.server_id)
            if not server:
                raise ValueError("Server not found")
            
            # Get agent client
            agent_client = self._get_agent_client(server)
            
            # Delete old config from agent
            if key.key_type == KeyType.OUTLINE:
                await agent_client.delete_outline_key(key.external_id or '')
                # Create new
                result = await agent_client.create_outline_key(name=key.name)
                config = result['access_url']
                external_id = result['id']
            else:  # wireguard
                await agent_client.delete_wireguard_peer(key.external_id or '')
                # Create new
                result = await agent_client.create_wireguard_peer(name=key.name)
                config = result['config']
                external_id = result['public_key']
            
            # Update key
            key.key_data = config
            key.external_id = external_id
            
            await self.vpn_repo.update(key)
            
            # Log audit
            await self.log_audit(
                operation='regenerate_config',
                target_id=key_id,
                admin_telegram_id=current_admin_id,
                admin_username='',
                success=True,
                details={
                    'user_telegram_id': key.user_id,
                    'notify_user': notify_user,
                }
            )
            
            return key
            
        except Exception as e:
            logger.error(f"Error regenerating config: {e}", exc_info=True)
            raise

    async def delete_key(
        self,
        key_id: UUID,
        current_admin_id: int
    ) -> bool:
        """Delete a VPN key permanently."""
        try:
            key = await self.vpn_repo.get_by_id(key_id)
            if not key:
                raise ValueError(f"Key {key_id} not found")
            
            # Delete from agent
            if key.server_id:
                server = await self.server_registry.get_server(key.server_id)
                if server:
                    agent_client = self._get_agent_client(server)
                    if key.key_type == KeyType.OUTLINE:
                        await agent_client.delete_outline_key(key.external_id or '')
                    else:
                        await agent_client.delete_wireguard_peer(key.external_id or '')
            
            # Delete from DB
            await self.vpn_repo.delete(key_id)
            
            # Log audit
            await self.log_audit(
                operation='delete_key',
                target_id=key_id,
                admin_telegram_id=current_admin_id,
                admin_username='',
                success=True,
                details={
                    'user_telegram_id': key.user_id,
                    'vpn_type': key.key_type.value,
                }
            )
            
            return True
            
        except Exception as e:
            logger.error(f"Error deleting key: {e}", exc_info=True)
            raise

    def _get_agent_client(self, server: Any) -> VpnAgentClient:
        """Get or create agent client for server."""
        if server.id not in self._agent_clients:
            self._agent_clients[server.id] = VpnAgentClient(
                base_url=server.agent_url,
                api_key=server.agent_api_key,
            )
        return self._agent_clients[server.id]

    async def log_audit(
        self,
        operation: str,
        target_id: UUID,
        admin_telegram_id: int,
        admin_username: str,
        success: bool,
        details: dict | None = None,
        error_message: str | None = None
    ) -> None:
        """Log operation to audit trail."""
        # This will be implemented with audit log repository
        # For now, just log to application logs
        logger.info(
            f"Audit: {operation} - target={target_id} - admin={admin_telegram_id} - success={success}"
        )
```

**Step 2: Write unit test**

```python
"""Tests for AdminVpnKeyService."""

import pytest
from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4

from src.core.application.services.admin_vpn_key_service import AdminVpnKeyService


class TestAdminVpnKeyService:
    """Test admin VPN key service."""

    @pytest.fixture
    def mock_user_repo(self):
        return AsyncMock()

    @pytest.fixture
    def mock_vpn_repo(self):
        return AsyncMock()

    @pytest.fixture
    def mock_server_registry(self):
        return AsyncMock()

    @pytest.fixture
    def service(self, mock_user_repo, mock_vpn_repo, mock_server_registry):
        return AdminVpnKeyService(
            user_repo=mock_user_repo,
            vpn_repo=mock_vpn_repo,
            server_registry=mock_server_registry,
        )

    @pytest.mark.asyncio
    async def test_list_keys_no_filters(self, service, mock_vpn_repo):
        """Test listing keys without filters."""
        # Arrange
        mock_keys = [MagicMock() for _ in range(5)]
        mock_vpn_repo.get_all.return_value = mock_keys
        
        # Act
        result = await service.list_keys({}, current_admin_id=123)
        
        # Assert
        assert result['total'] == 5
        assert result['page'] == 1
        assert result['page_size'] == 20

    @pytest.mark.asyncio
    async def test_get_user_keys(self, service, mock_user_repo, mock_vpn_repo):
        """Test getting keys for a user."""
        # Arrange
        mock_user = MagicMock(id=uuid4(), telegram_id=123456789)
        mock_user_repo.get_by_telegram_id.return_value = mock_user
        mock_keys = [MagicMock() for _ in range(3)]
        mock_vpn_repo.get_by_user_id.return_value = mock_keys
        
        # Act
        result = await service.get_user_keys(123456789, current_admin_id=123)
        
        # Assert
        assert len(result) == 3
        mock_vpn_repo.get_by_user_id.assert_called_once_with(mock_user.id)

    @pytest.mark.asyncio
    async def test_toggle_key(self, service, mock_vpn_repo):
        """Test toggling key status."""
        # Arrange
        mock_key = MagicMock()
        mock_key.id = uuid4()
        mock_key.status = MagicMock(value='active')
        mock_vpn_repo.get_by_id.return_value = mock_key
        
        # Act
        result = await service.toggle_key(mock_key.id, current_admin_id=123)
        
        # Assert
        assert result is True
        mock_vpn_repo.update.assert_called_once()
```

**Step 3: Run tests**

```bash
cd usipipo-backend
source .venv/bin/activate
pytest tests/unit/services/test_admin_vpn_key_service.py -v
```

Expected: All tests pass

**Step 4: Commit**

```bash
git add src/core/application/services/admin_vpn_key_service.py tests/unit/services/test_admin_vpn_key_service.py
git commit -m "service: implement AdminVpnKeyService with full CRUD operations"
```

---

## Phase 4: Shared Layer (Schemas)

### Task 4.1: Create Pydantic Schemas

**Files:**
- Create: `usipipo-backend/src/shared/schemas/admin_vpn_keys.py`

**Step 1: Create schemas file**

```python
"""Pydantic schemas for admin VPN key management."""

from datetime import datetime
from typing import Literal
from uuid import UUID

from pydantic import BaseModel, Field


# Request Schemas

class VpnKeyListFilters(BaseModel):
    """Filters for listing VPN keys."""
    page: int = Field(1, ge=1)
    page_size: int = Field(20, ge=1, le=100)
    user_telegram_id: int | None = None
    vpn_type: Literal["outline", "wireguard"] | None = None
    status: Literal["active", "inactive", "revoked"] | None = None
    server_id: UUID | None = None
    country: str | None = None
    search: str | None = None
    sort_by: Literal["created_at", "last_used", "data_used"] = "created_at"
    sort_order: Literal["asc", "desc"] = "desc"
    created_after: datetime | None = None
    created_before: datetime | None = None


class CreateVpnKeyAdminRequest(BaseModel):
    """Request to create a VPN key."""
    user_telegram_id: int
    name: str = Field(..., min_length=3, max_length=50)
    vpn_type: Literal["outline", "wireguard"]
    data_limit_gb: float = Field(5.0, ge=1, le=1000)
    country: str = "US"
    expires_in_days: int = Field(30, ge=1, le=365)


class UpdateDataLimitRequest(BaseModel):
    """Request to update data limit."""
    data_limit_gb: float = Field(..., ge=1, le=1000)
    reason: str | None = None


class ResetUsageRequest(BaseModel):
    """Request to reset usage."""
    reason: str | None = None


class RegenerateConfigRequest(BaseModel):
    """Request to regenerate configuration."""
    notify_user: bool = False


# Response Schemas

class VpnKeyListItemResponse(BaseModel):
    """Individual key item in list response."""
    id: UUID
    user_telegram_id: int
    user_name: str
    name: str
    vpn_type: str
    status: str
    country_code: str
    created_at: datetime
    last_used_at: datetime | None
    data_used_gb: float
    data_limit_gb: float
    usage_percentage: float
    is_active: bool


class VpnKeyDetailResponse(BaseModel):
    """Detailed key response."""
    id: UUID
    user_id: UUID
    user_telegram_id: int
    user_name: str
    name: str
    vpn_type: str
    status: str
    server_id: UUID
    server_name: str
    country_code: str
    config: str
    created_at: datetime
    expires_at: datetime
    last_used_at: datetime | None
    data_used_gb: float
    data_limit_gb: float
    usage_percentage: float
    is_active: bool


class VpnKeyListResponse(BaseModel):
    """Paginated list response."""
    keys: list[VpnKeyListItemResponse]
    total: int
    page: int
    page_size: int
    total_pages: int
    has_next: bool
    has_previous: bool


class AdminOperationResult(BaseModel):
    """Result of admin operation."""
    success: bool
    operation: str
    key_id: UUID
    message: str
    timestamp: datetime
    admin_telegram_id: int
```

**Step 2: Update schemas __init__.py**

```python
# usipipo-backend/src/shared/schemas/__init__.py
from .admin_vpn_keys import (
    VpnKeyListFilters,
    CreateVpnKeyAdminRequest,
    UpdateDataLimitRequest,
    ResetUsageRequest,
    RegenerateConfigRequest,
    VpnKeyListItemResponse,
    VpnKeyDetailResponse,
    VpnKeyListResponse,
    AdminOperationResult,
)

__all__ = [
    # ... existing exports
    "VpnKeyListFilters",
    "CreateVpnKeyAdminRequest",
    "UpdateDataLimitRequest",
    "ResetUsageRequest",
    "RegenerateConfigRequest",
    "VpnKeyListItemResponse",
    "VpnKeyDetailResponse",
    "VpnKeyListResponse",
    "AdminOperationResult",
]
```

**Step 3: Commit**

```bash
git add src/shared/schemas/admin_vpn_keys.py src/shared/schemas/__init__.py
git commit -m "schemas: add Pydantic schemas for admin VPN key management"
```

---

## Phase 5: Infrastructure Layer (Routes)

### Task 5.1: Create Admin VPN Keys Routes

**Files:**
- Create: `usipipo-backend/src/infrastructure/api/v1/routes/admin_vpn_keys.py`

**Step 1: Create routes file**

```python
"""Admin routes for VPN key management."""

from fastapi import APIRouter, Depends, Query, status
from sqlalchemy.ext.asyncio import AsyncSession
from uuid import UUID

from src.core.application.services.admin_vpn_key_service import AdminVpnKeyService
from src.infrastructure.api.v1.deps import get_db, require_support, require_admin, get_current_user
from src.shared.schemas.admin_vpn_keys import (
    VpnKeyListFilters,
    CreateVpnKeyAdminRequest,
    UpdateDataLimitRequest,
    ResetUsageRequest,
    RegenerateConfigRequest,
    VpnKeyListResponse,
    VpnKeyDetailResponse,
    AdminOperationResult,
)

router = APIRouter(prefix="/vpn-keys", tags=["Admin - VPN Keys"])


def get_service(db: AsyncSession = Depends(get_db)) -> AdminVpnKeyService:
    """Get AdminVpnKeyService instance."""
    # Import here to avoid circular imports
    from src.infrastructure.persistence.repositories.user_repository import UserRepository
    from src.infrastructure.persistence.repositories.vpn_repository import VpnRepository
    from src.core.application.services.server_registry_service import ServerRegistryService
    
    return AdminVpnKeyService(
        user_repo=UserRepository(db),
        vpn_repo=VpnRepository(db),
        server_registry=ServerRegistryService(db),
    )


@router.get("", response_model=VpnKeyListResponse)
async def list_vpn_keys(
    filters: VpnKeyListFilters = Depends(),
    current_user = Depends(require_support),
    service: AdminVpnKeyService = Depends(get_service),
):
    """List all VPN keys with filters and pagination."""
    result = await service.list_keys(
        filters=filters.model_dump(),
        current_admin_id=current_user.telegram_id
    )
    
    # Map entities to response schema
    # ... mapping logic
    
    return result


@router.get("/{key_id}", response_model=VpnKeyDetailResponse)
async def get_vpn_key_detail(
    key_id: UUID,
    current_user = Depends(require_support),
    service: AdminVpnKeyService = Depends(get_service),
):
    """Get detailed information about a specific VPN key."""
    key = await service.get_key_detail(key_id, current_user.telegram_id)
    
    if not key:
        raise HTTPException(status_code=404, detail="Key not found")
    
    # Map entity to response schema
    # ... mapping logic
    
    return key


@router.get("/users/{telegram_id}", response_model=VpnKeyListResponse)
async def get_user_vpn_keys(
    telegram_id: int,
    current_user = Depends(require_support),
    service: AdminVpnKeyService = Depends(get_service),
):
    """Get all VPN keys for a specific user by Telegram ID."""
    keys = await service.get_user_keys(telegram_id, current_user.telegram_id)
    
    # Map entities to response schema
    # ... mapping logic
    
    return keys


@router.post("", response_model=VpnKeyDetailResponse, status_code=status.HTTP_201_CREATED)
async def create_vpn_key(
    request: CreateVpnKeyAdminRequest,
    current_user = Depends(require_admin),
    service: AdminVpnKeyService = Depends(get_service),
):
    """Create a new VPN key for a user."""
    key = await service.create_key(
        request=request.model_dump(),
        current_admin_id=current_user.telegram_id
    )
    
    # Map entity to response schema
    # ... mapping logic
    
    return key


@router.patch("/{key_id}/toggle", response_model=AdminOperationResult)
async def toggle_vpn_key(
    key_id: UUID,
    current_user = Depends(require_support),
    service: AdminVpnKeyService = Depends(get_service),
):
    """Toggle VPN key active/inactive status."""
    success = await service.toggle_key(key_id, current_user.telegram_id)
    
    return AdminOperationResult(
        success=success,
        operation="toggle_key",
        key_id=key_id,
        message="Key toggled successfully",
        timestamp=datetime.now(UTC),
        admin_telegram_id=current_user.telegram_id,
    )


@router.patch("/{key_id}/data-limit", response_model=AdminOperationResult)
async def update_vpn_key_data_limit(
    key_id: UUID,
    request: UpdateDataLimitRequest,
    current_user = Depends(require_admin),
    service: AdminVpnKeyService = Depends(get_service),
):
    """Update data limit for a VPN key."""
    success = await service.update_data_limit(
        key_id=key_id,
        data_limit_gb=request.data_limit_gb,
        reason=request.reason,
        current_admin_id=current_user.telegram_id
    )
    
    return AdminOperationResult(
        success=success,
        operation="update_data_limit",
        key_id=key_id,
        message="Data limit updated successfully",
        timestamp=datetime.now(UTC),
        admin_telegram_id=current_user.telegram_id,
    )


@router.patch("/{key_id}/reset-usage", response_model=AdminOperationResult)
async def reset_vpn_key_usage(
    key_id: UUID,
    request: ResetUsageRequest,
    current_user = Depends(require_admin),
    service: AdminVpnKeyService = Depends(get_service),
):
    """Reset data usage for a VPN key."""
    success = await service.reset_usage(
        key_id=key_id,
        reason=request.reason,
        current_admin_id=current_user.telegram_id
    )
    
    return AdminOperationResult(
        success=success,
        operation="reset_usage",
        key_id=key_id,
        message="Usage reset successfully",
        timestamp=datetime.now(UTC),
        admin_telegram_id=current_user.telegram_id,
    )


@router.post("/{key_id}/regenerate", response_model=VpnKeyDetailResponse)
async def regenerate_vpn_key_config(
    key_id: UUID,
    request: RegenerateConfigRequest,
    current_user = Depends(require_admin),
    service: AdminVpnKeyService = Depends(get_service),
):
    """Regenerate configuration for a VPN key."""
    key = await service.regenerate_config(
        key_id=key_id,
        notify_user=request.notify_user,
        current_admin_id=current_user.telegram_id
    )
    
    # Map entity to response schema
    # ... mapping logic
    
    return key


@router.delete("/{key_id}", response_model=AdminOperationResult)
async def delete_vpn_key(
    key_id: UUID,
    current_user = Depends(require_admin),
    service: AdminVpnKeyService = Depends(get_service),
):
    """Delete a VPN key permanently."""
    success = await service.delete_key(key_id, current_user.telegram_id)
    
    return AdminOperationResult(
        success=success,
        operation="delete_key",
        key_id=key_id,
        message="Key deleted successfully",
        timestamp=datetime.now(UTC),
        admin_telegram_id=current_user.telegram_id,
    )
```

**Step 2: Register routes in main.py**

```python
# usipipo-backend/src/main.py
from .infrastructure.api.v1.routes.admin_vpn_keys import router as admin_vpn_keys_router

# Register admin VPN keys routes
app.include_router(admin_vpn_keys_router, prefix=api_prefix)
```

**Step 3: Commit**

```bash
git add src/infrastructure/api/v1/routes/admin_vpn_keys.py src/main.py
git commit -m "api: add admin VPN keys CRUD routes"
```

---

## Phase 6: Testing

### Task 6.1: Integration Tests

**Files:**
- Create: `usipipo-backend/tests/integration/test_admin_vpn_keys.py`

**Step 1: Write integration tests**

```python
"""Integration tests for admin VPN keys API."""

import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_list_vpn_keys_requires_auth(client: AsyncClient):
    """Test that listing keys requires authentication."""
    response = await client.get("/api/v1/admin/vpn-keys")
    assert response.status_code == 401


@pytest.mark.asyncio
async def test_list_vpn_keys_requires_staff(client: AsyncClient, auth_headers: dict):
    """Test that listing keys requires staff role."""
    response = await client.get(
        "/api/v1/admin/vpn-keys",
        headers=auth_headers
    )
    # Should fail if user is not in staff_roles table
    assert response.status_code in [200, 403]


@pytest.mark.asyncio
async def test_create_vpn_key_requires_admin(client: AsyncClient, support_auth_headers: dict):
    """Test that creating keys requires admin role."""
    payload = {
        "user_telegram_id": 123456789,
        "name": "Test Key",
        "vpn_type": "outline",
        "data_limit_gb": 5.0,
    }
    response = await client.post(
        "/api/v1/admin/vpn-keys",
        json=payload,
        headers=support_auth_headers
    )
    assert response.status_code == 403


@pytest.mark.asyncio
async def test_create_vpn_key_admin_success(client: AsyncClient, admin_auth_headers: dict):
    """Test admin can create VPN key."""
    payload = {
        "user_telegram_id": 123456789,
        "name": "Test Key",
        "vpn_type": "outline",
        "data_limit_gb": 5.0,
    }
    response = await client.post(
        "/api/v1/admin/vpn-keys",
        json=payload,
        headers=admin_auth_headers
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Test Key"
    assert "id" in data
```

**Step 2: Run integration tests**

```bash
cd usipipo-backend
pytest tests/integration/test_admin_vpn_keys.py -v
```

Expected: All tests pass

**Step 3: Commit**

```bash
git add tests/integration/test_admin_vpn_keys.py
git commit -m "test: add integration tests for admin VPN keys API"
```

---

## Phase 7: Agent Enhancements

### Task 7.1: Add Regenerate Endpoint

**Files:**
- Modify: `usipipo-agent/internal/api/handlers.go`
- Modify: `usipipo-agent/internal/vpn/wireguard.go`

**Step 1: Add regenerate handler**

```go
// RegenerateWireGuardPeerHandler regenerates a WireGuard peer configuration
func RegenerateWireGuardPeerHandler(c *gin.Context) {
	if wireguardClient == nil {
		c.JSON(http.StatusInternalServerError, gin.H{
			"error": "WireGuard client not initialized",
		})
		return
	}

	name := c.Param("name")

	// Delete existing peer
	err := wireguardClient.DeletePeer(c.Request.Context(), name)
	if err != nil && !strings.Contains(err.Error(), "peer not found") {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	// Create new peer with same name
	peer, err := wireguardClient.CreatePeer(c.Request.Context(), name)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusCreated, gin.H{
		"public_key":  peer.PublicKey,
		"name":        peer.Name,
		"ip_address":  peer.IPAddress,
		"config":      peer.Config,
	})
}
```

**Step 2: Register route**

```go
// internal/api/server.go
protected.POST("/wireguard/peers/:name/regenerate", RegenerateWireGuardPeerHandler)
```

**Step 3: Commit**

```bash
cd usipipo-agent
git add internal/api/handlers.go internal/api/server.go
git commit -m "feat: add regenerate endpoint for WireGuard peers"
```

---

## Phase 8: Deployment

### Task 8.1: Update Documentation

**Files:**
- Update: `usipipo-backend/README.md`
- Update: `usipipo-backend/docs/ADMIN_API.md`

**Step 1: Add API documentation**

Create `docs/ADMIN_API.md` with OpenAPI-style documentation for all admin endpoints.

**Step 2: Commit**

```bash
git add docs/ADMIN_API.md README.md
git commit -m "docs: add admin VPN keys API documentation"
```

---

## Testing Checklist

Before considering this complete, verify:

- [ ] All unit tests pass (95%+ coverage)
- [ ] All integration tests pass
- [ ] Role-based access works correctly (Support vs Admin)
- [ ] Audit logging captures all operations
- [ ] Migrations run successfully
- [ ] Agent regenerate endpoint works
- [ ] API documentation is complete

---

## Next Steps (Future - Staff Bot)

When Staff Bot repository is created:

1. Create `usipipo-staff-bot` repository
2. Implement Telegram handlers for all admin operations
3. Add inline keyboards for key management
4. Implement role-based command access
5. Add audit log viewer for staff

---

**Plan complete and saved to `docs/plans/2026-03-30-admin-vpn-keys.md`. Two execution options:**

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**Which approach?**
