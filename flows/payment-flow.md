# 💰 Payment Flow - Flujo de Pagos

> Diagrama de flujo de pagos con Crypto y Telegram Stars

---

## 📐 Visión General

Flujos de pago del ecosistema uSipipo.

---

## ₿ Flujo de Pago con Crypto (USDT/USDC)

```
1. Usuario selecciona plan
2. Backend crea orden en TronDealer
3. Muestra wallet address y monto
4. Usuario envía crypto
5. TronDealer detecta pago
6. Webhook → Backend
7. Backend valida signature
8. Backend actualiza payment
9. Backend agrega data al usuario
10. Notificación de éxito
```

**Redes Soportadas:**
- BSC (BEP20)
- Ethereum (ERC20)
- Polygon
- TRON (TRC20)

---

## ⭐ Flujo de Pago con Telegram Stars

```
1. Usuario selecciona plan
2. Backend crea invoice de Telegram
3. Bot muestra invoice
4. Usuario paga con Stars
5. Telegram notifica pago
6. Backend actualiza payment
7. Backend agrega data al usuario
8. Notificación de éxito
```

**Conversión:**
- 120 Stars ≈ $1 USD

---

## 🔄 Reembolsos

```
Usuario solicita reembolso
    │
    ├─→ Admin revisa ticket
    │
    ├─→ Aprueba → Backend
    │   └─→ Crea refund en TronDealer/Telegram
    │
    ├─→ Backend actualiza payment
    │
    └─→ Notifica usuario
```

---

**Última actualización:** 2026-03-27
