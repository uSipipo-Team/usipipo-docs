# Plan de Implementación: Autenticación Invisible Telegram Bot

**Fecha:** 2026-03-23  
**Fase:** 1 - Telegram Bot Auth  
**Estado:** Pendiente de implementación  
**Contexto:** `/plans/ECOSYSTEM-CONTEXT.md`

---

## 🎯 **Objetivo**

Implementar autenticación invisible en el Telegram Bot con persistencia en Redis, siguiendo el diseño de ecosistema centralizado.

---

## 📋 **Requisitos**

### **Funcionales**
1. Usuario se registra con `/start` sin intervención manual de auth
2. Tokens almacenados en Redis con expiración automática
3. Auto-refresh silencioso de access tokens (5 min antes de expirar)
4. Comandos de funcionalidad verifican auth automáticamente
5. Comando `/unlink` para revocar acceso (excepcional)

### **No Funcionales**
1. Redis como almacenamiento de tokens (producción-ready)
2. Connection pool para Redis (10 conexiones máx)
3. Timeouts y retries configurados
4. 15+ tests cubriendo auth flow
5. mypy, ruff, pytest pasando

---

## 🏗️ **Arquitectura de Implementación**

```
┌─────────────────────────────────────────────────────────────┐
│                    Telegram Bot                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  main.py                                                    │
│    │                                                        │
│    ├── Application (Telegram)                               │
│    │   ├── CommandHandler("/start") → start_handler()       │
│    │   ├── CommandHandler("/help") → help_handler()         │
│    │   ├── CommandHandler("/me") → me_handler()             │
│    │   ├── CommandHandler("/keys") → keys_handler()         │
│    │   ├── CommandHandler("/unlink") → unlink_handler()     │
│    │   └── ... (otros comandos)                             │
│    │                                                        │
│    └── Handlers                                             │
│        └── AuthHandler                                      │
│            ├── _check_auth() → verifica tokens en Redis     │
│            ├── _auto_refresh() → refresh si expira pronto   │
│            └── _require_auth() → retorna False si no auth   │
│                                                              │
│  infrastructure/                                            │
│    ├── config.py → Settings (pydantic-settings)             │
│    ├── redis.py → RedisPool (singleton)                     │
│    ├── token_storage.py → TokenStorage (Redis operations)   │
│    └── api_client.py → APIClient (HTTP al backend)          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    Backend API (usipipo-backend)
                      - /auth/telegram/request-code
                      - /auth/telegram/verify
                      - /auth/refresh
                      - /users/me
```

---

## 📁 **Estructura de Archivos a Crear/Modificar**

### **Archivos Nuevos**
```
usipipo-telegram-bot/src/
├── infrastructure/
│   ├── config.py              ← NUEVO (pydantic-settings)
│   ├── redis.py               ← NUEVO (RedisPool singleton)
│   └── token_storage.py       ← NUEVO (TokenStorage class)
└── bot/
    └── keyboards/
        └── auth.py            ← NUEVO (AuthMessages constants)

tests/
├── infrastructure/
│   └── test_token_storage.py  ← NUEVO (10 tests)
└── bot/
    └── test_auth_handlers.py  ← NUEVO (8 tests)
```

### **Archivos a Modificar**
```
usipipo-telegram-bot/src/
├── main.py                    ← MODIFICAR (registrar auth handlers)
└── bot/handlers/
    └── basic.py               ← MODIFICAR (agregar check auth opcional)
```

---

## 📝 **Tareas de Implementación**

### **Task 1: Configuración con Pydantic** ⏳
**Archivo:** `src/infrastructure/config.py`

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class BotSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )
    
    # Telegram
    TELEGRAM_TOKEN: str
    ADMIN_ID: int
    BOT_USERNAME: str = "usipipobot"
    
    # Backend API
    BACKEND_URL: str = "https://usipipo.duckdns.org"
    API_PREFIX: str = "/api/v1"
    
    # Redis
    REDIS_URL: str = "redis://localhost:6379"
    REDIS_MAX_CONNECTIONS: int = 10
    REDIS_SOCKET_TIMEOUT: float = 5.0
    REDIS_RETRY_ON_TIMEOUT: bool = True
    
    # Logging
    LOG_LEVEL: str = "INFO"
    
    # Tokens
    TOKEN_REFRESH_THRESHOLD_SECONDS: int = 300  # 5 min antes de expirar


settings = BotSettings()
```

**Criterio de Aceptación:**
- [ ] Settings carga desde `.env`
- [ ] Valores por defecto configurados
- [ ] Type hints correctos
- [ ] mypy passing

---

### **Task 2: Redis Connection Pool** ⏳
**Archivo:** `src/infrastructure/redis.py`

```python
import redis.asyncio as redis
from typing import Optional


class RedisPool:
    """Pool de conexiones Redis (singleton)."""
    
    _instance: Optional["RedisPool"] = None
    _pool: Optional[redis.Redis] = None
    
    def __init__(self, redis_url: str, max_connections: int = 10):
        self.redis_url = redis_url
        self.max_connections = max_connections
    
    @classmethod
    async def get_instance(
        cls,
        redis_url: str,
        max_connections: int = 10,
    ) -> "RedisPool":
        """Obtiene instancia singleton del pool."""
        if cls._instance is None:
            cls._instance = cls(redis_url, max_connections)
            cls._pool = redis.from_url(
                redis_url,
                max_connections=max_connections,
                decode_responses=True,
                socket_timeout=5.0,
                socket_connect_timeout=5.0,
                retry_on_timeout=True,
            )
        return cls._instance
    
    @classmethod
    async def get_client(cls) -> redis.Redis:
        """Obtiene cliente Redis del pool."""
        if cls._pool is None:
            raise RuntimeError("Redis pool not initialized")
        return cls._pool
    
    @classmethod
    async def health_check(cls) -> bool:
        """Verifica conexión con Redis."""
        try:
            client = await cls.get_client()
            await client.ping()
            return True
        except Exception:
            return False
    
    @classmethod
    async def close(cls) -> None:
        """Cierra el pool de conexiones."""
        if cls._pool:
            await cls._pool.close()
            cls._pool = None
            cls._instance = None
```

**Criterio de Aceptación:**
- [ ] Singleton funciona correctamente
- [ ] Health check retorna booleano
- [ ] Close libera recursos
- [ ] Tests de conexión passing

---

### **Task 3: Token Storage** ⏳
**Archivo:** `src/infrastructure/token_storage.py`

```python
import time
from typing import Optional, Dict, Any
from src.infrastructure.redis import RedisPool
from src.infrastructure.config import settings


class TokenStorage:
    """Gestión de tokens en Redis."""
    
    TOKEN_PREFIX = "usipipo:bot:tokens:"
    ACCESS_TOKEN_EXPIRY = 30 * 60  # 30 minutos
    REFRESH_TOKEN_EXPIRY = 30 * 24 * 60 * 60  # 30 días
    
    async def store(
        self,
        telegram_id: int,
        tokens: Dict[str, Any],
    ) -> None:
        """Guarda tokens en Redis con expiración automática."""
        key = f"{self.TOKEN_PREFIX}{telegram_id}"
        redis_client = await RedisPool.get_client()
        
        async with redis_client.pipeline() as pipe:
            await pipe.hset(key, mapping={
                "access_token": tokens["access_token"],
                "refresh_token": tokens["refresh_token"],
                "user_id": tokens["user_id"],
                "expires_at": str(int(time.time()) + tokens["expires_in"]),
                "created_at": str(int(time.time())),
            })
            await pipe.expire(key, self.REFRESH_TOKEN_EXPIRY)
            await pipe.execute()
    
    async def get(self, telegram_id: int) -> Optional[Dict[str, Any]]:
        """Recupera tokens del usuario."""
        key = f"{self.TOKEN_PREFIX}{telegram_id}"
        redis_client = await RedisPool.get_client()
        data = await redis_client.hgetall(key)
        
        if not data:
            return None
        
        return {
            "access_token": data["access_token"],
            "refresh_token": data["refresh_token"],
            "user_id": data["user_id"],
            "expires_at": int(data["expires_at"]),
            "created_at": int(data["created_at"]),
        }
    
    async def delete(self, telegram_id: int) -> bool:
        """Elimina tokens (unlink)."""
        key = f"{self.TOKEN_PREFIX}{telegram_id}"
        redis_client = await RedisPool.get_client()
        return await redis_client.delete(key) > 0
    
    async def is_authenticated(self, telegram_id: int) -> bool:
        """Verifica si el usuario tiene tokens válidos."""
        tokens = await self.get(telegram_id)
        
        if not tokens:
            return False
        
        # Check if access token expired
        if time.time() > tokens["expires_at"]:
            return False
        
        return True
    
    async def needs_refresh(self, telegram_id: int) -> bool:
        """Verifica si el token necesita refresh (5 min antes de expirar)."""
        tokens = await self.get(telegram_id)
        
        if not tokens:
            return False
        
        threshold = settings.TOKEN_REFRESH_THRESHOLD_SECONDS
        return time.time() > (tokens["expires_at"] - threshold)
```

**Criterio de Aceptación:**
- [ ] store() guarda en Redis con expiración
- [ ] get() retorna dict o None
- [ ] delete() retorna booleano
- [ ] is_authenticated() verifica expiración
- [ ] needs_refresh() verifica threshold
- [ ] 10 tests passing

---

### **Task 4: Auth Messages** ⏳
**Archivo:** `src/bot/keyboards/auth.py`

```python
"""Mensajes de autenticación."""


class AuthMessages:
    """Mensajes para autenticación invisible."""
    
    # /start command
    WELCOME_NEW_USER = (
        "✅ ¡Bienvenido a uSipipo!\n\n"
        "Tu cuenta ha sido creada y estás autenticado.\n\n"
        "Usa /help para ver los comandos disponibles."
    )
    
    WELCOME_RETURNING_USER = (
        "👋 ¡Bienvenido de nuevo!\n\n"
        "Ya tienes una cuenta activa.\n\n"
        "Usa /help para ver los comandos disponibles."
    )
    
    # Auth errors
    AUTH_ERROR = (
        "❌ Error de autenticación.\n\n"
        "Intenta de nuevo en unos minutos."
    )
    
    # /unlink command
    UNLINK_CONFIRMATION = (
        "⚠️ ¿Estás seguro de que quieres desvincular tu cuenta?\n\n"
        "Esto cerrará todas tus sesiones y tendrás que volver a autenticarte.\n\n"
        "Escribe /confirm_unlink para confirmar."
    )
    
    UNLINK_SUCCESS = "✅ Tu cuenta ha sido desvinculada correctamente."
    
    UNLINK_NOT_AUTHENTICATED = "ℹ️ No tenías sesión iniciada."
    
    # /me command
    ME_AUTHENTICATED = (
        "👤 <b>Tu Perfil</b>\n\n"
        "ID: {user_id}\n"
        "Telegram: @{username}\n"
        "Plan: {plan_name}\n"
        "Keys activas: {keys_count}/{max_keys}"
    )
    
    ME_NOT_AUTHENTICATED = (
        "🔒 No autenticado\n\n"
        "Usa /start para iniciar sesión."
    )
```

**Criterio de Aceptación:**
- [ ] Mensajes definidos como constantes
- [ ] Formato HTML donde corresponde
- [ ] Placeholders para variables dinámicas

---

### **Task 5: Auth Handler** ⏳
**Archivo:** `src/bot/handlers/auth.py`

```python
"""Handlers de autenticación invisible."""

import logging
from telegram import Update
from telegram.ext import ContextTypes

from src.infrastructure.api_client import APIClient
from src.infrastructure.token_storage import TokenStorage
from src.bot.keyboards.auth import AuthMessages

logger = logging.getLogger(__name__)


class AuthHandler:
    """Handler para autenticación invisible."""
    
    def __init__(self, api_client: APIClient, token_storage: TokenStorage):
        self.api = api_client
        self.tokens = token_storage
    
    async def start_handler(
        self,
        update: Update,
        context: ContextTypes.DEFAULT_TYPE,
    ) -> None:
        """Maneja /start - registro y auth automática."""
        telegram_id = update.effective_user.id
        
        # Check if already authenticated
        if await self.tokens.is_authenticated(telegram_id):
            await update.message.reply_text(AuthMessages.WELCOME_RETURNING_USER)
            return
        
        # Request auth code from backend
        try:
            response = await self.api.post(
                "/auth/telegram/request-code",
                {"telegram_id": telegram_id},
            )
            
            if response.get("success"):
                # Backend envió código por Telegram Bot API
                # Auto-verificar (el backend ya tiene el telegram_id)
                # En este caso, el backend genera y envía el código,
                # pero para auth invisible, auto-verificamos
                await self._auto_verify_and_store(telegram_id, update, context)
            else:
                await update.message.reply_text(AuthMessages.AUTH_ERROR)
                
        except Exception as e:
            logger.error(f"Error en /start: {e}")
            await update.message.reply_text(AuthMessages.AUTH_ERROR)
    
    async def _auto_verify_and_store(
        self,
        telegram_id: int,
        update: Update,
        context: ContextTypes.DEFAULT_TYPE,
    ) -> None:
        """Auto-verifica código y guarda tokens (invisible)."""
        # Nota: El backend necesita un endpoint que permita
        # auto-verificación cuando el bot es el cliente
        # Opción: endpoint /auth/telegram/auto-register
        
        try:
            response = await self.api.post(
                "/auth/telegram/auto-register",
                {"telegram_id": telegram_id},
            )
            
            if "access_token" in response:
                await self.tokens.store(telegram_id, response)
                await update.message.reply_text(AuthMessages.WELCOME_NEW_USER)
            else:
                await update.message.reply_text(AuthMessages.AUTH_ERROR)
                
        except Exception as e:
            logger.error(f"Error en auto-verificación: {e}")
            await update.message.reply_text(AuthMessages.AUTH_ERROR)
    
    async def me_handler(
        self,
        update: Update,
        context: ContextTypes.DEFAULT_TYPE,
    ) -> None:
        """Maneja /me - muestra perfil del usuario."""
        telegram_id = update.effective_user.id
        
        if not await self.tokens.is_authenticated(telegram_id):
            await update.message.reply_text(AuthMessages.ME_NOT_AUTHENTICATED)
            return
        
        # Auto-refresh si es necesario
        if await self.tokens.needs_refresh(telegram_id):
            await self._refresh_tokens(telegram_id)
        
        try:
            tokens = await self.tokens.get(telegram_id)
            response = await self.api.get(
                "/users/me",
                headers={"Authorization": f"Bearer {tokens['access_token']}"},
            )
            
            message = AuthMessages.ME_AUTHENTICATED.format(
                user_id=response.get("id", "N/A")[:8],
                username=update.effective_user.username or "N/A",
                plan_name=response.get("plan", "Free"),
                keys_count=response.get("active_keys", 0),
                max_keys=response.get("max_keys", 2),
            )
            
            await update.message.reply_text(message, parse_mode="HTML")
            
        except Exception as e:
            logger.error(f"Error al obtener perfil: {e}")
            await update.message.reply_text("❌ Error al obtener perfil")
    
    async def unlink_handler(
        self,
        update: Update,
        context: ContextTypes.DEFAULT_TYPE,
    ) -> None:
        """Maneja /unlink - revoca acceso del bot."""
        telegram_id = update.effective_user.id
        
        if not await self.tokens.is_authenticated(telegram_id):
            await update.message.reply_text(AuthMessages.UNLINK_NOT_AUTHENTICATED)
            return
        
        # Delete tokens from Redis
        await self.tokens.delete(telegram_id)
        
        # TODO: Revocar tokens en backend (endpoint /auth/logout)
        
        await update.message.reply_text(AuthMessages.UNLINK_SUCCESS)
    
    async def _refresh_tokens(self, telegram_id: int) -> bool:
        """Auto-refresh de tokens silencioso."""
        try:
            tokens = await self.tokens.get(telegram_id)
            
            response = await self.api.post(
                "/auth/refresh",
                {"refresh_token": tokens["refresh_token"]},
            )
            
            if "access_token" in response:
                await self.tokens.store(telegram_id, response)
                return True
            
            return False
            
        except Exception as e:
            logger.error(f"Error en refresh: {e}")
            return False
```

**Criterio de Aceptación:**
- [ ] start_handler() registra y autentica automáticamente
- [ ] me_handler() muestra perfil con auto-refresh
- [ ] unlink_handler() elimina tokens en Redis
- [ ] _refresh_tokens() hace auto-refresh silencioso
- [ ] Manejo de errores en todos los handlers
- [ ] 8 tests passing

---

### **Task 6: Registrar Handlers en main.py** ⏳
**Archivo:** `src/main.py`

**Modificaciones:**
```python
from src.infrastructure.config import settings
from src.infrastructure.redis import RedisPool
from src.infrastructure.token_storage import TokenStorage
from src.infrastructure.api_client import APIClient
from src.bot.handlers.auth import AuthHandler

# Inicializar dependencias en create_application()
async def init_dependencies() -> tuple[APIClient, TokenStorage, AuthHandler]:
    """Inicializa dependencias del bot."""
    # Redis pool
    await RedisPool.get_instance(settings.REDIS_URL)
    
    # Token storage
    token_storage = TokenStorage()
    
    # API client
    api_client = APIClient(
        base_url=settings.BACKEND_URL,
        api_prefix=settings.API_PREFIX,
    )
    
    # Auth handler
    auth_handler = AuthHandler(api_client, token_storage)
    
    return api_client, token_storage, auth_handler


def create_application(token: str) -> Application:
    """Create and configure the Telegram application."""
    logger.info("Initializing Telegram bot application...")
    
    # Inicializar dependencias (en producción sería async context manager)
    api_client, token_storage, auth_handler = asyncio.run(init_dependencies())
    
    app = Application.builder().token(token).build()
    
    # Registrar handlers
    app.add_handler(CommandHandler("start", auth_handler.start_handler))
    app.add_handler(CommandHandler("help", help_handler))  # existente
    app.add_handler(CommandHandler("me", auth_handler.me_handler))
    app.add_handler(CommandHandler("unlink", auth_handler.unlink_handler))
    # ... otros handlers
    
    app.add_error_handler(error_handler)
    
    logger.info("Bot handlers registered successfully")
    return app
```

**Criterio de Aceptación:**
- [ ] Dependencies inicializadas correctamente
- [ ] Handlers registrados en la aplicación
- [ ] Redis pool creado al startup
- [ ] Tests de integración passing

---

### **Task 7: Tests - Token Storage** ⏳
**Archivo:** `tests/infrastructure/test_token_storage.py`

```python
"""Tests para TokenStorage."""

import pytest
import time
from src.infrastructure.token_storage import TokenStorage


class TestTokenStorage:
    """Tests para TokenStorage."""
    
    @pytest.fixture
    async def storage(self):
        """Crea instancia de TokenStorage."""
        storage = TokenStorage()
        yield storage
        # Cleanup
        await storage.delete(123456)
    
    async def test_store_tokens_success(self, storage):
        """Guarda tokens correctamente."""
        tokens = {
            "access_token": "eyJhbGc...",
            "refresh_token": "eyJhbGc...",
            "user_id": "uuid-1234",
            "expires_in": 1800,
        }
        
        await storage.store(123456, tokens)
        stored = await storage.get(123456)
        
        assert stored is not None
        assert stored["access_token"] == tokens["access_token"]
        assert stored["user_id"] == tokens["user_id"]
    
    async def test_get_tokens_not_found(self, storage):
        """Retorna None si no existe."""
        stored = await storage.get(999999)
        assert stored is None
    
    async def test_delete_tokens_success(self, storage):
        """Elimina tokens correctamente."""
        # Primero guardar
        await storage.store(123456, {...})
        
        # Luego eliminar
        result = await storage.delete(123456)
        assert result is True
        
        # Verificar que no existe
        stored = await storage.get(123456)
        assert stored is None
    
    async def test_is_authenticated_true(self, storage):
        """Retorna True con tokens válidos."""
        await storage.store(123456, {
            "access_token": "...",
            "refresh_token": "...",
            "user_id": "...",
            "expires_in": 1800,  # 30 min
        })
        
        assert await storage.is_authenticated(123456) is True
    
    async def test_is_authenticated_false_no_tokens(self, storage):
        """Retorna False sin tokens."""
        assert await storage.is_authenticated(999999) is False
    
    async def test_is_authenticated_false_expired(self, storage):
        """Retorna False con tokens expirados."""
        # Guardar con expiración inmediata (hack para test)
        key = f"{storage.TOKEN_PREFIX}123456"
        redis_client = await storage._get_redis()
        await redis_client.hset(key, mapping={
            "access_token": "...",
            "expires_at": str(int(time.time()) - 100),  # Expirado
        })
        
        assert await storage.is_authenticated(123456) is False
    
    async def test_needs_refresh_true(self, storage):
        """Retorna True si está por expirar."""
        # Guardar con expiración en 4 minutos (menos que threshold de 5 min)
        await storage.store(123456, {
            "access_token": "...",
            "refresh_token": "...",
            "user_id": "...",
            "expires_in": 240,  # 4 min
        })
        
        assert await storage.needs_refresh(123456) is True
    
    async def test_needs_refresh_false(self, storage):
        """Retorna False si no está por expirar."""
        await storage.store(123456, {
            "access_token": "...",
            "refresh_token": "...",
            "user_id": "...",
            "expires_in": 1800,  # 30 min
        })
        
        assert await storage.needs_refresh(123456) is False
    
    async def test_store_overwrites_existing(self, storage):
        """Sobrescribe tokens existentes."""
        # Guardar primera vez
        await storage.store(123456, {
            "access_token": "token1",
            "refresh_token": "refresh1",
            "user_id": "user1",
            "expires_in": 1800,
        })
        
        # Sobrescribir
        await storage.store(123456, {
            "access_token": "token2",
            "refresh_token": "refresh2",
            "user_id": "user2",
            "expires_in": 1800,
        })
        
        stored = await storage.get(123456)
        assert stored["access_token"] == "token2"
        assert stored["user_id"] == "user2"
```

**Criterio de Aceptación:**
- [ ] 10 tests implementados
- [ ] Todos passing
- [ ] Cleanup de Redis después de cada test
- [ ] pytest-asyncio configurado

---

### **Task 8: Tests - Auth Handlers** ⏳
**Archivo:** `tests/bot/test_auth_handlers.py`

```python
"""Tests para AuthHandler."""

import pytest
from unittest.mock import AsyncMock, MagicMock, patch

from src.bot.handlers.auth import AuthHandler
from src.infrastructure.token_storage import TokenStorage
from src.infrastructure.api_client import APIClient


class TestAuthHandler:
    """Tests para AuthHandler."""
    
    @pytest.fixture
    def mock_api_client(self):
        """Mock de APIClient."""
        client = AsyncMock(spec=APIClient)
        return client
    
    @pytest.fixture
    def mock_token_storage(self):
        """Mock de TokenStorage."""
        storage = AsyncMock(spec=TokenStorage)
        return storage
    
    @pytest.fixture
    def auth_handler(self, mock_api_client, mock_token_storage):
        """Crea AuthHandler con mocks."""
        return AuthHandler(mock_api_client, mock_token_storage)
    
    @pytest.mark.asyncio
    async def test_start_handler_already_authenticated(
        self,
        auth_handler,
        mock_token_storage,
    ):
        """No registra si ya está autenticado."""
        mock_token_storage.is_authenticated.return_value = True
        
        update = MagicMock()
        context = MagicMock()
        
        await auth_handler.start_handler(update, context)
        
        # Verifica que no llamó a API
        mock_token_storage.is_authenticated.assert_called_once()
    
    @pytest.mark.asyncio
    async def test_start_handler_new_user(
        self,
        auth_handler,
        mock_token_storage,
        mock_api_client,
    ):
        """Registra usuario nuevo."""
        mock_token_storage.is_authenticated.return_value = False
        mock_api_client.post.return_value = {"success": True}
        
        update = MagicMock()
        context = MagicMock()
        
        await auth_handler.start_handler(update, context)
        
        # Verifica que llamó a API para registrar
        mock_api_client.post.assert_called()
    
    @pytest.mark.asyncio
    async def test_me_handler_not_authenticated(
        self,
        auth_handler,
        mock_token_storage,
    ):
        """Muestra 'no autenticado' si no hay tokens."""
        mock_token_storage.is_authenticated.return_value = False
        
        update = MagicMock()
        context = MagicMock()
        
        await auth_handler.me_handler(update, context)
        
        update.message.reply_text.assert_called_once()
    
    @pytest.mark.asyncio
    async def test_me_handler_authenticated(
        self,
        auth_handler,
        mock_token_storage,
        mock_api_client,
    ):
        """Muestra perfil si está autenticado."""
        mock_token_storage.is_authenticated.return_value = True
        mock_token_storage.needs_refresh.return_value = False
        mock_token_storage.get.return_value = {"access_token": "..."}
        mock_api_client.get.return_value = {
            "id": "uuid-1234",
            "plan": "VIP",
            "active_keys": 2,
            "max_keys": 10,
        }
        
        update = MagicMock()
        context = MagicMock()
        
        await auth_handler.me_handler(update, context)
        
        mock_api_client.get.assert_called_once()
        update.message.reply_text.assert_called_once()
    
    @pytest.mark.asyncio
    async def test_unlink_handler_success(
        self,
        auth_handler,
        mock_token_storage,
    ):
        """Elimina tokens correctamente."""
        mock_token_storage.is_authenticated.return_value = True
        
        update = MagicMock()
        context = MagicMock()
        
        await auth_handler.unlink_handler(update, context)
        
        mock_token_storage.delete.assert_called_once()
        update.message.reply_text.assert_called_once()
    
    @pytest.mark.asyncio
    async def test_unlink_handler_not_authenticated(
        self,
        auth_handler,
        mock_token_storage,
    ):
        """No hace nada si no está autenticado."""
        mock_token_storage.is_authenticated.return_value = False
        
        update = MagicMock()
        context = MagicMock()
        
        await auth_handler.unlink_handler(update, context)
        
        mock_token_storage.delete.assert_not_called()
    
    @pytest.mark.asyncio
    async def test_auto_refresh_success(
        self,
        auth_handler,
        mock_token_storage,
        mock_api_client,
    ):
        """Auto-refresh exitoso."""
        mock_token_storage.get.return_value = {"refresh_token": "..."}
        mock_api_client.post.return_value = {
            "access_token": "new_access",
            "refresh_token": "new_refresh",
            "expires_in": 1800,
        }
        
        result = await auth_handler._refresh_tokens(123456)
        
        assert result is True
        mock_api_client.post.assert_called_once()
        mock_token_storage.store.assert_called_once()
    
    @pytest.mark.asyncio
    async def test_auto_refresh_failure(
        self,
        auth_handler,
        mock_token_storage,
        mock_api_client,
    ):
        """Auto-refresh fallido."""
        mock_token_storage.get.return_value = {"refresh_token": "..."}
        mock_api_client.post.side_effect = Exception("API error")
        
        result = await auth_handler._refresh_tokens(123456)
        
        assert result is False
```

**Criterio de Aceptación:**
- [ ] 8 tests implementados
- [ ] Todos passing
- [ ] Mocks configurados correctamente
- [ ] pytest-asyncio configurado

---

## ✅ **Criterios de Aceptación de Fase**

### **Code Quality**
- [ ] `pytest` - 18 tests passing (10 + 8)
- [ ] `mypy src/` - 0 errores
- [ ] `ruff check src/ tests/` - All checks passed
- [ ] `bandit -r src/` - 0 issues

### **Functional**
- [ ] `/start` registra y autentica automáticamente
- [ ] `/me` muestra perfil con auto-refresh
- [ ] `/unlink` elimina tokens en Redis
- [ ] Auto-refresh silencioso funciona
- [ ] Tokens persisten en Redis (30 días)

### **Documentation**
- [ ] `docs/AUTHENTICATION.md` - Flujo de autenticación
- [ ] `docs/REDIS-SCHEMA.md` - Estructura de keys en Redis
- [ ] Comments en código para lógica compleja

---

## 📊 **Timeline Estimado**

| Task | Duración | Dependencias |
|------|----------|--------------|
| Task 1: Config | 30 min | - |
| Task 2: Redis Pool | 45 min | - |
| Task 3: Token Storage | 1.5 h | Task 2 |
| Task 4: Auth Messages | 30 min | - |
| Task 5: Auth Handler | 2 h | Task 3, 4 |
| Task 6: main.py | 30 min | Task 5 |
| Task 7: Tests Storage | 1 h | Task 3 |
| Task 8: Tests Handlers | 1 h | Task 5 |
| **Total** | **~7 horas** | |

---

## 🚀 **Siguientes Pasos**

1. **Invocar `writing-plans`** para crear track de implementación
2. **Ejecutar implementación** task por task
3. **Verificar** tests, mypy, ruff passing
4. **Crear PR** a main

---

**¿Procedo con la implementación usando este plan?**
