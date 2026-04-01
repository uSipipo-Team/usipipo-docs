# 🔌 Backend API Reference - Referencia Completa de API

> Documentación completa de todos los endpoints de la API uSipipo

---

## 📊 Información General

| Campo | Valor |
|-------|-------|
| **Base URL** | `https://usipipo.duckdns.org/api/v1` |
| **Autenticación** | JWT Bearer Token |
| **Formato** | JSON |
| **Documentación Interactiva** | `/docs` (Swagger), `/redoc` (ReDoc) |

---

## 🔐 Autenticación

### **Esquema de Autenticación**

```
Authorization: Bearer <access_token>
```

**Tipos de Token:**
- **Access Token:** 30 minutos de validez
- **Refresh Token:** 30 días de validez

---

### **Obtener Token (Telegram)**

```http
POST /api/v1/auth/telegram/auto-register
Content-Type: application/json
Telegram-Init-Data: query_id=AAE...&user=%7B%22id%22%3A123456789%7D&hash=abc123...
```

**Request:**
- Header `Telegram-Init-Data`: Datos de inicialización de Telegram WebApp

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "telegram_id": 123456789,
      "username": "juanperez",
      "first_name": "Juan",
      "last_name": "Pérez",
      "is_admin": false,
      "balance_gb": 5.0,
      "total_purchased_gb": 0.0,
      "referral_code": "JUAN123",
      "referral_credits": 0,
      "purchase_count": 0,
      "loyalty_bonus_percent": 0,
      "created_at": "2026-03-27T10:30:00Z",
      "updated_at": "2026-03-27T10:30:00Z"
    },
    "tokens": {
      "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "token_type": "Bearer",
      "expires_in": 1800
    }
  }
}
```

---

### **Refresh Token**

```http
POST /api/v1/auth/refresh
Authorization: Bearer <refresh_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "Bearer",
    "expires_in": 1800
  }
}
```

---

### **Logout**

```http
POST /api/v1/auth/logout
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "message": "Sesión cerrada correctamente"
}
```

---

## 👥 Usuarios

### **Obtener Perfil**

```http
GET /api/v1/users/me
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "telegram_id": 123456789,
    "username": "juanperez",
    "first_name": "Juan",
    "last_name": "Pérez",
    "is_admin": false,
    "balance_gb": 5.0,
    "total_purchased_gb": 10.0,
    "referral_code": "JUAN123",
    "referral_credits": 15,
    "purchase_count": 2,
    "loyalty_bonus_percent": 5,
    "active_keys": 2,
    "total_keys": 3,
    "created_at": "2026-03-27T10:30:00Z",
    "updated_at": "2026-03-27T12:00:00Z"
  }
}
```

---

### **Actualizar Perfil**

```http
PUT /api/v1/users/me
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "first_name": "Juan Carlos",
  "last_name": "Pérez López"
}
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "telegram_id": 123456789,
    "username": "juanperez",
    "first_name": "Juan Carlos",
    "last_name": "Pérez López",
    ...
  }
}
```

---

### **Obtener Estadísticas de Usuario**

```http
GET /api/v1/users/me/stats
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "total_data_used_gb": 15.5,
    "total_data_purchased_gb": 25.0,
    "total_payments": 3,
    "total_spent_usd": 21.60,
    "average_monthly_spending_usd": 7.20,
    "referrals_count": 3,
    "referrals_with_purchase": 2,
    "account_age_days": 30,
    "last_activity": "2026-03-27T12:00:00Z"
  }
}
```

---

## 🔑 VPN Keys

### **Listar Keys**

```http
GET /api/v1/vpn/keys
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "user_id": "550e8400-e29b-41d4-a716-446655440000",
      "key_type": "wireguard",
      "status": "active",
      "name": "Home VPN",
      "key_data": "[Interface]\nPrivateKey = ...\nAddress = 10.0.0.2/24\n...",
      "external_id": null,
      "created_at": "2026-03-27T10:30:00Z",
      "used_bytes": 2684354560,
      "last_seen_at": "2026-03-27T12:00:00Z",
      "data_limit_bytes": 5368709120,
      "billing_reset_at": "2026-04-27T10:30:00Z",
      "expires_at": null,
      "used_gb": 2.5,
      "data_limit_gb": 5.0,
      "remaining_bytes": 2684354560,
      "is_active": true,
      "is_over_limit": false
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440002",
      "user_id": "550e8400-e29b-41d4-a716-446655440000",
      "key_type": "outline",
      "status": "active",
      "name": "Work VPN",
      "key_data": "ss://YWVzLTI1Ni1nY206cGFzc3dvcmQ@vpn.usipipo.com:8080#WorkVPN",
      "external_id": "outline_key_123",
      "created_at": "2026-03-25T08:00:00Z",
      "used_bytes": 1073741824,
      "last_seen_at": "2026-03-27T11:30:00Z",
      "data_limit_bytes": 5368709120,
      "billing_reset_at": "2026-04-25T08:00:00Z",
      "expires_at": null,
      "used_gb": 1.0,
      "data_limit_gb": 5.0,
      "remaining_bytes": 4294967296,
      "is_active": true,
      "is_over_limit": false
    }
  ]
}
```

---

### **Crear Nueva Key**

```http
POST /api/v1/vpn/keys
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "name": "Mobile VPN",
  "key_type": "wireguard"
}
```

**Response (201 Created):**
```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440003",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "key_type": "wireguard",
    "status": "active",
    "name": "Mobile VPN",
    "key_data": "[Interface]\nPrivateKey = ABC123...\nAddress = 10.0.0.3/24\nDNS = 1.1.1.1\n\n[Peer]\nPublicKey = XYZ789...\nEndpoint = vpn.usipipo.com:51820\nAllowedIPs = 0.0.0.0/0\nPersistentKeepalive = 25",
    "external_id": null,
    "created_at": "2026-03-27T12:30:00Z",
    "data_limit_bytes": 5368709120,
    "data_limit_gb": 5.0,
    "is_active": true
  },
  "message": "VPN key creada exitosamente. Se descontaron 5 GB de tu balance."
}
```

---

### **Obtener Key Específica**

```http
GET /api/v1/vpn/keys/{key_id}
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "key_type": "wireguard",
    "status": "active",
    "name": "Home VPN",
    "key_data": "[Interface]...",
    "used_gb": 2.5,
    "data_limit_gb": 5.0,
    "remaining_bytes": 2684354560,
    "is_active": true
  }
}
```

---

### **Eliminar Key**

```http
DELETE /api/v1/vpn/keys/{key_id}
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "message": "VPN key eliminada correctamente"
}
```

---

### **Obtener Métricas de Key**

```http
GET /api/v1/vpn/keys/{key_id}/metrics
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "key_id": "550e8400-e29b-41d4-a716-446655440001",
    "total_bytes_used": 2684354560,
    "total_bytes_uploaded": 1073741824,
    "total_bytes_downloaded": 1610612736,
    "sessions_count": 15,
    "last_session_start": "2026-03-27T10:00:00Z",
    "last_session_end": "2026-03-27T12:00:00Z",
    "last_session_duration_seconds": 7200,
    "average_session_duration_seconds": 3600,
    "peak_upload_mbps": 50.5,
    "peak_download_mbps": 120.3,
    "daily_usage": [
      {"date": "2026-03-25", "bytes": 536870912},
      {"date": "2026-03-26", "bytes": 1073741824},
      {"date": "2026-03-27", "bytes": 1073741824}
    ]
  }
}
```

---

### **Resetear Ciclo de Billing**

```http
POST /api/v1/vpn/keys/{key_id}/reset
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "key_id": "550e8400-e29b-41d4-a716-446655440001",
    "used_bytes": 0,
    "billing_reset_at": "2026-04-27T12:30:00Z"
  },
  "message": "Ciclo de billing reseteado exitosamente"
}
```

---

## 🌐 Server Selection

### **GET /api/v1/vpn/servers**

Get list of available VPN servers for user selection with real-time load indicators.

**Authentication:** Required (user JWT token)

**Query Parameters:**
- `protocol` (required, string): Protocol type - `"outline"` or `"wireguard"`

**Success Response (200 OK):**

```json
{
  "servers": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "US-East-1",
      "country_code": "US",
      "country_name": "United States",
      "city": "New York",
      "load_percentage": 23,
      "load_level": "low",
      "status": "online"
    },
    {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "name": "DE-Frankfurt-2",
      "country_code": "DE",
      "country_name": "Germany",
      "city": "Frankfurt",
      "load_percentage": 67,
      "load_level": "medium",
      "status": "online"
    }
  ],
  "recommended": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "US-East-1",
      "country_code": "US",
      "country_name": "United States",
      "city": "New York",
      "load_percentage": 23,
      "load_level": "low",
      "status": "online"
    }
  ]
}
```

**Response Fields:**
- `servers`: Array of all available servers matching protocol
- `recommended`: Array of top 5 servers with lowest load (best choices for users)
- `load_percentage`: Server load as percentage (0-100)
- `load_level`: Human-readable load indicator with emoji:
  - 🟢 `"low"`: 0-50% connections used
  - 🟡 `"medium"`: 51-80% connections used
  - 🔴 `"high"`: 81-100% connections used
- `status`: Server status (`"online"`, `"offline"`, `"maintenance"`)

**Error Responses:**

**400 Bad Request - Invalid Protocol**
```json
{
  "detail": "Invalid protocol: invalid. Must be 'outline' or 'wireguard'"
}
```

**401 Unauthorized**
```json
{
  "detail": "Not authenticated"
}
```

**Example Request:**
```bash
curl -X GET "http://localhost:8000/api/v1/vpn/servers?protocol=outline" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Use Case:**
This endpoint is used by the Telegram bot to display available servers to users during VPN key creation. Users can see server load in real-time and select the best server for their needs. The `recommended` array helps users quickly identify the least loaded servers for optimal performance.

---

## 💳 Pagos

### **Listar Pagos**

```http
GET /api/v1/payments
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440010",
      "user_id": "550e8400-e29b-41d4-a716-446655440000",
      "amount_usd": 7.20,
      "gb_purchased": 10.0,
      "method": "crypto_usdt",
      "status": "completed",
      "crypto_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
      "crypto_network": "BSC",
      "telegram_star_invoice_id": null,
      "created_at": "2026-03-20T10:00:00Z",
      "expires_at": "2026-03-20T10:30:00Z",
      "paid_at": "2026-03-20T10:15:00Z",
      "transaction_hash": "0xabc123def456..."
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 1,
    "total_pages": 1
  }
}
```

---

### **Crear Pago con Crypto**

```http
POST /api/v1/payments/crypto
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "package_type": "one_month",
  "token": "USDT",
  "network": "BSC"
}
```

**Response (201 Created):**
```json
{
  "status": "success",
  "data": {
    "payment_id": "550e8400-e29b-41d4-a716-446655440010",
    "order_id": "usipipo_order_123",
    "amount_usd": 7.20,
    "amount_crypto": 7.20,
    "token": "USDT",
    "network": "BSC",
    "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "expires_at": "2026-03-27T11:00:00Z",
    "time_remaining_seconds": 1785,
    "status": "pending"
  },
  "message": "Envía exactamente 7.20 USDT (red BSC) a la dirección mostrada. La orden expira en 29:45."
}
```

---

### **Crear Pago con Telegram Stars**

```http
POST /api/v1/payments/stars
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "package_type": "one_month"
}
```

**Response (201 Created):**
```json
{
  "status": "success",
  "data": {
    "payment_id": "550e8400-e29b-41d4-a716-446655440010",
    "invoice_id": "telegram_invoice_123",
    "amount_stars": 360,
    "amount_usd": 7.20,
    "package_type": "one_month",
    "invoice_payload": "invoice_payload_xyz",
    "status": "pending"
  },
  "message": "Invoice de Telegram creada. Completa el pago en Telegram."
}
```

---

### **Solicitar Reembolso**

```http
POST /api/v1/payments/{payment_id}/refund
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "reason": "El servicio no funciona correctamente"
}
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "payment_id": "550e8400-e29b-41d4-a716-446655440010",
    "refund_id": "refund_123",
    "amount_usd": 7.20,
    "status": "pending",
    "reason": "El servicio no funciona correctamente",
    "created_at": "2026-03-27T12:30:00Z"
  },
  "message": "Solicitud de reembolso creada. Será revisada en 24-48 horas."
}
```

---

## 📦 Suscripciones

### **Listar Planes Disponibles**

```http
GET /api/v1/subscriptions/plans
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": [
    {
      "plan_type": "one_month",
      "name": "1 Mes",
      "price_usd": 7.20,
      "price_stars": 360,
      "data_gb": 10,
      "vpn_keys": 5,
      "duration_days": 30,
      "features": [
        "10 GB de data",
        "Hasta 5 VPN keys",
        "Soporte prioritario",
        "Data rollover"
      ]
    },
    {
      "plan_type": "three_months",
      "name": "3 Meses",
      "price_usd": 19.20,
      "price_stars": 960,
      "data_gb": 30,
      "vpn_keys": 5,
      "duration_days": 90,
      "discount_percent": 11,
      "features": [
        "30 GB de data",
        "Hasta 5 VPN keys",
        "Soporte prioritario",
        "Data rollover",
        "Ahorras 11%"
      ]
    },
    {
      "plan_type": "six_months",
      "name": "6 Meses",
      "price_usd": 31.20,
      "price_stars": 1560,
      "data_gb": 60,
      "vpn_keys": 5,
      "duration_days": 180,
      "discount_percent": 28,
      "features": [
        "60 GB de data",
        "Hasta 5 VPN keys",
        "Soporte prioritario",
        "Data rollover",
        "Ahorras 28%"
      ]
    }
  ]
}
```

---

### **Obtener Suscripción Activa**

```http
GET /api/v1/subscriptions/me
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440020",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "plan_type": "one_month",
    "stars_paid": 360,
    "payment_id": "550e8400-e29b-41d4-a716-446655440010",
    "starts_at": "2026-03-20T10:15:00Z",
    "expires_at": "2026-04-20T10:15:00Z",
    "is_active": true,
    "is_expired": false,
    "days_remaining": 24,
    "is_expiring_soon": false,
    "created_at": "2026-03-20T10:15:00Z",
    "updated_at": "2026-03-20T10:15:00Z"
  }
}
```

---

### **Activar Suscripción**

```http
POST /api/v1/subscriptions/activate
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "payment_id": "550e8400-e29b-41d4-a716-446655440010",
  "transaction_id": "tx_123"
}
```

**Response (201 Created):**
```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440020",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "plan_type": "one_month",
    "stars_paid": 360,
    "payment_id": "550e8400-e29b-41d4-a716-446655440010",
    "starts_at": "2026-03-27T12:30:00Z",
    "expires_at": "2026-04-27T12:30:00Z",
    "is_active": true
  },
  "message": "Suscripción activada exitosamente"
}
```

---

## 📊 Facturación por Consumo

### **Listar Ciclos de Billing**

```http
GET /api/v1/billing/cycles
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440030",
      "user_id": 123456789,
      "started_at": "2026-02-27T00:00:00Z",
      "ended_at": "2026-03-27T00:00:00Z",
      "status": "closed",
      "mb_consumed": 5120.5,
      "total_cost_usd": 1.25,
      "price_per_mb_usd": 0.000244140625,
      "is_active": false,
      "is_closed": true,
      "is_paid": true,
      "gb_consumed": 5.0
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440031",
      "user_id": 123456789,
      "started_at": "2026-03-27T00:00:00Z",
      "ended_at": null,
      "status": "active",
      "mb_consumed": 1024.0,
      "total_cost_usd": 0.25,
      "price_per_mb_usd": 0.000244140625,
      "is_active": true,
      "is_closed": false,
      "is_paid": false,
      "gb_consumed": 1.0
    }
  ]
}
```

---

### **Obtener Invoice**

```http
GET /api/v1/billing/cycles/{cycle_id}/invoice
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "invoice_id": "550e8400-e29b-41d4-a716-446655440040",
    "billing_id": "550e8400-e29b-41d4-a716-446655440030",
    "user_id": 123456789,
    "amount_usd": 1.25,
    "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "payment_method": "crypto",
    "status": "paid",
    "expires_at": "2026-03-27T00:30:00Z",
    "paid_at": "2026-03-27T00:15:00Z",
    "transaction_hash": "0xabc123...",
    "created_at": "2026-03-27T00:00:00Z",
    "is_pending": false,
    "is_paid": true,
    "is_expired": false,
    "is_usdt_payment": true,
    "time_remaining_seconds": 0,
    "time_remaining_formatted": "00:00"
  }
}
```

---

### **Pagar Invoice**

```http
POST /api/v1/billing/cycles/{cycle_id}/pay
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "payment_method": "crypto",
  "transaction_hash": "0xabc123..."
}
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "invoice_id": "550e8400-e29b-41d4-a716-446655440040",
    "status": "paid",
    "paid_at": "2026-03-27T12:30:00Z",
    "transaction_hash": "0xabc123..."
  },
  "message": "Invoice pagada exitosamente"
}
```

---

## 🎫 Tickets de Soporte

### **Listar Tickets**

```http
GET /api/v1/tickets
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440050",
      "user_id": 123456789,
      "category": "vpn_fail",
      "priority": "high",
      "subject": "No puedo conectar mi VPN",
      "status": "open",
      "ticket_number": "T-1234567",
      "is_open": true,
      "is_resolved": false,
      "is_closed": false,
      "created_at": "2026-03-27T10:00:00Z",
      "updated_at": "2026-03-27T10:00:00Z",
      "resolved_at": null,
      "resolved_by": null,
      "admin_notes": null
    }
  ]
}
```

---

### **Crear Ticket**

```http
POST /api/v1/tickets
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "category": "vpn_fail",
  "priority": "high",
  "subject": "No puedo conectar mi VPN",
  "message": "Hola, intento conectar mi VPN WireGuard pero recibo error de timeout. Ya verifiqué mi conexión a internet y funciona bien. ¿Qué puedo hacer?"
}
```

**Response (201 Created):**
```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440050",
    "user_id": 123456789,
    "category": "vpn_fail",
    "priority": "high",
    "subject": "No puedo conectar mi VPN",
    "status": "open",
    "ticket_number": "T-1234567",
    "created_at": "2026-03-27T12:30:00Z"
  },
  "message": "Ticket creado exitosamente. Un agente te responderá pronto."
}
```

---

### **Enviar Mensaje en Ticket**

```http
POST /api/v1/tickets/{ticket_id}/messages
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "message": "Gracias por la ayuda. Ya pude conectar siguiendo las instrucciones."
}
```

**Response (201 Created):**
```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440060",
    "ticket_id": "550e8400-e29b-41d4-a716-446655440050",
    "from_user_id": 123456789,
    "from_admin": false,
    "message": "Gracias por la ayuda. Ya pude conectar siguiendo las instrucciones.",
    "created_at": "2026-03-27T14:00:00Z"
  }
}
```

---

### **Actualizar Estado de Ticket**

```http
PUT /api/v1/tickets/{ticket_id}/status
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "status": "resolved"
}
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440050",
    "status": "resolved",
    "resolved_at": "2026-03-27T15:00:00Z",
    "resolved_by": 1
  },
  "message": "Ticket marcado como resuelto"
}
```

---

## 📦 Paquetes de Datos

### **Listar Opciones de Paquetes**

```http
GET /api/v1/data-packages/options
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": [
    {
      "package_type": "basic",
      "name": "BASIC",
      "data_gb": 5,
      "price_usd": 2.50,
      "price_stars": 300,
      "validity_days": 30,
      "description": "5 GB de data adicional"
    },
    {
      "package_type": "estandar",
      "name": "ESTANDAR",
      "data_gb": 15,
      "price_usd": 7.50,
      "price_stars": 900,
      "validity_days": 30,
      "description": "15 GB de data adicional"
    },
    {
      "package_type": "avanzado",
      "name": "AVANZADO",
      "data_gb": 30,
      "price_usd": 15.00,
      "price_stars": 1800,
      "validity_days": 30,
      "description": "30 GB de data adicional"
    },
    {
      "package_type": "premium",
      "name": "PREMIUM",
      "data_gb": 50,
      "price_usd": 25.00,
      "price_stars": 3000,
      "validity_days": 30,
      "description": "50 GB de data adicional"
    },
    {
      "package_type": "unlimited",
      "name": "UNLIMITED",
      "data_gb": -1,
      "price_usd": 50.00,
      "price_stars": 6000,
      "validity_days": 30,
      "description": "Data ilimitada por 30 días"
    }
  ]
}
```

---

### **Comprar Paquete**

```http
POST /api/v1/data-packages/purchase
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "package_type": "basic",
  "payment_method": "stars"
}
```

**Response (201 Created):**
```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440070",
    "user_id": 123456789,
    "package_type": "basic",
    "data_limit_bytes": 5368709120,
    "stars_paid": 300,
    "expires_at": "2026-04-27T12:30:00Z",
    "is_active": true,
    "data_limit_gb": 5.0,
    "remaining_bytes": 5368709120,
    "purchased_at": "2026-03-27T12:30:00Z"
  },
  "message": "Paquete de datos comprado exitosamente. 5 GB agregados a tu cuenta."
}
```

---

## 🏛️ Admin (Solo Admins)

### **Listar Usuarios**

```http
GET /api/v1/admin/users?page=1&per_page=20
Authorization: Bearer <admin_access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": [
    {
      "user_id": "550e8400-e29b-41d4-a716-446655440000",
      "telegram_id": 123456789,
      "username": "juanperez",
      "first_name": "Juan",
      "last_name": "Pérez",
      "total_keys": 3,
      "active_keys": 2,
      "total_deposited": 10.0,
      "referral_credits": 15,
      "registration_date": "2026-03-27T10:30:00Z",
      "last_activity": "2026-03-27T12:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

---

### **Obtener Estadísticas del Sistema**

```http
GET /api/v1/admin/stats
Authorization: Bearer <admin_access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "users": {
      "total": 150,
      "active_today": 45,
      "active_this_week": 120,
      "active_this_month": 150
    },
    "vpn_keys": {
      "total": 350,
      "active": 280,
      "expired": 50,
      "revoked": 20
    },
    "payments": {
      "total_today": 15,
      "total_this_week": 85,
      "total_this_month": 320,
      "revenue_today_usd": 108.00,
      "revenue_this_week_usd": 612.00,
      "revenue_this_month_usd": 2304.00
    },
    "vpn_servers": {
      "wireguard": {
        "is_healthy": true,
        "total_keys": 200,
        "active_keys": 180,
        "version": "1.0.20210914",
        "uptime": "15d 4h 32m"
      },
      "outline": {
        "is_healthy": true,
        "total_keys": 150,
        "active_keys": 100,
        "version": "1.6.10",
        "uptime": "10d 2h 15m"
      }
    }
  }
}
```

---

### **Revocar VPN Key**

```http
POST /api/v1/admin/keys/{key_id}/revoke
Authorization: Bearer <admin_access_token>
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "key_id": "550e8400-e29b-41d4-a716-446655440001",
    "previous_status": "active",
    "new_status": "revoked",
    "revoked_at": "2026-03-27T12:30:00Z",
    "revoked_by": 1
  },
  "message": "VPN key revocada exitosamente"
}
```

---

## 🔗 Webhooks

### **TronDealer Webhook**

```http
POST /api/v1/webhooks/trondealer
X-Webhook-Signature: <hmac-sha256-signature>
Content-Type: application/json

{
  "order_id": "usipipo_order_123",
  "amount": 7.20,
  "token": "USDT",
  "network": "BSC",
  "tx_hash": "0xabc123def456...",
  "status": "confirmed",
  "timestamp": "2026-03-27T10:15:00Z"
}
```

**Response (200 OK):**
```json
{
  "status": "success",
  "message": "Webhook procesado exitosamente"
}
```

---

## 📈 Códigos de Error

| Código | Error | Descripción |
|--------|-------|-------------|
| **400** | Bad Request | Datos inválidos en el request |
| **401** | Unauthorized | Token inválido o expirado |
| **403** | Forbidden | No tiene permisos para esta acción |
| **404** | Not Found | Recurso no encontrado |
| **409** | Conflict | Conflicto (ej: recurso ya existe) |
| **422** | Unprocessable Entity | Error de validación |
| **429** | Too Many Requests | Rate limit excedido |
| **500** | Internal Server Error | Error interno del servidor |

---

### **Ejemplo de Error Response**

```json
{
  "status": "error",
  "error": {
    "code": "insufficient_balance",
    "message": "Balance insuficiente. Necesitas 5 GB para crear una VPN key, pero solo tienes 3.0 GB.",
    "details": {
      "required_gb": 5.0,
      "current_balance_gb": 3.0,
      "action": "purchase_data_package"
    }
  }
}
```

---

## 📚 Recursos Relacionados

- [Ecosystem Architecture](../context/ecosystem-architecture.md)
- [Backend Stack](../technology/backend-stack.md)
- [Webhooks Integration](webhooks-integration.md)

---

**Última actualización:** 2026-03-31
