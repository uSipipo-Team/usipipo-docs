# Referrals Flow

> User journey for the Referrals feature in @usipipobot

---

## Overview

The Referrals feature allows users to invite friends and earn credits. Users can:
- View their referral statistics
- Share their referral link
- Apply referral codes from other users
- Redeem earned credits for data

---

## User Journey

### **Main Flow: Viewing Referrals**

```
┌─────────────────┐
│   Usuario       │
└────────┬────────┘
         │
         │ Envía /referidos
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ GET /api/v1/referrals/me
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  🎁 Programa de Referidos               │
│                                         │
│  Tu código: JUAN123                     │
│                                         │
│  📊 Estadísticas:                       │
│  • Amigos referidos: 5                  │
│  • Amigos activos: 3                    │
│  • Créditos ganados: 25 GB              │
│  • Créditos canjeados: 10 GB            │
│  • Créditos disponibles: 15 GB          │
│                                         │
│  [🎁 Canjear Créditos]                  │
│  [📝 Aplicar Código]                    │
│  [↩️ Volver]                            │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario selecciona opción
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Ejecuta acción seleccionada
         ▼
┌─────────────────┐
│   Usuario       │
│  (Acción)       │
└─────────────────┘
```

---

## States

### **Viewing Stats** (Initial State)

User is viewing their referral statistics and main menu.

**Entry:** User sends `/referidos`  
**Actions Available:**
- View stats (automatic)
- Click "Canjear Créditos"
- Click "Aplicar Código"
- Click "Volver"

**Exit Triggers:**
- User clicks "Canjear Créditos" → **Redeeming** state
- User clicks "Aplicar Código" → **Applying** state
- User clicks "Volver" → Main menu

---

### **Redeeming** (Selecting Credits to Redeem)

User is selecting how many credits to redeem for data.

**Entry:** User clicks "Canjear Créditos"  
**Actions Available:**
- Select credit amount (e.g., 5, 10, 15 GB)
- Confirm redemption
- Cancel and go back

**Flow:**
```
┌─────────────────┐
│  Redeeming      │
└────────┬────────┘
         │
         │ Usuario selecciona cantidad
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  🎁 Canjear Créditos                    │
│                                         │
│  Tienes 15 créditos disponibles         │
│                                         │
│  Opciones:                              │
│  • 5 créditos = 2.5 GB data             │
│  • 10 créditos = 5 GB data              │
│  • 15 créditos = 7.5 GB data            │
│                                         │
│  [Seleccionar 5] [Seleccionar 10]       │
│  [Seleccionar 15]                       │
│                                         │
│  [↩️ Volver]                            │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario selecciona opción
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  ✅ ¿Confirmar canje?                   │
│                                         │
│  Estás canjeando 10 créditos            │
│  Recibirás: 5 GB de data                │
│                                         │
│  Créditos restantes: 5                  │
│                                         │
│  [✅ Confirmar] [❌ Cancelar]            │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario confirma
         ▼
┌─────────────────┐
│  Backend        │
└────────┬────────┘
         │
         │ POST /api/v1/referrals/redeem
         │ Valida créditos disponibles
         │ Agrega data al usuario
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Muestra confirmación
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  ✅ ¡Canje Exitoso!                     │
│                                         │
│  Has canjeado 10 créditos               │
│  Recibiste: 5 GB de data                │
│                                         │
│  Créditos restantes: 5                  │
│                                         │
│  [🎁 Ver Estadísticas]                  │
│                                         │
└─────────────────────────────────────────┘
```

**Exit Triggers:**
- User confirms → Redemption processed → **Viewing Stats** state
- User cancels → **Viewing Stats** state

---

### **Applying** (Entering Referral Code)

User is entering a referral code from another user.

**Entry:** User clicks "Aplicar Código"  
**Actions Available:**
- Enter referral code
- Submit code
- Cancel and go back

**Flow:**
```
┌─────────────────┐
│  Applying       │
└────────┬────────┘
         │
         │ Bot solicita código
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  📝 Aplicar Código de Referido          │
│                                         │
│  ¿Tienes un código de referido?         │
│                                         │
│  Ingresa el código aquí:                │
│                                         │
│  [Escribe tu código...]                 │
│                                         │
│  [Enviar] [❌ Cancelar]                  │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario envía código
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Valida formato
         │ POST /api/v1/referrals/apply
         ▼
┌─────────────────┐
│  Backend        │
└────────┬────────┘
         │
         │ Valida código
         │ Verifica no sea el mismo usuario
         │ Agrega bono si es válido
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Muestra resultado
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  ✅ ¡Código Aplicado!                   │
│                                         │
│  Has aplicado el código: PEDRO456       │
│                                         │
│  🎁 Bono: 5 GB de data                  │
│                                         │
│  ¡Gracias por unirte!                   │
│                                         │
│  [🎁 Ver Estadísticas]                  │
│                                         │
└─────────────────────────────────────────┘
```

**Error Cases:**
```
┌─────────────────────────────────────────┐
│                                         │
│  ❌ Código Inválido                     │
│                                         │
│  El código "INVALID" no existe o        │
│  ha expirado.                           │
│                                         │
│  Por favor verifica e intenta de nuevo. │
│                                         │
│  [🔄 Intentar de Nuevo]                 │
│  [↩️ Volver]                            │
│                                         │
└─────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────┐
│                                         │
│  ❌ No Puedes Auto-Referirte            │
│                                         │
│  No puedes aplicar tu propio código     │
│  de referido.                           │
│                                         │
│  ¡Invita a tus amigos en su lugar!      │
│                                         │
│  [🎁 Ver Estadísticas]                  │
│                                         │
└─────────────────────────────────────────┘
```

**Exit Triggers:**
- Code applied successfully → **Viewing Stats** state
- Invalid code → Retry or **Viewing Stats** state
- User cancels → **Viewing Stats** state

---

## Callback Queries

### `referral_redeem_confirm:{credits}`

**Purpose:** User confirming credit redemption  
**Data:** `credits` (integer) - Number of credits to redeem  
**Handler:** `handle_redeem_confirm()`

**Flow:**
1. User clicks "Seleccionar X créditos"
2. Bot shows confirmation dialog
3. User clicks "Confirmar"
4. Bot calls `POST /api/v1/referrals/redeem`
5. Bot shows success/error message

---

### `referral_apply`

**Purpose:** User wants to apply a referral code  
**Data:** None  
**Handler:** `handle_apply_code()`

**Flow:**
1. User clicks "Aplicar Código"
2. Bot prompts for code input
3. User types code
4. Bot calls `POST /api/v1/referrals/apply`
5. Bot shows success/error message

---

### `referral_back`

**Purpose:** User wants to go back to main menu  
**Data:** None  
**Handler:** `handle_back()`

**Flow:**
1. User clicks "Volver"
2. Bot returns to main menu or previous state

---

## Messages

### REFERRAL_STATS

Shows user's referral statistics with action menu.

**Template:**
```
🎁 Programa de Referidos

Tu código: {code}

📊 Estadísticas:
• Amigos referidos: {total_referrals}
• Amigos activos: {active_referrals}
• Créditos ganados: {credits_earned} GB
• Créditos canjeados: {credits_redeemed} GB
• Créditos disponibles: {credits_available} GB

[🎁 Canjear Créditos]
[📝 Aplicar Código]
[↩️ Volver]
```

---

### INVITE_LINK

Shows user's referral link for sharing.

**Template:**
```
🔗 Tu Link de Referido

Comparte este link con tus amigos:

{link}

[📋 Copiar Link]
[📤 Compartir]
[↩️ Volver]

💡 Tip: Ganas 5 GB por cada amigo que se registre
y compre su primer plan.
```

---

### REDEEM_CONFIRMATION

Confirms credit redemption.

**Template:**
```
✅ ¡Canje Exitoso!

Has canjeado {credits} créditos
Recibiste: {data_awarded} GB de data

Créditos restantes: {remaining_credits}

[🎁 Ver Estadísticas]
```

---

### APPLY_SUCCESS

Confirms referral code application.

**Template:**
```
✅ ¡Código Aplicado!

Has aplicado el código: {code}

🎁 Bono: {bonus} GB de data

¡Gracias por unirte!

[🎁 Ver Estadísticas]
```

---

### APPLY_ERROR

Invalid or expired referral code.

**Template:**
```
❌ Código Inválido

El código "{code}" no existe o ha expirado.

Por favor verifica e intenta de nuevo.

[🔄 Intentar de Nuevo]
[↩️ Volver]
```

---

## Backend Integration

### API Calls

| Action | Endpoint | Method |
|--------|----------|--------|
| Get stats | `/api/v1/referrals/me` | GET |
| Apply code | `/api/v1/referrals/apply` | POST |
| Redeem credits | `/api/v1/referrals/redeem` | POST |

### Error Handling

| Error | Status Code | User Message |
|-------|-------------|--------------|
| Invalid code | 400 | "Código inválido o expirado" |
| Self-referral | 409 | "No puedes auto-referirte" |
| Insufficient credits | 400 | "Créditos insuficientes" |
| Already redeemed | 409 | "Ya aplicaste este código" |
| Network error | - | "Error de conexión. Intenta de nuevo." |

---

## Related Documentation

- [Phase 7 Design](../../plans/telegram-bot/2026-03-28-phase-7-referrals-tickets-design.md)
- [Tickets Flow](tickets-flow.md)
- [Telegram Bot Flow](telegram-bot-flow.md)

---

**Última actualización:** 2026-03-28
