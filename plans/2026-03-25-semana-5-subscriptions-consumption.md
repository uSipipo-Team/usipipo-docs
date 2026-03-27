# Week 5: Backend - Subscriptions & Consumption Billing

**Fecha:** 2026-03-20
**Estado:** En Progreso
**Branch:** `feature/week-5-subscriptions-consumption`
**Dependencias:** Week 4 completada ✅ (PR #3 merged)

---

## 📦 Entregables de la Semana

- [ ] `SubscriptionService` migrado desde monorepo
- [ ] `SubscriptionPaymentService` migrado desde monorepo
- [ ] `ConsumptionBillingService` migrado desde monorepo
- [ ] `ConsumptionInvoiceService` migrado desde monorepo
- [ ] Repository interfaces implementadas
- [ ] Endpoints de suscripciones
- [ ] Endpoints de consumo billing
- [ ] Tests de integración

---

## 🎯 Tareas Detalladas

### Día 1-2: Subscription Service

#### 1.1 Migrar SubscriptionService
- [ ] Copiar `subscription_service.py` desde monorepo
- [ ] Adaptar imports a nueva estructura hexagonal
- [ ] Actualizar para usar `usipipo-commons` entities
- [ ] Tests unitarios

#### 1.2 Migrar SubscriptionPaymentService
- [ ] Copiar `subscription_payment_service.py` desde monorepo
- [ ] Integrar con `CryptoPaymentService` existente
- [ ] Adaptar Telegram Stars invoice
- [ ] Tests unitarios

#### 1.3 Crear Repository Interfaces
- [ ] `ISubscriptionRepository` interface
- [ ] `ISubscriptionTransactionRepository` interface
- [ ] Implementaciones SQLAlchemy

#### 1.4 Endpoints de Suscripciones
- [ ] `GET /api/v1/subscriptions/plans` - Listar planes
- [ ] `POST /api/v1/subscriptions/activate` - Activar suscripción
- [ ] `GET /api/v1/subscriptions/me` - Obtener suscripción actual

---

### Día 3-4: Consumption Billing Service

#### 2.1 Migrar ConsumptionBillingService
- [ ] Copiar `consumption_billing_service.py` desde monorepo
- [ ] Migrar sub-servicios:
  - [ ] `ConsumptionActivationService`
  - [ ] `ConsumptionCycleService`
  - [ ] `ConsumptionBillingDTOS`
- [ ] Adaptar imports y dependencias

#### 2.2 Migrar ConsumptionInvoiceService
- [ ] Copiar `consumption_invoice_service.py` desde monorepo
- [ ] Integrar con payment gateways existentes
- [ ] Tests unitarios

#### 2.3 Crear Repository Interfaces
- [ ] `IConsumptionBillingRepository` interface
- [ ] `IConsumptionInvoiceRepository` interface
- [ ] Implementaciones SQLAlchemy

#### 2.4 Endpoints de Consumo
- [ ] `POST /api/v1/consumption/activate` - Activar modo consumo
- [ ] `POST /api/v1/consumption/usage` - Registrar uso de datos
- [ ] `GET /api/v1/consumption/summary` - Obtener resumen de consumo
- [ ] `POST /api/v1/consumption/cycle/close` - Cerrar ciclo

---

### Día 5: Tests + Documentación

#### 3.1 Tests de Integración
- [ ] Tests para subscription endpoints
- [ ] Tests para consumption endpoints
- [ ] Tests de integración con payment gateways

#### 3.2 Documentación
- [ ] OpenAPI/Swagger docs
- [ ] API documentation para bot team

---

## 📝 Notas de Implementación

### Subscription Plans (del monorepo):
```python
SUBSCRIPTION_OPTIONS = [
    {"name": "1 Month", "plan_type": "one_month", "stars": 360, "usdt": 2.99},
    {"name": "3 Months", "plan_type": "three_months", "stars": 960, "usdt": 7.99, "bonus": 11},
    {"name": "6 Months", "plan_type": "six_months", "stars": 1560, "usdt": 12.99, "bonus": 13},
]
```

### Consumption Pricing:
- Price per MB: $0.000244140625 USD (~$0.25/GB)
- Cycle duration: 30 days
- Invoice expiry: 30 minutes

### Entities Needed (ya en commons):
- ✅ `SubscriptionPlan`
- ✅ `SubscriptionTransaction`
- ✅ `ConsumptionBilling`
- ✅ `ConsumptionInvoice`
- ✅ `PlanType` enum
- ✅ `SubscriptionTransactionStatus` enum
- ✅ `BillingStatus` enum
- ✅ `InvoiceStatus` enum
- ✅ `PaymentMethod` enum

---

## ✅ Criterios de Aceptación

| Criterio | Estado |
|----------|--------|
| SubscriptionService migrado | ⏳ Pendiente |
| SubscriptionPaymentService migrado | ⏳ Pendiente |
| ConsumptionBillingService migrado | ⏳ Pendiente |
| ConsumptionInvoiceService migrado | ⏳ Pendiente |
| Repositorios implementados | ⏳ Pendiente |
| Endpoints de suscripciones | ⏳ Pendiente |
| Endpoints de consumo | ⏳ Pendiente |
| Tests pasando (>90%) | ⏳ Pendiente |
| OpenAPI docs actualizadas | ⏳ Pendiente |

---

## 🔗 Referencias

- Monorepo: `/home/mowgli/usipipobot/application/services/`
- Subscription entities: `usipipo-commons v0.4.1`
- Payment entities: `usipipo-commons v0.4.1`

---

**PRÓXIMA SESIÓN:** Comenzar migración de SubscriptionService
