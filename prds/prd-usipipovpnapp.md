# 📋 PRD - uSipipo VPN Android App

> Product Requirements Document de la Aplicación Android

---

## 📊 Información del Producto

| Campo | Valor |
|-------|-------|
| **Nombre** | uSipipo VPN Android |
| **Repositorio** | https://github.com/uSipipo-Team/usipipovpnapp |
| **Estado** | 🟡 Desarrollo (Fase 2/8) |
| **Versión** | v1.0.0+1 |
| **Tech Stack** | Flutter 3.16, Dart 3.11 |

---

## 🎯 Propósito

Cliente VPN nativo Android que permite conectar a servidores WireGuard y Outline con autenticación vía Telegram.

---

## 📱 Features Principales

### **1. Autenticación Híbrida**

**Flujo Principal (Telegram Widget):**
```
App → Telegram Login Widget → Telegram App
                              ↓
                      Autoriza login
                              ↓
App ← Deep link con credentials
```

**Flujo Fallback (Manual):**
```
App → Ingresar username + verification code
      ↓
Backend → Valida y retorna JWT
```

---

### **2. Conexión VPN**

**Protocolos Soportados:**
- ✅ WireGuard (wg-quick)
- ✅ Outline (Shadowsocks)

**Estados:**
- Desconectado
- Conectando
- Conectado
- Error

**UI:**
- Botón grande de connect/disconnect
- Selector de protocolo
- Selector de VPN key

---

### **3. Dashboard**

**Métricas en Tiempo Real:**
- Upload speed (Mbps)
- Download speed (Mbps)
- Data usada (MB/GB)
- Tiempo de sesión
- VPN key activa

**UI:**
```
┌─────────────────────────┐
│     🟢 CONECTADO        │
│                         │
│     ⬆️ 50 Mbps          │
│     ⬇️ 120 Mbps         │
│                         │
│     📊 2.5 GB usados    │
│     ⏱️ 1h 23m           │
│                         │
│   [Home VPN - WireGuard]│
│                         │
│      [DESCONECTAR]      │
└─────────────────────────┘
```

---

### **4. Gestión de VPN Keys**

**Features:**
- Listar keys del usuario
- Ver detalles (nombre, protocolo, data)
- Crear nueva key
- Eliminar key
- Copiar config al portapapeles

---

### **5. Notificaciones**

**Tipos:**
- Conexión exitosa/fallida
- Key por expirar (3 días)
- Límite de datos alcanzado
- Pago confirmado

**Implementación:**
- Firebase Cloud Messaging
- Notificaciones locales
- Foreground service

---

## 🏗️ Arquitectura

**Patrón:** Clean Architecture + Riverpod

```
lib/
├── core/
│   ├── constants/
│   └── theme/
├── domain/
│   └── entities/
│       ├── user.dart
│       ├── vpn_key.dart
│       ├── connection_status.dart
│       └── traffic_stats.dart
├── data/
│   ├── providers/
│   ├── repositories/
│   └── datasources/
└── presentation/
    ├── providers/
    ├── screens/
    └── widgets/
```

---

## 🔐 Permisos Android

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
```

---

## 🎨 Diseño

**Tema:** Cyberpunk Material Design 3

**Colores:**
- Primary: `#00F0FF`
- Secondary: `#FF00AA`
- Background: `#0A0A0F`

**Typography:**
- Headings: JetBrains Mono
- Body: Inter

---

## 📦 Dependencias Clave

```yaml
dependencies:
  flutter_riverpod: ^2.4.0
  go_router: ^12.0.0
  dio: ^5.3.0
  flutter_secure_storage: ^9.0.0
  firebase_messaging: ^14.7.0
  flutter_local_notifications: ^16.0.0
  connectivity_plus: ^5.0.0
```

---

## 🚀 Roadmap

### **Fase 1: Setup (✅ Completado)**
- [x] Project structure
- [x] Theme system
- [x] Domain entities

### **Fase 2: Data Layer (🟡 En progreso)**
- [ ] API client
- [ ] Repositories
- [ ] Local storage

### **Fase 3: Auth Flow**
- [ ] Telegram login
- [ ] Manual auth
- [ ] Token management

### **Fase 4: VPN Integration**
- [ ] WireGuard platform channel
- [ ] Outline platform channel
- [ ] Foreground service

### **Fase 5: UI Screens**
- [ ] Login screen
- [ ] Home screen
- [ ] Keys screen
- [ ] Settings screen

### **Fase 6: Testing**
- [ ] Unit tests
- [ ] Widget tests
- [ ] Integration tests

### **Fase 7: CI/CD**
- [ ] GitHub Actions
- [ ] Firebase App Distribution
- [ ] Play Store deployment

### **Fase 8: Release**
- [ ] Beta testing
- [ ] Production release
- [ ] Marketing

---

**Última actualización:** 2026-03-27
