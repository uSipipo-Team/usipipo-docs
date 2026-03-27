# 📋 PRD - uSipipo Telegram Bot

> Product Requirements Document del Bot de Telegram

---

## 📊 Información del Producto

| Campo | Valor |
|-------|-------|
| **Nombre** | uSipipo Telegram Bot |
| **Repositorio** | https://github.com/uSipipo-Team/usipipo-telegram-bot |
| **Bot** | @usipipobot |
| **Estado** | ✅ Producción |
| **Versión Actual** | v0.3.0 |
| **Tech Stack** | python-telegram-bot 22.7, httpx |

---

## 🎯 Propósito

Interface principal de usuario para el ecosistema uSipipo. Permite autenticación, gestión de VPN keys, pagos y soporte completamente desde Telegram.

---

## 🤖 Comandos Disponibles

| Comando | Descripción | Estado |
|---------|-------------|--------|
| `/start` | Registro y bienvenida | ✅ Implementado |
| `/me` | Ver perfil y VPN keys | ✅ Implementado |
| `/unlink` | Revocar acceso del bot | ✅ Implementado |
| `/help` | Mostrar comandos | ✅ Implementado |
| `/status` | Health check del servicio | ✅ Implementado |
| `/newkey` | Generar nueva VPN key | 🟡 Planificado |
| `/delkey` | Eliminar VPN key | 🟡 Planificado |
| `/qr` | Obtener QR de configuración | 🟡 Planificado |
| `/planes` | Ver planes disponibles | 🟡 Planificado |
| `/pago` | Registrar pago | 🟡 Planificado |
| `/soporte` | Contactar soporte | 🟡 Planificado |

---

## 🔄 Flujos Principales

### **1. Autenticación Invisible**

```
Usuario → /start → Bot
           ↓
Bot → POST /auth/telegram/auto-register → Backend
           ↓
Backend → Valida WebApp Init Data
           ↓
Backend → JWT tokens (access + refresh)
           ↓
Bot → Almacena en Redis (30d TTL)
           ↓
Bot → Mensaje de bienvenida
```

---

### **2. Ver Perfil**

```
Usuario → /me → Bot
           ↓
Bot → GET /users/me (con JWT) → Backend
           ↓
Backend → User data + VPN keys
           ↓
Bot → Muestra perfil con teclado inline
```

**Respuesta:**
```
👤 Juan Pérez
📊 Plan: Free
💾 Balance: 5.0 GB
🔑 VPN Keys: 2/5

[🔑 Mis Keys] [📊 Consumo] [💳 Pagos]
```

---

### **3. Generar VPN Key (Planificado)**

```
Usuario → /newkey → Bot
           ↓
Bot → Verifica balance >= 5 GB
           ↓
Bot → POST /vpn/keys → Backend
           ↓
Backend → Genera key (WireGuard/Outline)
           ↓
Bot → Muestra key + instrucciones
```

---

## 💾 Almacenamiento de Tokens

**Redis Structure:**
```
Key: usipipo:bot:tokens:{telegram_id}
Value: {
  "access_token": "...",
  "refresh_token": "...",
  "expires_at": "2026-03-27T11:00:00Z"
}
TTL: 30 días
```

**Auto-refresh:** 5 minutos antes de expiración

---

## 🎨 Teclados Inline

### **Main Keyboard**
```
[🔑 Mis Keys]  [📊 Consumo]
[💳 Pagos]    [🎫 Soporte]
[⚙️ Configuración]
```

### **VPN Keys Keyboard**
```
[📄 Key 1 - WireGuard]
[📄 Key 2 - Outline]
[➕ Nueva Key]
```

### **Payment Keyboard**
```
[💰 Crypto]  [⭐ Stars]
[📜 Historial]
[↩️ Volver]
```

---

## 📈 Roadmap

### **v0.4.0 (2026-04)**
- [ ] Gestión completa de VPN keys
- [ ] Pagos con Telegram Stars
- [ ] Sistema de tickets integrado

### **v0.5.0 (2026-05)**
- [ ] Notificaciones push
- [ ] Referidos desde bot
- [ ] Estadísticas de uso

---

**Última actualización:** 2026-03-27
