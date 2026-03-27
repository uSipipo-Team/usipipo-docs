# 🔄 Backend API Flow - Flujo de Autenticación y VPN

> Diagrama de flujo de autenticación y gestión de VPN keys en el backend

---

## 📐 Visión General

Este documento describe los flujos principales del backend API para autenticación y gestión de VPN keys.

---

## 🔐 Flujo de Autenticación

### **1. Auto-Registro vía Telegram**

```
┌─────────────┐
│   Usuario   │
└──────┬──────┘
       │ 1. Abre Telegram Bot
       │ 2. Envía /start con WebApp Init Data
       ▼
┌─────────────┐
│  Bot        │
│  (PTB)      │
└──────┬──────┘
       │ 3. POST /auth/telegram/auto-register
       │    Headers: Telegram-Init-Data
       ▼
┌─────────────┐
│  Backend    │
│  (FastAPI)  │
└──────┬──────┘
       │ 4. Validar signature
       │    - Parsear initData
       │    - Calcular hash con bot token
       │    - Verificar timestamp < 5 min
       ▼
┌─────────────┐
│  PostgreSQL │
└──────┬──────┘
       │ 5. SELECT user WHERE telegram_id = ?
       │
       ├─────┐
       │     │ Usuario no existe
       │     ▼
       │  6. INSERT user (telegram_id, username, balance_gb=5)
       │     INSERT referral_code (unique)
       │
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 7. Generar JWT tokens
       │    - access_token (30 min)
       │    - refresh_token (30 días)
       ▼
┌─────────────┐
│  Redis      │
└──────┬──────┘
       │ 8. STORE usipipo:bot:tokens:{telegram_id}
       │    TTL: 30 días
       │
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 9. Retornar {user, tokens}
       ▼
┌─────────────┐
│  Bot        │
└──────┬──────┘
       │ 10. Mensaje de bienvenida
       ▼
┌─────────────┐
│   Usuario   │
└─────────────┘
```

---

### **2. Token Refresh Automático**

```
┌─────────────┐
│  Client     │
└──────┬──────┘
       │ 1. Check: access_token expira en < 5 min
       │
       ▼
┌─────────────┐
│  Client     │
└──────┬──────┘
       │ 2. POST /auth/refresh
       │    Headers: Authorization: Bearer {refresh_token}
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 3. Validar refresh_token
       │    - Signature válida
       │    - No expirado
       │    - No revocado
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 4. Generar nuevo access_token
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 5. Retornar {access_token}
       ▼
┌─────────────┐
│  Client     │
└──────┬──────┘
       │ 6. Actualizar token en storage
       │
       │ (Silent, sin UX)
```

---

## 🔑 Flujo de Generación de VPN Key

### **WireGuard**

```
┌─────────────┐
│   Usuario   │
└──────┬──────┘
       │ 1. /newkey (Telegram) o POST /vpn/keys (API)
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 2. Verificar balance_gb >= 5
       │    SELECT balance_gb FROM users WHERE id = ?
       │
       │ Si balance < 5:
       │ ❌ Retornar error "Insufficient balance"
       │
       │ Si balance >= 5:
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 3. Generar WireGuard keys
       │    wg genkey → private_key
       │    wg pubkey → public_key
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 4. Generar configuración
       │    [Interface]
       │    PrivateKey = {private_key}
       │    Address = 10.0.0.X/24
       │    DNS = 1.1.1.1
       │
       │    [Peer]
       │    PublicKey = {server_public_key}
       │    Endpoint = vpn.usipipo.com:51820
       │    AllowedIPs = 0.0.0.0/0
       ▼
┌─────────────┐
│  PostgreSQL │
└──────┬──────┘
       │ 5. INSERT vpn_key
       │    - user_id
       │    - key_type = 'wireguard'
       │    - key_data = {config}
       │    - data_limit_bytes = 5 GB
       │
       │ UPDATE users
       │    SET balance_gb = balance_gb - 5
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 6. Retornar VPN key config
       ▼
┌─────────────┐
│   Usuario   │
└─────────────┘
```

---

### **Outline (Shadowsocks)**

```
┌─────────────┐
│   Usuario   │
└──────┬──────┘
       │ 1. POST /vpn/keys (protocol=outline)
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 2. Verificar balance_gb >= 5
       ▼
┌─────────────┐
│  Outline API│
└──────┬──────┘
       │ 3. POST /access-keys
       │    {
       │      "method": "aes-256-gcm",
       │      "name": "user-uuid",
       │      "limit": { "bytes": 5368709120 }
       │    }
       ▼
┌─────────────┐
│  Outline API│
└──────┬──────┘
       │ 4. Retornar
       │    {
       │      "id": "key_id",
       │      "accessUrl": "ss://..."
       │    }
       ▼
┌─────────────┐
│  PostgreSQL │
└──────┬──────┘
       │ 5. INSERT vpn_key
       │    - key_type = 'outline'
       │    - key_data = {accessUrl}
       │    - external_id = {key_id}
       ▼
┌─────────────┐
│   Usuario   │
└─────────────┘
```

---

## 💰 Flujo de Pago con Crypto

```
┌─────────────┐
│   Usuario   │
└──────┬──────┘
       │ 1. /pago (Telegram) o POST /payments/crypto (API)
       │    { package: "one_month", method: "USDT" }
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 2. POST https://api.trondealer.com/v2/order/create
       │    {
       │      "amount": 7.20,
       │      "token": "USDT",
       │      "network": "BSC",
       │      "order_id": "usipipo_{uuid}"
       │    }
       ▼
┌─────────────┐
│ TronDealer  │
└──────┬──────┘
       │ 3. Retornar
       │    {
       │      "order_id": "...",
       │      "wallet_address": "0x...",
       │      "amount": 7.20,
       │      "expires_at": "2026-03-27T11:00:00Z"
       │    }
       ▼
┌─────────────┐
│  PostgreSQL │
└──────┬──────┘
       │ 4. INSERT payment
       │    - status = 'pending'
       │    - crypto_address = {wallet_address}
       │    - expires_at = 30 min
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 5. Retornar datos de pago al usuario
       │    "Envía 7.20 USDT a: 0x..."
       │    "Expires en: 29:45"
       ▼
┌─────────────┐
│   Usuario   │
└──────┬──────┘
       │ 6. Realiza transferencia crypto
       │
       ▼
┌─────────────┐
│  Blockchain │
└──────┬──────┘
       │ 7. Transacción confirmada
       ▼
┌─────────────┐
│ TronDealer  │
└──────┬──────┘
       │ 8. POST /webhook/trondealer
       │    {
       │      "order_id": "...",
       │      "tx_hash": "0x...",
       │      "amount": 7.20,
       │      "status": "confirmed"
       │    }
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 9. Validar webhook signature
       │    - HMAC SHA-256
       │    - Token válido
       │    - Timestamp < 5 min
       ▼
┌─────────────┐
│  PostgreSQL │
└──────┬──────┘
       │ 10. UPDATE payment SET status = 'completed'
       │     UPDATE users SET balance_gb += 10
       │     INSERT subscription_plan (si aplica)
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 11. Notificar usuario (FCM / Telegram)
       │     "✅ Pago confirmado. 10 GB agregados."
       ▼
┌─────────────┐
│   Usuario   │
└─────────────┘
```

---

## 📊 Flujo de Facturación por Consumo

### **Ciclo de Billing**

```
┌─────────────┐
│  Scheduler  │
│  (Cron)     │
└──────┬──────┘
       │ 1. Daily job: 00:00 UTC
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 2. Para cada usuario con consumo > 0:
       │
       │ SELECT SUM(bytes_used) FROM vpn_keys
       │    WHERE user_id = ?
       │    AND billing_reset_at > NOW() - 30 days
       │
       ▼
┌─────────────┐
│  PostgreSQL │
└──────┬──────┘
       │ 3. Calcular consumo
       │    mb_consumed = total_bytes / 1024 / 1024
       │    total_cost = mb_consumed × 0.000244140625
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 4. Si cycle cerrado:
       │    INSERT consumption_invoice
       │    - amount_usd = {total_cost}
       │    - expires_at = 30 min
       │    - wallet_address = {payment_wallet}
       │
       ▼
┌─────────────┐
│   Usuario   │
└──────┬──────┘
       │ 5. Notificación
       │    "Tu factura de consumo: $5.50 USD"
       │    "Paga antes de: 29:45"
       │
       │ [Pagar con Crypto] [Pagar con Stars]
       ▼
┌─────────────┐
│   Usuario   │
└──────┬──────┘
       │ 6. Realiza pago
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 7. UPDATE invoice SET status = 'paid'
       │    UPDATE consumption_billing SET status = 'paid'
       │    RESET vpn_keys.used_bytes = 0
       │    UPDATE vpn_keys.billing_reset_at = NOW() + 30 days
       ▼
┌─────────────┐
│   Usuario   │
└─────────────┘
```

---

## 🎫 Flujo de Ticket de Soporte

```
┌─────────────┐
│   Usuario   │
└──────┬──────┘
       │ 1. /soporte o POST /tickets
       │    {
       │      category: "vpn_fail",
       │      priority: "high",
       │      subject: "No puedo conectar"
       │    }
       ▼
┌─────────────┐
│  PostgreSQL │
└──────┬──────┘
       │ 2. INSERT ticket
       │    - ticket_number = 'T-' + random(7 digits)
       │    - status = 'open'
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 3. Notificar admins (email/Telegram)
       │    "Nuevo ticket: T-1234567"
       │    "Categoría: VPN_FAIL"
       ▼
┌─────────────┐
│    Admin    │
└──────┬──────┘
       │ 4. Revisa ticket en admin panel
       │    GET /admin/tickets/{id}
       ▼
┌─────────────┐
│    Admin    │
└──────┬──────┘
       │ 5. POST /tickets/{id}/messages
       │    {
       │      message: "Hola, ¿qué error ves?",
       │      from_admin: true
       │    }
       ▼
┌─────────────┐
│  Backend    │
└──────┬──────┘
       │ 6. Notificar usuario (Telegram)
       │    "Nuevo mensaje en tu ticket T-1234567"
       ▼
┌─────────────┐
│   Usuario   │
└──────┬──────┘
       │ 7. Responde mensaje
       ▼
┌─────────────┐
│    Admin    │
└──────┬──────┘
       │ 8. Resuelve ticket
       │    PUT /tickets/{id}/status
       │    { status: "resolved" }
       │
       ▼
┌─────────────┐
│   Usuario   │
└──────┬──────┘
       │ 9. Notificación de resolución
       │    "Tu ticket ha sido resuelto"
       │    [Calificar atención]
```

---

## 📚 Recursos Relacionados

- [Ecosystem Architecture](../context/ecosystem-architecture.md)
- [Backend API Reference](../apis/backend-api-reference.md)

---

**Última actualización:** 2026-03-27
