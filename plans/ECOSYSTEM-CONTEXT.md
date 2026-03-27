# Contexto del Ecosistema uSipipo

**Última Actualización:** 2026-03-24  
**Versión:** 1.1.0  
**Estado:** Producción  
**Ubicación:** `/plans/ECOSYSTEM-CONTEXT.md` (fuente única de verdad)

---

## 🎯 **Propósito del Ecosistema**

uSipipo es un ecosistema **centralizado en el backend** que provee servicios de VPN management, pagos, suscripciones, y soporte a través de múltiples plataformas:

- **Telegram Bot** - Interfaz principal nativa
- **Mini App Web** - Telegram WebApp integrada
- **Android App** - Aplicación móvil nativa
- **Desktop App** - Aplicación de escritorio

**Principio Fundamental:** El usuario se registra **UNA vez** y usa todo el ecosistema con la misma identidad.

---

## 🏗️ **Arquitectura Centralizada**

```
┌─────────────────────────────────────────────────────────────────┐
│                    BACKEND CENTRALIZADO                         │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  USER IDENTITY (Central)                                │   │
│  │  - user_id: UUID                                        │   │
│  │  - telegram_id: PRIMARY IDENTIFIER                      │   │
│  │  - email: optional (web registration)                   │   │
│  │  - password_hash: optional (web registration)           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  ACTIVE SESSIONS (Redis)                                │   │
│  │  - telegram:{telegram_id} → tokens                      │   │
│  │  - miniapp:{telegram_id} → tokens                       │   │
│  │  - android:{user_id} → tokens                           │   │
│  │  - desktop:{user_id} → tokens                           │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  Telegram     │    │  Mini App     │    │  Android/     │
│  Bot          │    │  (Web)        │    │  Desktop      │
│               │    │               │    │               │
│ /start → auth │    │ initData →    │    │ Login →       │
│ One-time      │    │ auto-register │    │ Telegram      │
│               │    │ or login      │    │ OAuth flow    │
└───────────────┘    └───────────────┘    └───────────────┘
```

---

## 🔐 **Flujos de Autenticación por Plataforma**

### **1. Telegram Bot (Primera Vez)**

**Flujo:**
```
Usuario → /start → Backend genera código → Bot envía código → 
Usuario /verify → Backend guarda tokens en Redis → 
Usuario registrado PARA SIEMPRE
```

**Características:**
- ✅ Auth invisible (el usuario no percibe el proceso)
- ✅ One-time registration
- ✅ Tokens persistentes en Redis (30 días refresh token)
- ✅ Auto-refresh silencioso cuando expira access token
- ✅ No hay comandos `/login` o `/logout` visibles

**Comandos:**
| Comando | Descripción | Auth Required |
|---------|-------------|---------------|
| `/start` | Registro inicial + auth automática | ❌ (auto) |
| `/help` | Muestra comandos | ❌ |
| `/keys` | Ver/gestionar VPN keys | ✅ (auto-check) |
| `/newkey` | Crear nueva VPN key | ✅ (auto-check) |
| `/payments` | Ver pagos y facturas | ✅ (auto-check) |
| `/tickets` | Soporte técnico | ✅ (auto-check) |
| `/me` | Perfil de usuario | ✅ (auto-check) |
| `/unlink` | Revocar acceso del bot | ✅ |

---

### **2. Mini App Web (Telegram WebApp)**

**Flujo:**
```
Mini App abre → Telegram envía initData → 
Backend valida initData con Telegram API → 
Si usuario existe → Login automático
Si no existe → Registro automático → Dashboard
```

**Características:**
- ✅ Usa `initData` de Telegram WebApp
- ✅ Validación HMAC del initData en backend
- ✅ Registro automático si no existe
- ✅ Dashboard inmediato sin login manual

**Endpoints:**
- `POST /auth/telegram/webapp` - Valida initData y retorna tokens
- `GET /users/me` - Obtiene perfil con access token

---

### **3. Android/Desktop App**

**Flujo:**
```
App abre → Usuario elige "Login con Telegram" → 
Muestra QR o código → Usuario introduce en bot → 
Backend vincula dispositivo → Tokens guardados en OS keychain → 
Dashboard con VPN keys → Botón "Conectar"
```

**Características:**
- ✅ OAuth flow vía Telegram Bot
- ✅ Tokens almacenados en OS keychain (Android Keystore / Windows Credential Manager)
- ✅ Dashboard con lista de VPN keys (WireGuard, Outline, etc.)
- ✅ Botón "Conectar" inicia VPN con la key seleccionada

**Endpoints:**
- `POST /auth/device/request` - Solicita código de vinculación
- `POST /auth/device/verify` - Verifica código y obtiene tokens
- `GET /vpn/keys` - Obtiene keys del usuario
- `POST /vpn/keys/{id}/connect` - Inicia conexión (log en backend)

---

## 📦 **Estructura de Tokens (Redis)**

### **Keys en Redis**

```redis
# Identidad del usuario
usipipo:users:{user_id}:
  - telegram_id: 1058749165
  - email: optional
  - created_at: timestamp
  - status: active|banned|deleted

# Sesiones por plataforma
usipipo:sessions:{user_id}:{platform}:
  - telegram → {access_token, refresh_token, expires_at}
  - miniapp → {access_token, refresh_token, expires_at}
  - android → {access_token, refresh_token, expires_at}
  - desktop → {access_token, refresh_token, expires_at}
  - web → {access_token, refresh_token, expires_at}

# Dispositivos vinculados
usipipo:devices:{user_id}:
  - [{device_id, device_name, platform, last_seen, ip_address}]
```

### **Token Lifecycle**

| Plataforma | Access Token | Refresh Token | Auto-Refresh |
|------------|--------------|---------------|--------------|
| Telegram Bot | 30 min | 30 días | ✅ Sí |
| Mini App | 30 min | 30 días | ✅ Sí |
| Android | 30 min | 30 días | ✅ Sí |
| Desktop | 30 min | 30 días | ✅ Sí |
| Web | 30 min | 7 días | ❌ Re-login |

---

## 🎨 **Principios de Diseño**

### **1. Auth Invisible**
- El usuario **nunca** debe percibir el proceso de autenticación
- No hay `/login` o `/logout` en el bot
- Auto-refresh silencioso de tokens
- Si el token expira, se renueva automáticamente sin molestar al usuario

### **2. Persistencia Total**
- Una vez registrado, el usuario **nunca** se desregistra
- Los tokens persisten por 30 días (refresh token)
- El usuario puede revocar acceso con `/unlink` (caso excepcional)

### **3. Multi-Plataforma Nativa**
- Cada plataforma usa sus mecanismos nativos de auth
- Telegram Bot: comandos
- Mini App: initData
- Android/Desktop: OAuth flow + OS keychain

### **4. Backend como Fuente de Verdad**
- Todas las plataformas consultan el mismo backend
- El backend gestiona identidad centralizada
- Redis para sesiones distribuidas

---

## 📋 **Comandos del Bot - Detalle**

### **Comandos de Registro/Auth**

#### `/start`
**Propósito:** Registro inicial y autenticación automática

**Flujo:**
1. Bot detecta si `telegram_id` ya está registrado
2. Si no existe → Backend genera código de 6 dígitos
3. Backend envía código por Telegram Bot API
4. Bot auto-verifica código (sin intervención del usuario)
5. Backend guarda tokens en Redis
6. Bot muestra bienvenida y comandos disponibles

**Mensaje:**
```
✅ ¡Bienvenido a uSipipo!

Tu cuenta ha sido creada y estás autenticado.

Usa /help para ver los comandos disponibles.
```

---

### **Comandos de Funcionalidad**

#### `/keys`
**Propósito:** Ver y gestionar VPN keys del usuario

**Pre-condición:** Usuario autenticado (auto-check)

**Respuesta:**
```
🔑 Tus VPN Keys

WireGuard:
  - Key 1: Activa (23 días restantes)
  - Key 2: Expirada

Outline:
  - Key 1: Activa (15 días restantes)

Usa /newkey para crear una nueva key.
```

#### `/newkey`
**Propósito:** Crear nueva VPN key

**Flujo:**
1. Verifica límite de keys por plan
2. Pide protocolo (WireGuard / Outline)
3. Backend genera key
4. Bot muestra QR o archivo de configuración

#### `/payments`
**Propósito:** Ver historial de pagos y facturas

**Respuesta:**
```
💰 Tus Pagos

Últimos pagos:
  - 2026-03-01: 10 Stars (Plan VIP)
  - 2026-02-01: 10 Stars (Plan VIP)

Próximo pago: 2026-04-01
```

#### `/tickets`
**Propósito:** Soporte técnico

**Comandos relacionados:**
- `/newticket` - Crear nuevo ticket
- `/mytickets` - Ver tickets abiertos

#### `/me`
**Propósito:** Ver perfil de usuario

**Respuesta:**
```
👤 Tu Perfil

ID: uuid-1234-5678
Telegram: @username
Email: usuario@email.com
Plan: VIP
Keys activas: 2/10
Desde: 2026-01-15
```

#### `/unlink`
**Propósito:** Revocar acceso del bot (logout forzado)

**Advertencia:**
```
⚠️ ¿Estás seguro de que quieres desvincular tu cuenta?

Esto cerrará todas tus sesiones y tendrás que volver a autenticarte.

Escribe /confirm_unlink para confirmar.
```

---

## 🛠️ **Infraestructura Técnica**

### **Backend Endpoints**

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/auth/telegram/request-code` | POST | Solicita código de auth |
| `/auth/telegram/verify` | POST | Verifica código y retorna tokens |
| `/auth/telegram/webapp` | POST | Valida initData de Mini App |
| `/auth/device/request` | POST | Solicita código para dispositivo |
| `/auth/device/verify` | POST | Verifica código de dispositivo |
| `/auth/refresh` | POST | Renueva tokens con refresh token |
| `/auth/logout` | POST | Revoca tokens |
| `/users/me` | GET | Obtiene perfil del usuario |
| `/vpn/keys` | GET | Obtiene VPN keys del usuario |
| `/vpn/keys` | POST | Crea nueva VPN key |

### **Redis Configuration**

```python
REDIS_URL = "redis://localhost:6379"
REDIS_MAX_CONNECTIONS = 10
REDIS_SOCKET_TIMEOUT = 5.0
REDIS_RETRY_ON_TIMEOUT = True
```

### **Token Storage (Bot)**

```python
class TokenStorage:
    """Gestión de tokens en Redis."""
    
    async def store(self, telegram_id: int, tokens: dict) -> None:
        """Guarda tokens con expiración automática."""
        
    async def get(self, telegram_id: int) -> Optional[dict]:
        """Recupera tokens del usuario."""
        
    async def delete(self, telegram_id: int) -> bool:
        """Elimina tokens (unlink)."""
        
    async def is_authenticated(self, telegram_id: int) -> bool:
        """Verifica si el usuario tiene tokens válidos."""
        
    async def refresh_if_needed(self, telegram_id: int) -> bool:
        """Auto-refresh si token está por expirar."""
```

---

## 📊 **Estados del Usuario**

| Estado | Descripción | Acciones Permitidas |
|--------|-------------|---------------------|
| **No registrado** | telegram_id no existe en backend | `/start`, `/help` |
| **Registrado** | Usuario en backend con tokens válidos | Todos los comandos |
| **Token expirado** | Refresh token expirado (> 30 días) | Auto-re-auth en `/start` |
| **Baneado** | Usuario baneado por admin | Ninguna (mensaje de baneo) |

---

## 🔒 **Seguridad**

### **Token Security**
- Access tokens: JWT con expiración de 30 minutos
- Refresh tokens: JWT con expiración de 30 días
- Tokens almacenados en Redis con expiración automática
- Auto-refresh silencioso 5 minutos antes de expirar

### **Session Security**
- Cada plataforma tiene su propia sesión
- Dispositivos vinculados se registran en backend
- Usuario puede ver sesiones activas (futuro `/sessions`)
- Usuario puede revocar sesiones individualmente (futuro)

### **Rate Limiting**
- `/auth/*`: 5 requests por minuto
- Comandos normales: 30 requests por minuto
- Admin commands: 60 requests por minuto

### **Telegram initData Validation**
```python
# Validación HMAC-SHA256 del initData de Telegram WebApp
def validate_telegram_init_data(init_data: str, token: str) -> bool:
    """
    Valida initData de Telegram WebApp.
    
    Args:
        init_data: Telegram WebApp initData string
        token: Telegram bot token
    
    Returns:
        bool: True si es válido, False si no
    """
    # Implementación en backend: src/shared/security/telegram_auth.py
```

---

## 📈 **Roadmap**

### **Fase 1: Telegram Bot Auth** ✅ (Completada)
- [x] Diseño de arquitectura centralizada
- [x] Documento de contexto creado
- [x] TokenStorage con Redis
- [x] AuthHandler invisible
- [x] Auto-refresh de tokens
- [x] Comandos de funcionalidad (/me, /unlink)
- [x] CI/CD configurado
- [x] Integration tests con backend de producción
- [x] Branch protection habilitada

### **Fase 2: VPN Key Management (Bot)** 🟡 (En curso)
- [ ] Comando /keys (listar VPN keys)
- [ ] Comando /newkey (crear VPN key)
- [ ] Comando /delkey (eliminar VPN key)
- [ ] Comando /qr (mostrar QR)
- [ ] Integración con backend VPN endpoints

### **Fase 3: Mini App Web** 📋 (Planeada)
- [ ] Validación de initData en backend
- [ ] Registro automático de usuarios
- [ ] Dashboard de VPN keys
- [ ] Botón "Conectar" (log en backend)

### **Fase 4: Android/Desktop** 📋 (Planeada)
- [ ] OAuth flow vía bot
- [ ] Almacenamiento en OS keychain
- [ ] Dashboard nativo con VPN keys
- [ ] Conexión VPN real

### **Fase 5: Gestión de Sesiones** 📋 (Planeada)
- [ ] Comando `/sessions` (ver dispositivos)
- [ ] Revocar sesión por dispositivo
- [ ] Notificación de nuevo dispositivo
- [ ] Límite de dispositivos activos

---

## 📚 **Documentación Relacionada**

### **En este directorio (`/plans/`)**
- `sk-prompting.md` - Prompt para continuar la migración
- `MIGRATION-PROGRESS.md` - Progreso de migración monorepo → multi-repo
- `2026-03-23-telegram-bot-phase-1.md` - Plan de Fase 1 del bot
- `2026-03-18-semana-1-cimientos.md` - Semana 1: Cimientos del backend

### **En los repositorios**
- **Backend:** `usipipo-backend/docs/` - Documentación técnica de API
- **Bot:** `usipipo-telegram-bot/docs/` - Documentación del bot
- **Commons:** `usipipo-commons/README.md` - Librería compartida (PyPI)

---

## 🔄 **Historial de Cambios**

| Fecha | Versión | Cambios |
|-------|---------|---------|
| 2026-03-24 | 1.1.0 | Telegram Bot Auth invisible + Integration tests + CI/CD |
| 2026-03-23 | 1.0.0 | Documento inicial de contexto |

---

## ⚠️ **Importante**

**Este documento es la FUENTE ÚNICA DE VERDAD para el contexto del ecosistema.**

- No duplicar este documento en otros repositorios
- Actualizar aquí cuando se agreguen nuevas plataformas
- Los repositorios individuales deben referenciar este documento
- Para cambios de arquitectura, actualizar primero este documento

**Ubicación:** `/home/mowgli/usipipo/plans/ECOSYSTEM-CONTEXT.md`
