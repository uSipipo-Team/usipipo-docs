# Semana 4: Bot de Telegram - Refactor a APIs

**Fecha:** 8-14 Abril 2026  
**Objetivo:** Refactorizar el bot para que consuma APIs del backend en vez de usar lógica embebida

**Dependencias:** Semana 3 completada ✅ (Backend con todos los endpoints funcionando)

---

## 📦 Entregables de la Semana

- [ ] `usipipo-telegram-bot` con estructura basada en features
- [ ] Cliente HTTP hacia backend (`BackendClient`)
- [ ] Auth con JWT desde Telegram initData
- [ ] Todos los handlers refactorizados para consumir APIs
- [ ] Tests del bot con mock del backend
- [ ] Docker compose funcional para el bot

---

## 🎯 Tareas Detalladas

### Día 1-2: Estructura Base + Cliente HTTP

#### 1.1 Clonar y estructurar repositorio del bot

```bash
cd /home/mowgli
git clone git@github.com:usipipo/usipipo-telegram-bot.git
cd usipipo-telegram-bot

# Inicializar con uv
uv init --name usipipo-telegram-bot

# Crear estructura basada en features
mkdir -p src/core/domain/entities
mkdir -p src/features/start
mkdir -p src/features/key_management/{handlers,keyboards,messages}
mkdir -p src/features/payments/{handlers,keyboards,messages}
mkdir -p src/features/referrals/{handlers,keyboards,messages}
mkdir -p src/features/tickets/{handlers,keyboards,messages}
mkdir -p src/features/admin/{handlers,keyboards,messages}
mkdir -p src/features/user_management/{handlers,keyboards,messages}
mkdir -p src/infrastructure/api/{endpoints,services}
mkdir -p src/infrastructure/bot
mkdir -p src/infrastructure/config
mkdir -p src/shared
mkdir -p tests/unit
mkdir -p tests/integration
```

#### 1.2 Crear cliente HTTP hacia backend

**Archivo:** `src/infrastructure/api/client.py`
```python
from typing import Optional
from uuid import UUID

import httpx

from ...core.config import settings


class BackendClient:
    """Cliente HTTP para comunicar con el backend."""
    
    def __init__(self, base_url: str, telegram_init_data: str):
        self.base_url = base_url
        self.token = self._generate_jwt_from_telegram(telegram_init_data)
        self.http = httpx.AsyncClient(
            base_url=base_url,
            headers={
                "Authorization": f"Bearer {self.token}",
                "Content-Type": "application/json",
            },
            timeout=30.0,
        )
    
    def _generate_jwt_from_telegram(self, init_data: str) -> str:
        """
        Genera JWT validando initData de Telegram.
        
        NOTA: En producción, esto debería llamar al backend para autenticar.
        Aquí lo hacemos directo si compartimos el JWT_SECRET.
        """
        # Opción 1: Llamar al backend para autenticar
        # response = httpx.post(
        #     f"{self.base_url}/api/v1/auth/telegram",
        #     json={"init_data": init_data},
        # )
        # response.raise_for_status()
        # return response.json()["access_token"]
        
        # Opción 2: Generar JWT directamente (requiere compartir secret)
        # (Solo para desarrollo, no recomendado en producción)
        import jwt
        from urllib.parse import parse_qs
        import hashlib
        import hmac
        
        # Validar initData primero
        parsed = parse_qs(init_data)
        data = {k: v[0] for k, v in parsed.items()}
        received_hash = data.pop("hash", None)
        
        # Verificar hash (mismo algoritmo que el backend)
        data_check_string = "\n".join(f"{k}={v}" for k, v in sorted(data.items()))
        secret_key = hmac.new(
            b"WebAppData",
            settings.TELEGRAM_TOKEN.encode(),
            hashlib.sha256,
        ).digest()
        calculated_hash = hmac.new(
            secret_key,
            data_check_string.encode(),
            hashlib.sha256,
        ).hexdigest()
        
        if not hmac.compare_digest(calculated_hash, received_hash):
            raise ValueError("Invalid Telegram initData")
        
        # Extraer user info
        import json
        user = json.loads(data.get("user", "{}"))
        telegram_id = int(user["id"])
        
        # Generar JWT (necesita UUID del usuario, que obtenemos del backend)
        # Por simplicidad, usamos telegram_id como sub
        payload = {
            "sub": f"telegram_{telegram_id}",
            "telegram_id": telegram_id,
            "exp": datetime.utcnow() + timedelta(hours=24),
            "iat": datetime.utcnow(),
        }
        
        return jwt.encode(payload, settings.JWT_SECRET, algorithm="HS256")
    
    async def close(self):
        """Cierra el cliente HTTP."""
        await self.http.aclose()
```

#### 1.3 Crear endpoints del cliente

**Archivo:** `src/infrastructure/api/endpoints/users.py`
```python
from typing import Optional, Dict, Any


class UsersEndpoint:
    """Endpoints de usuarios del backend."""
    
    def __init__(self, client: "BackendClient"):
        self.client = client
    
    async def get_me(self) -> Optional[Dict[str, Any]]:
        """Obtiene perfil del usuario actual."""
        response = await self.client.http.get("/api/v1/users/me")
        if response.status_code == 200:
            return response.json()
        return None
    
    async def update_me(self, data: Dict[str, Any]) -> Optional[Dict[str, Any]]:
        """Actualiza perfil del usuario."""
        response = await self.client.http.put(
            "/api/v1/users/me",
            json=data,
        )
        if response.status_code == 200:
            return response.json()
        return None
```

**Archivo:** `src/infrastructure/api/endpoints/vpn.py`
```python
from typing import List, Optional, Dict, Any
from uuid import UUID


class VpnEndpoint:
    """Endpoints de VPN del backend."""
    
    def __init__(self, client: "BackendClient"):
        self.client = client
    
    async def list_keys(self) -> List[Dict[str, Any]]:
        """Lista todas las claves VPN del usuario."""
        response = await self.client.http.get("/api/v1/vpn/keys")
        if response.status_code == 200:
            return response.json()
        return []
    
    async def create_key(
        self,
        name: str,
        vpn_type: str,
        data_limit_gb: float = 5.0,
    ) -> Optional[Dict[str, Any]]:
        """Crea una nueva clave VPN."""
        response = await self.client.http.post(
            "/api/v1/vpn/keys",
            json={
                "name": name,
                "vpn_type": vpn_type,
                "data_limit_gb": data_limit_gb,
            },
        )
        if response.status_code == 201:
            return response.json()
        return None
    
    async def get_key(self, key_id: UUID) -> Optional[Dict[str, Any]]:
        """Obtiene detalles de una clave."""
        response = await self.client.http.get(f"/api/v1/vpn/keys/{key_id}")
        if response.status_code == 200:
            return response.json()
        return None
    
    async def delete_key(self, key_id: UUID) -> bool:
        """Elimina una clave VPN."""
        response = await self.client.http.delete(f"/api/v1/vpn/keys/{key_id}")
        return response.status_code == 204
    
    async def get_key_config(self, key_id: UUID) -> Optional[str]:
        """Obtiene configuración de clave (WireGuard config)."""
        response = await self.client.http.get(f"/api/v1/vpn/keys/{key_id}/config")
        if response.status_code == 200:
            data = response.json()
            return data.get("config")
        return None
```

**Archivo:** `src/infrastructure/api/endpoints/payments.py`
```python
from typing import List, Optional, Dict, Any
from uuid import UUID


class PaymentsEndpoint:
    """Endpoints de pagos del backend."""
    
    def __init__(self, client: "BackendClient"):
        self.client = client
    
    async def create_crypto_payment(
        self,
        amount_usd: float,
        gb_purchased: float,
        network: str = "BSC",
    ) -> Optional[Dict[str, Any]]:
        """Crea un pago con criptomoneda."""
        response = await self.client.http.post(
            "/api/v1/payments/crypto",
            json={
                "amount_usd": amount_usd,
                "gb_purchased": gb_purchased,
                "network": network,
            },
        )
        if response.status_code == 201:
            return response.json()
        return None
    
    async def create_stars_payment(
        self,
        amount_usd: float,
        gb_purchased: float,
    ) -> Optional[Dict[str, Any]]:
        """Crea un pago con Telegram Stars."""
        response = await self.client.http.post(
            "/api/v1/payments/stars",
            json={
                "amount_usd": amount_usd,
                "gb_purchased": gb_purchased,
            },
        )
        if response.status_code == 201:
            return response.json()
        return None
    
    async def get_history(self) -> List[Dict[str, Any]]:
        """Obtiene historial de pagos."""
        response = await self.client.http.get("/api/v1/payments/history")
        if response.status_code == 200:
            return response.json()
        return []
```

**Archivo:** `src/infrastructure/api/endpoints/billing.py`
```python
from typing import Optional, Dict, Any
from uuid import UUID


class BillingEndpoint:
    """Endpoints de billing del backend."""
    
    def __init__(self, client: "BackendClient"):
        self.client = client
    
    async def get_usage(self) -> Optional[Dict[str, Any]]:
        """Obtiene consumo de datos del usuario."""
        response = await self.client.http.get("/api/v1/billing/usage")
        if response.status_code == 200:
            return response.json()
        return None
    
    async def get_key_usage(self, key_id: UUID) -> Optional[Dict[str, Any]]:
        """Obtiene consumo de una clave específica."""
        response = await self.client.http.get(f"/api/v1/billing/usage/{key_id}")
        if response.status_code == 200:
            return response.json()
        return None
```

**Archivo:** `src/infrastructure/api/endpoints/referrals.py`
```python
from typing import Optional, Dict, Any, List


class ReferralsEndpoint:
    """Endpoints de referidos del backend."""
    
    def __init__(self, client: "BackendClient"):
        self.client = client
    
    async def get_referral_code(self) -> Optional[str]:
        """Obtiene código de referido del usuario."""
        response = await self.client.http.get("/api/v1/referrals/code")
        if response.status_code == 200:
            data = response.json()
            return data.get("referral_code")
        return None
    
    async def get_referrals(self) -> List[Dict[str, Any]]:
        """Obtiene lista de referidos."""
        response = await self.client.http.get("/api/v1/referrals")
        if response.status_code == 200:
            return response.json()
        return []
```

#### 1.4 Unificar todos los endpoints

**Archivo:** `src/infrastructure/api/client.py` (actualizar)
```python
# ... (código anterior)

from .endpoints.users import UsersEndpoint
from .endpoints.vpn import VpnEndpoint
from .endpoints.payments import PaymentsEndpoint
from .endpoints.billing import BillingEndpoint
from .endpoints.referrals import ReferralsEndpoint


class BackendClient:
    """Cliente HTTP para comunicar con el backend."""
    
    def __init__(self, base_url: str, telegram_init_data: str):
        self.base_url = base_url
        self.token = self._generate_jwt_from_telegram(telegram_init_data)
        self.http = httpx.AsyncClient(
            base_url=base_url,
            headers={
                "Authorization": f"Bearer {self.token}",
                "Content-Type": "application/json",
            },
            timeout=30.0,
        )
        
        # Endpoints
        self.users = UsersEndpoint(self)
        self.vpn = VpnEndpoint(self)
        self.payments = PaymentsEndpoint(self)
        self.billing = BillingEndpoint(self)
        self.referrals = ReferralsEndpoint(self)
    
    # ... (resto del código)
```

---

### Día 3-4: Refactorizar Handlers del Bot

#### 2.1 Handler de /start

**Archivo:** `src/features/start/handler.py`
```python
from telegram import Update
from telegram.ext import CommandHandler, ContextTypes

from ...infrastructure.bot import get_bot_client
from .messages import StartMessages
from .keyboards import StartKeyboards


async def handle_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Maneja el comando /start."""
    user = update.effective_user
    
    # Obtener initData para auth
    init_data = update.message.web_app_data.data if update.message.web_app_data else None
    
    async with get_bot_client(init_data) as backend:
        # Obtener perfil del usuario
        profile = await backend.users.get_me()
        
        if profile:
            # Usuario existente
            message = StartMessages.welcome_back(
                first_name=user.first_name,
                balance_gb=profile["balance_gb"],
            )
        else:
            # Usuario nuevo
            message = StartMessages.welcome_new(first_name=user.first_name)
        
        keyboard = StartKeyboards.main_menu()
        
        await update.message.reply_text(
            text=message,
            reply_markup=keyboard,
            parse_mode="HTML",
        )
```

**Archivo:** `src/features/start/messages.py`
```python
from usipipo_commons.constants import FREE_GB


class StartMessages:
    """Mensajes del feature de start."""
    
    @staticmethod
    def welcome_new(first_name: str) -> str:
        """Mensaje de bienvenida para usuario nuevo."""
        return (
            f"👋 ¡Hola <b>{first_name}</b>! Bienvenido a <b>uSipipo VPN</b>\n\n"
            f"🎁 Te regalamos <b>{FREE_GB} GB gratis</b> para comenzar.\n\n"
            f"¿Qué puedes hacer aquí?\n"
            f"• Crear claves VPN (WireGuard y Outline)\n"
            f"• Comprar GB adicionales\n"
            f"• Invitar amigos y ganar GB gratis\n"
            f"• Gestionar tu perfil y consumo\n\n"
            f"Usa el menú de abajo para comenzar. 🚀"
        )
    
    @staticmethod
    def welcome_back(first_name: str, balance_gb: float) -> str:
        """Mensaje de bienvenida para usuario existente."""
        return (
            f"👋 ¡Hola de nuevo <b>{first_name}</b>!\n\n"
            f"💰 Tu saldo actual: <b>{balance_gb} GB</b>\n\n"
            f"¿En qué puedo ayudarte hoy?"
        )
```

#### 2.2 Handler de gestión de claves

**Archivo:** `src/features/key_management/handlers/create_key.py`
```python
from telegram import Update, CallbackQuery
from telegram.ext import CallbackQueryHandler, ContextTypes

from ....infrastructure.bot import get_bot_client
from ....infrastructure.api.exceptions import (
    VpnKeyLimitReachedError,
    BackendApiError,
)
from ..messages import KeyManagementMessages
from ..keyboards import KeyManagementKeyboards


async def handle_create_key_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Maneja callback para crear nueva clave."""
    query: CallbackQuery = update.callback_query
    await query.answer()
    
    init_data = _get_init_data_from_query(query)
    
    async with get_bot_client(init_data) as backend:
        # Mostrar opciones de tipo de VPN
        keyboard = KeyManagementKeyboards.vpn_type_selection()
        message = KeyManagementMessages.select_vpn_type()
        
        await query.edit_message_text(
            text=message,
            reply_markup=keyboard,
            parse_mode="HTML",
        )


async def handle_vpn_type_selected(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Maneja selección de tipo de VPN."""
    query: CallbackQuery = update.callback_query
    await query.answer()
    
    vpn_type = query.data.split(":")[1]  # "vpn_type:wireguard"
    init_data = _get_init_data_from_query(query)
    
    async with get_bot_client(init_data) as backend:
        try:
            # Crear clave
            key = await backend.vpn.create_key(
                name=f"Key-{context.user_data.get('key_counter', 1)}",
                vpn_type=vpn_type,
                data_limit_gb=5.0,
            )
            
            message = KeyManagementMessages.key_created(
                key_name=key["name"],
                vpn_type=key["vpn_type"],
                data_limit_gb=key["data_limit_gb"],
            )
            
            # Mostrar config o QR
            keyboard = KeyManagementKeyboards.key_actions(key["id"])
            
            await query.edit_message_text(
                text=message,
                reply_markup=keyboard,
                parse_mode="HTML",
            )
            
        except VpnKeyLimitReachedError:
            message = KeyManagementMessages.key_limit_reached()
            await query.edit_message_text(text=message, parse_mode="HTML")
        
        except BackendApiError as e:
            message = KeyManagementMessages.creation_failed(error=str(e))
            await query.edit_message_text(text=message, parse_mode="HTML")
```

**Archivo:** `src/features/key_management/handlers/view_keys.py`
```python
from telegram import Update, CallbackQuery
from telegram.ext import CallbackQueryHandler, ContextTypes

from ....infrastructure.bot import get_bot_client
from ..messages import KeyManagementMessages
from ..keyboards import KeyManagementKeyboards


async def handle_view_keys(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Maneja comando para ver claves."""
    query: CallbackQuery = update.callback_query
    await query.answer()
    
    init_data = _get_init_data_from_query(query)
    
    async with get_bot_client(init_data) as backend:
        keys = await backend.vpn.list_keys()
        
        if not keys:
            message = KeyManagementMessages.no_keys()
            await query.edit_message_text(text=message, parse_mode="HTML")
            return
        
        message = KeyManagementMessages.keys_list(keys)
        keyboard = KeyManagementKeyboards.keys_list(keys)
        
        await query.edit_message_text(
            text=message,
            reply_markup=keyboard,
            parse_mode="HTML",
        )
```

#### 2.3 Handler de pagos

**Archivo:** `src/features/payments/handlers/buy_gb.py`
```python
from telegram import Update, CallbackQuery
from telegram.ext import CallbackQueryHandler, ContextTypes

from ....infrastructure.bot import get_bot_client
from ..messages import PaymentsMessages
from ..keyboards import PaymentsKeyboards


async def handle_buy_gb(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Maneja comando para comprar GB."""
    query: CallbackQuery = update.callback_query
    await query.answer()
    
    init_data = _get_init_data_from_query(query)
    
    async with get_bot_client(init_data) as backend:
        # Obtener consumo actual
        usage = await backend.billing.get_usage()
        
        message = PaymentsMessages.buy_gb_menu(
            balance_gb=usage["balance_gb"],
            usage_percentage=usage["usage_percentage"],
        )
        
        keyboard = PaymentsKeyboards.payment_methods()
        
        await query.edit_message_text(
            text=message,
            reply_markup=keyboard,
            parse_mode="HTML",
        )


async def handle_crypto_payment_selected(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Maneja selección de pago con crypto."""
    query: CallbackQuery = update.callback_query
    await query.answer()
    
    amount_usd = float(query.data.split(":")[1])  # "crypto:10.0"
    gb_purchased = amount_usd * 2  # $1 = 2 GB
    init_data = _get_init_data_from_query(query)
    
    async with get_bot_client(init_data) as backend:
        payment = await backend.payments.create_crypto_payment(
            amount_usd=amount_usd,
            gb_purchased=gb_purchased,
            network="BSC",
        )
        
        message = PaymentsMessages.crypto_payment_instructions(
            amount_usd=payment["amount_usd"],
            crypto_address=payment["crypto_address"],
            network=payment["crypto_network"],
            expires_at=payment["expires_at"],
        )
        
        # Mostrar QR con dirección crypto
        qr_image = _generate_qr_code(payment["crypto_address"])
        
        await query.edit_message_media(
            media=InputMediaPhoto(media=qr_image, caption=message),
            parse_mode="HTML",
        )
```

#### 2.4 Handler de referidos

**Archivo:** `src/features/referrals/handlers/referral.py`
```python
from telegram import Update, CallbackQuery
from telegram.ext import CallbackQueryHandler, ContextTypes

from ....infrastructure.bot import get_bot_client
from ..messages import ReferralMessages
from ..keyboards import ReferralKeyboards


async def handle_referral(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Maneja comando /referir."""
    query: CallbackQuery = update.callback_query
    await query.answer()
    
    init_data = _get_init_data_from_query(query)
    
    async with get_bot_client(init_data) as backend:
        referral_code = await backend.referrals.get_referral_code()
        referrals = await backend.referrals.get_referrals()
        
        message = ReferralMessages.referral_info(
            referral_code=referral_code,
            referrals_count=len(referrals),
            total_earned_gb=sum(r["bonus_gb"] for r in referrals),
        )
        
        keyboard = ReferralKeyboards.referral_actions(referral_code)
        
        await query.edit_message_text(
            text=message,
            reply_markup=keyboard,
            parse_mode="HTML",
        )
```

---

### Día 5: Utilitarios del Bot

#### 3.1 Factory del cliente backend

**Archivo:** `src/infrastructure/bot/__init__.py`
```python
from contextlib import asynccontextmanager
from typing import Optional

from .application import create_bot_application
from ..api.client import BackendClient
from ...core.config import settings


@asynccontextmanager
async def get_bot_client(telegram_init_data: Optional[str] = None):
    """
    Context manager para obtener cliente del backend.
    
    Usage:
        async with get_bot_client(init_data) as backend:
            user = await backend.users.get_me()
    """
    if not telegram_init_data:
        raise ValueError("Telegram initData required")
    
    client = BackendClient(
        base_url=settings.BACKEND_URL,
        telegram_init_data=telegram_init_data,
    )
    
    try:
        yield client
    finally:
        await client.close()
```

#### 3.2 Obtener initData desde callbacks

**Archivo:** `src/infrastructure/bot/utils.py`
```python
from telegram import CallbackQuery


def _get_init_data_from_query(query: CallbackQuery) -> str:
    """
    Obtiene initData desde un CallbackQuery.
    
    NOTA: Esto requiere que el frontend (Telegram WebApp) envíe el initData.
    En producción, usar WebApp.initData directamente.
    """
    # Opción 1: Desde web_app_data si está disponible
    if query.message and query.message.web_app_data:
        return query.message.web_app_data.data
    
    # Opción 2: Desde el contexto del usuario (almacenado temporalmente)
    # (Requiere guardar initData en context.user_data al iniciar)
    
    # Opción 3: Pedir al usuario que inicie desde la Mini App
    raise ValueError("InitData not available. Please start from Mini App.")
```

#### 3.3 Configurar settings

**Archivo:** `src/infrastructure/config/settings.py`
```python
from pydantic_settings import BaseSettings, Field


class BotSettings(BaseSettings):
    """Configuración del bot."""
    
    # Telegram
    TELEGRAM_TOKEN: str = Field(..., description="Token del bot de Telegram")
    BOT_USERNAME: str = Field(default="usipipo_bot", description="Username del bot")
    
    # Backend
    BACKEND_URL: str = Field(
        default="http://localhost:8000",
        description="URL del backend API",
    )
    
    # JWT (compartido con backend para desarrollo)
    JWT_SECRET: str = Field(..., description="Secret para JWT")
    
    # Feature flags
    ENABLE_ADMIN_PANEL: bool = Field(default=True, description="Activar panel de admin")
    ENABLE_CRYPTO_PAYMENTS: bool = Field(default=True, description="Activar pagos crypto")
    ENABLE_STARS_PAYMENTS: bool = Field(default=True, description="Activar Telegram Stars")
    
    class Config:
        env_file = ".env"
        case_sensitive = True


settings = BotSettings()
```

---

### Día 6: Tests del Bot

#### 4.1 Tests unitarios con mocks

**Archivo:** `tests/unit/features/test_start_handler.py`
```python
import pytest
from unittest.mock import AsyncMock, MagicMock, patch

from src.features.start.handler import handle_start


@pytest.mark.asyncio
async def test_handle_start_new_user():
    """Test de /start para usuario nuevo."""
    # Mock update
    update = MagicMock()
    update.effective_user.first_name = "Test"
    update.message.web_app_data = None
    
    context = MagicMock()
    
    # Mock backend client
    with patch("src.features.start.handler.get_bot_client") as mock_client_ctx:
        mock_backend = AsyncMock()
        mock_backend.users.get_me.return_value = None  # Usuario nuevo
        mock_client_ctx.return_value.__aenter__.return_value = mock_backend
        
        await handle_start(update, context)
        
        # Verificar que se llamó a reply_text con mensaje de bienvenida
        update.message.reply_text.assert_called_once()
        call_args = update.message.reply_text.call_args
        assert "Bienvenido" in call_args[1]["text"] or "Bienvenido" in call_args[0][0]
```

**Archivo:** `tests/unit/features/test_vpn_handler.py`
```python
import pytest
from unittest.mock import AsyncMock, MagicMock, patch

from src.features.key_management.handlers.create_key import handle_create_key_callback


@pytest.mark.asyncio
async def test_handle_create_key_success():
    """Test de creación de clave exitosa."""
    update = MagicMock()
    update.callback_query.data = "create_key"
    update.callback_query.answer = AsyncMock()
    update.callback_query.edit_message_text = AsyncMock()
    
    context = MagicMock()
    
    with patch("src.features.key_management.handlers.create_key.get_bot_client") as mock_client_ctx:
        mock_backend = AsyncMock()
        mock_backend.vpn.create_key.return_value = {
            "id": "uuid-123",
            "name": "Test Key",
            "vpn_type": "wireguard",
            "data_limit_gb": 5.0,
        }
        mock_client_ctx.return_value.__aenter__.return_value = mock_backend
        
        await handle_create_key_callback(update, context)
        
        # Verificar que se creó la clave
        mock_backend.vpn.create_key.assert_called_once()
        update.callback_query.edit_message_text.assert_called_once()
```

---

### Día 7: Docker + Deploy

#### 5.1 Crear docker-compose.yml

**Archivo:** `docker-compose.yml`
```yaml
version: '3.8'

services:
  bot:
    build: .
    environment:
      - TELEGRAM_TOKEN=${TELEGRAM_TOKEN}
      - BACKEND_URL=http://backend:8000
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - backend
    restart: unless-stopped

  # Opcional: ejecutar backend localmente para testing
  # backend:
  #   image: usipipo-backend:latest
  #   environment:
  #     - DATABASE_URL=postgresql+asyncpg://usipipo:pass@db:5432/usipipo
  #   depends_on:
  #     - db
  #     - redis
```

#### 5.2 Crear Dockerfile

**Archivo:** `Dockerfile`
```dockerfile
FROM python:3.13-slim

WORKDIR /app

# Instalar uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

# Copiar dependencias
COPY pyproject.toml uv.lock ./

# Instalar dependencias
RUN uv sync --frozen --no-dev

# Copiar código
COPY src/ ./src/

# Comando
CMD ["uv", "run", "python", "-m", "src.infrastructure.bot.application"]
```

---

## ✅ Criterios de Aceptación

- [ ] `usipipo-telegram-bot` tiene estructura basada en features
- [ ] `BackendClient` funciona y autentica con JWT
- [ ] Todos los handlers principales refactorizados:
  - [ ] /start
  - [ ] /keys (ver claves)
  - [ ] /newkey (crear clave)
  - [ ] /buy (comprar GB)
  - [ ] /referir (referidos)
- [ ] Tests unitarios pasando (mínimo 70% coverage)
- [ ] Docker compose funcional
- [ ] El bot puede conectarse al backend en producción

---

## 📚 Recursos

- [python-telegram-bot v21 docs](https://docs.python-telegram-bot.org/)
- [httpx async client](https://www.python-httpx.org/async/)
- [Telegram WebApp initData](https://core.telegram.org/bots/webapps#validating-data-received-via-the-mini-app)

---

## 🔄 Dependencias para Semana 5

La Semana 5 necesita:
- ✅ Bot consumiendo APIs del backend
- ✅ Auth JWT funcionando
- ✅ Backend estable en producción o staging

---

## 📝 Notas

- El bot ya NO debe importar nada de `application/services` o `domain/entities` del backend
- Toda comunicación con el backend debe ser vía HTTP
- Los errores de API deben manejarse gracefulmente (reintentos, mensajes de error claros)
- Considerar rate limiting en el bot para evitar saturar el backend
