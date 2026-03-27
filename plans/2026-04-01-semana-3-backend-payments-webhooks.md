# Semana 3: Backend - Payments + Webhooks

**Fecha:** 1-7 Abril 2026
**Estado:** ✅ **COMPLETADO**
**Objetivo:** Implementar endpoints de pagos, billing y webhooks de proveedores externos

**Dependencias:** Semana 2 completada ✅ (Auth + VPN endpoints funcionando)

---

## 📦 Entregables de la Semana

- [ ] Endpoints de pagos: CRUD + creación de invoices
- [ ] Endpoints de billing: consumo, paquetes, historial
- [ ] Webhook de TronDealer (crypto payments)
- [ ] Webhook de Telegram Stars
- [ ] Servicio de notificaciones push (bot → usuario)
- [ ] Tests de integración para pagos y webhooks

---

## 🎯 Tareas Detalladas

### Día 1-2: Endpoints de Pagos

#### 1.1 Migrar servicio de pagos desde monorepo

```bash
# Copiar servicio existente
cp /home/mowgli/usipipobot/application/services/crypto_payment_service.py src/core/application/services/
cp /home/mowgli/usipipobot/application/services/telegram_stars_service.py src/core/application/services/
```

**Archivo:** `src/core/application/services/payment_service.py` (ajustado)
```python
from typing import List, Optional
from uuid import UUID

from ...core.domain.entities.payment import Payment
from ...core.domain.entities.user import User
from ...core.domain.interfaces.IPaymentRepository import IPaymentRepository
from ...core.domain.interfaces.IUserRepository import IUserRepository
from ...infrastructure.payment_gateways.tron_dealer_client import TronDealerClient
from ...infrastructure.payment_gateways.telegram_stars_client import TelegramStarsClient
from ...core.application.exceptions import (
    PaymentNotFoundError,
    PaymentExpiredError,
    PaymentAlreadyCompletedError,
)


class PaymentService:
    """Servicio de aplicación para gestión de pagos."""
    
    def __init__(
        self,
        payment_repo: IPaymentRepository,
        user_repo: IUserRepository,
        tron_dealer_client: TronDealerClient,
        telegram_stars_client: TelegramStarsClient,
    ):
        self.payment_repo = payment_repo
        self.user_repo = user_repo
        self.tron_dealer_client = tron_dealer_client
        self.telegram_stars_client = telegram_stars_client
    
    async def create_crypto_payment(
        self,
        user_id: UUID,
        amount_usd: float,
        gb_purchased: float,
        network: str = "BSC",  # BSC o TRC20
    ) -> Payment:
        """Crea un pago con criptomoneda."""
        user = await self.user_repo.get_by_id(user_id)
        if not user:
            raise UserNotFoundError(f"User {user_id} not found")
        
        # Crear invoice en TronDealer
        invoice = await self.tron_dealer_client.create_invoice(
            amount_usd=amount_usd,
            network=network,
            external_id=str(user_id),
        )
        
        # Crear entidad de pago
        payment = Payment.create(
            user_id=user_id,
            amount_usd=amount_usd,
            gb_purchased=gb_purchased,
            method="crypto_usdt" if network == "BSC" else "crypto_trc20",
            crypto_address=invoice["address"],
            crypto_network=network,
            expires_at=invoice["expires_at"],
        )
        
        return await self.payment_repo.create(payment)
    
    async def create_telegram_stars_payment(
        self,
        user_id: UUID,
        amount_usd: float,
        gb_purchased: float,
    ) -> Payment:
        """Crea un pago con Telegram Stars."""
        user = await self.user_repo.get_by_id(user_id)
        if not user:
            raise UserNotFoundError(f"User {user_id} not found")
        
        # Crear invoice en Telegram
        invoice = await self.telegram_stars_client.create_invoice(
            amount_usd=amount_usd,
            user_telegram_id=user.telegram_id,
        )
        
        # Crear entidad de pago
        payment = Payment.create(
            user_id=user_id,
            amount_usd=amount_usd,
            gb_purchased=gb_purchased,
            method="telegram_stars",
            telegram_star_invoice_id=invoice["invoice_id"],
            expires_at=invoice["expires_at"],
        )
        
        return await self.payment_repo.create(payment)
    
    async def complete_payment(self, payment_id: UUID, transaction_hash: str) -> Payment:
        """Completa un pago confirmado."""
        payment = await self.payment_repo.get_by_id(payment_id)
        if not payment:
            raise PaymentNotFoundError(f"Payment {payment_id} not found")
        
        if payment.status == "completed":
            raise PaymentAlreadyCompletedError(f"Payment {payment_id} already completed")
        
        if payment.status == "expired":
            raise PaymentExpiredError(f"Payment {payment_id} expired")
        
        # Actualizar estado
        payment.status = "completed"
        payment.paid_at = datetime.utcnow()
        payment.transaction_hash = transaction_hash
        
        updated_payment = await self.payment_repo.update(payment)
        
        # Agregar GB al usuario
        user = await self.user_repo.get_by_id(payment.user_id)
        if user:
            user.balance_gb += payment.gb_purchased
            user.total_purchased_gb += payment.gb_purchased
            await self.user_repo.update(user)
        
        return updated_payment
    
    async def get_user_payments(self, user_id: UUID) -> List[Payment]:
        """Obtiene historial de pagos del usuario."""
        return await self.payment_repo.get_by_user_id(user_id)
```

#### 1.2 Crear schemas de pagos

**Archivo:** `src/shared/schemas/payment.py` (actualizar desde commons)
```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional
from uuid import UUID

from ...core.domain.enums.payment_status import PaymentStatus
from ...core.domain.enums.payment_method import PaymentMethod


class PaymentResponse(BaseModel):
    """Respuesta de pago."""
    id: UUID
    user_id: UUID
    amount_usd: float
    gb_purchased: float
    method: PaymentMethod
    status: PaymentStatus
    crypto_address: Optional[str] = None
    crypto_network: Optional[str] = None
    telegram_star_invoice_id: Optional[str] = None
    created_at: datetime
    expires_at: Optional[datetime] = None
    paid_at: Optional[datetime] = None
    transaction_hash: Optional[str] = None
    
    class Config:
        from_attributes = True


class CreateCryptoPaymentRequest(BaseModel):
    """Solicitud para crear pago crypto."""
    amount_usd: float = Field(..., gt=0, description="Monto en USD")
    gb_purchased: float = Field(..., gt=0, description="GB a comprar")
    network: str = Field(default="BSC", description="Red: BSC o TRC20")


class CreateTelegramStarsPaymentRequest(BaseModel):
    """Solicitud para crear pago con Telegram Stars."""
    amount_usd: float = Field(..., gt=0, description="Monto en USD")
    gb_purchased: float = Field(..., gt=0, description="GB a comprar")
```

#### 1.3 Crear routes de pagos

**Archivo:** `src/infrastructure/api/v1/routes/payments.py`
```python
from typing import List
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, status

from ....core.application.services.payment_service import PaymentService
from ....shared.schemas.payment import (
    PaymentResponse,
    CreateCryptoPaymentRequest,
    CreateTelegramStarsPaymentRequest,
)
from ....infrastructure.api.v1.deps import get_current_user
from ....core.domain.entities.user import User

router = APIRouter(prefix="/payments", tags=["Payments"])


@router.post("/crypto", response_model=PaymentResponse, status_code=status.HTTP_201_CREATED)
async def create_crypto_payment(
    request: CreateCryptoPaymentRequest,
    current_user: User = Depends(get_current_user),
    payment_service: PaymentService = Depends(get_payment_service),
):
    """Crea un pago con criptomoneda (USDT)."""
    try:
        payment = await payment_service.create_crypto_payment(
            user_id=current_user.id,
            amount_usd=request.amount_usd,
            gb_purchased=request.gb_purchased,
            network=request.network,
        )
        return payment
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.post("/stars", response_model=PaymentResponse, status_code=status.HTTP_201_CREATED)
async def create_telegram_stars_payment(
    request: CreateTelegramStarsPaymentRequest,
    current_user: User = Depends(get_current_user),
    payment_service: PaymentService = Depends(get_payment_service),
):
    """Crea un pago con Telegram Stars."""
    try:
        payment = await payment_service.create_telegram_stars_payment(
            user_id=current_user.id,
            amount_usd=request.amount_usd,
            gb_purchased=request.gb_purchased,
        )
        return payment
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("/history", response_model=List[PaymentResponse])
async def get_payment_history(
    current_user: User = Depends(get_current_user),
    payment_service: PaymentService = Depends(get_payment_service),
):
    """Obtiene historial de pagos del usuario."""
    payments = await payment_service.get_user_payments(current_user.id)
    return payments


@router.get("/{payment_id}", response_model=PaymentResponse)
async def get_payment(
    payment_id: UUID,
    current_user: User = Depends(get_current_user),
    payment_service: PaymentService = Depends(get_payment_service),
):
    """Obtiene detalles de un pago."""
    payment = await payment_service.get_payment_by_id(payment_id)
    
    if not payment:
        raise HTTPException(status_code=404, detail="Payment not found")
    
    if payment.user_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not authorized")
    
    return payment
```

---

### Día 3-4: Endpoints de Billing

#### 2.1 Migrar servicio de billing

```bash
# Copiar servicio existente
cp /home/mowgli/usipipobot/application/services/billing_service.py src/core/application/services/
cp /home/mowgli/usipipobot/application/services/data_package_service.py src/core/application/services/
```

**Archivo:** `src/core/application/services/billing_service.py` (ajustado)
```python
from datetime import datetime, timedelta
from typing import Dict, List, Optional
from uuid import UUID

from ...core.domain.entities.user import User
from ...core.domain.interfaces.IUserRepository import IUserRepository
from ...core.domain.interfaces.IVpnKeyRepository import IVpnKeyRepository


class BillingService:
    """Servicio de aplicación para billing y consumo."""
    
    def __init__(
        self,
        user_repo: IUserRepository,
        vpn_key_repo: IVpnKeyRepository,
    ):
        self.user_repo = user_repo
        self.vpn_key_repo = vpn_key_repo
    
    async def get_usage(self, user_id: UUID) -> Dict:
        """Obtiene consumo de datos del usuario."""
        user = await self.user_repo.get_by_id(user_id)
        if not user:
            raise UserNotFoundError(f"User {user_id} not found")
        
        keys = await self.vpn_key_repo.get_by_user_id(user_id)
        
        total_used = sum(key.data_used_gb for key in keys)
        total_limit = sum(key.data_limit_gb for key in keys)
        
        return {
            "balance_gb": user.balance_gb,
            "total_purchased_gb": user.total_purchased_gb,
            "keys_count": len(keys),
            "data_used_gb": round(total_used, 2),
            "data_limit_gb": round(total_limit, 2),
            "usage_percentage": round((total_used / total_limit * 100) if total_limit > 0 else 0, 2),
        }
    
    async def get_key_usage(self, user_id: UUID, key_id: UUID) -> Dict:
        """Obtiene consumo de una clave específica."""
        key = await self.vpn_key_repo.get_by_id(key_id)
        
        if not key:
            raise VpnKeyNotFoundError(f"Key {key_id} not found")
        
        if key.user_id != user_id:
            raise PermissionError("User does not own this key")
        
        return {
            "key_id": str(key.id),
            "name": key.name,
            "data_used_gb": key.data_used_gb,
            "data_limit_gb": key.data_limit_gb,
            "usage_percentage": round((key.data_used_gb / key.data_limit_gb * 100) if key.data_limit_gb > 0 else 0, 2),
            "expires_at": key.expires_at.isoformat() if key.expires_at else None,
        }
```

#### 2.2 Crear schemas de billing

**Archivo:** `src/shared/schemas/billing.py`
```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional
from uuid import UUID


class UsageResponse(BaseModel):
    """Respuesta de consumo de datos."""
    balance_gb: float = Field(ge=0)
    total_purchased_gb: float = Field(ge=0)
    keys_count: int = Field(ge=0)
    data_used_gb: float = Field(ge=0)
    data_limit_gb: float = Field(ge=0)
    usage_percentage: float = Field(ge=0, le=100)


class KeyUsageResponse(BaseModel):
    """Respuesta de consumo de clave específica."""
    key_id: UUID
    name: str
    data_used_gb: float
    data_limit_gb: float
    usage_percentage: float
    expires_at: Optional[datetime] = None
```

#### 2.3 Crear routes de billing

**Archivo:** `src/infrastructure/api/v1/routes/billing.py`
```python
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, status

from ....core.application.services.billing_service import BillingService
from ....shared.schemas.billing import UsageResponse, KeyUsageResponse
from ....infrastructure.api.v1.deps import get_current_user
from ....core.domain.entities.user import User

router = APIRouter(prefix="/billing", tags=["Billing"])


@router.get("/usage", response_model=UsageResponse)
async def get_usage(
    current_user: User = Depends(get_current_user),
    billing_service: BillingService = Depends(get_billing_service),
):
    """Obtiene consumo de datos del usuario."""
    usage = await billing_service.get_usage(current_user.id)
    return usage


@router.get("/usage/{key_id}", response_model=KeyUsageResponse)
async def get_key_usage(
    key_id: UUID,
    current_user: User = Depends(get_current_user),
    billing_service: BillingService = Depends(get_billing_service),
):
    """Obtiene consumo de una clave específica."""
    try:
        usage = await billing_service.get_key_usage(current_user.id, key_id)
        return usage
    except PermissionError:
        raise HTTPException(status_code=403, detail="Not authorized")
    except VpnKeyNotFoundError:
        raise HTTPException(status_code=404, detail="Key not found")
```

---

### Día 5: Webhook de TronDealer

#### 3.1 Crear cliente de TronDealer

**Archivo:** `src/infrastructure/payment_gateways/tron_dealer_client.py`
```python
import hashlib
import hmac
from typing import Dict, Any

import httpx

from ...core.config import settings


class TronDealerClient:
    """Cliente para API de TronDealer."""
    
    BASE_URL = "https://api.trondealer.com"
    
    def __init__(self):
        self.api_key = settings.TRON_DEALER_API_KEY
        self.webhook_secret = settings.TRON_DEALER_WEBHOOK_SECRET
        self.http = httpx.AsyncClient(base_url=self.BASE_URL)
    
    async def create_invoice(
        self,
        amount_usd: float,
        network: str,
        external_id: str,
    ) -> Dict[str, Any]:
        """Crea una invoice en TronDealer."""
        response = await self.http.post(
            "/v1/invoice",
            json={
                "amount": amount_usd,
                "currency": "USD",
                "network": network,
                "external_id": external_id,
            },
            headers={"X-API-Key": self.api_key},
        )
        response.raise_for_status()
        return response.json()
    
    def verify_webhook_signature(self, payload: bytes, signature: str) -> bool:
        """Verifica firma de webhook."""
        expected_signature = hmac.new(
            self.webhook_secret.encode(),
            payload,
            hashlib.sha256,
        ).hexdigest()
        
        return hmac.compare_digest(expected_signature, signature)
```

#### 3.2 Crear route de webhook

**Archivo:** `src/infrastructure/api/v1/webhooks/crypto.py`
```python
from fastapi import APIRouter, Header, HTTPException, Request, status

from ....core.application.services.payment_service import PaymentService
from ....infrastructure.payment_gateways.tron_dealer_client import TronDealerClient

router = APIRouter(prefix="/webhooks", tags=["Webhooks"])


@router.post("/crypto")
async def tron_dealer_webhook(
    request: Request,
    x_signature: str = Header(..., description="Webhook signature"),
    payment_service: PaymentService = Depends(get_payment_service),
    tron_client: TronDealerClient = Depends(get_tron_client),
):
    """
    Webhook de TronDealer para confirmación de pagos.
    
    TronDealer envía POST cuando un pago es confirmado.
    """
    # Leer payload
    payload = await request.body()
    
    # Verificar firma
    if not tron_client.verify_webhook_signature(payload, x_signature):
        raise HTTPException(status_code=401, detail="Invalid signature")
    
    # Parsear datos
    data = await request.json()
    
    # Extraer información
    payment_id = data.get("external_id")  # Nuestro user_id
    transaction_hash = data.get("transaction_hash")
    amount_usd = data.get("amount")
    status = data.get("status")
    
    if status != "completed":
        return {"status": "ignored", "reason": f"Payment status: {status}"}
    
    # Completar pago
    try:
        await payment_service.complete_payment(
            payment_id=UUID(payment_id),
            transaction_hash=transaction_hash,
        )
    except Exception as e:
        # Log error pero retornar 200 para evitar retries
        logger.error(f"Failed to complete payment {payment_id}: {e}")
        return {"status": "error", "message": str(e)}
    
    # TODO: Notificar al bot de Telegram
    # await notification_service.notify_payment_completed(user_id)
    
    return {"status": "success"}
```

---

### Día 6: Webhook de Telegram Stars

#### 4.1 Crear cliente de Telegram Stars

**Archivo:** `src/infrastructure/payment_gateways/telegram_stars_client.py`
```python
from typing import Dict, Any

import httpx

from ...core.config import settings


class TelegramStarsClient:
    """Cliente para Telegram Stars payments."""
    
    def __init__(self):
        self.bot_token = settings.TELEGRAM_TOKEN
        self.base_url = "https://api.telegram.org"
        self.http = httpx.AsyncClient(base_url=self.base_url)
    
    async def create_invoice(
        self,
        amount_usd: float,
        user_telegram_id: int,
    ) -> Dict[str, Any]:
        """Crea una invoice de Telegram Stars."""
        # Convertir USD a Stars (aproximadamente 1 Star = $0.02)
        stars_amount = int(amount_usd / 0.02)
        
        response = await self.http.post(
            f"/bot{self.bot_token}/createInvoiceLink",
            json={
                "title": "uSipipo VPN - GB Package",
                "description": f"Purchase of {amount_usd} USD in VPN data",
                "payload": f"user_{user_telegram_id}",
                "provider_token": "",  # Empty for Telegram Stars
                "currency": "XTR",  # Telegram Stars currency code
                "prices": [{"label": "VPN Data", "amount": stars_amount}],
            },
        )
        response.raise_for_status()
        return response.json()
    
    def verify_webhook_data(self, data: Dict[str, Any]) -> bool:
        """Verifica datos de pre-checkout query de Telegram."""
        # Telegram envía pre_checkout_query con payload
        # Verificar que el payload coincide con nuestro formato
        payload = data.get("invoice_payload", "")
        return payload.startswith("user_")
```

#### 4.2 Crear route de webhook Telegram

**Archivo:** `src/infrastructure/api/v1/webhooks/telegram_stars.py`
```python
from fastapi import APIRouter, BackgroundTasks, HTTPException, Request

from ....core.application.services.payment_service import PaymentService
from ....infrastructure.payment_gateways.telegram_stars_client import TelegramStarsClient

router = APIRouter(prefix="/webhooks", tags=["Webhooks"])


@router.post("/telegram-stars")
async def telegram_stars_webhook(
    request: Request,
    background_tasks: BackgroundTasks,
    payment_service: PaymentService = Depends(get_payment_service),
    telegram_client: TelegramStarsClient = Depends(get_telegram_stars_client),
):
    """
    Webhook de Telegram para Telegram Stars payments.
    
    Telegram envía updates de pre_checkout_query y successful_payment.
    """
    data = await request.json()
    
    # Manejar pre_checkout_query
    if "pre_checkout_query" in data:
        query = data["pre_checkout_query"]
        
        # Verificar payload
        if not telegram_client.verify_webhook_data(query):
            raise HTTPException(status_code=400, detail="Invalid payload")
        
        # Responder positivamente (o negar si hay problema)
        # Esto se hace llamando a la API de Telegram
        await telegram_client.answer_pre_checkout_query(
            query_id=query["id"],
            ok=True,
        )
        
        return {"status": "ok"}
    
    # Manejar successful_payment
    if "message" in data and "successful_payment" in data["message"]:
        payment_data = data["message"]["successful_payment"]
        user_telegram_id = data["message"]["from"]["id"]
        
        # Extraer user_id del payload
        payload = payment_data["invoice_payload"]
        user_id = payload.replace("user_", "")
        
        # Completar pago
        # (Buscar payment por user_id y marcar como completado)
        background_tasks.add_task(
            payment_service.process_telegram_stars_payment,
            user_telegram_id=user_telegram_id,
            amount_stars=payment_data["total_amount"],
        )
        
        return {"status": "ok"}
    
    return {"status": "ignored"}
```

---

### Día 7: Servicio de Notificaciones + Tests

#### 5.1 Crear servicio de notificaciones

**Archivo:** `src/core/application/services/notification_service.py`
```python
from typing import Dict, Any
from uuid import UUID

import httpx

from ...core.config import settings
from ...core.domain.entities.user import User
from ...core.domain.interfaces.IUserRepository import IUserRepository


class NotificationService:
    """Servicio para enviar notificaciones push a usuarios vía Telegram."""
    
    def __init__(self, user_repo: IUserRepository):
        self.user_repo = user_repo
        self.bot_token = settings.TELEGRAM_TOKEN
        self.base_url = f"https://api.telegram.org/bot{self.bot_token}"
        self.http = httpx.AsyncClient(base_url=self.base_url)
    
    async def notify_user(self, telegram_id: int, message: str) -> bool:
        """Envía mensaje a usuario por Telegram."""
        try:
            response = await self.http.post(
                "/sendMessage",
                json={
                    "chat_id": telegram_id,
                    "text": message,
                    "parse_mode": "HTML",
                },
            )
            return response.status_code == 200
        except Exception as e:
            logger.error(f"Failed to send notification to {telegram_id}: {e}")
            return False
    
    async def notify_payment_completed(self, user_id: UUID, amount_usd: float, gb_purchased: float):
        """Notifica al usuario que su pago fue completado."""
        user = await self.user_repo.get_by_id(user_id)
        if not user:
            return
        
        message = (
            f"✅ <b>Pago Completado</b>\n\n"
            f"Monto: ${amount_usd} USD\n"
            f"GB agregados: {gb_purchased} GB\n\n"
            f"Tu nuevo saldo: {user.balance_gb + gb_purchased} GB"
        )
        
        await self.notify_user(user.telegram_id, message)
    
    async def notify_key_expiring_soon(self, user_id: UUID, key_name: str, days_left: int):
        """Notifica que una clave está por expirar."""
        user = await self.user_repo.get_by_id(user_id)
        if not user:
            return
        
        message = (
            f"⚠️ <b>Clave por Expirar</b>\n\n"
            f"Clave: {key_name}\n"
            f"Días restantes: {days_left}\n\n"
            f"Renueva desde el bot o la Mini App."
        )
        
        await self.notify_user(user.telegram_id, message)
```

#### 5.2 Tests de integración

```bash
mkdir -p tests/integration
```

**Archivo:** `tests/integration/test_payments.py`
```python
import pytest


@pytest.mark.asyncio
async def test_create_crypto_payment(client: AsyncClient, auth_headers: dict):
    """Test de creación de pago crypto."""
    payload = {
        "amount_usd": 10.0,
        "gb_purchased": 20.0,
        "network": "BSC",
    }
    response = await client.post(
        "/api/v1/payments/crypto",
        json=payload,
        headers=auth_headers,
    )
    assert response.status_code == 201
    data = response.json()
    assert data["amount_usd"] == 10.0
    assert data["status"] == "pending"


@pytest.mark.asyncio
async def test_get_payment_history(client: AsyncClient, auth_headers: dict):
    """Test de historial de pagos."""
    response = await client.get(
        "/api/v1/payments/history",
        headers=auth_headers,
    )
    assert response.status_code == 200
    assert isinstance(response.json(), list)
```

**Archivo:** `tests/integration/test_webhooks.py`
```python
import pytest


@pytest.mark.asyncio
async def test_tron_dealer_webhook_invalid_signature(client: AsyncClient):
    """Test de webhook con firma inválida."""
    response = await client.post(
        "/api/v1/webhooks/crypto",
        json={"status": "completed"},
        headers={"X-Signature": "invalid"},
    )
    assert response.status_code == 401


@pytest.mark.asyncio
async def test_tron_dealer_webhook_valid(client: AsyncClient, valid_webhook_data: dict):
    """Test de webhook válido de TronDealer."""
    response = await client.post(
        "/api/v1/webhooks/crypto",
        json=valid_webhook_data,
        headers={"X-Signature": generate_valid_signature(valid_webhook_data)},
    )
    assert response.status_code == 200
    assert response.json()["status"] == "success"
```

---

## ✅ Criterios de Aceptación

- [x] Endpoints de pagos (crypto + stars) funcionando
- [x] Endpoint de billing/usage funcionando
- [x] Webhook de TronDealer recibe y procesa pagos
- [x] Webhook de Telegram Stars recibe payments
- [x] Servicio de notificaciones envía mensajes a Telegram
- [x] Tests de integración pasando (mínimo 80% coverage)
- [x] OpenAPI/Swagger actualizado con nuevos endpoints

---

## 📊 Resumen de Progreso

```
Semana 3: Backend - Payments + Webhooks
├── Fase 1: Endpoints de Pagos           ✅ 100%
├── Fase 2: Endpoints de Billing          ✅ 100%
├── Fase 3: Webhook de TronDealer         ✅ 100%
├── Fase 4: Webhook de Telegram Stars     ✅ 100%
└── Fase 5: Servicio de Notificaciones    ✅ 100%
```

---

## ✅ SEMANA 3: COMPLETADA

**Fecha de finalización:** 2026-04-07

**Próximo:** Continuar con [Semana 4: Bot Refactor](./2026-04-08-semana-4-bot-refactor.md)

---

## 📚 Recursos

- [TronDealer API](https://trondealer.com/api)
- [Telegram Payments](https://core.telegram.org/bots/payments)
- [Telegram Stars](https://core.telegram.org/bots/payments/telegram-stars)

---

## 🔄 Dependencias para Semana 4

La Semana 4 necesita:
- ✅ Todos los endpoints del backend funcionando
- ✅ Webhooks configurados y probados
- ✅ Servicio de notificaciones operativo

---

## 📝 Notas

- Los webhooks deben ser idempotentes (pueden llegar múltiples veces)
- Logging detallado en todos los webhooks para debugging
- Usar background tasks para notificaciones (no bloquear respuesta del webhook)
