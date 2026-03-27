# 📋 PRD - uSipipo MiniApp Web

> Product Requirements Document de la Telegram Mini App

---

## 📊 Información del Producto

| Campo | Valor |
|-------|-------|
| **Nombre** | uSipipo MiniApp Web |
| **Repositorio** | https://github.com/uSipipo-Team/usipipo-miniapp-web |
| **Estado** | 🟡 Desarrollo |
| **Versión Actual** | v0.1.0 |
| **Tech Stack** | Flask 3.1+, Telegram WebApp API |

---

## 🎯 Propósito

Web app embebida en Telegram que permite gestionar VPN keys, ver consumo y realizar pagos sin salir del chat.

---

## 📱 Features

### **Fase 1 (MVP)**
- [ ] Autenticación vía Telegram WebApp
- [ ] Ver perfil y balance
- [ ] Listar VPN keys
- [ ] Ver consumo de datos

### **Fase 2**
- [ ] Generar nueva VPN key
- [ ] Eliminar VPN key
- [ ] Comprar paquete de datos
- [ ] Historial de pagos

### **Fase 3**
- [ ] Sistema de tickets
- [ ] Referidos
- [ ] Configuración de cuenta

---

## 🔌 Integración con Telegram

**WebApp API:**
```javascript
const tg = window.Telegram.WebApp;

// Obtener datos del usuario
const user = tg.initDataUnsafe.user;

// Enviar datos al backend
const hash = tg.initData; // Para validar
```

**Validación:**
- Backend valida `initData` con bot token
- Extrae telegram_id del payload
- Auto-registra o login de usuario

---

## 🎨 UI Components

**Telegram Native:**
- Usar Telegram Theme Colors
- MainButton para acciones principales
- BackButton para navegación
- Native alerts y confirms

**Custom:**
- Cards cyberpunk theme
- Tab bar inferior
- Loading states

---

## 📊 Arquitectura

```
src/
├── features/
│   ├── home/
│   ├── keys/
│   ├── billing/
│   └── profile/
├── infrastructure/
│   └── web/
│       ├── app.py
│       ├── templates/
│       └── static/
└── core/
    └── config/
```

---

## 🚀 Deployment

**Ruta:** `/miniapp/*`

**Proxy (Caddy):**
```
@miniapp path /miniapp/*
reverse_proxy @miniapp localhost:5001
```

---

**Última actualización:** 2026-03-27
