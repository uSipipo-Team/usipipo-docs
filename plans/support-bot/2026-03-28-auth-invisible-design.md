# uSipipo Support Bot - Plan de Autenticación Invisible

**Fecha:** 2026-03-28  
**Versión:** 1.0  
**Estado:** En implementación  

---

## 📋 Resumen Ejecutivo

Este documento describe la implementación de autenticación invisible para el uSipipo Support Bot (`@uSipipoSupport_Bot`), permitiendo que usuarios del bot principal (`@usipipobot`) accedan automáticamente al bot de soporte sin registro adicional.

---

## 🎯 Problema Identificado

### **Situación Actual**

1. **Comandos expuestos sin autenticación**: `/tickets`, `/nuevoticket` pueden ser llamados sin verificación
2. **`/start` no autentica**: El bot no registra automáticamente al usuario en el backend
3. **Logs muestran uso sin auth**: Usuario `1058749165` ejecutó comandos de tickets sin flujo de autenticación
4. **Inconsistencia con bot principal**: El bot principal (`@usipipobot`) sí implementa auth invisible

### **Evidencia de Logs**

```
timestamp="2026-03-28 14:17:09,339" User 1058749165 executed /start
timestamp="2026-03-28 14:17:16,880" User 1058749165 listing tickets
timestamp="2026-03-28 14:17:26,703" User 1058749165 creating ticket
```

**Problema**: El usuario puede ejecutar comandos pero no hay autenticación registrada en Redis.

---

## 🔍 Análisis de Root Cause

### **Investigación Sistemática**

#### **1. Backend - Endpoint Disponible**

El backend tiene endpoint de auto-registro implementado:

```python
# usipipo-backend/src/infrastructure/api/v1/routes/auth.py
@router.post("/telegram/auto-register")
async def auto_register_telegram(
    telegram_id: int,
    user_service: UserService
) -> AuthResponse:
    """Registro automático de usuario desde Telegram Bot."""
    user = await user_service.get_or_create_by_telegram(telegram_id)
    access_token, refresh_token = create_token_pair(user.id, user.telegram_id)
    return AuthResponse(...)
```

**Estado**: ✅ Funcional y probado

#### **2. Bot Principal - Auth Implementado**

```python
# usipipo-telegram-bot/src/bot/handlers/auth.py
class AuthHandler:
    async def start_handler(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        if await self.tokens.is_authenticated(telegram_id):
            await update.message.reply_text(WELCOME_RETURNING_USER)
            return
        
        await self._register_and_auth(telegram_id, update, context)
    
    async def _register_and_auth(self, telegram_id: int, update: Update, context):
        response = await self.api.post(
            "/auth/telegram/auto-register",
            {"telegram_id": telegram_id}
        )
        await self.tokens.store(telegram_id, response)
```

**Estado**: ✅ Implementado y en producción

#### **3. Support Bot - Auth Ausente**

```python
# usipipo-support-bot/src/main.py
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle /start command."""
    user = update.effective_user
    logger.info(f"User {user.id} executed /start")
    
    # ❌ NO hay autenticación
    await update.message.reply_text("👋 Hola...")

# usipipo-support-bot/src/bot/handlers/tickets.py
async def list_tickets(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await self.tokens.is_authenticated(telegram_id):
        await update.message.reply_text(TicketsMessages.Error.NOT_AUTHORIZED)
        return
    # ✅ Verifica auth pero usuario nunca se autentica
```

**Problema**: Los handlers verifican autenticación pero `/start` nunca autentica al usuario.

---

## 🏗️ Diseño de Solución

### **Arquitectura Propuesta**

```
┌─────────────────────────────────────────────────────────────┐
│                    Telegram @uSipipoSupport_Bot              │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ /start
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    AuthHandler.start_handler()               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 1. Check Redis: ¿Usuario autenticado?                │   │
│  │    ├─ SÍ → "Bienvenido de vuelta"                    │   │
│  │    └─ NO → Auto-registrar                            │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────────────────┬─────────────────────────────────────┘
                        │ POST /api/v1/auth/telegram/auto-register
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    Backend API (usipipo-backend)             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 1. Buscar usuario por telegram_id                    │   │
│  │ 2. Si no existe → Crear usuario                      │   │
│  │ 3. Generar JWT tokens (access + refresh)             │   │
│  │ 4. Retornar tokens                                   │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    TokenStorage (Redis)                      │
│  Key: "support_bot:tokens:{telegram_id}"                     │
│  Value: { access_token, refresh_token, expires_at, user_id } │
│  TTL: 30 días                                                │
└─────────────────────────────────────────────────────────────┘
```

### **Flujo de Autenticación**

#### **Usuario Nuevo (primer acceso)**

```
1. Usuario: /start
2. AuthHandler: Check Redis → No autenticado
3. AuthHandler: POST /auth/telegram/auto-register {telegram_id: 123456}
4. Backend: 
   - Buscar usuario por telegram_id → No encontrado
   - Crear usuario nuevo (telegram_id: 123456, plan: Free)
   - Generar JWT tokens
5. AuthHandler: Guardar tokens en Redis
6. AuthHandler: "¡Bienvenido! Tu cuenta ha sido creada"
```

#### **Usuario Existente (viene del bot principal)**

```
1. Usuario: /start (telegram_id: 1058749165)
2. AuthHandler: Check Redis → No autenticado (primera vez en support bot)
3. AuthHandler: POST /auth/telegram/auto-register {telegram_id: 1058749165}
4. Backend:
   - Buscar usuario por telegram_id → ENCONTRADO
   - Usuario ya existe (viene de @usipipobot)
   - Generar JWT tokens
5. AuthHandler: Guardar tokens en Redis
6. AuthHandler: "¡Bienvenido de vuelta! Tu cuenta está vinculada"
```

#### **Usuario Autenticado (retorno)**

```
1. Usuario: /start (dentro de 30 días)
2. AuthHandler: Check Redis → Autenticado ✓
3. AuthHandler: "¡Bienvenido de vuelta!"
4. Sin llamadas al backend
```

---

## 📦 Componentes a Implementar

### **1. AuthHandler (`src/bot/handlers/auth.py`)**

**Responsabilidad**: Gestionar autenticación invisible y auto-registro.

**Métodos:**
- `start_handler(update, context)` - Maneja `/start`
- `_register_and_auth(telegram_id, update, context)` - Auto-registro
- `me_handler(update, context)` - Muestra perfil (opcional)
- `unlink_handler(update, context)` - Revoca acceso (opcional)

**Dependencias:**
- `APIClient` - Para llamar al backend
- `TokenStorage` - Para guardar tokens en Redis

---

### **2. AuthMessages (`src/bot/keyboards/auth.py`)**

**Responsabilidad**: Mensajes de autenticación localizados (español).

**Mensajes:**
```python
class AuthMessages:
    WELCOME_NEW_USER = """
🎉 ¡Bienvenido a uSipipo Support!

Tu cuenta ha sido creada automáticamente.
Ahora podés gestionar tus tickets de soporte.

/comandos - Ver comandos disponibles
"""

    WELCOME_RETURNING_USER = """
👋 ¡Bienvenido de vuelta!

Tu cuenta está vinculada con @usipipobot.
¿En qué podemos ayudarte hoy?

/comandos - Ver comandos disponibles
"""

    AUTH_ERROR = """
❌ Error de autenticación

No pudimos autenticar tu cuenta.
Por favor intentá nuevamente en unos momentos.
"""

    NOT_AUTHENTICATED = """
⚠️ No estás autenticado

Por favor iniciá el bot con /start para acceder.
"""
```

---

### **3. Modificaciones a `src/main.py`**

**Cambios:**
```python
# ANTES
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    logger.info(f"User {user.id} executed /start")
    
    if update.message:
        await update.message.reply_text("👋 Hola...")

# DESPUÉS
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    auth_handler = _get_auth_handler()
    await auth_handler.start_handler(update, context)
```

---

### **4. Modificaciones a `src/bot/handlers/tickets.py`**

**Mejora de mensajes de error:**
```python
# ANTES
if not await self.tokens.is_authenticated(telegram_id):
    await update.message.reply_text(
        TicketsMessages.Error.NOT_AUTHORIZED,
        parse_mode="Markdown",
    )
    return

# DESPUÉS
if not await self.tokens.is_authenticated(telegram_id):
    await update.message.reply_text(
        TicketsMessages.Error.NOT_AUTHENTICATED_START_FIRST,
        parse_mode="Markdown",
    )
    return
```

---

## 🔐 Seguridad

### **Consideraciones**

1. **Telegram ID como identidad**: 
   - ✅ Único y verificado por Telegram
   - ✅ No puede ser falsificado (validado por Telegram API)

2. **Auto-registro automático**:
   - ✅ Cualquier usuario de Telegram puede acceder
   - ✅ Backend crea usuario con plan "Free" por defecto
   - ✅ Sin información sensible requerida

3. **Tokens JWT**:
   - ✅ Access token: 30 minutos
   - ✅ Refresh token: 30 días
   - ✅ Almacenados en Redis con TTL

4. **Backend verifica permisos**:
   - ✅ AuthHandler solo autentica
   - ✅ Backend verifica roles y permisos
   - ✅ Support bot solo accede a endpoints de tickets

---

## 🧪 Testing

### **Casos de Prueba**

| ID | Escenario | Resultado Esperado |
|----|-----------|-------------------|
| T1 | Usuario nuevo ejecuta `/start` | Se crea cuenta, tokens en Redis, mensaje bienvenida |
| T2 | Usuario existente (bot principal) ejecuta `/start` | Reutiliza cuenta, tokens en Redis, mensaje "bienvenido de vuelta" |
| T3 | Usuario autenticado ejecuta `/start` (dentro de 30 días) | Sin llamada al backend, mensaje "bienvenido de vuelta" |
| T4 | Usuario no autenticado ejecuta `/tickets` | Prompt "Usa /start para autenticarte" |
| T5 | Usuario autenticado ejecuta `/tickets` | Lista de tickets del usuario |

### **Verificación en Logs**

```bash
# Esperado después de /start
INFO - User 1058749165 executed /start
INFO - Auto-registered telegram user: uuid-xxx (telegram_id=1058749165)
INFO - Token storage: Saved tokens for telegram_id=1058749165
```

### **Verificación en Redis**

```bash
redis-cli
> KEYS support_bot:tokens:*
"support_bot:tokens:1058749165"
> GET support_bot:tokens:1058749165
{"access_token":"eyJ...","refresh_token":"eyJ...","expires_at":1711627200}
```

---

## 📊 Métricas de Éxito

| Métrica | Línea Base | Objetivo |
|---------|------------|----------|
| Usuarios autenticados en Redis | 0 | 100% de usuarios activos |
| Error "NOT_AUTHORIZED" en logs | Frecuente | Cero después de `/start` |
| Tiempo promedio de autenticación | N/A | < 500ms |
| Usuarios que abandonan por auth error | Desconocido | 0% |

---

## 🚀 Rollout Plan

### **Fase 1: Implementación** (Día 1)
- [ ] Crear `src/bot/handlers/auth.py`
- [ ] Crear `src/bot/keyboards/auth.py`
- [ ] Modificar `src/main.py` → `/start` usa AuthHandler
- [ ] Actualizar mensajes de error en tickets.py

### **Fase 2: Testing Local** (Día 1)
- [ ] Probar con usuario nuevo
- [ ] Probar con usuario existente (1058749165)
- [ ] Verificar tokens en Redis
- [ ] Verificar logs de autenticación

### **Fase 3: Deploy a Producción** (Día 2)
- [ ] Detener servicio actual
- [ ] Actualizar código
- [ ] Reiniciar servicio
- [ ] Monitorear logs de autenticación

### **Fase 4: Monitoreo** (Día 2-7)
- [ ] Verificar tasa de autenticación exitosa
- [ ] Monitorear errores de auth
- [ ] Revisar logs de usuarios nuevos

---

## 🔗 Referencias

### **Documentación Relacionada**
- [Backend API - Auth Endpoints](../apis/backend-api-reference.md#auth)
- [Support Bot Architecture](./ARCHITECTURE.md)
- [Support Bot Deployment](./DEPLOYMENT.md)

### **Código de Referencia**
- [Bot Principal - AuthHandler](https://github.com/uSipipo-Team/usipipo-telegram-bot/blob/main/src/bot/handlers/auth.py)
- [Backend - Auto-Register Endpoint](https://github.com/uSipipo-Team/usipipo-backend/blob/main/src/infrastructure/api/v1/routes/auth.py)

---

## 📝 Historial de Cambios

| Fecha | Versión | Cambio | Autor |
|-------|---------|--------|-------|
| 2026-03-28 | 1.0 | Documento inicial | uSipipo Team |

---

**Última Actualización:** 2026-03-28  
**Estado:** En implementación  
**Próximo Review:** Post-implementación
