# Telegram Bot Hexagonal Architecture Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Refactor Telegram Bot from flat structure to hexagonal architecture (Consumer Pattern) with backend as single source of truth.

**Architecture:** Hexagonal (Ports & Adapters) with 3 layers: Primary Adapters (Telegram handlers) → Application (Use Cases with data adaptation) → Secondary Adapters (Backend API, Redis). Bot consumes backend APIs only, no business logic duplication.

**Tech Stack:** Python 3.13, aiogram 3.x, httpx, redis, pydantic-settings, usipipo-commons>=0.12.0, pytest, mypy, ruff.

---

## Context Summary

Based on design document `/plans/2026-03-24-bot-hexagonal-architecture-design.md`:

### Key Principles:
1. **Bot as Consumer** - No business logic, only UI adapter
2. **Single Source of Truth** - Backend owns all data/entities
3. **Invisible Auth** - Auto-refresh tokens, no /login or /logout

### Current Structure (Pre-Refactor):
```
src/
├── bot/
│   ├── handlers/
│   └── keyboards/
└── infrastructure/
    ├── api_client.py
    ├── config.py
    ├── redis.py
    └── token_storage.py
```

### Target Structure (Post-Refactor):
```
src/
├── application/
│   ├── ports/
│   ├── use_cases/
│   └── dtos/
└── infrastructure/
    ├── primary_adapters/
    └── secondary_adapters/
```

---

## Phase 1: Foundation & Setup (Day 1)

### Task 1.1: Update pyproject.toml

**Files:**
- Modify: `usipipo-telegram-bot/pyproject.toml`

**Step 1: Update dependencies**

```toml
[project]
name = "usipipo-telegram-bot"
version = "0.3.0"
description = "Bot de Telegram para interacción con usuarios del ecosistema uSipipo"
requires-python = ">=3.13"
license = {text = "MIT"}
authors = [{name = "uSipipo Team", email = "usipipo@gmail.com"}]

dependencies = [
    "usipipo-commons>=0.12.0",
    "python-telegram-bot>=21.0",
    "pydantic-settings>=2.0.0",
    "redis>=7.3.0",
    "httpx>=0.27.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.0.0",
    "mypy>=1.0.0",
    "ruff>=0.1.0",
]
```

**Step 2: Install dependencies**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
uv sync
```

Expected: All dependencies installed successfully

**Step 3: Commit**

```bash
git add pyproject.toml
git commit -m "chore: update dependencies for hexagonal architecture"
```

---

### Task 1.2: Create Directory Structure

**Files:**
- Create: `usipipo-telegram-bot/src/application/__init__.py`
- Create: `usipipo-telegram-bot/src/application/ports/__init__.py`
- Create: `usipipo-telegram-bot/src/application/use_cases/__init__.py`
- Create: `usipipo-telegram-bot/src/application/dtos/__init__.py`
- Create: `usipipo-telegram-bot/src/infrastructure/primary_adapters/__init__.py`
- Create: `usipipo-telegram-bot/src/infrastructure/primary_adapters/telegram/__init__.py`
- Create: `usipipo-telegram-bot/src/infrastructure/primary_adapters/telegram/handlers/__init__.py`
- Create: `usipipo-telegram-bot/src/infrastructure/primary_adapters/telegram/keyboards/__init__.py`
- Create: `usipipo-telegram-bot/src/infrastructure/secondary_adapters/__init__.py`
- Create: `usipipo-telegram-bot/src/infrastructure/secondary_adapters/backend_api/__init__.py`
- Create: `usipipo-telegram-bot/src/infrastructure/secondary_adapters/redis/__init__.py`

**Step 1: Create directories**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot/src

# Application layer
mkdir -p application/ports
mkdir -p application/use_cases/auth
mkdir -p application/use_cases/vpn_keys
mkdir -p application/use_cases/payments
mkdir -p application/use_cases/referrals
mkdir -p application/dtos

# Infrastructure layer
mkdir -p infrastructure/primary_adapters/telegram/handlers
mkdir -p infrastructure/primary_adapters/telegram/keyboards
mkdir -p infrastructure/secondary_adapters/backend_api/endpoints
mkdir -p infrastructure/secondary_adapters/redis

# Config, logging, error_handling (already exist, move if needed)
mkdir -p infrastructure/config
mkdir -p infrastructure/logging
mkdir -p infrastructure/error_handling
```

**Step 2: Create __init__.py files**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot/src

# Application layer
touch application/__init__.py
touch application/ports/__init__.py
touch application/use_cases/__init__.py
touch application/use_cases/auth/__init__.py
touch application/use_cases/vpn_keys/__init__.py
touch application/use_cases/payments/__init__.py
touch application/use_cases/referrals/__init__.py
touch application/dtos/__init__.py

# Infrastructure layer
touch infrastructure/primary_adapters/__init__.py
touch infrastructure/primary_adapters/telegram/__init__.py
touch infrastructure/primary_adapters/telegram/handlers/__init__.py
touch infrastructure/primary_adapters/telegram/keyboards/__init__.py
touch infrastructure/secondary_adapters/__init__.py
touch infrastructure/secondary_adapters/backend_api/__init__.py
touch infrastructure/secondary_adapters/backend_api/endpoints/__init__.py
touch infrastructure/secondary_adapters/redis/__init__.py
```

**Step 3: Verify structure**

```bash
tree -L 4 -I '__pycache__|*.pyc'
```

Expected: Directory structure matches target

**Step 4: Commit**

```bash
git add -A
git commit -m "feat: create hexagonal architecture directory structure"
```

---

### Task 1.3: Define BackendApiPort

**Files:**
- Create: `usipipo-telegram-bot/src/application/ports/backend_api_port.py`
- Test: `usipipo-telegram-bot/tests/unit/application/ports/test_backend_api_port.py`

**Step 1: Write the port interface**

```python
"""Backend API Port - Contract for backend communication."""

from typing import List, Optional, Protocol
from uuid import UUID

from usipipo_commons.domain.entities import User, VpnKey, Payment


class BackendApiPort(Protocol):
    """
    Contrato para comunicación con el backend API.
    
    Define todos los métodos necesarios para interactuar
    con el backend centralizado de uSipipo.
    """

    # Auth endpoints
    async def auto_register(self, telegram_id: int) -> dict:
        """
        Auto-registra usuario y retorna tokens.
        
        Args:
            telegram_id: ID de Telegram del usuario
            
        Returns:
            dict con access_token, refresh_token, expires_in
        """

    async def refresh_tokens(self, refresh_token: str) -> dict:
        """
        Renueva tokens con refresh token.
        
        Args:
            refresh_token: Refresh token actual
            
        Returns:
            dict con nuevos access_token y refresh_token
        """

    async def get_user_profile(self, access_token: str) -> User:
        """
        Obtiene perfil del usuario.
        
        Args:
            access_token: JWT access token
            
        Returns:
            User entity con datos del usuario
        """

    # VPN Keys endpoints
    async def list_vpn_keys(self, access_token: str) -> List[VpnKey]:
        """
        Lista VPN keys del usuario.
        
        Args:
            access_token: JWT access token
            
        Returns:
            Lista de VpnKey entities
        """

    async def create_vpn_key(
        self,
        access_token: str,
        name: str,
        key_type: str,
        data_limit_gb: float = 5.0,
    ) -> VpnKey:
        """
        Crea nueva VPN key.
        
        Args:
            access_token: JWT access token
            name: Nombre de la key
            key_type: Tipo de VPN (wireguard, outline)
            data_limit_gb: Límite de datos en GB
            
        Returns:
            VpnKey entity creada
        """

    async def delete_vpn_key(
        self,
        access_token: str,
        key_id: UUID,
    ) -> bool:
        """
        Elimina VPN key.
        
        Args:
            access_token: JWT access token
            key_id: ID de la key a eliminar
            
        Returns:
            True si se eliminó correctamente
        """

    async def get_key_config(
        self,
        access_token: str,
        key_id: UUID,
    ) -> str:
        """
        Obtiene configuración de VPN key.
        
        Args:
            access_token: JWT access token
            key_id: ID de la key
            
        Returns:
        Configuración (WireGuard config o ss:// URL)
        """

    # Payments endpoints
    async def create_crypto_payment(
        self,
        access_token: str,
        amount_usd: float,
        gb_purchased: float,
    ) -> Payment:
        """
        Crea pago con criptomoneda.
        
        Args:
            access_token: JWT access token
            amount_usd: Monto en USD
            gb_purchased: GB comprados
            
        Returns:
            Payment entity creada
        """

    async def create_stars_payment(
        self,
        access_token: str,
        amount_usd: float,
        gb_purchased: float,
    ) -> Payment:
        """
        Crea pago con Telegram Stars.
        
        Args:
            access_token: JWT access token
            amount_usd: Monto en USD
            gb_purchased: GB comprados
            
        Returns:
            Payment entity creada
        """

    # Referrals endpoints
    async def get_referral_code(self, access_token: str) -> str:
        """
        Obtiene código de referido.
        
        Args:
            access_token: JWT access token
            
        Returns:
            Código de referido
        """

    async def get_referral_stats(self, access_token: str) -> dict:
        """
        Obtiene estadísticas de referidos.
        
        Args:
            access_token: JWT access token
            
        Returns:
            dict con stats (referrals_count, bonus_earned_gb, etc.)
        """
```

**Step 2: Run mypy to verify type hints**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
mypy src/application/ports/backend_api_port.py --no-error-summary
```

Expected: 0 errors

**Step 3: Commit**

```bash
git add src/application/ports/backend_api_port.py
git commit -m "feat: define BackendApiPort interface"
```

---

### Task 1.4: Define TokenStoragePort

**Files:**
- Create: `usipipo-telegram-bot/src/application/ports/token_storage_port.py`
- Test: `usipipo-telegram-bot/tests/unit/application/ports/test_token_storage_port.py`

**Step 1: Write the port interface**

```python
"""Token Storage Port - Contract for token caching."""

from typing import Optional, Protocol

from .backend_api_port import BackendApiPort


class TokenStoragePort(Protocol):
    """
    Contrato para almacenamiento de tokens en Redis.
    
    Proporciona caché local de tokens JWT para evitar
    llamadas innecesarias al backend.
    """

    async def store(
        self,
        telegram_id: int,
        access_token: str,
        refresh_token: str,
        expires_in: int,
    ) -> None:
        """
        Guarda tokens con expiración automática.
        
        Args:
            telegram_id: ID de Telegram del usuario
            access_token: JWT access token
            refresh_token: JWT refresh token
            expires_in: Segundos hasta expiración
        """

    async def get(self, telegram_id: int) -> Optional[dict]:
        """
        Recupera tokens del usuario.
        
        Args:
            telegram_id: ID de Telegram del usuario
            
        Returns:
            dict con access_token y refresh_token, o None
        """

    async def delete(self, telegram_id: int) -> bool:
        """
        Elimina tokens (unlink).
        
        Args:
            telegram_id: ID de Telegram del usuario
            
        Returns:
            True si se eliminó, False si no existía
        """

    async def is_authenticated(self, telegram_id: int) -> bool:
        """
        Verifica si el usuario tiene tokens válidos.
        
        Args:
            telegram_id: ID de Telegram del usuario
            
        Returns:
            True si tiene tokens válidos
        """

    async def refresh_if_needed(
        self,
        telegram_id: int,
        backend_api: BackendApiPort,
    ) -> bool:
        """
        Auto-refresh si token está por expirar (5 min).
        
        Args:
            telegram_id: ID de Telegram del usuario
            backend_api: BackendApiPort para refresh
            
        Returns:
            True si se hizo refresh, False si no era necesario
        """
```

**Step 2: Run mypy to verify type hints**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
mypy src/application/ports/token_storage_port.py --no-error-summary
```

Expected: 0 errors

**Step 3: Commit**

```bash
git add src/application/ports/token_storage_port.py
git commit -m "feat: define TokenStoragePort interface"
```

---

### Task 1.5: Define Custom Exceptions

**Files:**
- Create: `usipipo-telegram-bot/src/infrastructure/error_handling/exceptions.py`
- Test: `usipipo-telegram-bot/tests/unit/infrastructure/error_handling/test_exceptions.py`

**Step 1: Write exception hierarchy**

```python
"""Custom exceptions for Telegram Bot."""


class BotError(Exception):
    """Excepción base del bot."""

    def __init__(self, message: str, original_exception: Optional[Exception] = None):
        super().__init__(message)
        self.message = message
        self.original_exception = original_exception


class AuthenticationError(BotError):
    """
    Error de autenticación.
    
    Se lanza cuando el usuario no está autenticado o
    los tokens son inválidos.
    """


class BackendConnectionError(BotError):
    """
    Error de conexión con el backend.
    
    Se lanza cuando no se puede conectar con el backend
    o la respuesta es inválida.
    """


class ValidationError(BotError):
    """
    Error de validación de datos.
    
    Se lanza cuando los datos de entrada son inválidos.
    """


class NotFoundError(BotError):
    """
    Recurso no encontrado.
    
    Se lanza cuando el backend retorna 404.
    """


class PermissionDeniedError(BotError):
    """
    Permiso denegado.
    
    Se lanza cuando el backend retorna 403.
    """


class RateLimitError(BotError):
    """
    Límite de tasa excedido.
    
    Se lanza cuando el backend retorna 429.
    """
```

**Step 2: Write tests**

```python
"""Tests for custom exceptions."""

import pytest

from src.infrastructure.error_handling.exceptions import (
    BotError,
    AuthenticationError,
    BackendConnectionError,
    ValidationError,
    NotFoundError,
    PermissionDeniedError,
    RateLimitError,
)


def test_bot_error_basic():
    """Test básico de BotError."""
    error = BotError("Test message")
    assert str(error) == "Test message"
    assert error.message == "Test message"
    assert error.original_exception is None


def test_bot_error_with_original():
    """Test de BotError con excepción original."""
    original = ValueError("Original error")
    error = BotError("Test message", original)
    assert error.message == "Test message"
    assert error.original_exception is original


def test_authentication_error():
    """Test de AuthenticationError."""
    error = AuthenticationError("Not authenticated")
    assert isinstance(error, BotError)
    assert "Not authenticated" in str(error)


def test_backend_connection_error():
    """Test de BackendConnectionError."""
    error = BackendConnectionError("Backend unavailable")
    assert isinstance(error, BotError)


def test_validation_error():
    """Test de ValidationError."""
    error = ValidationError("Invalid data")
    assert isinstance(error, BotError)


def test_not_found_error():
    """Test de NotFoundError."""
    error = NotFoundError("Resource not found")
    assert isinstance(error, BotError)


def test_permission_denied_error():
    """Test de PermissionDeniedError."""
    error = PermissionDeniedError("Access denied")
    assert isinstance(error, BotError)


def test_rate_limit_error():
    """Test de RateLimitError."""
    error = RateLimitError("Too many requests")
    assert isinstance(error, BotError)
```

**Step 3: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
pytest tests/unit/infrastructure/error_handling/test_exceptions.py -v
```

Expected: 8 passed

**Step 4: Commit**

```bash
git add src/infrastructure/error_handling/exceptions.py tests/unit/infrastructure/error_handling/test_exceptions.py
git commit -m "feat: define custom exception hierarchy"
```

---

### Task 1.6: Define ErrorTranslator

**Files:**
- Create: `usipipo-telegram-bot/src/infrastructure/secondary_adapters/backend_api/error_translator.py`
- Test: `usipipo-telegram-bot/tests/unit/infrastructure/secondary_adapters/backend_api/test_error_translator.py`

**Step 1: Write error translator**

```python
"""HTTP to Domain Error Translator."""

import logging
from typing import Optional

from ....error_handling.exceptions import (
    BotError,
    AuthenticationError,
    BackendConnectionError,
    ValidationError,
    NotFoundError,
    PermissionDeniedError,
    RateLimitError,
)


logger = logging.getLogger(__name__)


class ErrorTranslator:
    """
    Traduce errores HTTP a mensajes amigables de Telegram.
    
    Mapea códigos de estado HTTP a excepciones de dominio
    con mensajes apropiados para usuarios de Telegram.
    """

    ERROR_MESSAGES = {
        400: "❌ Solicitud inválida. Verifica los datos e intenta nuevamente.",
        401: "❌ Sesión expirada. Por favor inicia sesión nuevamente.",
        403: "❌ Acceso denegado. No tienes permisos para esta acción.",
        404: "❌ No encontrado. El recurso solicitado no existe.",
        409: "❌ Conflicto. Ya existe un recurso con estos datos.",
        422: "❌ Datos inválidos. Verifica el formato e intenta nuevamente.",
        429: "⏳ Demasiadas solicitudes. Espera unos segundos e intenta nuevamente.",
        500: "❌ Error interno del servidor. Intenta nuevamente más tarde.",
        502: "❌ Servicio no disponible. Intenta nuevamente más tarde.",
        503: "❌ Servicio temporalmente no disponible. Intenta más tarde.",
    }

    @classmethod
    def translate(
        cls,
        status_code: int,
        detail: Optional[str] = None,
        original_exception: Optional[Exception] = None,
    ) -> BotError:
        """
        Traduce error HTTP a excepción de dominio.
        
        Args:
            status_code: Código de estado HTTP
            detail: Detalle opcional del error
            original_exception: Excepción original (opcional)
            
        Returns:
            Excepción de dominio apropiada
        """
        # Client errors (4xx)
        if status_code == 400:
            return ValidationError(
                cls.ERROR_MESSAGES[400],
                original_exception,
            )
        elif status_code == 401:
            return AuthenticationError(
                cls.ERROR_MESSAGES[401],
                original_exception,
            )
        elif status_code == 403:
            return PermissionDeniedError(
                cls.ERROR_MESSAGES[403],
                original_exception,
            )
        elif status_code == 404:
            return NotFoundError(
                cls.ERROR_MESSAGES[404],
                original_exception,
            )
        elif status_code == 409:
            return ValidationError(
                cls.ERROR_MESSAGES[409],
                original_exception,
            )
        elif status_code == 422:
            return ValidationError(
                cls.ERROR_MESSAGES[422],
                original_exception,
            )
        elif status_code == 429:
            return RateLimitError(
                cls.ERROR_MESSAGES[429],
                original_exception,
            )
        
        # Server errors (5xx)
        elif status_code >= 500:
            # Log detailed error for server errors
            logger.error(
                f"Backend server error {status_code}: {detail or 'No detail'}",
                exc_info=original_exception,
            )
            return BackendConnectionError(
                cls.ERROR_MESSAGES.get(
                    status_code,
                    f"❌ Error inesperado (código {status_code}). Intenta nuevamente.",
                ),
                original_exception,
            )
        
        # Unknown errors
        else:
            return BotError(
                f"❌ Error inesperado (código {status_code}). Intenta nuevamente.",
                original_exception,
            )

    @classmethod
    def translate_connection_error(
        cls,
        original_exception: Exception,
    ) -> BackendConnectionError:
        """
        Traduce error de conexión a excepción de dominio.
        
        Args:
            original_exception: Excepción original (httpx.ConnectError, etc.)
            
        Returns:
            BackendConnectionError
        """
        return BackendConnectionError(
            "❌ No se pudo conectar con el backend. "
            "Verifica tu conexión e intenta nuevamente.",
            original_exception,
        )
```

**Step 2: Write tests**

```python
"""Tests for ErrorTranslator."""

import pytest

from src.infrastructure.secondary_adapters.backend_api.error_translator import (
    ErrorTranslator,
)
from src.infrastructure.error_handling.exceptions import (
    AuthenticationError,
    BackendConnectionError,
    NotFoundError,
    PermissionDeniedError,
    RateLimitError,
    ValidationError,
)


def test_translate_400_validation_error():
    """Test de traducción de error 400."""
    error = ErrorTranslator.translate(400)
    assert isinstance(error, ValidationError)
    assert "Solicitud inválida" in error.message


def test_translate_401_authentication_error():
    """Test de traducción de error 401."""
    error = ErrorTranslator.translate(401)
    assert isinstance(error, AuthenticationError)
    assert "Sesión expirada" in error.message


def test_translate_403_permission_denied():
    """Test de traducción de error 403."""
    error = ErrorTranslator.translate(403)
    assert isinstance(error, PermissionDeniedError)
    assert "Acceso denegado" in error.message


def test_translate_404_not_found():
    """Test de traducción de error 404."""
    error = ErrorTranslator.translate(404)
    assert isinstance(error, NotFoundError)
    assert "No encontrado" in error.message


def test_translate_429_rate_limit():
    """Test de traducción de error 429."""
    error = ErrorTranslator.translate(429)
    assert isinstance(error, RateLimitError)
    assert "Demasiadas solicitudes" in error.message


def test_translate_500_server_error():
    """Test de traducción de error 500."""
    error = ErrorTranslator.translate(500)
    assert isinstance(error, BackendConnectionError)
    assert "Error interno" in error.message


def test_translate_with_detail():
    """Test de traducción con detalle."""
    error = ErrorTranslator.translate(400, detail="Email inválido")
    assert isinstance(error, ValidationError)
    # Los errores de cliente muestran el detalle
    assert "Email inválido" in str(error)


def test_translate_unknown_error():
    """Test de traducción de error desconocido."""
    error = ErrorTranslator.translate(418)  # I'm a teapot
    assert isinstance(error, Exception)
    assert "418" in str(error)


def test_translate_connection_error():
    """Test de traducción de error de conexión."""
    original = ConnectionError("Network unreachable")
    error = ErrorTranslator.translate_connection_error(original)
    assert isinstance(error, BackendConnectionError)
    assert "No se pudo conectar" in error.message
    assert error.original_exception is original
```

**Step 3: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
pytest tests/unit/infrastructure/secondary_adapters/backend_api/test_error_translator.py -v
```

Expected: 9 passed

**Step 4: Commit**

```bash
git add src/infrastructure/secondary_adapters/backend_api/error_translator.py tests/unit/infrastructure/secondary_adapters/backend_api/test_error_translator.py
git commit -m "feat: implement ErrorTranslator for HTTP to domain error mapping"
```

---

## Phase 2: Auth Use Cases (Day 2)

### Task 2.1: Implement AutoRegisterUser Use Case

**Files:**
- Create: `usipipo-telegram-bot/src/application/use_cases/auth/auto_register_user.py`
- Test: `usipipo-telegram-bot/tests/unit/application/use_cases/auth/test_auto_register_user.py`

**Step 1: Write the use case**

```python
"""AutoRegisterUser Use Case."""

from typing import NamedTuple

from ...ports.backend_api_port import BackendApiPort
from ...ports.token_storage_port import TokenStoragePort


class AutoRegisterUserResult(NamedTuple):
    """Resultado del caso de uso."""
    success: bool
    message: str
    user_id: str


class AutoRegisterUser:
    """
    Caso de uso: Auto-registrar usuario con Telegram ID.
    
    Flujo:
    1. Llama al backend para auto-registrar usuario
    2. Guarda tokens en Redis
    3. Retorna resultado
    """

    def __init__(
        self,
        backend_api: BackendApiPort,
        token_storage: TokenStoragePort,
    ):
        self.backend_api = backend_api
        self.token_storage = token_storage

    async def execute(self, telegram_id: int) -> AutoRegisterUserResult:
        """
        Ejecuta el caso de uso.
        
        Args:
            telegram_id: ID de Telegram del usuario
            
        Returns:
            AutoRegisterUserResult con el resultado
            
        Raises:
            BackendConnectionError: Si el backend no responde
        """
        # 1. Auto-registrar en backend
        tokens = await self.backend_api.auto_register(telegram_id)
        
        # 2. Guardar tokens en Redis
        await self.token_storage.store(
            telegram_id,
            tokens["access_token"],
            tokens["refresh_token"],
            tokens["expires_in"],
        )
        
        # 3. Retornar resultado
        return AutoRegisterUserResult(
            success=True,
            message="✅ ¡Bienvenido a uSipipo!\n\n"
                    "Tu cuenta ha sido creada y estás autenticado.\n\n"
                    "Usa /help para ver los comandos disponibles.",
            user_id=tokens.get("user_id", "unknown"),
        )
```

**Step 2: Write tests**

```python
"""Tests for AutoRegisterUser use case."""

import pytest
from unittest.mock import AsyncMock

from src.application.use_cases.auth.auto_register_user import (
    AutoRegisterUser,
    AutoRegisterUserResult,
)


@pytest.fixture
def mock_backend_api():
    """Mock de BackendApiPort."""
    return AsyncMock()


@pytest.fixture
def mock_token_storage():
    """Mock de TokenStoragePort."""
    return AsyncMock()


@pytest.mark.asyncio
async def test_auto_register_success(
    mock_backend_api: AsyncMock,
    mock_token_storage: AsyncMock,
):
    """Test de auto-registro exitoso."""
    # Arrange
    telegram_id = 1058749165
    tokens = {
        "access_token": "test_access",
        "refresh_token": "test_refresh",
        "expires_in": 1800,
        "user_id": "uuid-1234",
    }
    mock_backend_api.auto_register.return_value = tokens
    
    use_case = AutoRegisterUser(mock_backend_api, mock_token_storage)
    
    # Act
    result = await use_case.execute(telegram_id)
    
    # Assert
    assert isinstance(result, AutoRegisterUserResult)
    assert result.success is True
    assert "¡Bienvenido" in result.message
    assert result.user_id == "uuid-1234"
    
    # Verify backend was called
    mock_backend_api.auto_register.assert_called_once_with(telegram_id)
    
    # Verify tokens were stored
    mock_token_storage.store.assert_called_once_with(
        telegram_id,
        "test_access",
        "test_refresh",
        1800,
    )
```

**Step 3: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
pytest tests/unit/application/use_cases/auth/test_auto_register_user.py -v
```

Expected: 1 passed

**Step 4: Commit**

```bash
git add src/application/use_cases/auth/auto_register_user.py tests/unit/application/use_cases/auth/test_auto_register_user.py
git commit -m "feat: implement AutoRegisterUser use case"
```

---

## Phase 3: Backend API Adapter (Day 3)

### Task 3.1: Implement HTTP Client

**Files:**
- Create: `usipipo-telegram-bot/src/infrastructure/secondary_adapters/backend_api/http_client.py`
- Test: `usipipo-telegram-bot/tests/unit/infrastructure/secondary_adapters/backend_api/test_http_client.py`

**Step 1: Write HTTP client wrapper**

```python
"""HTTP Client for Backend API."""

import httpx
from typing import Optional


class BackendHttpClient:
    """
    Cliente HTTP para comunicar con el backend.
    
    Wrapper alrededor de httpx.AsyncClient con configuración
    específica para el backend de uSipipo.
    """

    def __init__(
        self,
        base_url: str,
        timeout: float = 30.0,
    ):
        """
        Inicializa el cliente HTTP.
        
        Args:
            base_url: URL base del backend
            timeout: Timeout en segundos
        """
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout
        self._client: Optional[httpx.AsyncClient] = None

    async def get_client(self) -> httpx.AsyncClient:
        """
        Obtiene o crea el cliente HTTP.
        
        Returns:
            httpx.AsyncClient configurado
        """
        if self._client is None or self._client.is_closed:
            self._client = httpx.AsyncClient(
                base_url=self.base_url,
                timeout=httpx.Timeout(self.timeout),
                headers={
                    "Content-Type": "application/json",
                    "Accept": "application/json",
                },
            )
        return self._client

    async def close(self) -> None:
        """Cierra el cliente HTTP."""
        if self._client and not self._client.is_closed:
            await self._client.aclose()
            self._client = None

    async def __aenter__(self) -> "BackendHttpClient":
        """Context manager entry."""
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        """Context manager exit."""
        await self.close()
```

**Step 2: Write tests**

```python
"""Tests for BackendHttpClient."""

import pytest
import httpx

from src.infrastructure.secondary_adapters.backend_api.http_client import (
    BackendHttpClient,
)


@pytest.mark.asyncio
async def test_http_client_creation():
    """Test de creación de cliente HTTP."""
    client = BackendHttpClient("http://localhost:8001")
    assert client.base_url == "http://localhost:8001"
    assert client.timeout == 30.0


@pytest.mark.asyncio
async def test_http_client_get_client():
    """Test de obtención de cliente."""
    client = BackendHttpClient("http://localhost:8001")
    httpx_client = await client.get_client()
    assert isinstance(httpx_client, httpx.AsyncClient)
    assert not httpx_client.is_closed


@pytest.mark.asyncio
async def test_http_client_close():
    """Test de cierre de cliente."""
    client = BackendHttpClient("http://localhost:8001")
    await client.get_client()
    await client.close()
    assert client._client is None or client._client.is_closed


@pytest.mark.asyncio
async def test_http_client_context_manager():
    """Test de context manager."""
    async with BackendHttpClient("http://localhost:8001") as client:
        httpx_client = await client.get_client()
        assert isinstance(httpx_client, httpx.AsyncClient)
    
    # After context, client should be closed
    assert client._client is None or client._client.is_closed
```

**Step 3: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
pytest tests/unit/infrastructure/secondary_adapters/backend_api/test_http_client.py -v
```

Expected: 4 passed

**Step 4: Commit**

```bash
git add src/infrastructure/secondary_adapters/backend_api/http_client.py tests/unit/infrastructure/secondary_adapters/backend_api/test_http_client.py
git commit -m "feat: implement BackendHttpClient wrapper"
```

---

## Summary of Remaining Phases

### Phase 4: Redis Adapter (Day 4)
- Implement RedisAdapter (TokenStoragePort)
- Implement RedisPool
- Tests: 10+

### Phase 5: VPN Key Management (Day 5-6)
- Implement ListVpnKeys, CreateVpnKey, DeleteVpnKey use cases
- Implement VpnKeyHandler, VpnKeyKeyboard
- Tests: 15+

### Phase 6: Payments and Referrals (Day 7-8)
- Implement CreateCryptoPayment, CreateStarsPayment use cases
- Implement GetReferralCode use case
- Handlers and keyboards
- Tests: 15+

### Phase 7: Integration (Day 9)
- Configure Dependency Injection
- Integrate all handlers in main.py
- Integration tests: 10+

### Phase 8: Merge to Main (Day 10)
- Update documentation
- Update CHANGELOG.md
- Merge to main
- Release v0.3.0

---

## Testing Strategy

### Unit Tests (Application Layer)
- Mock ports, test use cases in isolation
- Verify data adaptation logic
- Verify error handling

### Integration Tests (Backend Real)
- Test with production backend
- Verify end-to-end flows
- Test Redis integration

### Code Quality
```bash
# Run all checks
ruff check src/ tests/
mypy src/
pytest tests/ --cov=src --cov-report=term-missing
bandit -r src/
```

Expected: 0 errors, >80% coverage

---

**Plan complete and saved to `plans/2026-03-24-hexagonal-architecture-implementation-plan.md`. Two execution options:**

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**Which approach?**
