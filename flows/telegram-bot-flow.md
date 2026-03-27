# 🔄 Telegram Bot Flow - Journey del Usuario

> Diagrama de flujo de usuario en el bot de Telegram

---

## 📐 Visión General

Este documento describe el journey completo de un usuario en el bot de Telegram @usipipobot.

---

## 🚀 Flujo de Onboarding

### **Primer Uso (/start)**

```
┌─────────────────┐
│   Usuario       │
│  (Nuevo)        │
└────────┬────────┘
         │
         │ 1. Abre Telegram
         │ 2. Busca @usipipobot
         │ 3. Toca "Iniciar" o envía /start
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ 4. Muestra mensaje de bienvenida
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  👋 ¡Hola! Bienvenido a uSipipo VPN     │
│                                         │
│  Soy tu asistente virtual. Puedo       │
│  ayudarte con:                          │
│                                         │
│  🔑 Generar VPN keys                    │
│  💳 Ver tu saldo                        │
│  📊 Ver tu consumo                      │
│  🎫 Soporte técnico                     │
│                                         │
│  ¿Qué necesitas?                        │
│                                         │
│  [🔑 Mis Keys]  [💳 Pagos]             │
│  [📊 Consumo]   [🎫 Soporte]            │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario selecciona opción
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ 5. Ejecuta acción seleccionada
         ▼
┌─────────────────┐
│   Usuario       │
│  (Registrado)   │
└─────────────────┘
```

---

## 🔑 Flujo de Generación de VPN Key

```
┌─────────────────┐
│   Usuario       │
└────────┬────────┘
         │
         │ Toca [🔑 Mis Keys]
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ GET /users/me/vpn-keys
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  🔑 Tus VPN Keys                        │
│                                         │
│  Tienes 2/5 keys activas                │
│                                         │
│  📄 Home VPN - WireGuard                │
│     2.5 GB / 5 GB usados                │
│     Expira: 2026-04-27                  │
│     [📋 Copiar] [🗑️ Eliminar]           │
│                                         │
│  📄 Work VPN - Outline                  │
│     1.0 GB / 5 GB usados                │
│     Expira: 2026-04-25                  │
│     [📋 Copiar] [🗑️ Eliminar]           │
│                                         │
│  [➕ Nueva Key]                         │
│                                         │
└─────────────────────────────────────────┐
         │
         │ Usuario toca [➕ Nueva Key]
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Verifica balance >= 5 GB
         │
         ├──────────────┐
         │              │
    Balance >= 5    Balance < 5
         │              │
         ▼              ▼
┌─────────────┐  ┌──────────────────┐
│  Generar    │  │  ❌ Saldo        │
│  key        │  │  insuficiente    │
│             │  │                  │
│             │  │  Tienes 3.0 GB   │
│             │  │  Necesitas 5 GB  │
│             │  │                  │
│             │  │  [💳 Recargar]   │
│             │  │  [↩️ Volver]     │
│             │  └──────────────────┘
│             │
│  ✅ Key creada
│
│  📄 New VPN - WireGuard
│
│  Data: 5 GB
│  Expira: 2026-04-27
│
│  [📋 Copiar Config]
│  [📱 Ver QR]
│  [↩️ Volver]
│
└─────────────┘
```

---

## 💳 Flujo de Pago

```
┌─────────────────┐
│   Usuario       │
└────────┬────────┘
         │
         │ Toca [💳 Pagos] o /pago
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  💳 Métodos de Pago                     │
│                                         │
│  Selecciona método:                     │
│                                         │
│  [₿ Crypto]      [⭐ Telegram Stars]   │
│                                         │
│  [📜 Historial]  [↩️ Volver]            │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario selecciona Crypto
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  ₿ Pago con Crypto                      │
│                                         │
│  Planes disponibles:                    │
│                                         │
│  📦 1 Mes - $7.20 USD (360 Stars)       │
│     10 GB de data                       │
│     [Seleccionar]                       │
│                                         │
│  📦 3 Meses - $19.20 USD (960 Stars)    │
│     30 GB de data                       │
│     [Seleccionar]                       │
│                                         │
│  📦 6 Meses - $31.20 USD (1560 Stars)   │
│     60 GB de data                       │
│     [Seleccionar]                       │
│                                         │
│  [↩️ Volver]                            │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario selecciona "1 Mes"
         ▼
┌─────────────────┐
│  Backend        │
└────────┬────────┘
         │
         │ Crea orden en TronDealer
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  💰 Realiza tu pago                     │
│                                         │
│  Envía exactamente:                     │
│  7.20 USDT                              │
│                                         │
│  Red: BSC (BEP20)                       │
│  Address:                               │
│  0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb
│                                         │
│  ⏰ Expires en: 29:45                   │
│                                         │
│  [📋 Copiar Address]                    │
│  [🔄 Verificar Pago]                    │
│  [❌ Cancelar]                          │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario realiza transferencia
         │
         │ Bot monitorea blockchain
         ▼
┌─────────────────┐
│  TronDealer     │
└────────┬────────┘
         │
         │ Detecta pago
         │ Webhook → Backend
         ▼
┌─────────────────┐
│  Backend        │
└────────┬────────┘
         │
         │ Valida webhook
         │ Actualiza payment.status
         │ Agrega 10 GB al usuario
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Notifica usuario
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  ✅ ¡Pago Confirmado!                   │
│                                         │
│  Gracias por tu compra                  │
│                                         │
│  📦 Plan: 1 Mes                         │
│  💰 Monto: 7.20 USDT                    │
│  💾 Data: 10 GB agregados               │
│                                         │
│  Tu nueva VPN key está lista            │
│  [🔑 Ver Keys]                          │
│                                         │
└─────────────────────────────────────────┘
```

---

## 📊 Flujo de Ver Consumo

```
┌─────────────────┐
│   Usuario       │
└────────┬────────┘
         │
         │ Toca [📊 Consumo] o /me
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  👤 Juan Pérez                          │
│  @juanperez                             │
│                                         │
│  📊 Plan: Free                          │
│  💾 Balance: 5.0 GB                     │
│  🔑 Keys activas: 2/5                   │
│                                         │
│  ─────────────────────────────────      │
│                                         │
│  📈 Consumo este ciclo:                 │
│                                         │
│  Home VPN    ████████░░  2.5/5 GB       │
│  Work VPN    ██░░░░░░░░  1.0/5 GB       │
│                                         │
│  Total: 3.5 GB / 10 GB                  │
│                                         │
│  📅 Ciclo reset: 2026-04-27             │
│                                         │
│  [🔑 Mis Keys]  [💳 Recargar]           │
│                                         │
└─────────────────────────────────────────┘
```

---

## 🎫 Flujo de Soporte

```
┌─────────────────┐
│   Usuario       │
└────────┬────────┘
         │
         │ Toca [🎫 Soporte] o /soporte
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  🎫 Centro de Soporte                   │
│                                         │
│  ¿En qué podemos ayudarte?              │
│                                         │
│  Selecciona categoría:                  │
│                                         │
│  🌐 VPN no funciona                     │
│  💳 Problema con pago                   │
│  👤 Mi cuenta                           │
│  📝 Otro                                │
│                                         │
│  [📜 Mis tickets]  [↩️ Volver]          │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario selecciona "VPN no funciona"
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  🌐 VPN no funciona                     │
│                                         │
│  Por favor describe tu problema:        │
│                                         │
│  [Escribe tu mensaje aquí...]           │
│                                         │
│  [Enviar]  [Cancelar]                   │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario escribe mensaje
         ▼
┌─────────────────┐
│  Backend        │
└────────┬────────┘
         │
         │ Crea ticket
         │ Asigna número: T-1234567
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  ✅ Ticket Creado                       │
│                                         │
│  Número: T-1234567                      │
│  Categoría: VPN_FAIL                    │
│  Prioridad: HIGH                        │
│                                         │
│  Un agente te responderá pronto         │
│                                         │
│  [📜 Ver Tickets]                       │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Admin responde (backend)
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Notifica usuario
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  💬 Nuevo mensaje en tu ticket          │
│                                         │
│  Ticket: T-1234567                      │
│                                         │
│  Agente: Carlos                         │
│  "Hola Juan, ¿qué error ves cuando     │
│   intentas conectar?"                   │
│                                         │
│  [Responder]  [Ver Ticket]              │
│                                         │
└─────────────────────────────────────────┘
```

---

## 🔗 Flujo de Referidos

```
┌─────────────────┐
│   Usuario       │
└────────┬────────┘
         │
         │ /referir
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  🎁 Programa de Referidos               │
│                                         │
│  ¡Gana 5 GB por cada amigo!             │
│                                         │
│  Tu código: JUAN123                     │
│                                         │
│  Amigos referidos: 3                    │
│  GB ganados: 15 GB                      │
│                                         │
│  Amigos que compraron: 2                │
│  GB bonus: 10 GB                        │
│                                         │
│  [📋 Copiar Código]                     │
│  [🔗 Compartir Link]                    │
│  [📊 Ver Estadísticas]                  │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario comparte código
         ▼
┌─────────────────┐
│  Amigo          │
└────────┬────────┘
         │
         │ Se registra con código
         ▼
┌─────────────────┐
│  Backend        │
└────────┬────────┘
         │
         │ Vincula referido
         ▼
┌─────────────────┐
│  Amigo          │
└────────┬────────┘
         │
         │ Compra primer plan
         ▼
┌─────────────────┐
│  Backend        │
└────────┬────────┘
         │
         │ Detecta primera compra
         │ Agrega 5 GB al referidor
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Notifica usuario
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  🎉 ¡Ganaste 5 GB!                      │
│                                         │
│  Tu amigo @amigo compró su primer plan  │
│                                         │
│  GB ganados: 5 GB                       │
│  Total referido: 15 GB                  │
│                                         │
│  ¡Sigue invitando!                      │
│                                         │
└─────────────────────────────────────────┘
```

---

## 📊 Métricas del Bot

| Métrica | Objetivo | Actual |
|---------|----------|--------|
| **Usuarios activos** | 1,000 MAU | 200 |
| **Comandos/día** | 5,000 | 800 |
| **Conversión a pago** | 12% | 8% |
| **Tickets creados/mes** | 100 | 25 |
| **Respuesta < 1h** | 90% | 95% |

---

## 📚 Recursos Relacionados

- [Backend API Flow](backend-api-flow.md)
- [Ecosystem Architecture](../context/ecosystem-architecture.md)

---

**Última actualización:** 2026-03-27
