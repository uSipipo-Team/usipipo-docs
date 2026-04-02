# User Profile "Mis Datos" Feature Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement a fully functional "Mis Datos" feature that displays a rich, professional user profile with balance, referral stats, and loyalty information.

**Architecture:** Follow Clean Architecture patterns with a new handler module that fetches user data from the backend API via the existing adapter, formats it using professional message templates, and displays it with an inline keyboard.

**Tech Stack:** Python 3.13, python-telegram-bot v20+, httpx (via existing APIClient), existing BackendApiAdapter, existing TokenStorage

---

## Task 1: Create Message Templates Module

**Files:**
- Create: `src/bot/keyboards/messages_user_profile.py`
- Test: `tests/unit/bot/keyboards/test_messages_user_profile.py`

**Step 1: Create the messages file with professional templates**

```python
"""Mensajes para perfil de usuario."""

from datetime import datetime


class UserProfileMessages:
    """Mensajes para mostrar el perfil del usuario."""

    class Profile:
        """Mensajes del perfil de usuario."""

        HEADER = "👤 **Tu Perfil uSipipo**\n\n"

        PERSONAL_INFO = (
            "📋 **Información Personal**\n"
            "├─ Usuario: @{username}\n"
            "├─ Nombre: {full_name}\n"
            "└─ ID Telegram: `{telegram_id}`\n\n"
        )

        BALANCE_INFO = (
            "💰 **Balance y Datos**\n"
            "├─ Saldo Actual: {balance_gb} GB\n"
            "├─ Total Comprado: {total_purchased_gb} GB\n"
            "└─ Claves VPN Activas: {vpn_keys_count}\n\n"
        )

        REFERRAL_INFO = (
            "🎁 **Programa de Referidos**\n"
            "├─ Código: `{referral_code}`\n"
            "├─ Referidos: {referrals_count} usuarios\n"
            "└─ Créditos Ganados: {referral_credits} GB\n\n"
        )

        LOYALTY_INFO = (
            "⭐ **Programa de Lealtad**\n"
            "├─ Nivel: {loyalty_level} ({loyalty_bonus}% bonus)\n"
            "├─ Compras Realizadas: {purchase_count}\n"
            "└─ Bonus Bienvenida: {welcome_bonus_status}\n\n"
        )

        ACCOUNT_INFO = (
            "📅 **Información de Cuenta**\n"
            "├─ Creada: {created_at}\n"
            "└─ Última Actualización: {updated_at}\n\n"
        )

        TIP = "💡 *Consejo: Invita más amigos para aumentar tu bonus de lealtad*"

        FULL_PROFILE = HEADER + PERSONAL_INFO + BALANCE_INFO + REFERRAL_INFO + LOYALTY_INFO + ACCOUNT_INFO + TIP

    class Error:
        """Mensajes de error."""

        NOT_AUTHENTICATED = (
            "🔒 **No autenticado**\n\n"
            "Debes iniciar sesión para ver tu perfil.\n\n"
            "💡 Usa /start para autenticarte."
        )

        API_ERROR = (
            "❌ **Error al cargar perfil**\n\n"
            "No se pudo obtener tu información.\n\n"
            "🔄 Intenta nuevamente en un momento."
        )

        GENERIC_ERROR = (
            "❌ **Error del sistema**\n\n"
            "Ocurrió un error inesperado.\n\n"
            "💡 Intenta nuevamente o contacta al soporte."
        )

    @staticmethod
    def format_personal_info(username: str | None, first_name: str | None, 
                             last_name: str | None, telegram_id: int) -> str:
        """Formatea información personal manejando valores nulos."""
        username_str = username if username else "No disponible"
        full_name = " ".join(filter(None, [first_name or "", last_name or ""])) or "No disponible"
        
        return UserProfileMessages.Profile.PERSONAL_INFO.format(
            username=username_str,
            full_name=full_name,
            telegram_id=telegram_id,
        )

    @staticmethod
    def format_balance_info(balance_gb: float, total_purchased_gb: float, 
                           vpn_keys_count: int) -> str:
        """Formatea información de balance."""
        return UserProfileMessages.Profile.BALANCE_INFO.format(
            balance_gb=f"{balance_gb:.2f}",
            total_purchased_gb=f"{total_purchased_gb:.2f}",
            vpn_keys_count=vpn_keys_count,
        )

    @staticmethod
    def format_referral_info(referral_code: str, referrals_count: int, 
                            referral_credits: float) -> str:
        """Formatea información de referidos."""
        return UserProfileMessages.Profile.REFERRAL_INFO.format(
            referral_code=referral_code,
            referrals_count=referrals_count,
            referral_credits=f"{referral_credits:.1f}",
        )

    @staticmethod
    def format_loyalty_info(loyalty_bonus_percent: int, purchase_count: int, 
                           welcome_bonus_used: bool) -> str:
        """Formatea información de lealtad."""
        # Determine loyalty level based on bonus percent
        if loyalty_bonus_percent >= 10:
            level = "Platinum"
        elif loyalty_bonus_percent >= 7:
            level = "Gold"
        elif loyalty_bonus_percent >= 5:
            level = "Silver"
        elif loyalty_bonus_percent >= 3:
            level = "Bronze"
        else:
            level = "Standard"
        
        welcome_status = "✅ Usado" if welcome_bonus_used else "⏳ Disponible"
        
        return UserProfileMessages.Profile.LOYALTY_INFO.format(
            loyalty_level=level,
            loyalty_bonus=loyalty_bonus_percent,
            purchase_count=purchase_count,
            welcome_bonus_status=welcome_status,
        )

    @staticmethod
    def format_account_info(created_at: datetime, updated_at: datetime) -> str:
        """Formatea información de cuenta."""
        created_str = created_at.strftime("%d %b %Y")
        updated_str = updated_at.strftime("%d %b %Y")
        
        return UserProfileMessages.Profile.ACCOUNT_INFO.format(
            created_at=created_str,
            updated_at=updated_str,
        )
```

**Step 2: Create test file for message formatting**

```python
"""Tests for UserProfileMessages."""

import pytest
from datetime import datetime
from src.bot.keyboards.messages_user_profile import UserProfileMessages


class TestUserProfileMessages:
    """Tests for message formatting."""

    def test_format_personal_info_with_all_data(self):
        """Test formatting with complete user data."""
        result = UserProfileMessages.format_personal_info(
            username="testuser",
            first_name="Test",
            last_name="User",
            telegram_id=123456789,
        )
        
        assert "@testuser" in result
        assert "Test User" in result
        assert "`123456789`" in result

    def test_format_personal_info_with_missing_username(self):
        """Test formatting when username is None."""
        result = UserProfileMessages.format_personal_info(
            username=None,
            first_name="Test",
            last_name="User",
            telegram_id=123456789,
        )
        
        assert "No disponible" in result
        assert "Test User" in result

    def test_format_balance_info(self):
        """Test balance formatting."""
        result = UserProfileMessages.format_balance_info(
            balance_gb=15.5,
            total_purchased_gb=50.0,
            vpn_keys_count=3,
        )
        
        assert "15.50 GB" in result
        assert "50.00 GB" in result
        assert "3" in result

    def test_format_loyalty_info_gold_level(self):
        """Test loyalty level determination for Gold."""
        result = UserProfileMessages.format_loyalty_info(
            loyalty_bonus_percent=7,
            purchase_count=5,
            welcome_bonus_used=True,
        )
        
        assert "Gold" in result
        assert "7" in result
        assert "✅ Usado" in result

    def test_format_account_info(self):
        """Test account date formatting."""
        created = datetime(2024, 1, 15, 10, 30)
        updated = datetime(2025, 3, 28, 14, 45)
        
        result = UserProfileMessages.format_account_info(created, updated)
        
        assert "15 Jan 2024" in result
        assert "28 Mar 2025" in result
```

**Step 3: Run tests to verify they pass**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
pytest tests/unit/bot/keyboards/test_messages_user_profile.py -v
```

Expected: All tests PASS

**Step 4: Commit**

```bash
git add src/bot/keyboards/messages_user_profile.py tests/unit/bot/keyboards/test_messages_user_profile.py
git commit -m "feat: add user profile message templates"
```

---

## Task 2: Create Keyboard Module

**Files:**
- Create: `src/bot/keyboards/user_profile.py`
- Test: `tests/unit/bot/keyboards/test_user_profile_keyboard.py`

**Step 1: Create the keyboard file**

```python
"""Teclado para perfil de usuario."""

from telegram import InlineKeyboardButton, InlineKeyboardMarkup


class UserProfileKeyboard:
    """Teclado para mostrar perfil de usuario."""

    @staticmethod
    def profile_menu() -> InlineKeyboardMarkup:
        """
        Retorna teclado para el menú de perfil.

        Returns:
            InlineKeyboardMarkup: Teclado con botón para volver al menú principal
        """
        keyboard = [
            [InlineKeyboardButton("🔙 Volver al Menú Principal", callback_data="main_menu")],
        ]
        return InlineKeyboardMarkup(keyboard)
```

**Step 2: Create test file**

```python
"""Tests for UserProfileKeyboard."""

import pytest
from src.bot.keyboards.user_profile import UserProfileKeyboard


class TestUserProfileKeyboard:
    """Tests for user profile keyboard."""

    def test_profile_menu_structure(self):
        """Test that profile menu has correct structure."""
        keyboard = UserProfileKeyboard.profile_menu()
        
        assert isinstance(keyboard, InlineKeyboardMarkup)
        assert len(keyboard.inline_keyboard) == 1
        assert len(keyboard.inline_keyboard[0]) == 1
        
        button = keyboard.inline_keyboard[0][0]
        assert button.text == "🔙 Volver al Menú Principal"
        assert button.callback_data == "main_menu"
```

**Step 3: Run tests**

```bash
pytest tests/unit/bot/keyboards/test_user_profile_keyboard.py -v
```

Expected: PASS

**Step 4: Commit**

```bash
git add src/bot/keyboards/user_profile.py tests/unit/bot/keyboards/test_user_profile_keyboard.py
git commit -m "feat: add user profile keyboard"
```

---

## Task 3: Create Handler Module

**Files:**
- Create: `src/bot/handlers/user_profile.py`
- Test: `tests/bot/test_user_profile_handlers.py`

**Step 1: Create the handler file**

```python
"""Handler para perfil de usuario."""

import logging
from typing import TYPE_CHECKING, Any

from telegram import Update
from telegram.ext import CallbackQueryHandler, CommandHandler, ContextTypes

from src.bot.keyboards.messages_user_profile import UserProfileMessages
from src.bot.keyboards.user_profile import UserProfileKeyboard
from src.bot.keyboards.main_menu import MainMenuKeyboard
from src.bot.keyboards.main import BasicMessages

if TYPE_CHECKING:
    from src.infrastructure.api_client import APIClient
    from src.infrastructure.token_storage import TokenStorage

logger = logging.getLogger(__name__)


class UserProfileHandler:
    """Handler para mostrar perfil de usuario."""

    def __init__(self, api_client: "APIClient", token_storage: "TokenStorage"):
        self.api = api_client
        self.tokens = token_storage

    async def _get_auth_headers(self, telegram_id: int) -> dict[str, str]:
        """Obtiene headers de autenticación para el usuario."""
        tokens = await self.tokens.get(telegram_id)
        if not tokens:
            raise PermissionError("User not authenticated")
        return {"Authorization": f"Bearer {tokens['access_token']}"}

    async def _safe_answer_query(self, query: Any) -> None:
        """Responde a callback query de forma segura."""
        try:
            await query.answer()
        except Exception as e:
            logger.error(f"Error answering query: {e}")

    async def _safe_edit_message(
        self,
        query: Any,
        context: ContextTypes.DEFAULT_TYPE,
        text: str,
        reply_markup: Any = None,
        parse_mode: str = "Markdown",
    ) -> None:
        """Edita mensaje de forma segura."""
        try:
            await query.edit_message_text(
                text=text,
                reply_markup=reply_markup,
                parse_mode=parse_mode,
            )
        except Exception as e:
            logger.error(f"Error editing message: {e}")

    async def show_user_profile(self, update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
        """
        Muestra el perfil del usuario con información detallada.

        Args:
            update: Update de Telegram
            context: Contexto del bot
        """
        query = update.callback_query
        if query is None:
            logger.warning("show_user_profile called without callback_query")
            return

        await self._safe_answer_query(query)
        
        if update.effective_user is None:
            return
            
        telegram_id = update.effective_user.id
        logger.info(f"User {telegram_id} viewing profile")

        try:
            # Check authentication
            if not await self.tokens.is_authenticated(telegram_id):
                await self._safe_edit_message(
                    query,
                    context,
                    UserProfileMessages.Error.NOT_AUTHENTICATED,
                    UserProfileKeyboard.profile_menu(),
                )
                return

            # Get user profile from backend
            headers = await self._get_auth_headers(telegram_id)
            
            # Get user data
            user_response = await self.api.get("/users/me", headers=headers)
            
            # Get VPN keys count (optional, don't fail if not available)
            try:
                vpn_keys = await self.api.get("/vpn/keys", headers=headers)
                vpn_keys_count = len(vpn_keys) if isinstance(vpn_keys, list) else 0
            except Exception:
                vpn_keys_count = 0

            # Build profile message
            message = UserProfileMessages.Profile.HEADER
            
            # Personal info
            message += UserProfileMessages.format_personal_info(
                username=user_response.get("username"),
                first_name=user_response.get("first_name"),
                last_name=user_response.get("last_name"),
                telegram_id=telegram_id,
            )

            # Balance info
            message += UserProfileMessages.format_balance_info(
                balance_gb=user_response.get("balance_gb", 0),
                total_purchased_gb=user_response.get("total_purchased_gb", 0),
                vpn_keys_count=vpn_keys_count,
            )

            # Referral info
            message += UserProfileMessages.format_referral_info(
                referral_code=user_response.get("referral_code", "N/A"),
                referrals_count=user_response.get("referred_users_with_purchase", 0),
                referral_credits=user_response.get("referral_credits", 0),
            )

            # Loyalty info
            message += UserProfileMessages.format_loyalty_info(
                loyalty_bonus_percent=user_response.get("loyalty_bonus_percent", 0),
                purchase_count=user_response.get("purchase_count", 0),
                welcome_bonus_used=user_response.get("welcome_bonus_used", False),
            )

            # Account info
            from datetime import datetime
            created_at = datetime.fromisoformat(user_response.get("created_at", "2024-01-01"))
            updated_at = datetime.fromisoformat(user_response.get("updated_at", "2024-01-01"))
            message += UserProfileMessages.format_account_info(created_at, updated_at)

            # Add tip
            message += "\n" + UserProfileMessages.Profile.TIP

            await self._safe_edit_message(
                query,
                context,
                message,
                UserProfileKeyboard.profile_menu(),
            )

        except PermissionError:
            logger.warning(f"Unauthenticated user {telegram_id} tried to view profile")
            await self._safe_edit_message(
                query,
                context,
                UserProfileMessages.Error.NOT_AUTHENTICATED,
                UserProfileKeyboard.profile_menu(),
            )
        except Exception as e:
            logger.error(f"Error showing user profile: {e}")
            await self._safe_edit_message(
                query,
                context,
                UserProfileMessages.Error.API_ERROR,
                UserProfileKeyboard.profile_menu(),
            )


def get_user_profile_handlers(api_client: "APIClient", token_storage: "TokenStorage"):
    """Retorna los handlers para perfil de usuario."""
    handler = UserProfileHandler(api_client, token_storage)

    return [
        CallbackQueryHandler(handler.show_user_profile, pattern="^show_usage$"),
    ]
```

**Step 2: Create test file**

```python
"""Tests for user profile handlers."""

import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from telegram import Update, CallbackQuery, User
from src.bot.handlers.user_profile import UserProfileHandler


class TestUserProfileHandler:
    """Tests for UserProfileHandler."""

    @pytest.fixture
    def mock_api_client(self):
        """Mock API client."""
        client = AsyncMock()
        return client

    @pytest.fixture
    def mock_token_storage(self):
        """Mock token storage."""
        storage = AsyncMock()
        storage.is_authenticated = AsyncMock(return_value=True)
        storage.get = AsyncMock(return_value={"access_token": "test_token"})
        return storage

    @pytest.fixture
    def handler(self, mock_api_client, mock_token_storage):
        """Create handler with mocked dependencies."""
        return UserProfileHandler(mock_api_client, mock_token_storage)

    @pytest.mark.asyncio
    async def test_show_user_profile_authenticated(self, handler, mock_api_client):
        """Test showing profile for authenticated user."""
        # Mock user data
        mock_api_client.get = AsyncMock(return_value={
            "username": "testuser",
            "first_name": "Test",
            "last_name": "User",
            "balance_gb": 15.5,
            "total_purchased_gb": 50.0,
            "referral_code": "ABC123",
            "referred_users_with_purchase": 5,
            "referral_credits": 2.5,
            "loyalty_bonus_percent": 7,
            "purchase_count": 8,
            "welcome_bonus_used": True,
            "created_at": "2024-01-15T10:30:00",
            "updated_at": "2025-03-28T14:45:00",
        })

        # Mock update
        callback_query = MagicMock(spec=CallbackQuery)
        callback_query.answer = AsyncMock()
        callback_query.edit_message_text = AsyncMock()
        callback_query.from_user = User(id=123456789, is_bot=False)
        
        update = MagicMock(spec=Update)
        update.callback_query = callback_query
        update.effective_user = callback_query.from_user

        context = MagicMock()

        # Call handler
        await handler.show_user_profile(update, context)

        # Verify API was called
        mock_api_client.get.assert_called()
        # Verify message was edited
        callback_query.edit_message_text.assert_called_once()

    @pytest.mark.asyncio
    async def test_show_user_profile_not_authenticated(self, handler, mock_token_storage):
        """Test showing profile for unauthenticated user."""
        mock_token_storage.is_authenticated = AsyncMock(return_value=False)

        callback_query = MagicMock(spec=CallbackQuery)
        callback_query.answer = AsyncMock()
        callback_query.edit_message_text = AsyncMock()
        callback_query.from_user = User(id=123456789, is_bot=False)
        
        update = MagicMock(spec=Update)
        update.callback_query = callback_query
        update.effective_user = callback_query.from_user

        context = MagicMock()

        await handler.show_user_profile(update, context)

        # Verify error message shown
        callback_query.edit_message_text.assert_called_once()
        call_args = callback_query.edit_message_text.call_args
        assert "No autenticado" in call_args[1]["text"]
```

**Step 3: Run tests**

```bash
pytest tests/bot/test_user_profile_handlers.py -v
```

Expected: PASS

**Step 4: Commit**

```bash
git add src/bot/handlers/user_profile.py tests/bot/test_user_profile_handlers.py
git commit -m "feat: add user profile handler"
```

---

## Task 4: Register Handler in main.py

**Files:**
- Modify: `src/main.py:285-295`

**Step 1: Add handler registration**

Find the section where callback handlers are registered (after line 285) and add:

```python
# Register callback handler for user profile
from src.bot.handlers.user_profile import get_user_profile_handlers
for handler in get_user_profile_handlers(_api_client, _token_storage):
    app.add_handler(handler)
```

Add this after the referrals callback handlers and before the main menu handler.

**Step 2: Run linting**

```bash
ruff check src/main.py
```

Expected: No errors

**Step 3: Commit**

```bash
git add src/main.py
git commit -m "feat: register user profile handler"
```

---

## Task 5: Integration Testing

**Files:**
- Modify: `test_bot_backend_integration.py` (if exists)

**Step 1: Test the complete flow manually**

```bash
# Start the bot in development mode
cd /home/mowgli/usipipo/usipipo-telegram-bot
python src/main.py
```

**Step 2: Verify in Telegram**

1. Start a conversation with the bot
2. Use `/start` to authenticate
3. Press the "💾 Mis Datos" button
4. Verify the profile displays correctly with all sections

**Step 3: Test error cases**

1. Test without authentication (should show "No autenticado")
2. Test with API down (should show "Error al cargar perfil")

---

## Task 6: Documentation Update

**Files:**
- Modify: `README.md`

**Step 1: Add feature to README**

Add to the features list:

```markdown
- 👤 **Perfil de Usuario**: Ver información detallada del usuario, balance, referidos y lealtad
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add user profile feature to README"
```

---

## Summary

This plan implements a complete "Mis Datos" feature with:
- Professional message templates with proper formatting
- Clean keyboard with back navigation
- Robust handler with authentication and error handling
- Full test coverage
- Integration with existing backend API

**Total Tasks:** 6
**Estimated Time:** 30-45 minutes
**Files Created:** 4
**Files Modified:** 2
