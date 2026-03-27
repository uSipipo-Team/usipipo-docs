# 🔌 VPN Connection Flow - Flujo de Conexión VPN

> Diagrama de flujo de establecimiento de conexión VPN

---

## 📐 Visión General

Flujo de conexión a servidores VPN usando WireGuard y Outline.

---

## 🔵 Flujo WireGuard

### **1. Generación de Keys**

```
Backend → wg genkey → private_key
Backend → wg pubkey → public_key
Backend → Genera config
```

### **2. Configuración del Cliente**

```
[Interface]
PrivateKey = <client_private_key>
Address = 10.0.0.X/24
DNS = 1.1.1.1

[Peer]
PublicKey = <server_public_key>
Endpoint = vpn.usipipo.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

### **3. Conexión**

```
Cliente → wg-quick up wg0
        │
        ├─→ Crea interfaz tun0
        │
        ├─→ Handshake con servidor
        │   ├─→ Noise Protocol
        │   └─→ Intercambio de keys
        │
        ├─→ Estado: Connected
        │
        └─→ Traffic flow
            ├─→ Encrypt (ChaCha20)
            └─→ Route por tun0
```

---

## 🟠 Flujo Outline (Shadowsocks)

### **1. Generación de URL**

```
Outline API → createAccessKey
            │
            ├─→ method: aes-256-gcm
            ├─→ password: random
            └─→ port: random

URL: ss://base64(method:password)@hostname:port/#name
```

### **2. Conexión**

```
Cliente → Parse ss:// URL
        │
        ├─→ Conecta a hostname:port
        │
        ├─→ Handshake Shadowsocks
        │   ├─→ Cifrado AES-256-GCM
        │   └─→ Autenticación
        │
        ├─→ Estado: Connected
        │
        └─→ Traffic flow
            ├─→ Encrypt payload
            └─→ Send por SOCKS5
```

---

## 📊 Monitoreo de Conexión

```
Connected
    │
    ├─→ Ping cada 10s
    │   └─→ Si timeout → Reconnect
    │
    ├─→ Check data usage
    │   └─→ Si > limit → Disconnect
    │
    ├─→ Check expiry
    │   └─→ Si expired → Disconnect
    │
    └─→ Update stats UI
```

---

## 🔌 Reconexión Automática

```
Connection Lost
    │
    ├─→ Intent 1 (inmediato)
    │   └─→ Si falla → wait 5s
    │
    ├─→ Intent 2
    │   └─→ Si falla → wait 30s
    │
    ├─→ Intent 3
    │   └─→ Si falla → notify user
    │
    └─→ Give up after 5 intents
```

---

**Última actualización:** 2026-03-27
