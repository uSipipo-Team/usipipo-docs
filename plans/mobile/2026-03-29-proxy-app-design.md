# uSipipo Proxy Android App - Architecture Design

**Date:** 2026-03-29
**Status:** Approved ✅
**Repository:** `usipipo-mobile` (NEW)
**Replaces:** `usipipovpnapp` (Flutter → Native Android)

---

## 🎯 Objective

Build a production-ready Android VPN client using **Go + Kotlin** architecture for maximum performance, precise byte counting (consumption billing), and full control over Android's VpnService API.

**App Name:** uSipipo Proxy
**Package:** `com.usipipo.proxy`
**Target:** Android 10+ (API 29+)

---

## 🏗️ Architecture Overview

### **Technology Stack**

| Layer | Technology | Purpose |
|-------|------------|---------|
| **VPN Engine** | Go 1.21+ + wireguard-go | WireGuard tunnel, byte counting, encryption |
| **Integration** | gomobile bind | Go → Kotlin bindings (.aar library) |
| **VPN Service** | Kotlin + Android VpnService | Background VPN service, tunnel management |
| **UI** | Kotlin + Jetpack Compose | Modern declarative UI |
| **State Management** | Kotlin Flow + StateFlow | Reactive state management |
| **Navigation** | Jetpack Navigation | Type-safe navigation |
| **Build System** | Gradle Kotlin DSL + GitHub Actions | 100% CI/CD builds |

### **Architecture Diagram**

```
┌─────────────────────────────────────────────────────────┐
│                  uSipipo Proxy (Android)                │
├─────────────────────────────────────────────────────────┤
│  UI Layer (Kotlin + Jetpack Compose)                    │
│  ├─ MainActivity                                        │
│  ├─ VpnStatusScreen                                     │
│  ├─ ServerSelector                                      │
│  └─ ConsumptionDashboard                                │
├─────────────────────────────────────────────────────────┤
│  Service Layer (Kotlin)                                 │
│  ├─ VpnService (Android VpnService)                     │
│  ├─ ConnectionManager                                   │
│  └─ UsageMonitor                                        │
├─────────────────────────────────────────────────────────┤
│  Go Engine (gomobile bind → .aar)                       │
│  ├─ WireGuardClient (wireguard-go)                      │
│  ├─ OutlineClient (Shadowsocks)                         │
│  ├─ UsageCounter (bytes TX/RX)                          │
│  └─ APIClient (backend API calls)                       │
└─────────────────────────────────────────────────────────┘
```

---

## 📁 Repository Structure

```
usipipo-mobile/
├── android/
│   ├── app/
│   │   ├── src/main/
│   │   │   ├── java/com/usipipo/proxy/
│   │   │   │   ├── MainActivity.kt
│   │   │   │   ├── MainApplication.kt
│   │   │   │   ├── ui/                    # Jetpack Compose UI
│   │   │   │   │   ├── theme/
│   │   │   │   │   ├── components/
│   │   │   │   │   ├── screens/
│   │   │   │   │   └── navigation/
│   │   │   │   ├── service/               # VpnService
│   │   │   │   │   ├── VpnService.kt
│   │   │   │   │   ├── VpnConfig.kt
│   │   │   │   │   └── ConnectionManager.kt
│   │   │   │   ├── data/                  # API clients
│   │   │   │   │   ├── ApiClient.kt
│   │   │   │   │   └── models/
│   │   │   │   └── util/                  # Utilities
│   │   │   │       └── Preferences.kt
│   │   │   ├── res/
│   │   │   └── AndroidManifest.xml
│   │   └── build.gradle.kts
│   ├── build.gradle.kts
│   └── gradle.properties
│
├── go/
│   ├── vpn/
│   │   ├── wireguard.go          # WireGuard tunnel implementation
│   │   ├── outline.go            # Outline/Shadowsocks client
│   │   ├── usage.go              # Byte counter (TX/RX)
│   │   ├── api.go                # Backend API client
│   │   └── config.go             # VPN configuration parsing
│   ├── main.go                   # gomobile entry point
│   ├── go.mod
│   └── go.sum
│
├── scripts/
│   ├── build-go.sh               # gomobile bind → .aar
│   ├── build-apk.sh              # Full APK build
│   └── sign-release.sh           # Sign release APK
│
├── docs/
│   ├── ARCHITECTURE.md
│   ├── BUILD.md
│   ├── DEPLOYMENT.md
│   └── GOMOBILE_SETUP.md
│
├── .github/workflows/
│   ├── ci.yml                    # PR checks (lint + build)
│   ├── build-debug.yml           # Debug APK on PR merge
│   └── release.yml               # Release APK on tag
│
├── .gitignore
├── README.md
└── BUILD.bazel (optional)
```

---

## 🔧 Go VPN Engine

### **Dependencies**

```go
module github.com/uSipipo-Team/usipipo-mobile/go

go 1.21

require (
    golang.zx2c4.com/wireguard v0.0.0-20230704
    github.com/go-resty/resty/v2 v2.11.0
    golang.org/x/crypto v0.17.0
)
```

### **Exported Functions (via gomobile bind)**

```go
// main.go - gomobile entry point
package main

import (
    "encoding/json"
    "github.com/uSipipo-Team/usipipo-mobile/go/vpn"
)

// VpnConfig represents VPN configuration from backend
type VpnConfig struct {
    Endpoint    string `json:"endpoint"`
    PublicKey   string `json:"public_key"`
    PrivateKey  string `json:"private_key"`
    Address     string `json:"address"`
    DNS         string `json:"dns"`
    Protocol    string `json:"protocol"` // "wireguard" or "outline"
}

//export StartVPN
func StartVPN(configJSON string) int {
    var config VpnConfig
    if err := json.Unmarshal([]byte(configJSON), &config); err != nil {
        return 1 // Error parsing config
    }
    
    if config.Protocol == "wireguard" {
        return vpn.StartWireGuard(config)
    } else if config.Protocol == "outline" {
        return vpn.StartOutline(config)
    }
    
    return 1 // Unknown protocol
}

//export StopVPN
func StopVPN() int {
    return vpn.StopVPN()
}

//export GetBytesTransferred
func GetBytesTransferred() (tx, rx uint64) {
    return vpn.GetTXBytes(), vpn.GetRXBytes()
}

//export GetConnectionStatus
func GetConnectionStatus() int {
    return vpn.GetStatus() // 0=disconnected, 1=connecting, 2=connected, 3=error
}

//export SetBackendConfig
func SetBackendConfig(baseURL, apiKey string) {
    vpn.SetBackendConfig(baseURL, apiKey)
}
```

### **WireGuard Implementation**

```go
// vpn/wireguard.go
package vpn

import (
    "golang.zx2c4.com/wireguard/tun"
    "golang.zx2c4.com/wireguard/device"
    "golang.zx2c4.com/wireguard/conn"
    "io"
    "net/netip"
    "sync/atomic"
)

var (
    dev        *device.Device
    tunDevice  tun.Device
    txBytes    uint64
    rxBytes    uint64
    status     int32 // 0=disconnected, 1=connecting, 2=connected
)

func StartWireGuard(config VpnConfig) int {
    atomic.StoreInt32(&status, 1) // connecting
    
    // Create TUN device
    var err error
    tunDevice, err = tun.CreateTUN("usipipo0", 1420)
    if err != nil {
        atomic.StoreInt32(&status, 3) // error
        return 1
    }
    
    // Create WireGuard device
    dev = device.NewDevice(
        tunDevice,
        conn.NewDefaultBind(),
        device.NewLogger(&logWriter{}, device.LogLevelVerbose),
    )
    
    // Configure WireGuard
    wgConfig := fmt.Sprintf(
        "[Interface]\nPrivateKey = %s\nAddress = %s\nDNS = %s\n\n"+
        "[Peer]\nPublicKey = %s\nEndpoint = %s\nAllowedIPs = 0.0.0.0/0\n",
        config.PrivateKey,
        config.Address,
        config.DNS,
        config.PublicKey,
        config.Endpoint,
    )
    
    if err := dev.IpcSet(wgConfig); err != nil {
        atomic.StoreInt32(&status, 3)
        return 1
    }
    
    if err := dev.Up(); err != nil {
        atomic.StoreInt32(&status, 3)
        return 1
    }
    
    // Start byte counter
    go countBytes()
    
    atomic.StoreInt32(&status, 2) // connected
    return 0
}

func StopVPN() int {
    if dev != nil {
        dev.Close()
        dev = nil
    }
    if tunDevice != nil {
        tunDevice.Close()
        tunDevice = nil
    }
    atomic.StoreInt32(&status, 0) // disconnected
    return 0
}

func countBytes() {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()
    
    for range ticker.C {
        if tunDevice == nil {
            break
        }
        
        stats, err := tunDevice.Stats()
        if err != nil {
            continue
        }
        
        atomic.StoreUint64(&txBytes, stats.TxBytes)
        atomic.StoreUint64(&rxBytes, stats.RxBytes)
    }
}

func GetTXBytes() uint64 {
    return atomic.LoadUint64(&txBytes)
}

func GetRXBytes() uint64 {
    return atomic.LoadUint64(&rxBytes)
}

func GetStatus() int {
    return int(atomic.LoadInt32(&status))
}
```

---

## 📱 Kotlin Android Layer

### **VpnService Implementation**

```kotlin
// service/VpnService.kt
package com.usipipo.proxy.service

import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.content.Intent
import android.net.VpnService
import android.os.Build
import android.os.IBinder
import androidx.core.app.NotificationCompat
import com.usipipo.proxy.MainActivity
import com.usipipo.proxy.R
import go.usipipo.Usipipo // Generated by gomobile
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

class VpnService : VpnService() {
    
    companion object {
        private const val NOTIFICATION_CHANNEL_ID = "vpn_service_channel"
        private const val NOTIFICATION_ID = 1001
    }
    
    private val serviceScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
    
    private val _connectionState = MutableStateFlow(ConnectionState.DISCONNECTED)
    val connectionState: StateFlow<ConnectionState> = _connectionState.asStateFlow()
    
    private val _bytesTransferred = MutableStateFlow(BytesTransferred(tx = 0u, rx = 0u))
    val bytesTransferred: StateFlow<BytesTransferred> = _bytesTransferred.asStateFlow()
    
    private var usageMonitorJob: Job? = null
    
    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val action = intent?.action
        
        when (action) {
            ACTION_CONNECT -> {
                val configJson = intent.getStringExtra(EXTRA_CONFIG_JSON)
                configJson?.let { connect(it) }
            }
            ACTION_DISCONNECT -> disconnect()
        }
        
        return START_STICKY
    }
    
    private fun connect(configJson: String) {
        _connectionState.value = ConnectionState.CONNECTING
        
        // Prepare VpnService.Builder
        val builder = Builder()
            .addAddress("10.0.0.2", 32)
            .addDnsServer("1.1.1.1")
            .addRoute("0.0.0.0", 0)
            .setSession("uSipipo Proxy")
            .setBlocking(false)
        
        // Establish VPN tunnel
        val vpnInterface = builder.establish()
        if (vpnInterface == null) {
            _connectionState.value = ConnectionState.ERROR
            stopSelf()
            return
        }
        
        // Start Go VPN engine
        val result = Usipipo.startVPN(configJson)
        
        if (result == 0) {
            _connectionState.value = ConnectionState.CONNECTED
            startForeground(NOTIFICATION_ID, createNotification())
            startUsageMonitor()
        } else {
            _connectionState.value = ConnectionState.ERROR
            stopSelf()
        }
    }
    
    private fun disconnect() {
        usageMonitorJob?.cancel()
        Usipipo.stopVPN()
        _connectionState.value = ConnectionState.DISCONNECTED
        stopForeground(STOP_FOREGROUND_REMOVE)
        stopSelf()
    }
    
    private fun startUsageMonitor() {
        usageMonitorJob = serviceScope.launch {
            while (isActive && _connectionState.value == ConnectionState.CONNECTED) {
                delay(1000) // Update every second
                
                val tx = Usipipo.getBytesTransferred().component1()
                val rx = Usipipo.getBytesTransferred().component2()
                
                _bytesTransferred.value = BytesTransferred(tx = tx, rx = rx)
                
                // Report to backend every 30 seconds
                if (System.currentTimeMillis() % 30000 < 1000) {
                    reportUsageToBackend(tx, rx)
                }
            }
        }
    }
    
    private fun reportUsageToBackend(tx: UInt, rx: UInt) {
        // TODO: Call backend API to report usage
        // This is critical for consumption billing
    }
    
    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                NOTIFICATION_CHANNEL_ID,
                "VPN Service",
                NotificationManager.IMPORTANCE_LOW
            ).apply {
                description = "VPN connection status"
            }
            
            val notificationManager = getSystemService(NotificationManager::class.java)
            notificationManager.createNotificationChannel(channel)
        }
    }
    
    private fun createNotification(): Notification {
        val pendingIntent = PendingIntent.getActivity(
            this,
            0,
            Intent(this, MainActivity::class.java),
            PendingIntent.FLAG_IMMUTABLE
        )
        
        return NotificationCompat.Builder(this, NOTIFICATION_CHANNEL_ID)
            .setContentTitle("uSipipo Proxy")
            .setContentText("VPN connected")
            .setSmallIcon(R.drawable.ic_vpn_notification)
            .setContentIntent(pendingIntent)
            .setOngoing(true)
            .build()
    }
    
    override fun onDestroy() {
        super.onDestroy()
        disconnect()
        serviceScope.cancel()
    }
    
    override fun onBind(intent: Intent?): IBinder? {
        return null
    }
    
    enum class ConnectionState {
        DISCONNECTED,
        CONNECTING,
        CONNECTED,
        ERROR
    }
    
    data class BytesTransferred(val tx: UInt, val rx: UInt)
    
    companion object {
        const val ACTION_CONNECT = "com.usipipo.proxy.CONNECT"
        const val ACTION_DISCONNECT = "com.usipipo.proxy.DISCONNECT"
        const val EXTRA_CONFIG_JSON = "config_json"
    }
}
```

---

## 🔄 GitHub Actions CI/CD

### **Workflow 1: CI (on PR)**

```yaml
name: CI

on:
  pull_request:
    branches: [ main ]

jobs:
  lint-go:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.21'
    - name: Lint Go
      run: |
        cd go
        go fmt ./...
        go vet ./...

  lint-kotlin:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Set up JDK
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
    - name: Lint Kotlin
      run: |
        cd android
        ./gradlew ktlintCheck

  build-go:
    runs-on: ubuntu-latest
    needs: [lint-go]
    steps:
    - uses: actions/checkout@v4
    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.21'
    - name: Install gomobile
      run: |
        go install golang.org/x/mobile/cmd/gomobile@latest
        go install golang.org/x/mobile/cmd/gobind@latest
    - name: Build Go .aar
      run: |
        cd go
        gomobile bind -target=android -o usipipo.aar -androidapi 29 .
    - name: Upload .aar artifact
      uses: actions/upload-artifact@v4
      with:
        name: usipipo-lib
        path: go/usipipo.aar

  build-apk:
    runs-on: ubuntu-latest
    needs: [lint-kotlin, build-go]
    steps:
    - uses: actions/checkout@v4
    - name: Set up JDK
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
    - name: Download .aar artifact
      uses: actions/download-artifact@v4
      with:
        name: usipipo-lib
        path: android/app/libs/
    - name: Build Debug APK
      run: |
        cd android
        ./gradlew assembleDebug
    - name: Upload APK artifact
      uses: actions/upload-artifact@v4
      with:
        name: debug-apk
        path: android/app/build/outputs/apk/debug/app-debug.apk
```

### **Workflow 2: Release (on tag)**

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build-release:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.21'
    - name: Set up JDK
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
    - name: Install gomobile
      run: |
        go install golang.org/x/mobile/cmd/gomobile@latest
        go install golang.org/x/mobile/cmd/gobind@latest
    - name: Build Go .aar (optimized)
      run: |
        cd go
        gomobile bind -target=android -o usipipo.aar -androidapi 29 -ldflags="-s -w" .
    - name: Copy .aar to Android project
      run: cp go/usipipo.aar android/app/libs/
    - name: Build Release APK
      run: |
        cd android
        ./gradlew assembleRelease
    - name: Sign APK
      env:
        KEYSTORE_B64: ${{ secrets.RELEASE_KEYSTORE_B64 }}
        KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
        KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
        KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
      run: |
        echo "$KEYSTORE_B64" | base64 -d > android/app/release-key.jks
        cd android
        # Sign APK (script implementation)
    - name: Create GitHub Release
      uses: softprops/action-gh-release@v1
      with:
        files: android/app/build/outputs/apk/release/app-release.apk
        generate_release_notes: true
```

---

## 🔐 Security Considerations

### **API Key Storage**
- Use Android Keystore for JWT tokens
- Never store API keys in SharedPreferences
- Use `EncryptedSharedPreferences` for sensitive config

### **Go Engine Security**
- Validate all input from Kotlin layer
- Use constant-time comparison for keys
- Zero out sensitive memory after use

### **Network Security**
- Enforce HTTPS for all backend calls
- Certificate pinning for backend API
- Use Android's Network Security Config

---

## 📊 Metrics & Monitoring

### **Byte Counting (Critical for Billing)**

```go
// vpn/usage.go
package vpn

import (
    "sync/atomic"
    "time"
)

var (
    txBytes uint64
    rxBytes uint64
)

func countBytes() {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()
    
    for range ticker.C {
        if tunDevice == nil {
            break
        }
        
        stats, err := tunDevice.Stats()
        if err != nil {
            continue
        }
        
        atomic.StoreUint64(&txBytes, stats.TxBytes)
        atomic.StoreUint64(&rxBytes, stats.RxBytes)
    }
}

func GetTXBytes() uint64 {
    return atomic.LoadUint64(&txBytes)
}

func GetRXBytes() uint64 {
    return atomic.LoadUint64(&rxBytes)
}

func ResetCounters() {
    atomic.StoreUint64(&txBytes, 0)
    atomic.StoreUint64(&rxBytes, 0)
}
```

### **Backend Reporting**

```kotlin
// service/UsageReporter.kt
class UsageReporter(
    private val apiClient: ApiClient,
    private val preferences: Preferences
) {
    suspend fun reportUsage(txBytes: UInt, rxBytes: UInt) {
        val userId = preferences.getUserId()
        val sessionId = preferences.getSessionId()
        
        apiClient.postUsage(
            userId = userId,
            sessionId = sessionId,
            txBytes = txBytes.toLong(),
            rxBytes = rxBytes.toLong(),
            timestamp = System.currentTimeMillis()
        )
    }
}
```

---

## 🧪 Testing Strategy

### **Go Tests**
```bash
cd go
go test ./vpn/... -v -cover
```

### **Kotlin Tests**
```bash
cd android
./gradlew testDebugUnitTest
./gradlew connectedAndroidTest
```

### **Integration Tests**
- Mock VpnService for UI tests
- Test Go ↔ Kotlin boundary
- End-to-end connection flow

---

## 📅 Implementation Phases

### **Phase 1: Foundation (Week 1-2)**
- [ ] Set up repository structure
- [ ] Configure gomobile build pipeline
- [ ] Create basic VpnService skeleton
- [ ] Implement Go WireGuard client

### **Phase 2: Core VPN (Week 3-4)**
- [ ] Integrate Go .aar with Kotlin
- [ ] Implement connection flow
- [ ] Add byte counting
- [ ] Backend API integration

### **Phase 3: UI (Week 5-6)**
- [ ] Jetpack Compose UI
- [ ] Connection status screen
- [ ] Server selector
- [ ] Consumption dashboard

### **Phase 4: Polish (Week 7-8)**
- [ ] Error handling
- [ ] Notifications
- [ ] Settings
- [ ] Performance optimization

### **Phase 5: Release (Week 9)**
- [ ] Beta testing
- [ ] Bug fixes
- [ ] Play Store listing
- [ ] Production release

---

## 🔗 Related Documentation

- **Backend API:** `/apis/backend-api-reference.md`
- **VPN Connection Flow:** `/flows/vpn-connection-flow.md`
- **Mobile App Flow:** `/flows/mobile-app-flow.md`
- **PRD (Old Flutter):** `/prds/prd-usipipovpnapp.md` (DEPRECATED)

---

## ✅ Approval

**Design approved by:** @mowgliph
**Date:** 2026-03-29
**Next Step:** Invoke `writing-plans` skill for implementation plan

---

**Last Updated:** 2026-03-29
**Status:** Ready for Implementation ✅
