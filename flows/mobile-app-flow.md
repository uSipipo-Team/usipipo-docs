# 🔄 Mobile App Flow - Flujo de Conexión VPN

> Diagrama de flujo de conexión VPN en la app Android

---

## 📐 Visión General

Flujo de conexión VPN en la app Android de uSipipo.

---

## 🔐 Flujo de Login

```
App → Select Login Method
      ├─→ Telegram Widget → Telegram App → Auth → Deep Link → App
      └─→ Manual → Username + Code → Backend → JWT → App

App → Store JWT in flutter_secure_storage
App → Navigate to Home
```

---

## 🔑 Flujo de Conexión

```
Home Screen
    │
    ├─→ Select VPN Key (WireGuard/Outline)
    │
    ├─→ Press Connect Button
    │
    ├─→ Start Foreground Service
    │
    ├─→ Platform Channel → Native VPN
    │   ├─→ WireGuard: wg-quick up
    │   └─→ Outline: Shadowsocks tunnel
    │
    ├─→ Connection State: Connecting
    │
    ├─→ Success → State: Connected
    │   └─→ Show stats (upload/download)
    │
    └─→ Error → State: Error
        └─→ Show error message
```

---

## 📊 Monitoreo en Tiempo Real

```
Connected State
    │
    ├─→ Poll traffic stats (cada 1s)
    │   ├─→ RX bytes
    │   └─→ TX bytes
    │
    ├─→ Calculate speed
    │   ├─→ Upload Mbps
    │   └─→ Download Mbps
    │
    ├─→ Update UI
    │
    └─→ Check limits
        ├─→ Data limit reached → Disconnect
        └─→ Key expired → Disconnect
```

---

## 🔌 Desconexión

```
User Presses Disconnect
    │
    ├─→ Stop Foreground Service
    │
    ├─→ Platform Channel → Native VPN
    │   └─→ wg-quick down / stop tunnel
    │
    ├─→ State: Disconnected
    │
    └─→ Show session summary
        ├─→ Duration
        ├─→ Data used
        └─→ Avg speed
```

---

**Última actualización:** 2026-03-27
