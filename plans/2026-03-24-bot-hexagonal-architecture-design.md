# Telegram Bot - Arquitectura Hexagonal (Consumer Pattern)

**Fecha:** 2026-03-24  
**Versión:** 1.0.0  
**Estado:** Diseño Aprobado  
**Branch:** `feature/hexagonal-architecture`  
**Ubicación:** `/plans/2026-03-24-bot-hexagonal-architecture-design.md`

---

## 🎯 Propósito

Este documento describe el diseño de arquitectura hexagonal para el **Telegram Bot** del ecosistema uSipipo, estableciendo los principios, patrones y convenciones para implementar el bot como un **consumidor del backend centralizado**.

---

## 📜 Principios Fundamentales

### 1. **Bot como Consumidor (Consumer Pattern)**

El bot **NO** tiene lógica de negocio propia. Es un **adaptador de UI** que:
- ✅ Muestra datos del backend
- ✅ Envía comandos al backend
- ✅ Traduce respuestas HTTP → Mensajes de Telegram
- ✅ NO duplica validaciones del backend
- ✅ NO tiene entidades propias (usa `usipipo-commons`)

```
┌─────────────────────────────────────────────────────────┐
│                 BACKEND (Cerebro)                        │
│  - TODA la lógica de negocio                            │
│  - Entidades: User, VpnKey, Payment, etc.               │
│  - Reglas de validación                                 │
│  - Persistencia (PostgreSQL + Redis)                    │
└─────────────────────────────────────────────────────────┘
                        │ HTTP/REST
                        ▼
┌─────────────────────────────────────────────────────────┐
│              TELEGRAM BOT (UI/UX)                        │
│  - Solo muestra datos del backend                       │
│  - Solo envía comandos al backend                       │
│  - Es un "adaptador" de Telegram                        │
└─────────────────────────────────────────────────────────┘
```

### 2. **Fuente Única de Verdad**

El backend es la **única fuente de verdad** para:
- Identidad de usuarios
- Estado de VPN keys
- Historial de pagos
- Suscripciones activas
- Tickets de soporte

El bot **solo cachéa** tokens de acceso en Redis local para rendimiento.

### 3. **Auth Invisible**

El usuario **nunca** debe percibir el proceso de autenticación:
- No hay `/login` o `/logout`
- Auto-refresh silencioso de tokens
- Re-auth automático si expira sesión

---

## 🏗️ Arquitectura Hexagonal

### Vista General

```
┌─────────────────────────────────────────────────────────────────┐
│                    TELEGRAM BOT (Consumer)                       │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  PRIMARY ADAPTERS (Driving)                               │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │  Telegram Handlers                                  │  │ │
│  │  │  - AuthHandler (/start, /me, /unlink)               │  │ │
│  │  │  - VpnKeyHandler (/keys, /newkey, /delkey)          │  │ │
│  │  │  - PaymentHandler (/payments)                       │  │ │
│  │  │  - ReferralHandler (/referrals)                     │  │ │
│  │  │  - TicketHandler (/tickets)                         │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │  Keyboards (UI components)                          │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  APPLICATION LAYER (Orchestration + Adaptation)           │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │  Use Cases (con adaptación de datos)                │  │ │
│  │  │  - AutoRegisterUser → UserDto                       │  │ │
│  │  │  - GetUserProfile → ProfileMessage                  │  │ │
│  │  │  - ListVpnKeys → KeysListMessage                    │  │ │
│  │  │  - CreatePayment → PaymentConfirmationMessage       │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │  Ports (Interfaces)                                 │  │ │
│  │  │  - BackendApiPort                                   │  │ │
│  │  │  - TokenStoragePort                                 │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  SECONDARY ADAPTERS (Driven)                              │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │  BackendApiAdapter (httpx)                          │  │ │
│  │  │  - HTTP Client hacia backend                        │  │ │
│  │  │  - Traduce errores HTTP → Domain Errors             │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │  RedisAdapter (caché local de tokens)               │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │ HTTP/REST
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              BACKEND (Cerebro - Lógica Real)                    │
│  - Entidades: User, VpnKey, Payment, etc.                       │
│  - Reglas de negocio                                            │
│  - Validaciones                                                 │
│  - Persistencia (PostgreSQL + Redis)                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📁 Estructura de Directorios

### Estructura Hexagonal

```
usipipo-telegram-bot/
├── src/
│   ├── __init__.py
│   ├── __main__.py
│   ├── main.py
│   │
│   ├── application/                      ← APLICACIÓN (Orquestación)
│   │   ├── __init__.py
│   │   ├── ports/                        ← Interfaces (contratos)
│   │   │   ├── __init__.py
│   │   │   ├── backend_api_port.py       ← Contrato para llamar al backend
│   │   │   └── token_storage_port.py     ← Contrato para caché de tokens
│   │   ├── use_cases/                    ← Casos de uso (con adaptación)
│   │   │   ├── __init__.py
│   │   │   ├── auth/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── auto_register_user.py
│   │   │   │   ├── refresh_token.py
│   │   │   │   └── get_user_profile.py
│   │   │   ├── vpn_keys/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── list_vpn_keys.py
│   │   │   │   ├── create_vpn_key.py
│   │   │   │   ├── delete_vpn_key.py
│   │   │   │   └── get_key_config.py
│   │   │   ├── payments/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── create_crypto_payment.py
│   │   │   │   └── create_stars_payment.py
│   │   │   └── referrals/
│   │   │       ├── __init__.py
│   │   │       ├── get_referral_code.py
│   │   │       └── get_referral_stats.py
│   │   └── dtos/                         ← DTOs para UI de Telegram
│   │       ├── __init__.py
│   │       ├── profile_message.py
│   │       ├── keys_list_message.py
│   │       └── payment_confirmation_message.py
│   │
│   └── infrastructure/                   ← INFRAESTRUCTURA (Adaptadores)
│       ├── __init__.py
│       ├── primary_adapters/             ← Driving (entradas)
│       │   ├── __init__.py
│       │   └── telegram/
│       │       ├── __init__.py
│       │       ├── handlers/
│       │       │   ├── __init__.py
│       │       │   ├── auth_handler.py
│       │       │   ├── basic_handler.py
│       │       │   ├── vpn_key_handler.py
│       │       │   ├── payment_handler.py
│       │       │   └── referral_handler.py
│       │       └── keyboards/
│       │           ├── __init__.py
│       │           ├── auth_keyboard.py
│       │           ├── main_menu_keyboard.py
│       │           ├── vpn_key_keyboard.py
│       │           └── payment_keyboard.py
│       ├── secondary_adapters/           ← Driven (salidas)
│       │   ├── __init__.py
│       │   ├── backend_api/
│       │   │   ├── __init__.py
│       │   │   ├── backend_api_adapter.py    ← Implementa BackendApiPort
│       │   │   ├── http_client.py            ← httpx.AsyncClient
│       │   │   ├── endpoints/
│       │   │   │   ├── auth_endpoints.py
│       │   │   │   ├── user_endpoints.py
│       │   │   │   ├── vpn_endpoints.py
│       │   │   │   └── payment_endpoints.py
│       │   │   └── error_translator.py       ← HTTP → Domain Errors
│       │   └── redis/
│       │       ├── __init__.py
│       │       ├── redis_adapter.py          ← Implementa TokenStoragePort
│       │       └── redis_pool.py             ← Connection pool
│       ├── config/
│       │   ├── __init__.py
│       │   └── settings.py
│       ├── logging/
│       │   ├── __init__.py
│       │   └── logger.py
│       └── error_handling/
│           ├── __init__.py
│           ├── exceptions.py
│           └── error_handler.py
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── unit/
│   │   ├── application/
│   │   │   └── test_use_cases.py
│   │   └── infrastructure/
│   │       ├── test_backend_api_adapter.py
│   │       └── test_redis_adapter.py
│   └── integration/
│       └── test_backend_integration.py
│
├── pyproject.toml
├── .env.example
├── .pre-commit-config.yaml
└── README.md
```

### Dominio (en usipipo-commons)

Las entidades **NO se duplican** en el bot. Se importan desde `usipipo-commons>=0.12.0`:

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

## 🔄 Flujo de Datos

### Ejemplo: Comando `/keys`

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Usuario envía: /keys                                         │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. VpnKeyHandler.handle_list_keys()                             │
│    - Obtiene telegram_id del usuario                            │
│    - Llama al Use Case: ListVpnKeys.execute(telegram_id)        │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. ListVpnKeys Use Case                                         │
│    a) Obtiene tokens de Redis (TokenStoragePort)                │
│    b) Verifica autenticación                                    │
│    c) Auto-refresh si token está por expirar                    │
│    d) Llama a BackendApiPort.list_vpn_keys(access_token)        │
│    e) Recibe lista de VpnKey (desde usipipo-commons)            │
│    f) ADAPTA a formato Telegram:                                │
│       - Construye mensaje formateado                            │
│       - Genera InlineKeyboardMarkup                             │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. BackendApiAdapter.list_vpn_keys()                            │
│    - HTTP GET /api/v1/vpn/keys                                  │
│    - Headers: Authorization: Bearer {access_token}              │
│    - Traduce errores HTTP → Domain Errors                       │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Backend retorna:                                             │
│    [VpnKey, VpnKey, ...]                                        │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. VpnKeyHandler envía mensaje:                                 │
│    "🔑 Tus VPN Keys                                              │
│                                                                  │
│    WireGuard:                                                    │
│      - Key 1: Activa (23 días restantes)                         │
│      - Key 2: Expirada                                           │
│                                                                  │
│    Outline:                                                      │
│      - Key 1: Activa (15 días restantes)                         │
│                                                                  │
│    [Botones inline para acciones]"                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔌 Ports (Interfaces)

### BackendApiPort

Define el contrato para comunicarse con el backend:

```python
from typing import Optional, List
from uuid import UUID

from usipipo_commons.domain.entities import User, VpnKey, Payment


class BackendApiPort(Protocol):
    """Contrato para comunicación con el backend API."""

    # Auth
    async def auto_register(self, telegram_id: int) -> dict:
        """Auto-registra usuario y retorna tokens."""

    async def refresh_tokens(self, refresh_token: str) -> dict:
        """Renueva tokens con refresh token."""

    async def get_user_profile(self, access_token: str) -> User:
        """Obtiene perfil del usuario."""

    # VPN Keys
    async def list_vpn_keys(self, access_token: str) -> List[VpnKey]:
        """Lista VPN keys del usuario."""

    async def create_vpn_key(
        self,
        access_token: str,
        name: str,
        key_type: str,
        data_limit_gb: float,
    ) -> VpnKey:
        """Crea nueva VPN key."""

    async def delete_vpn_key(
        self,
        access_token: str,
        key_id: UUID,
    ) -> bool:
        """Elimina VPN key."""

    async def get_key_config(
        self,
        access_token: str,
        key_id: UUID,
    ) -> str:
        """Obtiene configuración de VPN key."""

    # Payments
    async def create_crypto_payment(
        self,
        access_token: str,
        amount_usd: float,
        gb_purchased: float,
    ) -> Payment:
        """Crea pago con criptomoneda."""

    async def create_stars_payment(
        self,
        access_token: str,
        amount_usd: float,
        gb_purchased: float,
    ) -> Payment:
        """Crea pago con Telegram Stars."""

    # Referrals
    async def get_referral_code(self, access_token: str) -> str:
        """Obtiene código de referido."""

    async def get_referral_stats(
        self,
        access_token: str,
    ) -> dict:
        """Obtiene estadísticas de referidos."""
```

### TokenStoragePort

Define el contrato para caché local de tokens:

```python
from typing import Optional


class TokenStoragePort(Protocol):
    """Contrato para almacenamiento de tokens en Redis."""

    async def store(
        self,
        telegram_id: int,
        access_token: str,
        refresh_token: str,
        expires_in: int,
    ) -> None:
        """Guarda tokens con expiración automática."""

    async def get(
        self,
        telegram_id: int,
    ) -> Optional[dict]:
        """Recupera tokens del usuario."""

    async def delete(self, telegram_id: int) -> bool:
        """Elimina tokens (unlink)."""

    async def is_authenticated(self, telegram_id: int) -> bool:
        """Verifica si el usuario tiene tokens válidos."""

    async def refresh_if_needed(
        self,
        telegram_id: int,
        backend_api: BackendApiPort,
    ) -> bool:
        """Auto-refresh si token está por expirar (5 min)."""
```

---

## 📦 Use Cases (Con Adaptación)

Los Use Cases **NO son simples orquestadores**. Tienen responsabilidad de:
1. Orquestar llamadas al backend
2. **Adaptar datos** a formato específico para Telegram UI
3. Traducir errores a mensajes amigables

### Ejemplo: ListVpnKeys

```python
from typing import List, NamedTuple

from usipipo_commons.domain.entities import VpnKey
from usipipo_commons.domain.enums import KeyType, KeyStatus
from usipipo_commons.constants import FREE_KEYS_LIMIT

from ..ports.backend_api_port import BackendApiPort
from ..ports.token_storage_port import TokenStoragePort
from ..dtos.keys_list_message import KeysListMessage


class ListVpnKeysResult(NamedTuple):
    """Resultado del caso de uso."""
    message: str
    keyboard: InlineKeyboardMarkup
    keys_count: int


class ListVpnKeys:
    """Caso de uso: Listar VPN keys del usuario."""

    def __init__(
        self,
        backend_api: BackendApiPort,
        token_storage: TokenStoragePort,
    ):
        self.backend_api = backend_api
        self.token_storage = token_storage

    async def execute(self, telegram_id: int) -> ListVpnKeysResult:
        """
        Ejecuta el caso de uso.

        Args:
            telegram_id: ID de Telegram del usuario

        Returns:
            ListVpnKeysResult con mensaje y teclado formateados

        Raises:
            AuthenticationError: Si el usuario no está autenticado
            BackendConnectionError: Si el backend no responde
        """
        # 1. Obtener tokens
        tokens = await self.token_storage.get(telegram_id)
        if not tokens:
            raise AuthenticationError("Usuario no autenticado")

        # 2. Auto-refresh si es necesario
        await self.token_storage.refresh_if_needed(
            telegram_id,
            self.backend_api,
        )

        # 3. Obtener tokens actualizados
        tokens = await self.token_storage.get(telegram_id)
        access_token = tokens["access_token"]

        # 4. Llamar al backend
        try:
            keys: List[VpnKey] = await self.backend_api.list_vpn_keys(
                access_token,
            )
        except BackendConnectionError:
            raise BackendConnectionError(
                "No se pudo conectar con el backend. "
                "Intente nuevamente en unos segundos."
            )

        # 5. ADAPTAR: Construir mensaje para Telegram
        message_builder = KeysListMessageBuilder()
        message = message_builder.build(keys)
        keyboard = message_builder.build_keyboard(keys)

        return ListVpnKeysResult(
            message=message,
            keyboard=keyboard,
            keys_count=len(keys),
        )


class KeysListMessageBuilder:
    """Construye mensajes formateados para Telegram."""

    def build(self, keys: List[VpnKey]) -> str:
        """Construye mensaje con lista de keys."""
        if not keys:
            return (
                "🔑 Tus VPN Keys\n\n"
                "Aún no tienes claves VPN creadas.\n\n"
                "Usa /newkey para crear tu primera clave."
            )

        # Agrupar por tipo
        wireguard_keys = [k for k in keys if k.key_type == KeyType.WIREGUARD]
        outline_keys = [k for k in keys if k.key_type == KeyType.OUTLINE]

        lines = ["🔑 Tus VPN Keys\n"]

        if wireguard_keys:
            lines.append("🟦 *WireGuard:*")
            for key in wireguard_keys:
                status_emoji = "✅" if key.status == KeyStatus.ACTIVE else "❌"
                days_remaining = self._calculate_days_remaining(key.expires_at)
                lines.append(
                    f"  {status_emoji} {key.name}: "
                    f"{'Activa' if key.is_active else 'Expirada'} "
                    f"({days_remaining} días restantes)"
                )
            lines.append("")

        if outline_keys:
            lines.append("🟠 *Outline:*")
            for key in outline_keys:
                status_emoji = "✅" if key.status == KeyStatus.ACTIVE else "❌"
                days_remaining = self._calculate_days_remaining(key.expires_at)
                lines.append(
                    f"  {status_emoji} {key.name}: "
                    f"{'Activa' if key.is_active else 'Expirada'} "
                    f"({days_remaining} días restantes)"
                )

        lines.append(f"\nUsa /newkey para crear una nueva clave.")

        return "\n".join(lines)

    def build_keyboard(self, keys: List[VpnKey]) -> InlineKeyboardMarkup:
        """Construye teclado inline con acciones."""
        keyboard = []

        # Botón para crear nueva key
        keyboard.append([
            InlineKeyboardButton(
                text="➕ Crear Nueva Key",
                callback_data="vpn_key:create",
            )
        ])

        # Botones para cada key activa
        for key in keys:
            if key.is_active:
                keyboard.append([
                    InlineKeyboardButton(
                        text=f"🔑 {key.name}",
                        callback_data=f"vpn_key:info:{key.id}",
                    )
                ])

        return InlineKeyboardMarkup(keyboard)

    def _calculate_days_remaining(
        self,
        expires_at: Optional[datetime],
    ) -> int:
        """Calcula días restantes hasta expiración."""
        if not expires_at:
            return 0

        now = datetime.now(timezone.utc)
        delta = expires_at - now
        return max(0, delta.days)
```

---

## 🚫 Manejo de Errores

### Estrategia: Traducción Completa

Los errores HTTP del backend se traducen a **mensajes amigables de Telegram**:

```python
class ErrorTranslator:
    """Traduce errores HTTP a mensajes de Telegram."""

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
    def translate(cls, status_code: int, detail: Optional[str] = None) -> str:
        """
        Traduce error HTTP a mensaje de Telegram.

        Args:
            status_code: Código de estado HTTP
            detail: Detalle opcional del error

        Returns:
            Mensaje formateado para Telegram
        """
        base_message = cls.ERROR_MESSAGES.get(
            status_code,
            f"❌ Error inesperado (código {status_code}). Intenta nuevamente.",
        )

        if detail and status_code >= 500:
            # No mostrar detalles técnicos al usuario en errores de servidor
            logger.error(f"Backend error {status_code}: {detail}")
            return base_message

        if detail and status_code < 500:
            # Mostrar detalle para errores de cliente
            return f"{base_message}\n\nDetalle: {detail}"

        return base_message
```

### Jerarquía de Excepciones

```python
class BotError(Exception):
    """Excepción base del bot."""


class AuthenticationError(BotError):
    """Error de autenticación."""


class BackendConnectionError(BotError):
    """Error de conexión con el backend."""


class ValidationError(BotError):
    """Error de validación de datos."""


class NotFoundError(BotError):
    """Recurso no encontrado."""


class PermissionDeniedError(BotError):
    """Permiso denegado."""
```

---

## 🧪 Estrategia de Testing

### Pirámide de Tests

```
                    ┌───────────┐
                   │    E2E    │  ← Integration tests con backend real
                  │─────────────│
                 │  Integration  │ ← Tests de adaptadores
                │─────────────────│
               │     Unit Tests    │ ← Tests de Use Cases y handlers
              └─────────────────────┘
```

### Tests Unitarios (Application Layer)

```python
import pytest
from unittest.mock import AsyncMock, MagicMock

from src.application.use_cases.vpn_keys.list_vpn_keys import (
    ListVpnKeys,
    ListVpnKeysResult,
)
from src.application.ports.backend_api_port import BackendApiPort
from src.application.ports.token_storage_port import TokenStoragePort
from src.infrastructure.error_handling.exceptions import AuthenticationError


@pytest.fixture
def mock_backend_api() -> AsyncMock:
    """Mock de BackendApiPort."""
    return AsyncMock(spec=BackendApiPort)


@pytest.fixture
def mock_token_storage() -> AsyncMock:
    """Mock de TokenStoragePort."""
    return AsyncMock(spec=TokenStoragePort)


@pytest.mark.asyncio
async def test_list_vpn_keys_success(
    mock_backend_api: AsyncMock,
    mock_token_storage: AsyncMock,
):
    """Test de caso de uso exitoso."""
    # Arrange
    telegram_id = 1058749165
    tokens = {
        "access_token": "test_access_token",
        "refresh_token": "test_refresh_token",
    }
    mock_token_storage.get.return_value = tokens
    mock_token_storage.refresh_if_needed = AsyncMock()

    mock_keys = [
        VpnKey(
            id=uuid4(),
            user_id=uuid4(),
            name="Test Key",
            key_type=KeyType.WIREGUARD,
            status=KeyStatus.ACTIVE,
        ),
    ]
    mock_backend_api.list_vpn_keys.return_value = mock_keys

    use_case = ListVpnKeys(mock_backend_api, mock_token_storage)

    # Act
    result = await use_case.execute(telegram_id)

    # Assert
    assert isinstance(result, ListVpnKeysResult)
    assert result.keys_count == 1
    assert "🔑 Tus VPN Keys" in result.message
    mock_backend_api.list_vpn_keys.assert_called_once_with(
        "test_access_token",
    )


@pytest.mark.asyncio
async def test_list_vpn_keys_not_authenticated(
    mock_backend_api: AsyncMock,
    mock_token_storage: AsyncMock,
):
    """Test cuando usuario no está autenticado."""
    # Arrange
    telegram_id = 1058749165
    mock_token_storage.get.return_value = None

    use_case = ListVpnKeys(mock_backend_api, mock_token_storage)

    # Act & Assert
    with pytest.raises(AuthenticationError):
        await use_case.execute(telegram_id)
```

### Tests de Integración (Backend Real)

```python
import pytest
import httpx

from src.infrastructure.secondary_adapters.backend_api.backend_api_adapter import (
    BackendApiAdapter,
)


@pytest.mark.asyncio
async def test_backend_api_list_vpn_keys():
    """Test de integración con backend real."""
    # Arrange
    backend_url = "http://localhost:8001"
    access_token = "test_token_from_backend"

    adapter = BackendApiAdapter(backend_url, httpx.AsyncClient())

    # Act
    keys = await adapter.list_vpn_keys(access_token)

    # Assert
    assert isinstance(keys, list)
    assert all(isinstance(key, VpnKey) for key in keys)
```

---

## 📊 Dependencias

### pyproject.toml

```toml
[project]
name = "usipipo-telegram-bot"
version = "0.3.0"
description = "Bot de Telegram para interacción con usuarios del ecosistema uSipipo"
requires-python = ">=3.13"
license = {text = "MIT"}
authors = [{name = "uSipipo Team", email = "usipipo@gmail.com"}]

dependencies = [
    "usipipo-commons>=0.12.0",      # ← Entidades compartidas
    "python-telegram-bot>=21.0",    # ← SDK de Telegram
    "pydantic-settings>=2.0.0",     # ← Configuración
    "redis>=7.3.0",                 # ← Caché de tokens
    "httpx>=0.27.0",                # ← Cliente HTTP
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

---

## 🔗 Relación con Otros Documentos

### Documentos de Contexto

| Documento | Ubicación | Propósito |
|-----------|-----------|-----------|
| ECOSYSTEM-CONTEXT.md | `/plans/` | Contexto general del ecosistema |
| LEGACY-BOT-MIGRATION-SUMMARY.md | `/plans/` | Migración desde monorepo |
| sk-prompting.md | `/plans/` | Prompt para continuar migración |
| MIGRATION-PROGRESS.md | `/plans/` | Progreso de migración |

### Documentos de Implementación

| Documento | Ubicación | Propósito |
|-----------|-----------|-----------|
| Este documento | `/plans/` | Diseño de arquitectura hexagonal |
| 2026-03-23-telegram-bot-phase-1.md | `/plans/` | Fase 1: Auth invisible |
| 2026-04-08-semana-4-bot-refactor.md | `/plans/` | Refactor a APIs |

---

## 📈 Roadmap de Implementación

### Fase 1: Cimientos (Día 1)
- [ ] Crear estructura de directorios `application/`, `infrastructure/`
- [ ] Actualizar `pyproject.toml` con `usipipo-commons>=0.12.0`
- [ ] Definir `BackendApiPort` y `TokenStoragePort`
- [ ] Tests de configuración (5+ tests)

### Fase 2: Auth Use Cases (Día 2)
- [ ] `AutoRegisterUser` use case
- [ ] `RefreshToken` use case
- [ ] `GetUserProfile` use case
- [ ] Tests de use cases (10+ tests)

### Fase 3: Backend Adapter (Día 3)
- [ ] `BackendApiAdapter` (implementa `BackendApiPort`)
- [ ] `HttpClient` (httpx wrapper)
- [ ] `ErrorTranslator` (HTTP → Domain errors)
- [ ] Tests de adaptador (10+ tests)

### Fase 4: Redis Adapter (Día 4)
- [ ] `RedisAdapter` (implementa `TokenStoragePort`)
- [ ] `RedisPool` (connection pool)
- [ ] Tests de Redis adapter (5+ tests)

### Fase 5: VPN Key Management (Día 5-6)
- [ ] `ListVpnKeys` use case
- [ ] `CreateVpnKey` use case
- [ ] `DeleteVpnKey` use case
- [ ] `VpnKeyHandler` (Telegram handler)
- [ ] `VpnKeyKeyboard` (inline keyboard)
- [ ] Tests (15+ tests)

### Fase 6: Payments y Referrals (Día 7-8)
- [ ] `CreateCryptoPayment` use case
- [ ] `CreateStarsPayment` use case
- [ ] `GetReferralCode` use case
- [ ] Handlers y keyboards
- [ ] Tests (15+ tests)

### Fase 7: Integración (Día 9)
- [ ] Configurar Dependency Injection
- [ ] Integrar todos los handlers en `main.py`
- [ ] Tests de integración (10+ tests)
- [ ] Code review y refactor final

### Fase 8: Merge a Main (Día 10)
- [ ] Documentación actualizada
- [ ] CHANGELOG.md actualizado
- [ ] Merge a `main`
- [ ] Release v0.3.0

---

## ✅ Criterios de Aceptación

- [ ] Estructura hexagonal implementada
- [ ] Ports definidos como interfaces (Protocol)
- [ ] Use Cases con adaptación de datos
- [ ] Adapters implementados (Backend API, Redis)
- [ ] Handlers de Telegram integrados
- [ ] Tests unitarios por capa (60+ tests)
- [ ] Tests de integración (10+ tests)
- [ ] Manejo de errores con traducción completa
- [ ] Documentación actualizada
- [ ] CI/CD configurado
- [ ] mypy, ruff, bandit sin errores

---

## 🎯 Beneficios Esperados

| Beneficio | Descripción |
|-----------|-------------|
| **Testabilidad** | Mocks fáciles de ports para tests unitarios |
| **Mantenibilidad** | Cambios localizados en capas específicas |
| **Escalabilidad** | Nuevas features sin romper existing code |
| **Flexibilidad** | Cambiar infraestructura sin tocar aplicación |
| **Claridad** | Código auto-documentado por responsabilidades |
| **Reusabilidad** | Use Cases pueden ser usados por otros clients |

---

## 📚 Glosario

| Término | Definición |
|---------|------------|
| **Port** | Interface que define un contrato (ej: `BackendApiPort`) |
| **Adapter** | Implementación concreta de un port |
| **Use Case** | Caso de uso que orquesta lógica de aplicación |
| **DTO** | Data Transfer Object para transferencia de datos |
| **Consumer Pattern** | Patrón donde el bot solo consume backend |
| **Auth Invisible** | Autenticación que el usuario no percibe |

---

**Última Actualización:** 2026-03-24  
**Próxima Revisión:** 2026-03-25 (después de Fase 1)  
**Responsable:** uSipipo Team
