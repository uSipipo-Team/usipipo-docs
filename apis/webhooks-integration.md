# 🔗 Webhooks Integration Guide

> Guía completa de integración de webhooks para el ecosistema uSipipo

---

## 📊 Visión General

Los webhooks permiten que servicios externos notifiquen eventos asíncronos al backend de uSipipo, principalmente para confirmación de pagos y eventos de Telegram.

---

## 🏦 TronDealer Webhook

### **Configuración**

**URL del Webhook:**
```
POST https://usipipo.duckdns.org/api/v1/webhooks/trondealer
```

**Configurar en TronDealer Dashboard:**
1. Ir a Settings → Webhooks
2. Agregar URL: `https://usipipo.duckdns.org/api/v1/webhooks/trondealer`
3. Secret: `<generar_secret>`
4. Eventos: `payment.confirmed`, `payment.pending`, `payment.failed`

---

### **Seguridad**

**Headers:**
```
X-Webhook-Signature: sha256=<hmac_signature>
X-Webhook-Timestamp: 1679904000
Content-Type: application/json
```

**Validación del Signature:**
```python
import hmac
import hashlib
from datetime import datetime, timezone

def validate_trondealer_webhook(payload: str, signature: str, timestamp: str, secret: str) -> bool:
    # Verificar timestamp (no más de 5 minutos)
    webhook_time = datetime.fromtimestamp(int(timestamp), tz=timezone.utc)
    now = datetime.now(timezone.utc)
    
    if (now - webhook_time).total_seconds() > 300:
        return False  # Timestamp muy antiguo
    
    # Calcular HMAC
    message = f"{timestamp}.{payload}".encode('utf-8')
    expected_signature = hmac.new(
        secret.encode('utf-8'),
        message,
        hashlib.sha256
    ).hexdigest()
    
    # Comparar signatures
    return hmac.compare_digest(f"sha256={expected_signature}", signature)
```

---

### **Payload de Payment Confirmed**

```json
{
  "event": "payment.confirmed",
  "data": {
    "order_id": "usipipo_order_123",
    "merchant_order_id": "550e8400-e29b-41d4-a716-446655440010",
    "amount": 7.20,
    "token": "USDT",
    "network": "BSC",
    "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "tx_hash": "0xabc123def456789...",
    "confirmations": 12,
    "required_confirmations": 10,
    "status": "confirmed",
    "paid_at": "2026-03-27T10:15:00Z",
    "timestamp": "2026-03-27T10:20:00Z"
  }
}
```

---

### **Respuesta Exitosa**

```json
{
  "status": "success",
  "message": "Webhook procesado exitosamente"
}
```

**Códigos HTTP:**
- `200 OK` - Webhook procesado correctamente
- `400 Bad Request` - Payload inválido
- `401 Unauthorized` - Signature inválido
- `404 Not Found` - Order no encontrada
- `500 Internal Server Error` - Error interno

---

### **Manejo de Reintentos**

TronDealer reintentará el webhook si:
- No recibe respuesta 200 OK
- Timeout > 30 segundos
- Error 5xx del servidor

**Política de reintentos:**
- Intento 1: Inmediato
- Intento 2: 1 minuto
- Intento 3: 5 minutos
- Intento 4: 15 minutos
- Intento 5: 1 hora
- Después: Marcar como fallido

---

## ⭐ Telegram Stars Webhook

### **Configuración**

**URL del Webhook:**
```
POST https://usipipo.duckdns.org/api/v1/webhooks/telegram
```

**Configurar en BotFather:**
1. `/setwebhook`
2. URL: `https://usipipo.duckdns.org/api/v1/webhooks/telegram`
3. Secret token: `<telegram_secret_token>`

---

### **Payload de PreCheckoutQuery**

```json
{
  "update_id": 123456789,
  "pre_checkout_query": {
    "id": "pre_checkout_query_123",
    "from": {
      "id": 123456789,
      "is_bot": false,
      "first_name": "Juan",
      "last_name": "Pérez",
      "username": "juanperez",
      "language_code": "es"
    },
    "currency": "XTR",
    "total_amount": 360,
    "invoice_payload": "invoice_payload_xyz",
    "shipping_option_id": null,
    "order_info": null
  }
}
```

---

### **Respuesta a PreCheckoutQuery**

```python
from telegram import Update
from telegram.ext import ContextTypes

async def pre_checkout_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Callback para PreCheckoutQuery"""
    query = update.pre_checkout_query
    
    # Validar payload
    if query.invoice_payload == "invoice_payload_xyz":
        # Aceptar pago
        await query.answer(ok=True)
    else:
        # Rechazar pago
        await query.answer(
            ok=False,
            error_message="Hubo un error con tu pago. Por favor intenta de nuevo."
        )
```

---

### **Payload de SuccessfulPayment**

```json
{
  "update_id": 123456790,
  "message": {
    "message_id": 456,
    "from": {
      "id": 123456789,
      "is_bot": false,
      "first_name": "Juan",
      "username": "juanperez"
    },
    "chat": {
      "id": 123456789,
      "first_name": "Juan",
      "username": "juanperez",
      "type": "private"
    },
    "date": 1679904000,
    "successful_payment": {
      "currency": "XTR",
      "total_amount": 360,
      "invoice_payload": "invoice_payload_xyz",
      "telegram_payment_charge_id": "telegram_charge_123",
      "provider_payment_charge_id": null
    }
  }
}
```

---

### **Manejo de SuccessfulPayment**

```python
from telegram import Update
from telegram.ext import ContextTypes

async def successful_payment(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Manejador de pago exitoso"""
    payment = update.message.successful_payment
    
    # Extraer datos del payload
    payload = json.loads(payment.invoice_payload)
    user_id = payload['user_id']
    package_type = payload['package_type']
    
    # Actualizar base de datos
    await db.payments.update().where(
        db.payments.c.telegram_payment_charge_id == payment.telegram_payment_charge_id
    ).values(
        status='completed',
        paid_at=datetime.now(timezone.utc)
    ).execute()
    
    # Agregar data al usuario
    await add_data_to_user(user_id, package_type)
    
    # Notificar usuario
    await update.message.reply_text(
        "✅ ¡Pago exitoso!\n\n"
        f"Montón: {payment.total_amount} Stars\n"
        f"Paquete: {package_type}\n\n"
        "Tu data ha sido agregada a tu cuenta."
    )
```

---

## 🔔 Firebase Cloud Messaging (FCM) Webhook

### **Configuración**

**Service Account:**
1. Ir a Firebase Console → Project Settings → Service Accounts
2. Generar nueva clave privada
3. Descargar JSON
4. Guardar en `/etc/usipipo/fcm-service-account.json`

---

### **Enviar Notificación Push**

```python
from firebase_admin import messaging

def send_push_notification(
    token: str,
    title: str,
    body: str,
    data: dict = None
) -> messaging.SendResponse:
    """Enviar notificación push"""
    
    message = messaging.Message(
        notification=messaging.Notification(
            title=title,
            body=body,
        ),
        data=data or {},
        token=token,
        android=messaging.AndroidConfig(
            priority='high',
            notification=messaging.AndroidNotification(
                sound='default',
                color='#00F0FF',
            )
        ),
        apns=messaging.APNSConfig(
            payload=messaging.APNSPayload(
                aps=messaging.Aps(
                    sound='default',
                    content_available=True,
                )
            )
        ),
    )
    
    return messaging.send(message)
```

---

### **Casos de Uso**

**1. Pago Confirmado:**
```python
send_push_notification(
    token=user.fcm_token,
    title="✅ Pago Confirmado",
    body=f"Tu pago de ${amount} USD ha sido confirmado. {data_gb} GB agregados.",
    data={
        "type": "payment_confirmed",
        "payment_id": payment_id,
        "amount": str(amount),
        "data_gb": str(data_gb),
    }
)
```

**2. Key por Expirar:**
```python
send_push_notification(
    token=user.fcm_token,
    title="⚠️ VPN por Expirar",
    body=f"Tu VPN key expira en {days_remaining} días. Renueva para continuar.",
    data={
        "type": "key_expiring",
        "key_id": key_id,
        "days_remaining": str(days_remaining),
    }
)
```

**3. Límite de Datos Alcanzado:**
```python
send_push_notification(
    token=user.fcm_token,
    title="📊 Límite de Datos Alcanzado",
    body="Has alcanzado tu límite de datos. Compra más data o espera el próximo ciclo.",
    data={
        "type": "data_limit_reached",
        "key_id": key_id,
        "used_gb": str(used_gb),
    }
)
```

---

## 📊 Monitoreo de Webhooks

### **Logs de Webhooks**

```python
import logging

webhook_logger = logging.getLogger('webhooks')

def log_webhook_event(event_type: str, payload: dict, status: str, details: str = ""):
    """Log de evento de webhook"""
    webhook_logger.info(
        f"Webhook Event: {event_type} | Status: {status} | Details: {details}",
        extra={
            'event_type': event_type,
            'payload': payload,
            'status': status,
        }
    )
```

---

### **Métricas a Monitorear**

| Métrica | Descripción | Alerta |
|---------|-------------|--------|
| **Webhook Success Rate** | % de webhooks procesados exitosamente | < 95% |
| **Average Processing Time** | Tiempo promedio de procesamiento | > 5s |
| **Failed Webhooks** | Número de webhooks fallidos | > 10/hora |
| **Duplicate Webhooks** | Webhooks duplicados detectados | > 5/hora |

---

## 🧪 Testing de Webhooks

### **Testing Local con ngrok**

```bash
# Instalar ngrok
npm install -g ngrok

# Exponer puerto local
ngrok http 8000

# URL generada: https://abc123.ngrok.io
# Configurar en TronDealer/Telegram
```

---

### **Payloads de Test**

**TronDealer Test:**
```bash
curl -X POST http://localhost:8000/api/v1/webhooks/trondealer \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Signature: sha256=test" \
  -H "X-Webhook-Timestamp: $(date +%s)" \
  -d '{
    "event": "payment.confirmed",
    "data": {
      "order_id": "test_order_123",
      "amount": 7.20,
      "token": "USDT",
      "network": "BSC",
      "tx_hash": "0xtest123",
      "status": "confirmed"
    }
  }'
```

---

### **Herramientas de Testing**

| Herramienta | Propósito |
|-------------|-----------|
| **ngrok** | Exponer localhost a internet |
| **Webhook.site** | Testing básico de webhooks |
| **Postman** | Testing manual de payloads |
| **jq** | Parsear JSON en terminal |

---

## 🔐 Mejores Prácticas de Seguridad

### **1. Validar Siempre el Signature**

```python
# ❌ MAL - No validar signature
@app.post("/webhooks/trondealer")
async def handle_webhook(payload: dict):
    process_payment(payload)  # ¡Inseguro!

# ✅ BIEN - Validar signature
@app.post("/webhooks/trondealer")
async def handle_webhook(request: Request, payload: dict):
    signature = request.headers.get("X-Webhook-Signature")
    timestamp = request.headers.get("X-Webhook-Timestamp")
    
    if not validate_signature(payload, signature, timestamp):
        raise HTTPException(status_code=401, detail="Invalid signature")
    
    process_payment(payload)
```

---

### **2. Usar HTTPS Siempre**

```python
# Forzar HTTPS en producción
@app.middleware("http")
async def enforce_https(request: Request, call_next):
    if settings.APP_ENV == "production" and request.url.scheme != "https":
        return JSONResponse(
            status_code=301,
            content={"message": "HTTPS required"},
            headers={"Location": request.url.replace(scheme="https")}
        )
    return await call_next(request)
```

---

### **3. Rate Limiting para Webhooks**

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/webhooks/trondealer")
@limiter.limit("100/minute")
async def handle_trondealer_webhook(request: Request, payload: dict):
    ...
```

---

### **4. Idempotencia**

```python
# Usar order_id para idempotencia
async def process_payment(payload: dict):
    order_id = payload['data']['order_id']
    
    # Verificar si ya fue procesado
    existing = await db.payments.find_one({"tron_dealer_order_id": order_id})
    
    if existing and existing['status'] == 'completed':
        return {"status": "success", "message": "Ya procesado"}  # Idempotente
    
    # Procesar pago
    ...
```

---

## 📚 Recursos Relacionados

- [Backend API Reference](backend-api-reference.md)
- [Payment Flow](../flows/payment-flow.md)
- [Security Best Practices](../technology/backend-stack.md#seguridad)

---

**Última actualización:** 2026-03-27
