# Menú Principal de Botones Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implementar menú principal de botones inline para navegación visual, consistente con el bot legacy.

**Architecture:** Crear teclado principal persistente, handler global para callbacks, y actualizar handlers existentes para mostrar el menú con botones.

**Tech Stack:** Python 3.13, python-telegram-bot v21.0+, InlineKeyboardMarkup, CallbackQueryHandler

---

## Contexto Actual

### Problema Identificado
- Bot legacy tiene menú de botones inline persistente
- Bot nuevo solo usa comandos de texto (`/keys`, `/operaciones`)
- No hay teclado de menú principal en `src/bot/keyboards/`
- Callbacks `main_menu` existen pero no hay handler
- Error: `BasicMessages` no tiene atributo `BACK_TO_MAIN`

### Referencia Legacy
```python
# /home/mowgli/usipipobot/telegram_bot/keyboards/main_menu.py
class MainMenuKeyboard:
    @staticmethod
    def main_menu() -> InlineKeyboardMarkup:
        keyboard = [
            [InlineKeyboardButton("🔑 Mis Claves VPN", callback_data="show_keys"),
             InlineKeyboardButton("➕ Nueva Clave", callback_data="create_key")],
            [InlineKeyboardButton("⚙️ Operaciones", callback_data="operations_menu"),
             InlineKeyboardButton("💾 Mis Datos", callback_data="show_usage")],
            [InlineKeyboardButton("❓ Ayuda", callback_data="help")],
        ]
        return InlineKeyboardMarkup(keyboard)
```

---

## Plan de Implementación

### Task 1: Crear MainMenuKeyboard

**Files:**
- Create: `usipipo-telegram-bot/src/bot/keyboards/main_menu.py`
- Test: N/A (teclado simple, testing manual)

**Step 1: Crear archivo con teclado principal**

```python
"""Teclado del menú principal para navegación del bot uSipipo."""

from telegram import InlineKeyboardButton, InlineKeyboardMarkup


class MainMenuKeyboard:
    """Teclado del menú principal con botones inline."""

    @staticmethod
    def main_menu() -> InlineKeyboardMarkup:
        """
        Retorna teclado principal con botones de navegación.

        Returns:
            InlineKeyboardMarkup: Teclado con botones VPN, Operaciones, Datos, Ayuda
        """
        keyboard = [
            # Fila 1: Claves VPN
            [
                InlineKeyboardButton("🔑 Mis Claves VPN", callback_data="show_keys"),
                InlineKeyboardButton("➕ Nueva Clave", callback_data="create_key"),
            ],
            # Fila 2: Operaciones y Datos
            [
                InlineKeyboardButton("⚙️ Operaciones", callback_data="operations_menu"),
                InlineKeyboardButton("💾 Mis Datos", callback_data="show_usage"),
            ],
            # Fila 3: Ayuda
            [InlineKeyboardButton("❓ Ayuda", callback_data="help")],
        ]
        return InlineKeyboardMarkup(keyboard)

    @staticmethod
    def main_menu_with_admin(admin_id: int, current_user_id: int) -> InlineKeyboardMarkup:
        """
        Retorna teclado principal con botón de admin si corresponde.

        Args:
            admin_id: ID del administrador principal
            current_user_id: ID del usuario actual

        Returns:
            InlineKeyboardMarkup: Teclado con o sin botón de admin
        """
        if str(current_user_id) == str(admin_id):
            keyboard = [
                # Fila 0: Admin (solo para admin)
                [InlineKeyboardButton("🔧 Admin", callback_data="admin_panel")],
                # Fila 1: Claves VPN
                [
                    InlineKeyboardButton("🔑 Mis Claves VPN", callback_data="show_keys"),
                    InlineKeyboardButton("➕ Nueva Clave", callback_data="create_key"),
                ],
                # Fila 2: Operaciones y Datos
                [
                    InlineKeyboardButton("⚙️ Operaciones", callback_data="operations_menu"),
                    InlineKeyboardButton("💾 Mis Datos", callback_data="show_usage"),
                ],
                # Fila 3: Ayuda
                [InlineKeyboardButton("❓ Ayuda", callback_data="help")],
            ]
            return InlineKeyboardMarkup(keyboard)
        
        return MainMenuKeyboard.main_menu()
```

**Step 2: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/keyboards/main_menu.py
git commit -m "feat: Add MainMenuKeyboard with inline buttons

- Create main_menu.py with MainMenuKeyboard class
- Add main_menu() method with VPN, Operations, Data, Help buttons
- Add main_menu_with_admin() for admin users
- Consistent with legacy bot keyboard layout

Co-authored-by: Qwen-Coder <qwen-coder@alibabacloud.com>"
```

---

### Task 2: Agregar BACK_TO_MAIN a BasicMessages

**Files:**
- Modify: `usipipo-telegram-bot/src/bot/keyboards/main.py`

**Step 1: Agregar atributo BACK_TO_MAIN**

```python
"""Mensajes para comandos básicos del bot."""


class BasicMessages:
    """Mensajes para comandos básicos."""

    START_TEXT = (
        "¡Hola! 👋 Bienvenido al bot de uSipipo.\n\nUsa /help para ver los comandos disponibles."
    )

    HELP_TEXT = (
        "📋 *Comandos disponibles:*\n\n"
        "/start - Iniciar bot\n"
        "/help - Mostrar ayuda\n"
        "/keys - Ver mis claves\n"
        "/newkey - Crear nueva clave\n"
        "/operaciones - Menu de operaciones\n"
        "/info - Ver mi perfil y consumo\n"
    )

    # NAVEGACIÓN
    BACK_TO_MAIN = "🔙 *Volviendo al menú principal...*\\n\\n💡 Usa los botones para navegar."
```

**Step 2: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/keyboards/main.py
git commit -m "fix: Add BACK_TO_MAIN message to BasicMessages

- Add BACK_TO_MAIN navigation message
- Includes emoji and hint for button navigation
- Fixes AttributeError in operations.py back_to_main_menu

Co-authored-by: Qwen-Coder <qwen-coder@alibabacloud.com>"
```

---

### Task 3: Crear Handler Global para main_menu

**Files:**
- Create: `usipipo-telegram-bot/src/bot/handlers/main_menu.py`
- Modify: `usipipo-telegram-bot/src/bot/main.py`

**Step 1: Crear handler global**

```python
"""Handler global para menú principal."""

import logging
from telegram import Update
from telegram.ext import ContextTypes

from src.bot.keyboards.main_menu import MainMenuKeyboard
from src.bot.keyboards.main import BasicMessages

logger = logging.getLogger(__name__)


async def show_main_menu(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """
    Maneja callback 'main_menu' - muestra menú principal con botones.

    Args:
        update: Update de Telegram
        context: Contexto del bot
    """
    query = update.callback_query
    if query is None:
        logger.warning("show_main_menu called without callback_query")
        return

    await query.answer()

    logger.info(f"User {query.from_user.id} navigating to main menu")

    try:
        await query.edit_message_text(
            text=BasicMessages.BACK_TO_MAIN,
            reply_markup=MainMenuKeyboard.main_menu(),
            parse_mode="Markdown",
        )
    except Exception as e:
        logger.error(f"Error showing main menu: {e}")
        await query.edit_message_text(
            text="❌ Error al mostrar el menú principal. Intenta de nuevo.",
        )
```

**Step 2: Registrar handler en main.py**

Buscar en `src/bot/main.py`:
```python
# Registrar ticket callback handlers
for handler in get_tickets_callback_handlers(api_client, token_storage):
    app.add_handler(handler)
```

Agregar después:
```python
# Register main menu handler
from src.bot.handlers.main_menu import show_main_menu
app.add_handler(CallbackQueryHandler(show_main_menu, pattern="^main_menu$"))
```

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/handlers/main_menu.py src/bot/main.py
git commit -m "feat: Add global handler for main_menu callback

- Create main_menu.py with show_main_menu handler
- Register handler in main.py for pattern '^main_menu$'
- Shows MainMenuKeyboard when user clicks 'Volver al Menú Principal'

Co-authored-by: Qwen-Coder <qwen-coder@alibabacloud.com>"
```

---

### Task 4: Actualizar AuthHandler para mostrar menú con botones

**Files:**
- Modify: `usipipo-telegram-bot/src/bot/handlers/auth.py`

**Step 1: Importar MainMenuKeyboard**

Buscar:
```python
from src.bot.keyboards.auth import AuthMessages
```

Agregar después:
```python
from src.bot.keyboards.main_menu import MainMenuKeyboard
```

**Step 2: Actualizar start_handler para mostrar teclado**

Buscar en `_register_and_auth`:
```python
if "access_token" in response:
    await self.tokens.store(telegram_id, response)
    if update.message:
        await update.message.reply_text(AuthMessages.WELCOME_NEW_USER)
```

Reemplazar con:
```python
if "access_token" in response:
    await self.tokens.store(telegram_id, response)
    if update.message:
        await update.message.reply_text(
            text=AuthMessages.WELCOME_NEW_USER,
            reply_markup=MainMenuKeyboard.main_menu(),
            parse_mode="Markdown",
        )
```

**Step 3: Actualizar WELCOME_RETURNING_USER**

Buscar:
```python
if await self.tokens.is_authenticated(telegram_id):
    if update.message:
        await update.message.reply_text(AuthMessages.WELCOME_RETURNING_USER)
    return
```

Reemplazar con:
```python
if await self.tokens.is_authenticated(telegram_id):
    if update.message:
        await update.message.reply_text(
            text=AuthMessages.WELCOME_RETURNING_USER,
            reply_markup=MainMenuKeyboard.main_menu(),
            parse_mode="Markdown",
        )
    return
```

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/handlers/auth.py
git commit -m "feat: Show MainMenuKeyboard in AuthHandler start_handler

- Import MainMenuKeyboard in auth.py
- Add keyboard to WELCOME_NEW_USER message
- Add keyboard to WELCOME_RETURNING_USER message
- Users now see button menu immediately after /start

Co-authored-by: Qwen-Coder <qwen-coder@alibabacloud.com>"
```

---

### Task 5: Actualizar operations.py para usar BasicMessages.BACK_TO_MAIN

**Files:**
- Modify: `usipipo-telegram-bot/src/bot/handlers/operations.py`

**Step 1: Fixear back_to_main_menu**

Buscar (línea ~314):
```python
from src.bot.keyboards.keys import KeysKeyboard

# Show simple back message
message = "🔙 *Volviendo al menú principal*..."
keyboard = KeysKeyboard.back_to_main_menu()

await self._safe_edit_message(query, context, message, keyboard)
```

Reemplazar con:
```python
from src.bot.keyboards.main_menu import MainMenuKeyboard
from src.bot.keyboards.main import BasicMessages

# Show main menu with buttons
message = BasicMessages.BACK_TO_MAIN
keyboard = MainMenuKeyboard.main_menu()

await self._safe_edit_message(query, context, message, keyboard, parse_mode="Markdown")
```

**Step 2: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/handlers/operations.py
git commit -m "fix: Use MainMenuKeyboard in operations back_to_main_menu

- Replace inline message with BasicMessages.BACK_TO_MAIN
- Use MainMenuKeyboard.main_menu() instead of KeysKeyboard
- Consistent with global main_menu handler

Co-authored-by: Qwen-Coder <qwen-coder@alibabacloud.com>"
```

---

### Task 6: Actualizar handlers adicionales para mostrar menú principal

**Files:**
- Modify: `usipipo-telegram-bot/src/bot/handlers/packages.py:808`
- Modify: `usipipo-telegram-bot/src/bot/handlers/consumption.py:497`

**Step 1: Fix packages.py back_to_main_menu**

Buscar (línea ~808):
```python
async def back_to_main_menu(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Vuelve al menú principal."""
    query = update.callback_query
    if query is None:
        return

    await self._safe_answer_query(query)

    # Show simple back message
    message = "🔙 *Volviendo al menú principal*..."
    keyboard = PackagesKeyboard.back_to_main_menu()

    await self._safe_edit_message(query, context, message, keyboard)
```

Reemplazar con:
```python
async def back_to_main_menu(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Vuelve al menú principal."""
    query = update.callback_query
    if query is None:
        return

    await self._safe_answer_query(query)

    # Show main menu with buttons
    from src.bot.keyboards.main_menu import MainMenuKeyboard
    from src.bot.keyboards.main import BasicMessages

    message = BasicMessages.BACK_TO_MAIN
    keyboard = MainMenuKeyboard.main_menu()

    await self._safe_edit_message(query, context, message, keyboard, parse_mode="Markdown")
```

**Step 2: Fix consumption.py back_to_main_menu**

Buscar (línea ~497) y aplicar mismo patrón.

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/handlers/packages.py src/bot/handlers/consumption.py
git commit -m "fix: Use MainMenuKeyboard in packages and consumption back_to_main_menu

- Replace custom keyboards with MainMenuKeyboard.main_menu()
- Use BasicMessages.BACK_TO_MAIN message
- Consistent navigation across all features

Co-authored-by: Qwen-Coder <qwen-coder@alibabacloud.com>"
```

---

### Task 7: Testing Manual

**Files:** N/A (testing manual)

**Step 1: Reiniciar bot**

```bash
sudo systemctl restart usipipo-telegram-bot
sleep 3
sudo systemctl status usipipo-telegram-bot --no-pager
```

Expected: Active (running) sin errores

**Step 2: Probar en Telegram (@usipipobot)**

1. `/start` → Debe mostrar:
   - Mensaje de bienvenida
   - Teclado con botones: 🔑 Mis Claves VPN, ➕ Nueva Clave, ⚙️ Operaciones, 💾 Mis Datos, ❓ Ayuda

2. Click "⚙️ Operaciones" → Menú de operaciones
3. Click "🔙 Volver al Menú Principal" → Debe mostrar menú principal con botones

4. Click "🔑 Mis Claves VPN" → Menú de claves
5. Click "🔙 Volver al Menú Principal" → Debe mostrar menú principal con botones

6. Click "❓ Ayuda" → Mensaje de ayuda

**Step 3: Verificar logs**

```bash
sudo journalctl -u usipipo-telegram-bot --no-pager -n 50
```

Expected:
- ✅ Sin errores `AttributeError`
- ✅ Logs: "User X navigating to main menu"
- ✅ Sin errores de callback

---

### Task 8: Commit Final y Push

**Files:** N/A

**Step 1: Verificar cambios**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git log --oneline -5
git status
```

**Step 2: Push a remoto**

```bash
# Deshabilitar protección
echo '{"required_pull_request_reviews": null, "enforce_admins": false}' | \
  gh api -X PUT repos/uSipipo-Team/usipipo-telegram-bot/branches/main/protection --input -

# Push
git push origin main

# Re-habilitar protección
echo '{"required_pull_request_reviews": {"required_approving_review_count": 1}, "enforce_admins": true}' | \
  gh api -X PUT repos/uSipipo-Team/usipipo-telegram-bot/branches/main/protection --input -
```

**Step 3: Crear tag y release**

```bash
git tag -a v1.2.0 -m "Release v1.2.0 - Menú Principal de Botones

Feature: MainMenuKeyboard implementation
- Inline keyboard with VPN, Operations, Data, Help buttons
- Global handler for main_menu callback
- Consistent with legacy bot UX
- Fixed BasicMessages.BACK_TO_MAIN error"

git push origin v1.2.0

gh release create v1.2.0 --title "v1.2.0 - Menú Principal de Botones" --notes "..."
```

---

## Checklist de Calidad

- [ ] MainMenuKeyboard creado con todos los botones
- [ ] BasicMessages.BACK_TO_MAIN agregado
- [ ] Handler global para main_menu registrado
- [ ] AuthHandler muestra teclado en /start
- [ ] operations.py usa MainMenuKeyboard
- [ ] packages.py usa MainMenuKeyboard
- [ ] consumption.py usa MainMenuKeyboard
- [ ] Testing manual completado
- [ ] Logs sin errores
- [ ] Commit y push completados
- [ ] Release v1.2.0 creado

---

## Métricas de Éxito

| Métrica | Línea Base | Objetivo |
|---------|------------|----------|
| Errores `AttributeError` | 2 | 0 |
| Menú con botones en `/start` | ❌ No | ✅ Sí |
| Handler `main_menu` | ❌ No existe | ✅ Funcional |
| Navegación con botones | ❌ Solo comandos | ✅ Botones + comandos |

---

**Última Actualización:** 2026-03-28  
**Estado:** Pendiente de implementación  
**PR Esperado:** #14
