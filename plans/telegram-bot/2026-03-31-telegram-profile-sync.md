# Telegram Profile Sync Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task.

**Goal:** Fix incorrect profile data in Telegram bot by implementing two-phase approach: (1) update auto-register to capture Telegram profile data at registration time, and (2) add profile sync endpoint for ongoing updates.

**Architecture:** 
- Phase 1: Enhance `/auth/telegram/auto-register` to accept optional Telegram profile data (username, first_name, last_name) from the bot
- Phase 2: Create new `/users/sync-telegram-profile` endpoint for periodic profile updates
- Bot layer: Update `/start` handler to capture and send Telegram user data, add periodic sync on menu access

**Tech Stack:** Python 3.13, FastAPI, asyncpg, PostgreSQL, python-telegram-bot, Pydantic schemas, JWT auth

**Key Design Principles:**
- Backward compatibility: auto-register must still work with just telegram_id
- Idempotent operations: sync endpoint can be called multiple times safely
- Minimal API calls: batch profile data with auth when possible
- Clean separation: auth concerns separate from profile management

---

## Phase 1: Backend Auto-Register Enhancement

### Task 1: Update TelegramAutoRegisterRequest Schema

**Files:**
- Modify: `usipipo-backend/src/shared/schemas/auth.py`

**Step 1: Read current schema**

```bash
cd /home/mowgli/usipipo/usipipo-backend
grep -A 10 "class TelegramAutoRegisterRequest" src/shared/schemas/auth.py
```

**Step 2: Update schema to include optional Telegram profile fields**

```python
class TelegramAutoRegisterRequest(BaseModel):
    """
    Solicitud para auto-registro desde Telegram Bot.
    
    Attributes:
        telegram_id: ID único de Telegram del usuario
        username: Username de Telegram (opcional, puede cambiar)
        first_name: Nombre del usuario (opcional)
        last_name: Apellido del usuario (opcional)
    """
    telegram_id: int
    username: str | None = None
    first_name: str | None = None
    last_name: str | None = None
    
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "telegram_id": 123456789,
                    "username": "develop",
                    "first_name": "Developer",
                    "last_name": "User"
                }
            ]
        }
    }
```

**Step 3: Run type checking**

```bash
cd /home/mowgli/usipipo/usipipo-backend
uv run mypy src/shared/schemas/auth.py
```

Expected: PASS (no type errors)

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/shared/schemas/auth.py
git commit -m "feat: add Telegram profile fields to auto-register request schema"
```

---

### Task 2: Update Auto-Register Route Handler

**Files:**
- Modify: `usipipo-backend/src/infrastructure/api/v1/routes/auth.py`

**Step 1: Read current auto_register_telegram function (lines 217-260)**

```bash
cd /home/mowgli/usipipo/usipipo-backend
sed -n '217,260p' src/infrastructure/api/v1/routes/auth.py
```

**Step 2: Update handler to pass Telegram profile data to user_service**

```python
@router.post(
    "/telegram/auto-register",
    response_model=AuthResponse,
    status_code=status.HTTP_201_CREATED,
)
@limiter.limit(settings.RATE_LIMIT_AUTH)
async def auto_register_telegram(
    request: Request,
    auto_register: TelegramAutoRegisterRequest,
    user_service: UserService = Depends(get_user_service),
):
    """
    Registro automático de usuario desde Telegram Bot.

    Crea o busca usuario por telegram_id y retorna tokens JWT.
    Usado para autenticación invisible en el bot.
    
    Si se proporcionan datos de perfil de Telegram (username, first_name, last_name),
    se actualiza la información del usuario existente o se usa para crear el nuevo usuario.

    Args:
        request: Request object
        auto_register: telegram_id y datos de perfil opcionales
        user_service: Servicio de usuarios

    Returns:
        AuthResponse: Tokens de acceso

    Note:
        Este endpoint es usado exclusivamente por el Telegram Bot
        para autenticación invisible (sin intervención del usuario).
    """
    # Get or create user by telegram_id with profile data
    user = await user_service.get_or_create_by_telegram(
        telegram_id=auto_register.telegram_id,
        username=auto_register.username,
        first_name=auto_register.first_name,
        last_name=auto_register.last_name,
    )

    # Generate token pair
    access_token, refresh_token = create_token_pair(user.id, user.telegram_id)

    logger.info(
        f"Auto-registered telegram user: {user.id} (telegram_id={auto_register.telegram_id}, username={auto_register.username})"
    )

    return AuthResponse(
        access_token=access_token,
        refresh_token=refresh_token,
        token_type="bearer",
        expires_in=settings.JWT_ACCESS_TOKEN_EXPIRE_MINUTES * 60,
        user_id=str(user.id),
    )
```

**Step 3: Run tests for auth routes**

```bash
cd /home/mowgli/usipipo/usipipo-backend
uv run pytest tests/unit/api/v1/routes/test_auth.py -v -k auto_register
```

Expected: PASS (existing tests should still pass due to optional fields)

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/infrastructure/api/v1/routes/auth.py
git commit -m "feat: pass Telegram profile data to user service in auto-register"
```

---

### Task 3: Update UserService.get_or_create_by_telegram Logic

**Files:**
- Modify: `usipipo-backend/src/core/application/services/user_service.py`

**Step 1: Read current get_or_create_by_telegram method (lines 144-210)**

```bash
cd /home/mowgli/usipipo/usipipo-backend
sed -n '144,210p' src/core/application/services/user_service.py
```

**Step 2: Update method to properly handle profile updates**

```python
async def get_or_create_by_telegram(
    self,
    telegram_id: int,
    username: str | None = None,
    first_name: str | None = None,
    last_name: str | None = None,
    referral_code: str | None = None,
) -> User:
    """
    Obtiene usuario por Telegram ID o crea uno nuevo si no existe.
    
    Si se proporcionan datos de perfil de Telegram, actualiza el usuario existente
    o usa los datos para crear el nuevo usuario.

    Args:
        telegram_id: El ID de Telegram del usuario
        username: Username de Telegram (actualiza si cambia)
        first_name: Nombre del usuario
        last_name: Apellido del usuario
        referral_code: Código de referido opcional

    Returns:
        El usuario existente o el nuevo usuario creado
    """
    # Intentar obtener usuario existente
    existing_user = await self.get_by_telegram_id(telegram_id)
    if existing_user:
        # Actualizar información de perfil si se proporcionó y es diferente
        updated = False
        new_username = username if username is not None else existing_user.username
        new_first_name = first_name if first_name is not None else existing_user.first_name
        new_last_name = last_name if last_name is not None else existing_user.last_name
        
        # Detectar si hay cambios en el perfil
        if (username is not None and username != existing_user.username) or \
           (first_name is not None and first_name != existing_user.first_name) or \
           (last_name is not None and last_name != existing_user.last_name):
            updated = True
            logger.info(
                f"Updating profile for telegram_id={telegram_id}: "
                f"username={existing_user.username}->{new_username}, "
                f"first_name={existing_user.first_name}->{new_first_name}, "
                f"last_name={existing_user.last_name}->{new_last_name}"
            )
        
        updated_user = await self.user_repo.update(
            User(
                id=existing_user.id,
                telegram_id=telegram_id,
                username=new_username,
                first_name=new_first_name,
                last_name=new_last_name,
                is_admin=existing_user.is_admin,
                created_at=existing_user.created_at,
                updated_at=datetime.utcnow() if updated else existing_user.updated_at,
                balance_gb=existing_user.balance_gb,
                total_purchased_gb=existing_user.total_purchased_gb,
                referral_code=existing_user.referral_code,
                referred_by=existing_user.referred_by,
            )
        )
        return updated_user

    # Generar código de referido único si no se proporciona
    # Formato: ref_ + 16 chars hex (max 20 chars para caber en DB)
    if not referral_code:
        referral_code = f"ref_{uuid.uuid4().hex[:16]}"

    # Crear nuevo usuario con datos de perfil de Telegram
    new_user = User(
        id=uuid.uuid4(),
        telegram_id=telegram_id,
        username=username,
        first_name=first_name,
        last_name=last_name,
        is_admin=False,
        created_at=datetime.utcnow(),
        updated_at=datetime.utcnow(),
        balance_gb=FREE_GB,  # 5 GB gratis por defecto
        total_purchased_gb=0.0,
        referral_code=referral_code,
        referred_by=None,
    )

    return await self.user_repo.create(new_user)
```

**Step 3: Add import for logger if not present**

```bash
cd /home/mowgli/usipipo/usipipo-backend
head -20 src/core/application/services/user_service.py
```

If logger not imported, add at top of file:
```python
import logging
logger = logging.getLogger(__name__)
```

**Step 4: Run unit tests for user_service**

```bash
cd /home/mowgli/usipipo/usipipo-backend
uv run pytest tests/unit/application/services/test_user_service.py -v -k get_or_create
```

Expected: PASS

**Step 5: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/core/application/services/user_service.py
git commit -m "feat: update user profile on auto-register when Telegram data changes"
```

---

## Phase 2: Profile Sync Endpoint

### Task 4: Create Profile Sync Request Schema

**Files:**
- Create: `usipipo-backend/src/shared/schemas/users.py`

**Step 1: Create new schema file**

```python
"""Schemas para gestión de usuarios."""

from pydantic import BaseModel, Field


class TelegramProfileSyncRequest(BaseModel):
    """
    Solicitud para sincronizar perfil de Telegram.
    
    Attributes:
        telegram_id: ID de Telegram del usuario
        username: Username actual de Telegram
        first_name: Nombre actual del usuario
        last_name: Apellido actual del usuario
    """
    telegram_id: int = Field(..., description="Telegram user ID")
    username: str | None = Field(None, description="Telegram username")
    first_name: str | None = Field(None, description="User first name")
    last_name: str | None = Field(None, description="User last name")
    
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "telegram_id": 123456789,
                    "username": "develop",
                    "first_name": "Developer",
                    "last_name": "User"
                }
            ]
        }
    }


class ProfileSyncResponse(BaseModel):
    """
    Respuesta de sincronización de perfil.
    
    Attributes:
        success: Si la sincronización fue exitosa
        user_id: ID del usuario actualizado
        updated: Si se actualizaron datos (True) o ya estaban actualizados (False)
    """
    success: bool
    user_id: str
    updated: bool
    
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "success": True,
                    "user_id": "550e8400-e29b-41d4-a716-446655440000",
                    "updated": True
                }
            ]
        }
    }
```

**Step 2: Run type checking**

```bash
cd /home/mowgli/usipipo/usipipo-backend
uv run mypy src/shared/schemas/users.py
```

Expected: PASS

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/shared/schemas/users.py
git commit -m "feat: add schemas for Telegram profile sync endpoint"
```

---

### Task 5: Create Profile Sync Route

**Files:**
- Modify: `usipipo-backend/src/infrastructure/api/v1/routes/users.py`

**Step 1: Read current users.py routes**

```bash
cd /home/mowgli/usipipo/usipipo-backend
cat src/infrastructure/api/v1/routes/users.py
```

**Step 2: Add profile sync endpoint**

```python
"""Routes para gestión de usuarios."""

import logging
from fastapi import APIRouter, Depends, HTTPException, status
from usipipo_commons.domain.entities.user import User

from src.core.application.services.user_service import UserService
from src.infrastructure.api.v1.deps import get_current_user, get_user_service
from src.shared.schemas.users import TelegramProfileSyncRequest, ProfileSyncResponse

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/users", tags=["Users"])


@router.get(
    "/me",
    status_code=status.HTTP_200_OK,
)
async def get_current_user_profile(
    current_user: User = Depends(get_current_user),
):
    """
    Obtiene el perfil del usuario autenticado.

    Args:
        current_user: Usuario autenticado (inyectado por JWT)

    Returns:
        dict: Perfil del usuario
    """
    return {
        "id": str(current_user.id),
        "telegram_id": current_user.telegram_id,
        "username": current_user.username,
        "first_name": current_user.first_name,
        "last_name": current_user.last_name,
        "is_admin": current_user.is_admin,
        "balance_gb": current_user.balance_gb,
        "total_purchased_gb": current_user.total_purchased_gb,
        "referral_code": current_user.referral_code,
        "referral_credits": current_user.referral_credits,
        "purchase_count": current_user.purchase_count,
        "loyalty_bonus_percent": current_user.loyalty_bonus_percent,
        "created_at": current_user.created_at.isoformat() if current_user.created_at else None,
        "updated_at": current_user.updated_at.isoformat() if current_user.updated_at else None,
        "referred_by": str(current_user.referred_by) if current_user.referred_by else None,
    }


@router.post(
    "/sync-telegram-profile",
    response_model=ProfileSyncResponse,
    status_code=status.HTTP_200_OK,
)
async def sync_telegram_profile(
    sync_data: TelegramProfileSyncRequest,
    user_service: UserService = Depends(get_user_service),
):
    """
    Sincroniza el perfil de Telegram del usuario.
    
    Este endpoint permite actualizar el perfil del usuario (username, first_name, last_name)
    basado en los datos actuales de Telegram. Se debe llamar periódicamente para mantener
    el perfil sincronizado cuando el usuario cambia su username o nombre en Telegram.
    
    Args:
        sync_data: Datos de perfil de Telegram
        user_service: Servicio de usuarios

    Returns:
        ProfileSyncResponse: Confirmación de sincronización

    Raises:
        HTTPException: 404 si el usuario no existe
    """
    # Buscar usuario por telegram_id
    user = await user_service.get_by_telegram_id(sync_data.telegram_id)
    
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User with telegram_id {sync_data.telegram_id} not found",
        )
    
    # Verificar si hay cambios en el perfil
    has_changes = (
        (sync_data.username is not None and sync_data.username != user.username) or
        (sync_data.first_name is not None and sync_data.first_name != user.first_name) or
        (sync_data.last_name is not None and sync_data.last_name != user.last_name)
    )
    
    if has_changes:
        logger.info(
            f"Syncing profile for telegram_id={sync_data.telegram_id}: "
            f"username={user.username}->{sync_data.username}, "
            f"first_name={user.first_name}->{sync_data.first_name}, "
            f"last_name={user.last_name}->{sync_data.last_name}"
        )
        
        # Actualizar perfil
        await user_service.update_user(
            user_id=user.id,
            username=sync_data.username,
            first_name=sync_data.first_name,
            last_name=sync_data.last_name,
        )
    
    return ProfileSyncResponse(
        success=True,
        user_id=str(user.id),
        updated=has_changes,
    )
```

**Step 3: Run tests for users routes**

```bash
cd /home/mowgli/usipipo/usipipo-backend
uv run pytest tests/unit/api/v1/routes/test_users.py -v
```

Expected: PASS (or create test if doesn't exist)

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/infrastructure/api/v1/routes/users.py
git commit -m "feat: add /users/sync-telegram-profile endpoint for periodic profile updates"
```

---

### Task 6: Add UserService.update_user Method (if not complete)

**Files:**
- Modify: `usipipo-backend/src/core/application/services/user_service.py`

**Step 1: Check if update_user method exists and supports partial updates**

```bash
cd /home/mowgli/usipipo/usipipo-backend
sed -n '240,290p' src/core/application/services/user_service.py
```

**Step 2: Verify update_user signature supports partial updates**

The method should already exist (from earlier read). Verify it accepts optional username, first_name, last_name parameters. If not, the method signature is already correct from the existing code.

**Step 3: Add integration test for profile sync flow**

Create: `usipipo-backend/tests/integration/test_profile_sync.py`

```python
"""Tests de integración para sincronización de perfil de Telegram."""

import pytest
from uuid import uuid4
from datetime import datetime
from usipipo_commons.domain.entities.user import User


@pytest.mark.asyncio
async def test_profile_sync_updates_user_data(
    user_service,
    test_telegram_id=999888777,
):
    """Prueba que sync actualiza datos del usuario correctamente."""
    # Crear usuario de prueba
    test_user = User(
        id=uuid4(),
        telegram_id=test_telegram_id,
        username="old_username",
        first_name="Old",
        last_name="Name",
        is_admin=False,
        created_at=datetime.utcnow(),
        updated_at=datetime.utcnow(),
        balance_gb=5.0,
        total_purchased_gb=0.0,
        referral_code="ref_test123",
        referred_by=None,
    )
    await user_service.user_repo.create(test_user)
    
    # Sincronizar con nuevos datos
    updated_user = await user_service.update_user(
        user_id=test_user.id,
        username="new_username",
        first_name="New",
        last_name="Name",
    )
    
    # Verificar actualizaciones
    assert updated_user.username == "new_username"
    assert updated_user.first_name == "New"
    assert updated_user.last_name == "Name"
    # Verificar que otros campos no cambiaron
    assert updated_user.balance_gb == 5.0
    assert updated_user.telegram_id == test_telegram_id


@pytest.mark.asyncio
async def test_profile_sync_partial_update(user_service, test_telegram_id=999888776):
    """Prueba que sync permite actualizaciones parciales."""
    # Crear usuario de prueba
    test_user = User(
        id=uuid4(),
        telegram_id=test_telegram_id,
        username="test_user",
        first_name="Test",
        last_name="User",
        is_admin=False,
        created_at=datetime.utcnow(),
        updated_at=datetime.utcnow(),
        balance_gb=5.0,
        total_purchased_gb=0.0,
        referral_code="ref_test456",
        referred_by=None,
    )
    await user_service.user_repo.create(test_user)
    
    # Actualizar solo username
    updated_user = await user_service.update_user(
        user_id=test_user.id,
        username="updated_username",
        # first_name y last_name no se proporcionan
    )
    
    # Verificar que username cambió pero otros campos se mantuvieron
    assert updated_user.username == "updated_username"
    assert updated_user.first_name == "Test"
    assert updated_user.last_name == "User"


@pytest.mark.asyncio
async def test_get_or_create_with_profile_data(user_service, test_telegram_id=999888775):
    """Prueba get_or_create con datos de perfil de Telegram."""
    # Primer llamado - crea usuario
    user1 = await user_service.get_or_create_by_telegram(
        telegram_id=test_telegram_id,
        username="telegram_user",
        first_name="Telegram",
        last_name="User",
    )
    
    assert user1.username == "telegram_user"
    assert user1.first_name == "Telegram"
    
    # Segundo llamado - actualiza perfil
    user2 = await user_service.get_or_create_by_telegram(
        telegram_id=test_telegram_id,
        username="updated_user",
        first_name="Updated",
        last_name="Name",
    )
    
    # Debe ser el mismo usuario con datos actualizados
    assert user2.id == user1.id
    assert user2.username == "updated_user"
    assert user2.first_name == "Updated"
    assert user2.last_name == "Name"
```

**Step 4: Run integration tests**

```bash
cd /home/mowgli/usipipo/usipipo-backend
uv run pytest tests/integration/test_profile_sync.py -v
```

Expected: PASS

**Step 5: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/core/application/services/user_service.py tests/integration/test_profile_sync.py
git commit -m "test: add integration tests for profile sync functionality"
```

---

## Phase 3: Bot Implementation

### Task 7: Update Bot Auth Handler to Capture Telegram Profile Data

**Files:**
- Modify: `usipipo-telegram-bot/src/bot/handlers/auth.py`

**Step 1: Read current start_handler and _register_and_auth methods**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
sed -n '30,80p' src/bot/handlers/auth.py
```

**Step 2: Update _register_and_auth to capture and send Telegram profile data**

```python
async def _register_and_auth(
    self,
    telegram_id: int,
    update: Update,
    context: ContextTypes.DEFAULT_TYPE,
) -> None:
    """
    Registra y autentica usuario automáticamente.

    Captura datos de perfil de Telegram y los envía al backend
    para crear/actualizar el perfil del usuario.
    """
    try:
        # Capture Telegram profile data
        effective_user = update.effective_user
        telegram_profile = {
            "telegram_id": telegram_id,
            "username": effective_user.username if effective_user else None,
            "first_name": effective_user.first_name if effective_user else None,
            "last_name": effective_user.last_name if effective_user else None,
        }
        
        logger.info(
            f"Registering user: telegram_id={telegram_id}, "
            f"username={telegram_profile['username']}, "
            f"first_name={telegram_profile['first_name']}"
        )

        # Send profile data to backend
        response = await self.api.post(
            "/auth/telegram/auto-register",
            telegram_profile,  # Send complete profile
        )

        if "access_token" in response:
            await self.tokens.store(telegram_id, response)
            if update.message:
                await update.message.reply_text(
                    text=AuthMessages.WELCOME_NEW_USER,
                    reply_markup=MainMenuKeyboard.main_menu(),
                    parse_mode="Markdown",
                )
        else:
            if update.message:
                await update.message.reply_text(AuthMessages.AUTH_ERROR)

    except Exception as e:
        logger.error(f"Error en registro/auto-auth: {e}")
        if update.message:
            await update.message.reply_text(AuthMessages.AUTH_ERROR)
```

**Step 3: Run bot tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
uv run pytest tests/bot/test_handlers.py -v -k auth
```

Expected: PASS

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/handlers/auth.py
git commit -m "feat: capture and send Telegram profile data on auto-register"
```

---

### Task 8: Create Profile Sync Service in Bot

**Files:**
- Create: `usipipo-telegram-bot/src/bot/services/profile_sync.py`

**Step 1: Create profile sync service**

```python
"""Servicio para sincronización periódica de perfil de Telegram."""

import logging
from telegram import Update
from telegram.ext import ContextTypes

from src.infrastructure.api_client import APIClient
from src.infrastructure.token_storage import TokenStorage

logger = logging.getLogger(__name__)


class ProfileSyncService:
    """
    Servicio para sincronizar perfil de Telegram con el backend.
    
    Se llama periódicamente para mantener actualizados los datos
    del usuario (username, first_name, last_name) cuando el usuario
    los cambia en Telegram.
    """

    def __init__(self, api_client: APIClient, token_storage: TokenStorage):
        self.api = api_client
        self.tokens = token_storage

    async def sync_user_profile(self, update: Update, context: ContextTypes.DEFAULT_TYPE) -> bool:
        """
        Sincroniza perfil de Telegram con el backend.
        
        Args:
            update: Update de Telegram
            context: Contexto del bot
            
        Returns:
            bool: True si la sincronización fue exitosa, False si falló
        """
        if update.effective_user is None:
            logger.warning("sync_user_profile called without effective_user")
            return False

        telegram_id = update.effective_user.id

        # Check if authenticated
        if not await self.tokens.is_authenticated(telegram_id):
            logger.debug(f"User {telegram_id} not authenticated, skipping profile sync")
            return False

        try:
            # Get tokens
            tokens = await self.tokens.get(telegram_id)
            if tokens is None:
                return False

            # Prepare profile data
            profile_data = {
                "telegram_id": telegram_id,
                "username": update.effective_user.username,
                "first_name": update.effective_user.first_name,
                "last_name": update.effective_user.last_name,
            }

            # Call sync endpoint
            headers = {"Authorization": f"Bearer {tokens['access_token']}"}
            response = await self.api.post(
                "/users/sync-telegram-profile",
                profile_data,
                headers=headers,
            )

            if response.get("success"):
                updated = response.get("updated", False)
                if updated:
                    logger.info(
                        f"Profile synced for telegram_id={telegram_id}: "
                        f"username={profile_data['username']}"
                    )
                return True
            else:
                logger.warning(f"Profile sync failed for telegram_id={telegram_id}")
                return False

        except Exception as e:
            logger.error(f"Error syncing profile for telegram_id={telegram_id}: {e}")
            return False

    async def sync_on_menu_access(
        self,
        update: Update,
        context: ContextTypes.DEFAULT_TYPE,
    ) -> None:
        """
        Sincroniza perfil cuando el usuario accede al menú principal.
        
        Se llama de forma asíncrona para no bloquear la UI.
        """
        try:
            await self.sync_user_profile(update, context)
        except Exception as e:
            logger.error(f"Non-critical error in sync_on_menu_access: {e}")
            # No need to propagate error - sync is best-effort
```

**Step 2: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/services/profile_sync.py
git commit -m "feat: create ProfileSyncService for periodic Telegram profile updates"
```

---

### Task 9: Create Unit Tests for Profile Sync Service

**Files:**
- Create: `usipipo-telegram-bot/tests/unit/bot/services/test_profile_sync.py`

**Step 1: Create test file**

```python
"""Tests para ProfileSyncService."""

import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from telegram import User

from src.bot.services.profile_sync import ProfileSyncService
from src.infrastructure.api_client import APIClient
from src.infrastructure.token_storage import TokenStorage


@pytest.fixture
def mock_api_client():
    """Mock de APIClient."""
    client = AsyncMock(spec=APIClient)
    client.post = AsyncMock()
    client.get = AsyncMock()
    return client


@pytest.fixture
def mock_token_storage():
    """Mock de TokenStorage."""
    storage = AsyncMock(spec=TokenStorage)
    storage.is_authenticated = AsyncMock()
    storage.get = AsyncMock()
    storage.store = AsyncMock()
    storage.delete = AsyncMock()
    return storage


@pytest.fixture
def profile_sync_service(mock_api_client, mock_token_storage):
    """Crea ProfileSyncService con mocks."""
    return ProfileSyncService(mock_api_client, mock_token_storage)


@pytest.fixture
def mock_update():
    """Mock de Update de Telegram."""
    update = MagicMock()
    update.effective_user = User(
        id=123456789,
        is_bot=False,
        username="test_user",
        first_name="Test",
        last_name="User",
    )
    return update


@pytest.fixture
def mock_context():
    """Mock de ContextTypes."""
    return MagicMock()


@pytest.mark.asyncio
async def test_sync_user_profile_success(
    profile_sync_service,
    mock_update,
    mock_context,
    mock_token_storage,
    mock_api_client,
):
    """Prueba sincronización exitosa de perfil."""
    # Configurar mocks
    mock_token_storage.is_authenticated.return_value = True
    mock_token_storage.get.return_value = {
        "access_token": "test_token",
        "refresh_token": "test_refresh",
    }
    mock_api_client.post.return_value = {
        "success": True,
        "user_id": "550e8400-e29b-41d4-a716-446655440000",
        "updated": True,
    }

    # Ejecutar sync
    result = await profile_sync_service.sync_user_profile(mock_update, mock_context)

    # Verificar resultados
    assert result is True
    mock_api_client.post.assert_called_once()
    call_args = mock_api_client.post.call_args[0]
    assert call_args[0] == "/users/sync-telegram-profile"
    assert call_args[1]["telegram_id"] == 123456789
    assert call_args[1]["username"] == "test_user"


@pytest.mark.asyncio
async def test_sync_user_profile_not_authenticated(
    profile_sync_service,
    mock_update,
    mock_context,
    mock_token_storage,
):
    """Prueba que no sincroniza si no está autenticado."""
    mock_token_storage.is_authenticated.return_value = False

    result = await profile_sync_service.sync_user_profile(mock_update, mock_context)

    assert result is False
    mock_token_storage.is_authenticated.assert_called_once()


@pytest.mark.asyncio
async def test_sync_user_profile_api_error(
    profile_sync_service,
    mock_update,
    mock_context,
    mock_token_storage,
    mock_api_client,
):
    """Prueba manejo de error de API."""
    mock_token_storage.is_authenticated.return_value = True
    mock_token_storage.get.return_value = {
        "access_token": "test_token",
    }
    mock_api_client.post.side_effect = Exception("API Error")

    result = await profile_sync_service.sync_user_profile(mock_update, mock_context)

    assert result is False


@pytest.mark.asyncio
async def test_sync_user_profile_no_effective_user(
    profile_sync_service,
    mock_context,
):
    """Prueba manejo de effective_user None."""
    mock_update = MagicMock()
    mock_update.effective_user = None

    result = await profile_sync_service.sync_user_profile(mock_update, mock_context)

    assert result is False
```

**Step 2: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
uv run pytest tests/unit/bot/services/test_profile_sync.py -v
```

Expected: PASS

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add tests/unit/bot/services/test_profile_sync.py
git commit -m "test: add unit tests for ProfileSyncService"
```

---

### Task 10: Integrate Profile Sync into Main Menu Handler

**Files:**
- Modify: `usipipo-telegram-bot/src/bot/handlers/main_menu.py`

**Step 1: Read current main_menu.py**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
cat src/bot/handlers/main_menu.py
```

**Step 2: Add profile sync call when user accesses main menu**

```python
"""Handler para menú principal."""

import asyncio
import logging
from telegram import Update
from telegram.ext import ContextTypes

from src.bot.keyboards.main_menu import MainMenuKeyboard
from src.bot.services.profile_sync import ProfileSyncService
from src.infrastructure.api_client import APIClient
from src.infrastructure.token_storage import TokenStorage

logger = logging.getLogger(__name__)


async def show_main_menu(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE,
    api_client: APIClient | None = None,
    token_storage: TokenStorage | None = None,
) -> None:
    """
    Muestra menú principal y sincroniza perfil de Telegram.
    
    Args:
        update: Update de Telegram
        context: Contexto del bot
        api_client: Cliente de API (opcional, inyectado)
        token_storage: Almacenamiento de tokens (opcional, inyectado)
    """
    if update.effective_user is None:
        logger.warning("show_main_menu called without effective_user")
        return

    telegram_id = update.effective_user.id
    logger.info(f"User {telegram_id} accessing main menu")

    # Sync profile in background (non-blocking)
    if api_client and token_storage:
        profile_sync = ProfileSyncService(api_client, token_storage)
        # Fire and forget - don't block UI
        asyncio.create_task(
            profile_sync.sync_on_menu_access(update, context)
        )

    if update.callback_query:
        await update.callback_query.answer()
        await update.callback_query.edit_message_text(
            text=MainMenuKeyboard.MENU_TEXT,
            reply_markup=MainMenuKeyboard.main_menu(),
            parse_mode="Markdown",
        )
    elif update.message:
        await update.message.reply_text(
            text=MainMenuKeyboard.MENU_TEXT,
            reply_markup=MainMenuKeyboard.main_menu(),
            parse_mode="Markdown",
        )
```

**Note:** Need to add `import asyncio` at top of file.

**Step 3: Update main.py to pass dependencies to main_menu handler**

Modify: `usipipo-telegram-bot/src/main.py`

Find the main_menu handler registration and update:

```python
# Register main menu handler
from functools import partial
from src.bot.handlers.main_menu import show_main_menu
from telegram.ext import CallbackQueryHandler

# Create partial function with dependencies
main_menu_handler = partial(
    show_main_menu,
    api_client=_api_client,
    token_storage=_token_storage,
)
app.add_handler(CallbackQueryHandler(main_menu_handler, pattern="^main_menu$"))
```

**Step 4: Run bot tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
uv run pytest tests/bot/test_handlers.py -v
```

Expected: PASS

**Step 5: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/handlers/main_menu.py src/main.py
git commit -m "feat: sync Telegram profile when user accesses main menu"
```

---

### Task 11: Add Profile Sync to /start Handler (Returning Users)

**Files:**
- Modify: `usipipo-telegram-bot/src/bot/handlers/auth.py`

**Step 1: Update start_handler to sync profile for returning users**

```python
async def start_handler(
    self,
    update: Update,
    context: ContextTypes.DEFAULT_TYPE,
) -> None:
    """
    Maneja /start - registro y autenticación automática.
    
    Si el usuario ya está autenticado, muestra mensaje de bienvenida
    y sincroniza su perfil de Telegram.
    Si no, registra y autentica automáticamente.
    """
    if update.effective_user is None:
        logger.warning("start_handler called without effective_user")
        return

    telegram_id = update.effective_user.id

    # Check if already authenticated
    if await self.tokens.is_authenticated(telegram_id):
        logger.info(f"Returning user {telegram_id} - syncing profile")
        
        # Sync profile for returning user
        from src.bot.services.profile_sync import ProfileSyncService
        profile_sync = ProfileSyncService(self.api, self.tokens)
        asyncio.create_task(
            profile_sync.sync_on_menu_access(update, context)
        )
        
        if update.message:
            await update.message.reply_text(
                text=AuthMessages.WELCOME_RETURNING_USER,
                reply_markup=MainMenuKeyboard.main_menu(),
                parse_mode="Markdown",
            )
        return

    # Register and auto-authenticate new user
    await self._register_and_auth(telegram_id, update, context)
```

**Step 2: Add import for asyncio at top of file**

```python
import asyncio
import logging
from telegram import Update
# ... rest of imports
```

**Step 3: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
uv run pytest tests/bot/test_handlers.py -v -k start
```

Expected: PASS

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/handlers/auth.py
git commit -m "feat: sync profile for returning users on /start command"
```

---

## Phase 4: Testing & Verification

### Task 12: Create End-to-End Integration Test

**Files:**
- Create: `usipipo-telegram-bot/tests/integration/test_profile_flow.py`

**Step 1: Create comprehensive integration test**

```python
"""Tests de integración para flujo completo de perfil de Telegram."""

import pytest
from unittest.mock import AsyncMock, MagicMock
from telegram import User, Update
from telegram.ext import ContextTypes

from src.bot.handlers.auth import AuthHandler
from src.bot.handlers.user_profile import UserProfileHandler
from src.infrastructure.api_client import APIClient
from src.infrastructure.token_storage import TokenStorage


@pytest.fixture
def mock_telegram_user():
    """Mock de usuario de Telegram real."""
    return User(
        id=987654321,
        is_bot=False,
        username="develop",
        first_name="Developer",
        last_name="User",
    )


@pytest.fixture
def mock_update(mock_telegram_user):
    """Mock de Update con usuario real."""
    update = MagicMock(spec=Update)
    update.effective_user = mock_telegram_user
    update.message = MagicMock()
    update.message.reply_text = AsyncMock()
    update.callback_query = MagicMock()
    update.callback_query.answer = AsyncMock()
    update.callback_query.edit_message_text = AsyncMock()
    return update


@pytest.fixture
def mock_context():
    """Mock de contexto."""
    return MagicMock(spec=ContextTypes.DEFAULT_TYPE)


@pytest.fixture
def mock_api_client():
    """Mock de APIClient."""
    client = MagicMock(spec=APIClient)
    client.post = AsyncMock()
    client.get = AsyncMock()
    return client


@pytest.fixture
def mock_token_storage():
    """Mock de TokenStorage."""
    storage = MagicMock(spec=TokenStorage)
    storage.is_authenticated = AsyncMock()
    storage.get = AsyncMock()
    storage.store = AsyncMock()
    storage.delete = AsyncMock()
    storage.needs_refresh = AsyncMock()
    return storage


@pytest.mark.asyncio
async def test_new_user_registration_with_profile(
    mock_update,
    mock_context,
    mock_api_client,
    mock_token_storage,
):
    """Prueba registro de nuevo usuario con perfil de Telegram."""
    # Configurar mocks
    mock_token_storage.is_authenticated.return_value = False
    mock_api_client.post.return_value = {
        "access_token": "test_token",
        "refresh_token": "test_refresh",
        "expires_in": 3600,
        "user_id": "550e8400-e29b-41d4-a716-446655440000",
    }

    # Crear handler y ejecutar /start
    auth_handler = AuthHandler(mock_api_client, mock_token_storage)
    await auth_handler.start_handler(mock_update, mock_context)

    # Verificar que se envió perfil completo al backend
    mock_api_client.post.assert_called_once()
    call_args = mock_api_client.post.call_args[0]
    profile_data = call_args[1]
    
    assert profile_data["telegram_id"] == 987654321
    assert profile_data["username"] == "develop"
    assert profile_data["first_name"] == "Developer"
    assert profile_data["last_name"] == "User"
    
    # Verificar que se guardaron tokens
    mock_token_storage.store.assert_called_once()


@pytest.mark.asyncio
async def test_profile_display_shows_correct_data(
    mock_update,
    mock_context,
    mock_api_client,
    mock_token_storage,
):
    """Prueba que el perfil muestra datos correctos del usuario."""
    # Configurar mocks
    mock_token_storage.is_authenticated.return_value = True
    mock_token_storage.get.return_value = {
        "access_token": "test_token",
    }
    
    # Mock de respuesta del backend con datos reales del usuario
    mock_api_client.get.return_value = {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "telegram_id": 987654321,
        "username": "develop",
        "first_name": "Developer",
        "last_name": "User",
        "balance_gb": 5.0,
        "total_purchased_gb": 10.0,
        "referral_code": "ref_test123",
        "referral_credits": 2.5,
        "purchase_count": 2,
        "loyalty_bonus_percent": 5,
        "welcome_bonus_used": False,
        "created_at": "2024-01-01T00:00:00",
        "updated_at": "2024-01-15T00:00:00",
        "referred_users_with_purchase": 1,
    }

    # Crear handler y ejecutar show_user_profile
    profile_handler = UserProfileHandler(mock_api_client, mock_token_storage)
    await profile_handler.show_user_profile(mock_update, mock_context)

    # Verificar que se llamó a API para obtener perfil
    mock_api_client.get.assert_called()
    
    # Verificar que se mostró el mensaje con datos correctos
    mock_update.callback_query.edit_message_text.assert_called_once()
    message_text = mock_update.callback_query.edit_message_text.call_args[1]["text"]
    
    # Verificar que el mensaje contiene datos correctos
    assert "@develop" in message_text
    assert "Developer User" in message_text
    assert "987654321" in message_text


@pytest.mark.asyncio
async def test_profile_sync_on_menu_access(
    mock_update,
    mock_context,
    mock_api_client,
    mock_token_storage,
):
    """Prueba sincronización de perfil al acceder al menú."""
    # Configurar mocks
    mock_token_storage.is_authenticated.return_value = True
    mock_token_storage.get.return_value = {
        "access_token": "test_token",
    }
    mock_api_client.post.return_value = {
        "success": True,
        "user_id": "550e8400-e29b-41d4-a716-446655440000",
        "updated": True,
    }

    # Importar y crear servicio
    from src.bot.services.profile_sync import ProfileSyncService
    profile_sync = ProfileSyncService(mock_api_client, mock_token_storage)
    
    # Ejecutar sync
    result = await profile_sync.sync_user_profile(mock_update, mock_context)

    # Verificar resultados
    assert result is True
    mock_api_client.post.assert_called_once()
    call_args = mock_api_client.post.call_args[0]
    
    assert call_args[0] == "/users/sync-telegram-profile"
    profile_data = call_args[1]
    assert profile_data["telegram_id"] == 987654321
    assert profile_data["username"] == "develop"
```

**Step 2: Run integration tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
uv run pytest tests/integration/test_profile_flow.py -v
```

Expected: PASS

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add tests/integration/test_profile_flow.py
git commit -m "test: add end-to-end integration tests for Telegram profile flow"
```

---

### Task 13: Manual Testing Checklist

**Files:**
- Create: `usipipo-telegram-bot/docs/manual-testing-profile-sync.md`

**Step 1: Create manual testing guide**

```markdown
# Manual Testing Guide - Telegram Profile Sync

## Prerequisites
- Running backend instance
- Running Telegram bot instance
- Test Telegram account

## Test Cases

### TC1: New User Registration
1. Start bot with `/start` from new Telegram account
2. Verify welcome message appears
3. Click "Mis Datos" button
4. **Expected:** Username and name match Telegram profile exactly

### TC2: Existing User with Test Data
1. Use existing test user account (the one showing wrong data)
2. Send `/start` command
3. Click "Mis Datos" button
4. **Expected:** Profile now shows correct Telegram username and name

### TC3: Profile Update After Telegram Change
1. Change Telegram username in Telegram settings
2. Send `/start` to bot
3. Click "Mis Datos"
4. **Expected:** New username appears in profile

### TC4: Main Menu Access Sync
1. Send `/start` to open main menu
2. Wait 5 seconds
3. Click "Mis Datos"
4. **Expected:** Profile data is current

### TC5: Backend API Direct Test
```bash
# Test auto-register with profile data
curl -X POST http://localhost:8000/api/v1/auth/telegram/auto-register \
  -H "Content-Type: application/json" \
  -d '{
    "telegram_id": 987654321,
    "username": "test_user",
    "first_name": "Test",
    "last_name": "User"
  }'

# Test profile sync endpoint
curl -X POST http://localhost:8000/api/v1/users/sync-telegram-profile \
  -H "Content-Type: application/json" \
  -d '{
    "telegram_id": 987654321,
    "username": "updated_user",
    "first_name": "Updated",
    "last_name": "Name"
  }'

# Get user profile
curl -X GET http://localhost:8000/api/v1/users/me \
  -H "Authorization: Bearer <token_from_auto_register>"
```

**Expected:** Profile data updates correctly with each call
```

**Step 2: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add docs/manual-testing-profile-sync.md
git commit -m "docs: add manual testing guide for profile sync feature"
```

---

### Task 14: Run Full Test Suite

**Step 1: Run backend tests**

```bash
cd /home/mowgli/usipipo/usipipo-backend
uv run pytest tests/ -v --tb=short
```

Expected: All tests PASS

**Step 2: Run bot tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
uv run pytest tests/ -v --tb=short
```

Expected: All tests PASS

**Step 3: Run type checking**

```bash
cd /home/mowgli/usipipo/usipipo-backend
uv run mypy src/

cd /home/mowgli/usipipo/usipipo-telegram-bot
uv run mypy src/
```

Expected: No type errors

**Step 4: Run linters**

```bash
cd /home/mowgli/usipipo/usipipo-backend
uv run ruff check src/

cd /home/mowgli/usipipo/usipipo-telegram-bot
uv run ruff check src/
```

Expected: No linting errors

**Step 5: Commit all changes**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add .
git commit -m "chore: final cleanup for profile sync feature"

cd /home/mowgli/usipipo/usipipo-telegram-bot
git add .
git commit -m "chore: final cleanup for profile sync feature"
```

---

## Post-Implementation Verification

### Success Criteria

1. ✅ **New users:** Profile shows correct Telegram username and name immediately after `/start`
2. ✅ **Existing test users:** Profile updates to show correct Telegram data after `/start`
3. ✅ **Profile changes:** When user changes Telegram username, profile syncs on next `/start` or menu access
4. ✅ **No breaking changes:** Existing authentication flow continues to work
5. ✅ **All tests pass:** Unit, integration, and e2e tests pass
6. ✅ **Type safety:** No mypy errors
7. ✅ **Code quality:** No ruff linting errors

### Rollback Plan

If issues arise:

1. **Backend:** Revert commits from `git log` related to profile sync
2. **Bot:** Revert commits and redeploy previous version
3. **Database:** No schema changes needed, safe to rollback

---

## Future Enhancements (Not in Scope)

- Periodic background sync job (e.g., daily) for all users
- Profile change history/audit log
- Admin UI to view profile sync status
- Webhook-based sync when Telegram profile changes
```

**Step 2: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add docs/plans/2026-03-31-telegram-profile-sync.md
git commit -m "docs: add Telegram profile sync implementation plan"
```

---

**Plan complete and saved to `docs/plans/2026-03-31-telegram-profile-sync.md`. Two execution options:**

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**Which approach?**
