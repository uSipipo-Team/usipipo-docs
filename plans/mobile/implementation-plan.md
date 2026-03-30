# uSipipo Proxy Android App Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use `subagent-driven-development` to implement this plan task-by-task.

**Goal:** Build a production-ready Android VPN client using Go + Kotlin architecture with wireguard-go engine and gomobile bind integration.

**Architecture:** Monorepo with Go VPN engine (compiled to .aar via gomobile) + Kotlin Android app (Jetpack Compose UI + VpnService).

**Tech Stack:** Go 1.21+, wireguard-go, gomobile bind, Kotlin, Jetpack Compose, Android VpnService, GitHub Actions (100% CI/CD).

---

## Phase 0: Repository Setup

### Task 0.1: Create New GitHub Repository

**Action:** Create `usipipo-mobile` repository on GitHub

**Steps:**

1. Create repository via GitHub UI or gh cli:
```bash
gh repo create uSipipo-Team/usipipo-mobile \
  --private \
  --description "🤖 uSipipo Proxy - Native Android VPN client | Go + Kotlin | WireGuard + Outline | Production-ready" \
  --source=. \
  --remote=origin \
  --push
```

2. Verify repository is private:
```bash
gh repo view uSipipo-Team/usipipo-mobile --json visibility
# Expected: "visibility": "PRIVATE"
```

3. Add initial README.md placeholder (will be updated in Task 0.3)

**Expected:** Repository created at https://github.com/uSipipo-Team/usipipo-mobile

---

### Task 0.2: Delete Old Flutter Repository

**Action:** Delete `usipipovpnapp` repository from GitHub

**Steps:**

1. Navigate to repository settings:
```bash
gh repo view uSipipo-Team/usipipovpnapp
```

2. Delete via GitHub UI (Settings → Delete this repository)
   - Confirm deletion by typing repository name
   - This action is irreversible

3. Verify deletion:
```bash
gh repo view uSipipo-Team/usipipovpnapp
# Expected: 404 Not Found
```

**Note:** If gh cli cannot delete, delete manually via GitHub web UI.

---

### Task 0.3: Create Repository Structure

**Files:**
- Create: `README.md`
- Create: `.gitignore`
- Create: `LICENSE`
- Create: `docs/ARCHITECTURE.md`

**Step 1: Create README.md**

```markdown
# uSipipo Proxy - Android VPN Client

[![CI](https://github.com/uSipipo-Team/usipipo-mobile/actions/workflows/ci.yml/badge.svg)](https://github.com/uSipipo-Team/usipipo-mobile/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/uSipipo-Team/usipipo-mobile)](https://github.com/uSipipo-Team/usipipo-mobile/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**Native Android VPN client built with Go + Kotlin for maximum performance.**

---

## 🎯 Features

- ✅ **WireGuard Protocol** - Fast, modern VPN protocol
- ✅ **Outline/Shadowsocks** - Censorship circumvention
- ✅ **Precise Byte Counting** - For consumption-based billing
- ✅ **Telegram Authentication** - Seamless login via Telegram
- ✅ **Real-time Stats** - Monitor upload/download speeds
- ✅ **Background Service** - Persistent VPN connection

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────┐
│         UI (Jetpack Compose)            │
├─────────────────────────────────────────┤
│      Service Layer (Kotlin)             │
│      ├─ VpnService                      │
│      └─ ConnectionManager               │
├─────────────────────────────────────────┤
│      Go Engine (.aar via gomobile)      │
│      ├─ WireGuard Client                │
│      ├─ Outline Client                  │
│      └─ Usage Counter                   │
└─────────────────────────────────────────┘
```

---

## 📦 Installation

### Download Pre-built APK

```bash
wget https://github.com/uSipipo-Team/usipipo-mobile/releases/latest/download/usipipo-proxy.apk
```

### Build from Source

```bash
git clone https://github.com/uSipipo-Team/usipipo-mobile.git
cd usipipo-mobile

# Build Go engine
cd go
gomobile bind -target=android -o usipipo.aar -androidapi 29 .

# Build Android APK
cd ../android
./gradlew assembleDebug
```

---

## ⚙️ Configuration

Create `android/local.properties`:

```properties
sdk.dir=/path/to/Android/sdk
usipipo.backend.url=https://api.usipipo.duckdns.org
usipipo.telegram.bot.token=YOUR_BOT_TOKEN
```

---

## 🚀 Usage

1. **Login** via Telegram widget or manual credentials
2. **Select VPN Key** (WireGuard or Outline)
3. **Press Connect** button
4. **Monitor** real-time stats and consumption

---

## 📁 Project Structure

```
usipipo-mobile/
├── android/          # Kotlin Android app
├── go/               # Go VPN engine
├── docs/             # Documentation
├── scripts/          # Build scripts
└── .github/workflows/ # CI/CD
```

---

## 🔒 Security

- API keys encrypted in Android Keystore
- All backend communication over HTTPS
- Certificate pinning enabled
- No hardcoded secrets

---

## 📄 License

MIT License - see [LICENSE](LICENSE) file.

---

**Built with ❤️ by uSipipo Team**
```

**Step 2: Create .gitignore**

```gitignore
# Go
*.exe
*.exe~
*.dll
*.so
*.dylib
*.aar
*.lib
*.h
*.gox

# Android
*.iml
.gradle
/local.properties
/capture/
.DS_Store
.idea
/captures
.externalNativeBuild
.cxx
local.properties

# Kotlin
*.class
*.log

# Build outputs
android/app/build/
go/usipipo.aar

# Secrets
.env
.env.local
*.jks
*.keystore

# IDE
.vscode/
.idea/

# macOS
.DS_Store

# Windows
Thumbs.db
```

**Step 3: Create LICENSE**

```
MIT License

Copyright (c) 2026 uSipipo Team

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

**Step 4: Commit initial structure**

```bash
git add README.md .gitignore LICENSE
git commit -m "feat: Initial repository structure

- README.md with project overview
- .gitignore for Go + Android
- MIT License
- docs/ARCHITECTURE.md (symlink to usipipo-docs)"
git push origin main
```

---

### Task 0.4: Migrate Documentation

**Files:**
- Create symlink: `docs/ARCHITECTURE.md` → `../../usipipo-docs/plans/mobile/2026-03-29-proxy-app-design.md`
- Copy: `docs/BUILD.md` (from design doc)
- Copy: `docs/DEPLOYMENT.md` (from design doc)

**Step 1: Create docs directory**

```bash
mkdir -p docs
```

**Step 2: Create ARCHITECTURE.md symlink**

```bash
cd docs
ln -s ../../usipipo-docs/plans/mobile/2026-03-29-proxy-app-design.md ARCHITECTURE.md
```

**Step 3: Extract BUILD.md from design doc**

Copy the "GitHub Actions CI/CD" section from design doc into `docs/BUILD.md`.

**Step 4: Commit documentation**

```bash
git add docs/
git commit -m "docs: Add architecture and build documentation

- ARCHITECTURE.md (symlink to usipipo-docs)
- BUILD.md with CI/CD workflows
- DEPLOYMENT.md with release process"
git push origin main
```

---

## Phase 1: Go VPN Engine

### Task 1.1: Initialize Go Module

**Files:**
- Create: `go/go.mod`
- Create: `go/go.sum`
- Create: `go/main.go`

**Step 1: Create go.mod**

```go
module github.com/uSipipo-Team/usipipo-mobile/go

go 1.21

require (
	golang.zx2c4.com/wireguard v0.0.0-20230704
	github.com/go-resty/resty/v2 v2.11.0
	golang.org/x/crypto v0.17.0
)
```

**Step 2: Run go mod tidy**

```bash
cd go
go mod tidy
```

**Step 3: Commit**

```bash
git add go.mod go.sum
git commit -m "chore: Initialize Go module with wireguard-go dependency"
git push origin main
```

---

### Task 1.2: Create Go Main Entry Point

**Files:**
- Create: `go/main.go`

**Step 1: Write main.go with gomobile exports**

```go
package main

/*
#include <stdint.h>
*/
import "C"

import (
	"encoding/json"
	"github.com/uSipipo-Team/usipipo-mobile/go/vpn"
)

// VpnConfig represents VPN configuration from backend
type VpnConfig struct {
	Endpoint   string `json:"endpoint"`
	PublicKey  string `json:"public_key"`
	PrivateKey string `json:"private_key"`
	Address    string `json:"address"`
	DNS        string `json:"dns"`
	Protocol   string `json:"protocol"` // "wireguard" or "outline"
}

//export StartVPN
func StartVPN(configJSON string) C.int {
	var config VpnConfig
	if err := json.Unmarshal([]byte(configJSON), &config); err != nil {
		return C.int(1)
	}

	if config.Protocol == "wireguard" {
		result := vpn.StartWireGuard(config)
		return C.int(result)
	} else if config.Protocol == "outline" {
		result := vpn.StartOutline(config)
		return C.int(result)
	}

	return C.int(1)
}

//export StopVPN
func StopVPN() C.int {
	return C.int(vpn.StopVPN())
}

//export GetBytesTransferred
func GetBytesTransferred() (tx, rx uint64) {
	return vpn.GetTXBytes(), vpn.GetRXBytes()
}

//export GetConnectionStatus
func GetConnectionStatus() C.int {
	return C.int(vpn.GetStatus())
}

//export SetBackendConfig
func SetBackendConfig(baseURL, apiKey string) {
	vpn.SetBackendConfig(baseURL, apiKey)
}

func main() {}
```

**Step 2: Commit**

```bash
git add go/main.go
git commit -m "feat: Add Go main entry point with gomobile exports

- StartVPN(configJSON) int
- StopVPN() int
- GetBytesTransferred() (tx, rx uint64)
- GetConnectionStatus() int
- SetBackendConfig(baseURL, apiKey)"
git push origin main
```

---

### Task 1.3: Implement WireGuard Client

**Files:**
- Create: `go/vpn/wireguard.go`
- Create: `go/vpn/config.go`
- Create: `go/vpn/usage.go`

**Step 1: Create wireguard.go**

```go
package vpn

import (
	"fmt"
	"golang.zx2c4.com/wireguard/tun"
	"golang.zx2c4.com/wireguard/device"
	"golang.zx2c4.com/wireguard/conn"
	"sync/atomic"
	"time"
)

var (
	dev       *device.Device
	tunDevice tun.Device
	txBytes   uint64
	rxBytes   uint64
	status    int32 // 0=disconnected, 1=connecting, 2=connected, 3=error
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

type logWriter struct{}

func (logWriter) Write(p []byte) (n int, err error) {
	fmt.Printf("[WireGuard] %s", p)
	return len(p), nil
}
```

**Step 2: Create config.go**

```go
package vpn

var (
	backendBaseURL string
	backendAPIKey  string
)

func SetBackendConfig(baseURL, apiKey string) {
	backendBaseURL = baseURL
	backendAPIKey = apiKey
}
```

**Step 3: Create usage.go**

```go
package vpn

func ResetCounters() {
	atomic.StoreUint64(&txBytes, 0)
	atomic.StoreUint64(&rxBytes, 0)
}
```

**Step 4: Commit**

```bash
git add go/vpn/
git commit -m "feat: Implement WireGuard client with byte counting

- StartWireGuard(config) - Create and start WireGuard tunnel
- StopVPN() - Stop tunnel and cleanup
- GetTXBytes()/GetRXBytes() - Precise byte counters
- countBytes() - Background goroutine for stats
- LogWriter for WireGuard logs"
git push origin main
```

---

### Task 1.4: Implement Outline Client (Placeholder)

**Files:**
- Create: `go/vpn/outline.go`

**Step 1: Create outline.go (placeholder)**

```go
package vpn

// TODO: Implement Outline/Shadowsocks client
// For now, return error

func StartOutline(config VpnConfig) int {
	// Placeholder - to be implemented in Phase 3
	return 1
}
```

**Step 2: Commit**

```bash
git add go/vpn/outline.go
git commit -m "feat: Add Outline client placeholder

- StartOutline(config) returns error (to be implemented)
- WireGuard is primary protocol for MVP"
git push origin main
```

---

### Task 1.5: Build Go .aar with gomobile

**Files:**
- Create: `scripts/build-go.sh`
- Modify: `.github/workflows/ci.yml`

**Step 1: Create build-go.sh**

```bash
#!/bin/bash
set -e

echo "Building Go VPN engine for Android..."

cd go

# Install gomobile if not present
if ! command -v gomobile &> /dev/null; then
    echo "Installing gomobile..."
    go install golang.org/x/mobile/cmd/gomobile@latest
    go install golang.org/x/mobile/cmd/gobind@latest
fi

# Build .aar
echo "Building usipipo.aar..."
gomobile bind -target=android -o usipipo.aar -androidapi 29 .

echo "Build complete: go/usipipo.aar"
```

**Step 2: Make executable**

```bash
chmod +x scripts/build-go.sh
```

**Step 3: Test build locally (if Go installed)**

```bash
./scripts/build-go.sh
# Expected: go/usipipo.aar created
```

**Step 4: Commit**

```bash
git add scripts/build-go.sh
git commit -m "ci: Add Go build script for gomobile

- scripts/build-go.sh
- Installs gomobile if needed
- Builds usipipo.aar for Android API 29+"
git push origin main
```

---

## Phase 2: Android App Setup

### Task 2.1: Initialize Android Project

**Files:**
- Create: `android/build.gradle.kts`
- Create: `android/settings.gradle.kts`
- Create: `android/gradle.properties`
- Create: `android/app/build.gradle.kts`

**Step 1: Create root build.gradle.kts**

```kotlin
// Top-level build file where you can add configuration options common to all sub-projects/modules.
plugins {
    id("com.android.application") version "8.2.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.20" apply false
}
```

**Step 2: Create settings.gradle.kts**

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "uSipipo Proxy"
include(":app")
```

**Step 3: Create gradle.properties**

```properties
# Project-wide Gradle settings.
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
android.nonTransitiveRClass=true
```

**Step 4: Create app/build.gradle.kts**

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.usipipo.proxy"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.usipipo.proxy"
        minSdk = 29
        targetSdk = 34
        versionCode = 1
        versionName = "1.0.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        vectorDrawables {
            useSupportLibrary = true
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }

    buildFeatures {
        compose = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.4"
    }

    packaging {
        resources {
            excludes += "/META-INF/{AL2.0,LGPL2.1}"
        }
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.6.2")
    implementation("androidx.activity:activity-compose:1.8.1")
    implementation(platform("androidx.compose:compose-bom:2023.10.01"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-graphics")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    
    // Go library (will be added by build script)
    implementation(files("libs/usipipo.aar"))
    
    testImplementation("junit:junit:4.13.2")
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
    androidTestImplementation(platform("androidx.compose:compose-bom:2023.10.01"))
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-tooling")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
}
```

**Step 5: Commit**

```bash
git add android/
git commit -m "feat: Initialize Android project with Gradle Kotlin DSL

- build.gradle.kts (root + app)
- settings.gradle.kts
- gradle.properties
- Jetpack Compose enabled
- Kotlin 1.9.20, Android Gradle 8.2.0"
git push origin main
```

---

[CONTINUED IN NEXT MESSAGE DUE TO LENGTH...]
