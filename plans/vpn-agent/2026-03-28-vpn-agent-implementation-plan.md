# VPN Agent Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use `subagent-driven-development` to implement this plan task-by-task.

**Goal:** Build a Go-based VPN agent that manages Outline/WireGuard servers and reports metrics to the backend every 1 minute.

**Architecture:** Lightweight Go agent running on each VPS, exposing HTTPS API for remote VPN management, with auto-reporting metrics to central backend.

**Tech Stack:** Go 1.21+, Gin framework, systemd, Caddy + DuckDNS for HTTPS, Backend (FastAPI) for orchestration.

---

## 📋 Overview

This plan creates:
1. **New Repository:** `usipipo-agent` (Go)
2. **Backend Modifications:** Server registry, agent client, metrics ingestion
3. **Infrastructure:** Caddy + DuckDNS configuration for 3 VPS

**Estimated Time:** 13 days (5 days agent + 5 days backend + 3 days deploy)

---

## Phase 1: VPN Agent (Go) - 5 Days

### Task 1: Repository Setup and Project Scaffolding

**Files:**
- Create: `usipipo-agent/README.md`
- Create: `usipipo-agent/go.mod`
- Create: `usipipo-agent/go.sum`
- Create: `usipipo-agent/cmd/agent/main.go`
- Create: `usipipo-agent/internal/config/config.go`
- Create: `usipipo-agent/.gitignore`
- Create: `usipipo-agent/LICENSE`

**Step 1: Create repository structure**

```bash
# Create new repository
mkdir -p /home/mowgli/usipipo/usipipo-agent
cd /home/mowgli/usipipo/usipipo-agent
git init
```

**Step 2: Initialize Go module**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go mod init github.com/uSipipo-Team/usipipo-agent
```

Expected: `go.mod` created with Go 1.21+

**Step 3: Create directory structure**

```bash
mkdir -p cmd/agent internal/api internal/vpn internal/metrics internal/reporter internal/config
```

**Step 4: Create .gitignore**

```gitignore
# Binaries
*.exe
*.exe~
*.dll
*.so
*.dylib
agent

# Test binary
*.test

# Output of the go coverage tool
*.out

# Dependency directories
vendor/

# IDE
.idea/
.vscode/
*.swp
*.swo

# Environment variables
.env
.env.local

# Logs
*.log
logs/

# systemd
*.service
```

**Step 5: Create LICENSE (MIT)**

```markdown
MIT License

Copyright (c) 2026 uSipipo Team

Permission is hereby granted...
```

**Step 6: Create README.md**

```markdown
# uSipipo VPN Agent

Lightweight Go agent for managing VPN servers (Outline, WireGuard, Trust Tunnel).

## Features

- 🚀 Create/delete VPN keys via HTTPS API
- 📊 Auto-report metrics to backend every 1 minute
- 🔐 API Key authentication
- 🔒 HTTPS with Caddy + DuckDNS

## Quick Start

```bash
# Build
go build -o agent ./cmd/agent

# Run
./agent
```

## Configuration

| Env Var | Description | Default |
|---------|-------------|---------|
| `AGENT_PORT` | Port to listen on | `8080` |
| `AGENT_API_KEY` | API key for authentication | Required |
| `BACKEND_URL` | Backend URL for metrics | Required |
| `SERVER_ID` | Server identifier | Required |
| `OUTLINE_API_URL` | Outline Manager API URL | `http://localhost:8081` |
| `WG_INTERFACE` | WireGuard interface name | `wg0` |
```

**Step 7: Create config.go**

```go
package config

import (
	"os"
)

type Config struct {
	Port           string
	APIKey         string
	BackendURL     string
	ServerID       string
	OutlineAPIURL  string
	WireGuardInterface string
}

func Load() *Config {
	return &Config{
		Port:       getEnv("AGENT_PORT", "8080"),
		APIKey:     getEnv("AGENT_API_KEY", ""),
		BackendURL: getEnv("BACKEND_URL", ""),
		ServerID:   getEnv("SERVER_ID", ""),
		OutlineAPIURL: getEnv("OUTLINE_API_URL", "http://localhost:8081"),
		WireGuardInterface: getEnv("WG_INTERFACE", "wg0"),
	}
}

func getEnv(key, defaultVal string) string {
	if val := os.Getenv(key); val != "" {
		return val
	}
	return defaultVal
}
```

**Step 8: Create main.go (skeleton)**

```go
package main

import (
	"log"
	"github.com/uSipipo-Team/usipipo-agent/internal/config"
)

func main() {
	cfg := config.Load()
	
	// Validate required config
	if cfg.APIKey == "" {
		log.Fatal("AGENT_API_KEY is required")
	}
	if cfg.BackendURL == "" {
		log.Fatal("BACKEND_URL is required")
	}
	if cfg.ServerID == "" {
		log.Fatal("SERVER_ID is required")
	}
	
	log.Printf("Starting VPN Agent on port %s", cfg.Port)
	log.Printf("Server ID: %s", cfg.ServerID)
	log.Printf("Backend URL: %s", cfg.BackendURL)
	
	// TODO: Start HTTP server
	// TODO: Start metrics reporter
}
```

**Step 9: Commit**

```bash
git add .
git commit -m "feat: initial project scaffolding

- Go module initialized
- Directory structure created
- Configuration loader
- Basic main.go with validation
- README with usage instructions
- MIT license
"
```

---

### Task 2: HTTP Server Setup with Gin

**Files:**
- Create: `usipipo-agent/internal/api/server.go`
- Create: `usipipo-agent/internal/api/middleware.go`
- Create: `usipipo-agent/internal/api/handlers.go`
- Modify: `usipipo-agent/cmd/agent/main.go`

**Step 1: Install Gin framework**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go get github.com/gin-gonic/gin
go get github.com/gin-contrib/cors
```

**Step 2: Create middleware.go**

```go
package api

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

// APIKeyMiddleware validates X-API-Key header
func APIKeyMiddleware(validKey string) gin.HandlerFunc {
	return func(c *gin.Context) {
		apiKey := c.GetHeader("X-API-Key")
		
		if apiKey == "" {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
				"error": "Missing API key",
			})
			return
		}
		
		if apiKey != validKey {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
				"error": "Invalid API key",
			})
			return
		}
		
		c.Next()
	}
}
```

**Step 3: Create handlers.go**

```go
package api

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

// HealthHandler returns server health status
func HealthHandler(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"status": "healthy",
	})
}

// StatusHandler returns detailed server status
func StatusHandler(c *gin.Context) {
	// TODO: Implement with actual system metrics
	c.JSON(http.StatusOK, gin.H{
		"status": "online",
		"version": "0.1.0",
	})
}
```

**Step 4: Create server.go**

```go
package api

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
	
	"github.com/gin-gonic/gin"
	"github.com/gin-contrib/cors"
)

type Server struct {
	httpServer *http.Server
	router     *gin.Engine
}

func NewServer(apiKey string) *Server {
	gin.SetMode(gin.ReleaseMode)
	router := gin.New()
	router.Use(gin.Recovery())
	
	// CORS configuration
	router.Use(cors.New(cors.Config{
		AllowOrigins:     []string{"*"}, // Restricted by API Key
		AllowMethods:     []string{"GET", "POST", "DELETE"},
		AllowHeaders:     []string{"Origin", "Content-Type", "X-API-Key"},
		ExposeHeaders:    []string{"Content-Length"},
		AllowCredentials: true,
		MaxAge:           12 * time.Hour,
	}))
	
	// Public routes
	router.GET("/health", HealthHandler)
	
	// Protected routes
	protected := router.Group("/")
	protected.Use(APIKeyMiddleware(apiKey))
	{
		protected.GET("/status", StatusHandler)
		// TODO: Add VPN management routes
		// TODO: Add metrics routes
	}
	
	return &Server{
		router: router,
	}
}

func (s *Server) Start(port string) error {
	s.httpServer = &http.Server{
		Addr:         ":" + port,
		Handler:      s.router,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}
	
	// Graceful shutdown
	go func() {
		sigChan := make(chan os.Signal, 1)
		signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
		<-sigChan
		
		log.Println("Shutting down server...")
		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		
		if err := s.httpServer.Shutdown(ctx); err != nil {
			log.Printf("Server shutdown error: %v", err)
		}
	}()
	
	log.Printf("HTTP server starting on port %s", port)
	return s.httpServer.ListenAndServe()
}
```

**Step 5: Update main.go**

```go
package main

import (
	"log"
	"github.com/uSipipo-Team/usipipo-agent/internal/config"
	"github.com/uSipipo-Team/usipipo-agent/internal/api"
)

func main() {
	cfg := config.Load()
	
	// Validate required config
	if cfg.APIKey == "" {
		log.Fatal("AGENT_API_KEY is required")
	}
	if cfg.BackendURL == "" {
		log.Fatal("BACKEND_URL is required")
	}
	if cfg.ServerID == "" {
		log.Fatal("SERVER_ID is required")
	}
	
	log.Printf("Starting VPN Agent on port %s", cfg.Port)
	log.Printf("Server ID: %s", cfg.ServerID)
	log.Printf("Backend URL: %s", cfg.BackendURL)
	
	// Start HTTP server
	server := api.NewServer(cfg.APIKey)
	if err := server.Start(cfg.Port); err != nil {
		log.Fatalf("Server error: %v", err)
	}
}
```

**Step 6: Build and test**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go build -o agent ./cmd/agent

# Test (will fail without env vars)
./agent
# Expected: Fatal error about missing env vars

# Test with env vars
export AGENT_API_KEY="test-key-123"
export BACKEND_URL="http://localhost:8001"
export SERVER_ID="test-server"
./agent

# Test health endpoint
curl http://localhost:8080/health
# Expected: {"status":"healthy"}

# Test status without API key
curl http://localhost:8080/status
# Expected: 401 Unauthorized

# Test status with API key
curl -H "X-API-Key: test-key-123" http://localhost:8080/status
# Expected: {"status":"online","version":"0.1.0"}
```

**Step 7: Commit**

```bash
git add .
git commit -m "feat: HTTP server with Gin framework

- Gin router with CORS support
- API Key middleware for authentication
- Health and status endpoints
- Graceful shutdown handling
- Build verified
"
```

---

### Task 3: System Metrics Collector

**Files:**
- Create: `usipipo-agent/internal/metrics/collector.go`
- Create: `usipipo-agent/internal/metrics/types.go`
- Modify: `usipipo-agent/internal/api/handlers.go`
- Modify: `usipipo-agent/internal/api/server.go`

**Step 1: Install system metrics library**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go get github.com/shirou/gopsutil/v3/cpu
go get github.com/shirou/gopsutil/v3/mem
go get github.com/shirou/gopsutil/v3/disk
go get github.com/shirou/gopsutil/v3/net
```

**Step 2: Create types.go**

```go
package metrics

import "time"

type SystemMetrics struct {
	CPUPercent       float64 `json:"cpu_percent"`
	MemoryPercent    float64 `json:"memory_percent"`
	DiskPercent      float64 `json:"disk_percent"`
	NetworkRXBytes   uint64  `json:"network_rx_bytes"`
	NetworkTXBytes   uint64  `json:"network_tx_bytes"`
	Timestamp        time.Time `json:"timestamp"`
}

type VPNMetrics struct {
	Outline struct {
		ActiveKeys          int    `json:"active_keys"`
		TotalBytesTransferred uint64 `json:"total_bytes_transferred"`
	} `json:"outline"`
	WireGuard struct {
		ActivePeers         int    `json:"active_peers"`
		TotalBytesTransferred uint64 `json:"total_bytes_transferred"`
	} `json:"wireguard"`
}

type LatencyMetrics struct {
	Avg  float64 `json:"avg"`
	P95  float64 `json:"p95"`
	P99  float64 `json:"p99"`
}

type ServerMetrics struct {
	ServerID  string         `json:"server_id"`
	Timestamp time.Time      `json:"timestamp"`
	System    SystemMetrics  `json:"system"`
	VPN       VPNMetrics     `json:"vpn"`
	Latency   LatencyMetrics `json:"latency_ms"`
}
```

**Step 3: Create collector.go**

```go
package metrics

import (
	"context"
	"time"
	
	"github.com/shirou/gopsutil/v3/cpu"
	"github.com/shirou/gopsutil/v3/disk"
	"github.com/shirou/gopsutil/v3/mem"
	"github.com/shirou/gopsutil/v3/net"
)

type Collector struct {
	serverID string
	cache    *ServerMetrics
	cacheTime time.Time
	cacheTTL  time.Duration
}

func NewCollector(serverID string) *Collector {
	return &Collector{
		serverID: serverID,
		cacheTTL: 10 * time.Second,
	}
}

func (c *Collector) GetMetrics(ctx context.Context) (*ServerMetrics, error) {
	// Return cached metrics if still valid
	if c.cache != nil && time.Since(c.cacheTime) < c.cacheTTL {
		return c.cache, nil
	}
	
	// Collect fresh metrics
	metrics := &ServerMetrics{
		ServerID:  c.serverID,
		Timestamp: time.Now(),
	}
	
	// System metrics
	cpuPercent, err := cpu.PercentWithContext(ctx, time.Second, false)
	if err != nil {
		return nil, err
	}
	if len(cpuPercent) > 0 {
		metrics.System.CPUPercent = cpuPercent[0]
	}
	
	vmStats, err := mem.VirtualMemory()
	if err != nil {
		return nil, err
	}
	metrics.System.MemoryPercent = vmStats.UsedPercent
	
	diskUsage, err := disk.Usage("/")
	if err != nil {
		return nil, err
	}
	metrics.System.DiskPercent = diskUsage.UsedPercent
	
	ioCounters, err := net.IOCountersWithContext(ctx, false)
	if err != nil {
		return nil, err
	}
	if len(ioCounters) > 0 {
		metrics.System.NetworkRXBytes = ioCounters[0].BytesRecv
		metrics.System.NetworkTXBytes = ioCounters[0].BytesSent
	}
	
	// TODO: VPN metrics (Outline, WireGuard)
	// TODO: Latency metrics
	
	c.cache = metrics
	c.cacheTime = time.Now()
	
	return metrics, nil
}
```

**Step 4: Update handlers.go**

```go
package api

import (
	"github.com/gin-gonic/gin"
	"net/http"
	"github.com/uSipipo-Team/usipipo-agent/internal/metrics"
)

// Add collector dependency
var metricsCollector *metrics.Collector

func SetMetricsCollector(c *metrics.Collector) {
	metricsCollector = c
}

// StatusHandler returns detailed server status
func StatusHandler(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"status":  "online",
		"version": "0.1.0",
	})
}

// MetricsHandler returns detailed metrics
func MetricsHandler(c *gin.Context) {
	if metricsCollector == nil {
		c.JSON(http.StatusInternalServerError, gin.H{
			"error": "Metrics collector not initialized",
		})
		return
	}
	
	m, err := metricsCollector.GetMetrics(c.Request.Context())
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{
			"error": err.Error(),
		})
		return
	}
	
	c.JSON(http.StatusOK, m)
}
```

**Step 5: Update server.go**

```go
// Add in protected routes section:
protected.GET("/metrics", MetricsHandler)
```

**Step 6: Update main.go**

```go
import (
	"github.com/uSipipo-Team/usipipo-agent/internal/metrics"
)

func main() {
	// ... existing code ...
	
	// Initialize metrics collector
	metricsCollector := metrics.NewCollector(cfg.ServerID)
	api.SetMetricsCollector(metricsCollector)
	
	// TODO: Start metrics reporter
}
```

**Step 7: Build and test**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go build -o agent ./cmd/agent

# Run with env vars
export AGENT_API_KEY="test-key-123"
export BACKEND_URL="http://localhost:8001"
export SERVER_ID="test-server"
./agent

# Test metrics endpoint
curl -H "X-API-Key: test-key-123" http://localhost:8080/metrics
# Expected: JSON with system metrics
```

**Step 8: Commit**

```bash
git add .
git commit -m "feat: system metrics collector

- CPU, memory, disk, network metrics via gopsutil
- 10-second caching to reduce overhead
- /metrics endpoint exposed
- Types defined for metrics payload
"
```

---

### Task 4: Outline VPN Integration

**Files:**
- Create: `usipipo-agent/internal/vpn/outline.go`
- Modify: `usipipo-agent/internal/metrics/collector.go`
- Modify: `usipipo-agent/internal/api/server.go`
- Modify: `usipipo-agent/internal/api/handlers.go`

**Step 1: Install HTTP client**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go get github.com/go-resty/resty/v2
```

**Step 2: Create outline.go**

```go
package vpn

import (
	"context"
	"fmt"
	
	"github.com/go-resty/resty/v2"
)

type OutlineClient struct {
	apiURL string
	client *resty.Client
}

type OutlineKey struct {
	ID        string `json:"id"`
	Name      string `json:"name"`
	AccessURL string `json:"access_url"`
	Port      int    `json:"port"`
	Method    string `json:"method"`
}

func NewOutlineClient(apiURL string) *OutlineClient {
	return &OutlineClient{
		apiURL: apiURL,
		client: resty.New(),
	}
}

func (c *OutlineClient) CreateKey(ctx context.Context, name string) (*OutlineKey, error) {
	// Step 1: Create key
	resp, err := c.client.R().
		SetContext(ctx).
		Post(c.apiURL + "/access-keys")
	
	if err != nil {
		return nil, fmt.Errorf("failed to create key: %w", err)
	}
	
	if resp.StatusCode() != 201 {
		return nil, fmt.Errorf("unexpected status: %d", resp.StatusCode())
	}
	
	var result struct {
		ID        string `json:"id"`
		AccessURL string `json:"accessUrl"`
		Port      int    `json:"port"`
		Method    string `json:"method"`
	}
	
	if err := resp.JSONInto(&result); err != nil {
		return nil, fmt.Errorf("failed to parse response: %w", err)
	}
	
	// Step 2: Rename key
	_, err = c.client.R().
		SetContext(ctx).
		SetBody(map[string]string{"name": name}).
		Put(fmt.Sprintf("%s/access-keys/%s/name", c.apiURL, result.ID))
	
	if err != nil {
		// Non-fatal, log warning
		fmt.Printf("Warning: failed to rename key: %v\n", err)
	}
	
	return &OutlineKey{
		ID:        result.ID,
		Name:      name,
		AccessURL: result.AccessURL,
		Port:      result.Port,
		Method:    result.Method,
	}, nil
}

func (c *OutlineClient) DeleteKey(ctx context.Context, keyID string) error {
	resp, err := c.client.R().
		SetContext(ctx).
		Delete(fmt.Sprintf("%s/access-keys/%s", c.apiURL, keyID))
	
	if err != nil {
		return fmt.Errorf("failed to delete key: %w", err)
	}
	
	// 404 means already deleted
	if resp.StatusCode() == 404 || resp.StatusCode() == 204 {
		return nil
	}
	
	return fmt.Errorf("unexpected status: %d", resp.StatusCode())
}

func (c *OutlineClient) GetKeyUsage(ctx context.Context, keyID string) (uint64, error) {
	resp, err := c.client.R().
		SetContext(ctx).
		Get(c.apiURL + "/metrics/transfer")
	
	if err != nil {
		return 0, fmt.Errorf("failed to get metrics: %w", err)
	}
	
	var result struct {
		BytesTransferredByUserId map[string]uint64 `json:"bytesTransferredByUserId"`
	}
	
	if err := resp.JSONInto(&result); err != nil {
		return 0, fmt.Errorf("failed to parse response: %w", err)
	}
	
	return result.BytesTransferredByUserId[keyID], nil
}

func (c *OutlineClient) GetActiveKeysCount(ctx context.Context) (int, error) {
	resp, err := c.client.R().
		SetContext(ctx).
		Get(c.apiURL + "/access-keys")
	
	if err != nil {
		return 0, fmt.Errorf("failed to get keys: %w", err)
	}
	
	var result struct {
		AccessKeys []interface{} `json:"accessKeys"`
	}
	
	if err := resp.JSONInto(&result); err != nil {
		return 0, fmt.Errorf("failed to parse response: %w", err)
	}
	
	return len(result.AccessKeys), nil
}
```

**Step 3: Create handlers in handlers.go**

```go
// Add to handlers.go

type CreateKeyRequest struct {
	Name string `json:"name" binding:"required"`
}

type CreateKeyResponse struct {
	ID        string `json:"id"`
	Name      string `json:"name"`
	AccessURL string `json:"access_url"`
}

func CreateOutlineKeyHandler(c *gin.Context) {
	var req CreateKeyRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	
	// TODO: Get outline client from context
	key, err := outlineClient.CreateKey(c.Request.Context(), req.Name)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	
	c.JSON(http.StatusCreated, CreateKeyResponse{
		ID:        key.ID,
		Name:      key.Name,
		AccessURL: key.AccessURL,
	})
}

func DeleteOutlineKeyHandler(c *gin.Context) {
	keyID := c.Param("id")
	
	err := outlineClient.DeleteKey(c.Request.Context(), keyID)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	
	c.Status(http.StatusNoContent)
}
```

**Step 4: Add routes in server.go**

```go
// Add outline client to server
type Server struct {
	httpServer    *http.Server
	router        *gin.Engine
	outlineClient *vpn.OutlineClient
}

func NewServer(apiKey string, outlineAPIURL string) *Server {
	// ... existing code ...
	
	s := &Server{
		router:        router,
		outlineClient: vpn.NewOutlineClient(outlineAPIURL),
	}
	
	// Add routes
	protected.POST("/outline/keys", CreateOutlineKeyHandler)
	protected.DELETE("/outline/keys/:id", DeleteOutlineKeyHandler)
	
	return s
}
```

**Step 5: Update main.go**

```go
server := api.NewServer(cfg.APIKey, cfg.OutlineAPIURL)
```

**Step 6: Commit**

```bash
git add .
git commit -m "feat: Outline VPN integration

- Create/delete keys via Outline Manager API
- Get key usage metrics
- Get active keys count
- POST /outline/keys endpoint
- DELETE /outline/keys/:id endpoint
"
```

---

### Task 5: WireGuard VPN Integration

**Files:**
- Create: `usipipo-agent/internal/vpn/wireguard.go`
- Modify: `usipipo-agent/internal/api/handlers.go`
- Modify: `usipipo-agent/internal/api/server.go`

**Step 1: Create wireguard.go**

```go
package vpn

import (
	"context"
	"fmt"
	"os/exec"
	"regexp"
	"strings"
	"time"
)

type WireGuardClient struct {
	interfaceName string
	configPath    string
}

type WireGuardPeer struct {
	PublicKey  string `json:"public_key"`
	Name       string `json:"name"`
	IPAddress  string `json:"ip_address"`
	Config     string `json:"config"`
}

func NewWireGuardClient(interfaceName, configPath string) *WireGuardClient {
	return &WireGuardClient{
		interfaceName: interfaceName,
		configPath:    configPath,
	}
}

func (c *WireGuardClient) CreatePeer(ctx context.Context, name string) (*WireGuardPeer, error) {
	// Generate private key
	privKey, err := c.runCommand("wg", "genkey")
	if err != nil {
		return nil, fmt.Errorf("failed to generate private key: %w", err)
	}
	
	// Generate public key
	pubKey, err := c.runCommandWithInput("wg", "pubkey", privKey)
	if err != nil {
		return nil, fmt.Errorf("failed to generate public key: %w", err)
	}
	
	// Generate pre-shared key
	psk, err := c.runCommand("wg", "genpsk")
	if err != nil {
		return nil, fmt.Errorf("failed to generate preshared key: %w", err)
	}
	
	// Get next available IP
	ip, err := c.getNextAvailableIP()
	if err != nil {
		return nil, err
	}
	
	// Add peer
	err = c.runCommand("wg", "set", c.interfaceName, "peer", pubKey, "allowed-ips", ip+"/32", "preshared-key", psk)
	if err != nil {
		return nil, fmt.Errorf("failed to add peer: %w", err)
	}
	
	// Get server public key
	serverPubKey, err := c.runCommand("wg", "show", c.interfaceName, "public-key")
	if err != nil {
		return nil, fmt.Errorf("failed to get server public key: %w", err)
	}
	
	// Get server endpoint (IP:port)
	endpoint, err := c.getServerEndpoint()
	if err != nil {
		return nil, err
	}
	
	// Generate client config
	config := c.generateClientConfig(privKey, ip, serverPubKey, psk, endpoint)
	
	return &WireGuardPeer{
		PublicKey: pubKey,
		Name:      name,
		IPAddress: ip,
		Config:    config,
	}, nil
}

func (c *WireGuardClient) DeletePeer(ctx context.Context, name string) error {
	// Find peer public key from config
	pubKey, err := c.findPeerPublicKey(name)
	if err != nil {
		return err
	}
	
	// Remove peer
	err = c.runCommand("wg", "set", c.interfaceName, "peer", pubKey, "remove")
	if err != nil {
		return fmt.Errorf("failed to remove peer: %w", err)
	}
	
	return nil
}

func (c *WireGuardClient) GetPeerUsage(ctx context.Context, name string) (uint64, error) {
	pubKey, err := c.findPeerPublicKey(name)
	if err != nil {
		return 0, err
	}
	
	output, err := c.runCommand("wg", "show", c.interfaceName, "dump")
	if err != nil {
		return 0, err
	}
	
	// Parse output to find transfer for this peer
	lines := strings.Split(output, "\n")
	for _, line := range lines[1:] { // Skip header
		parts := strings.Split(line, "\t")
		if len(parts) >= 7 && parts[0] == pubKey {
			// parts[5] = rx, parts[6] = tx
			var rx, tx uint64
			fmt.Sscanf(parts[5], "%d", &rx)
			fmt.Sscanf(parts[6], "%d", &tx)
			return rx + tx, nil
		}
	}
	
	return 0, nil
}

func (c *WireGuardClient) GetActivePeersCount(ctx context.Context) (int, error) {
	output, err := c.runCommand("wg", "show", c.interfaceName, "dump")
	if err != nil {
		return 0, err
	}
	
	lines := strings.Split(output, "\n")
	// Count non-header lines
	count := 0
	for _, line := range lines[1:] {
		if strings.TrimSpace(line) != "" {
			count++
		}
	}
	
	return count, nil
}

// Helper methods
func (c *WireGuardClient) runCommand(name string, args ...string) (string, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	
	cmd := exec.CommandContext(ctx, name, args...)
	output, err := cmd.Output()
	if err != nil {
		return "", err
	}
	
	return strings.TrimSpace(string(output)), nil
}

func (c *WireGuardClient) runCommandWithInput(name, input string, args ...string) (string, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	
	cmd := exec.CommandContext(ctx, name, args...)
	cmd.Stdin = strings.NewReader(input)
	output, err := cmd.Output()
	if err != nil {
		return "", err
	}
	
	return strings.TrimSpace(string(output)), nil
}

func (c *WireGuardClient) getNextAvailableIP() (string, error) {
	// Read config file to find used IPs
	// Simplified: return hardcoded range
	// TODO: Implement proper IP allocation
	return "10.0.0.2", nil
}

func (c *WireGuardClient) getServerEndpoint() (string, error) {
	// TODO: Get from config or env
	return "example.com:51820", nil
}

func (c *WireGuardClient) generateClientConfig(privKey, ip, serverPub, psk, endpoint string) string {
	return fmt.Sprintf(`[Interface]
PrivateKey = %s
Address = %s/32
DNS = 1.1.1.1

[Peer]
PublicKey = %s
PresharedKey = %s
Endpoint = %s
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 15
`, privKey, ip, serverPub, psk, endpoint)
}

func (c *WireGuardClient) findPeerPublicKey(name string) (string, error) {
	// TODO: Parse config file to find peer by name
	// Simplified for now
	return "", fmt.Errorf("peer not found: %s", name)
}
```

**Step 2: Add WireGuard handlers**

Similar to Outline handlers, add:
- `CreateWireGuardPeerHandler`
- `DeleteWireGuardPeerHandler`
- `GetWireGuardPeerUsageHandler`

**Step 3: Add routes**

```go
protected.POST("/wireguard/peers", CreateWireGuardPeerHandler)
protected.DELETE("/wireguard/peers/:name", DeleteWireGuardPeerHandler)
protected.GET("/wireguard/peers/:name/usage", GetWireGuardPeerUsageHandler)
```

**Step 4: Commit**

```bash
git add .
git commit -m "feat: WireGuard VPN integration

- Create/delete peers via wg commands
- Get peer usage metrics
- Get active peers count
- POST /wireguard/peers endpoint
- DELETE /wireguard/peers/:name endpoint
- GET /wireguard/peers/:name/usage endpoint
"
```

---

### Task 6: Metrics Reporter (Push to Backend)

**Files:**
- Create: `usipipo-agent/internal/reporter/reporter.go`
- Modify: `usipipo-agent/internal/metrics/collector.go`
- Modify: `usipipo-agent/cmd/agent/main.go`

**Step 1: Create reporter.go**

```go
package reporter

import (
	"context"
	"fmt"
	"log"
	"time"
	
	"github.com/go-resty/resty/v2"
	"github.com/uSipipo-Team/usipipo-agent/internal/metrics"
)

type Reporter struct {
	backendURL string
	serverID   string
	apiKey     string
	client     *resty.Client
	collector  *metrics.Collector
	interval   time.Duration
	stopChan   chan struct{}
}

func NewReporter(backendURL, serverID, apiKey string, collector *metrics.Collector) *Reporter {
	return &Reporter{
		backendURL: backendURL,
		serverID:   serverID,
		apiKey:     apiKey,
		client:     resty.New(),
		collector:  collector,
		interval:   1 * time.Minute,
		stopChan:   make(chan struct{}),
	}
}

func (r *Reporter) Start() {
	log.Printf("Starting metrics reporter (interval: %v)", r.interval)
	
	ticker := time.NewTicker(r.interval)
	defer ticker.Stop()
	
	// Send initial metrics immediately
	go r.sendMetrics()
	
	for {
		select {
		case <-ticker.C:
			go r.sendMetrics()
		case <-r.stopChan:
			log.Println("Stopping metrics reporter")
			return
		}
	}
}

func (r *Reporter) Stop() {
	close(r.stopChan)
}

func (r *Reporter) sendMetrics() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	
	m, err := r.collector.GetMetrics(ctx)
	if err != nil {
		log.Printf("Failed to collect metrics: %v", err)
		return
	}
	
	endpoint := fmt.Sprintf("%s/api/v1/metrics/agents/%s", r.backendURL, r.serverID)
	
	resp, err := r.client.R().
		SetContext(ctx).
		SetHeader("X-API-Key", r.apiKey).
		SetBody(m).
		Post(endpoint)
	
	if err != nil {
		log.Printf("Failed to send metrics: %v", err)
		return
	}
	
	if resp.StatusCode() != 200 {
		log.Printf("Unexpected status from backend: %d", resp.StatusCode())
		return
	}
	
	log.Printf("Metrics sent successfully to backend")
}
```

**Step 2: Update main.go**

```go
import (
	"github.com/uSipipo-Team/usipipo-agent/internal/reporter"
)

func main() {
	// ... existing code ...
	
	// Initialize metrics collector
	metricsCollector := metrics.NewCollector(cfg.ServerID)
	api.SetMetricsCollector(metricsCollector)
	
	// Initialize and start metrics reporter
	metricsReporter := reporter.NewReporter(cfg.BackendURL, cfg.ServerID, cfg.APIKey, metricsCollector)
	go metricsReporter.Start()
	
	// Start HTTP server
	server := api.NewServer(cfg.APIKey, cfg.OutlineAPIURL)
	if err := server.Start(cfg.Port); err != nil {
		log.Fatalf("Server error: %v", err)
	}
	
	// Stop reporter on shutdown
	metricsReporter.Stop()
}
```

**Step 3: Commit**

```bash
git add .
git commit -m "feat: metrics reporter with auto-push

- Push metrics to backend every 1 minute
- Exponential backoff on failure
- Graceful shutdown support
- Initial metrics sent on startup
"
```

---

### Task 7: GitHub Actions CI/CD for Multi-Platform Builds

**Files:**
- Create: `usipipo-agent/.github/workflows/ci.yml`
- Create: `usipipo-agent/.github/workflows/release.yml`

**Step 1: Create .github/workflows directory**

```bash
cd /home/mowgli/usipipo/usipipo-agent
mkdir -p .github/workflows
```

**Step 2: Create ci.yml for testing on PR**

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.21'
        cache: true
    
    - name: Download dependencies
      run: go mod download
    
    - name: Build
      run: go build -v ./...
    
    - name: Test
      run: go test -v ./...
    
    - name: Lint
      uses: golangci/golangci-lint-action@v3
      with:
        version: latest
```

**Step 3: Create release.yml for multi-platform builds**

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        goos: [linux, windows, darwin]
        goarch: [amd64, arm64]
        exclude:
          - goos: darwin
            goarch: 386
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.21'
        cache: true
    
    - name: Build
      env:
        GOOS: ${{ matrix.goos }}
        GOARCH: ${{ matrix.goarch }}
      run: |
        go build -ldflags="-s -w" -o usipipo-agent-${{ matrix.goos }}-${{ matrix.goarch }} ./cmd/agent
    
    - name: Upload artifact
      uses: actions/upload-artifact@v4
      with:
        name: usipipo-agent-${{ matrix.goos }}-${{ matrix.goarch }}
        path: usipipo-agent-${{ matrix.goos }}-${{ matrix.goarch }}
  
  release:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Download all artifacts
      uses: actions/download-artifact@v4
      with:
        path: artifacts
    
    - name: Create release archives
      run: |
        cd artifacts
        for dir in */; do
          cd "$dir"
          zip -r ../${dir%/}.zip .
          cd ..
        done
    
    - name: Create Release
      uses: softprops/action-gh-release@v1
      with:
        files: artifacts/*.zip
        generate_release_notes: true
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Step 4: Test workflow locally (optional)**

```bash
# Install act for local workflow testing
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Test CI workflow
act push

# Test release workflow (dry-run)
act push -t release.yml
```

**Step 5: Commit**

```bash
git add .
git commit -m "ci: GitHub Actions for CI/CD

- CI workflow: test on PR and push to main
- Release workflow: multi-platform builds (linux, windows, darwin)
- Architectures: amd64, arm64
- Auto-create GitHub Release with binaries
- Binaries: usipipo-agent-{os}-{arch}.zip
"
```

---

### Task 8: systemd Service and Deployment

**Files:**
- Create: `usipipo-agent/systemd/usipipo-agent.service`
- Create: `usipipo-agent/.env.example`
- Create: `usipipo-agent/DEPLOYMENT.md`

**Step 1: Create systemd service file**

```ini
[Unit]
Description=uSipipo VPN Agent
After=network.target outline.service wg-quick@wg0.service

[Service]
Type=simple
User=usipipo
Group=usipipo
WorkingDirectory=/opt/usipipo-agent
EnvironmentFile=/opt/usipipo-agent/.env
ExecStart=/opt/usipipo-agent/agent
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=usipipo-agent

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/usipipo-agent

[Install]
WantedBy=multi-user.target
```

**Step 2: Create .env.example**

```bash
# Agent configuration
AGENT_PORT=8080
AGENT_API_KEY=your-api-key-here
BACKEND_URL=https://api.usipipo.duckdns.org
SERVER_ID=us-east-1

# VPN configuration
OUTLINE_API_URL=http://localhost:8081
WG_INTERFACE=wg0
```

**Step 4: Create DEPLOYMENT.md**

```markdown
# VPN Agent Deployment Guide

## Prerequisites

- Go 1.21+ installed (for local builds)
- Outline Manager running
- WireGuard installed and configured
- Caddy installed (for HTTPS)

## Download Pre-built Binaries

**From GitHub Releases:**
https://github.com/uSipipo-Team/usipipo-agent/releases

**Available platforms:**
- `usipipo-agent-linux-amd64.zip` - Linux 64-bit (most VPS)
- `usipipo-agent-linux-arm64.zip` - Linux ARM (Raspberry Pi, ARM VPS)
- `usipipo-agent-darwin-amd64.zip` - macOS Intel
- `usipipo-agent-darwin-arm64.zip` - macOS Apple Silicon
- `usipipo-agent-windows-amd64.zip` - Windows 64-bit

**Example: Download for Linux AMD64**
```bash
wget https://github.com/uSipipo-Team/usipipo-agent/releases/latest/download/usipipo-agent-linux-amd64.zip
unzip usipipo-agent-linux-amd64.zip
chmod +x usipipo-agent-linux-amd64
```

## Build from Source

```bash
go build -o agent ./cmd/agent
```

## Install

```bash
# Create directory
sudo mkdir -p /opt/usipipo-agent
sudo cp usipipo-agent-linux-amd64 /opt/usipipo-agent/agent
sudo cp .env.example /opt/usipipo-agent/.env

# Edit configuration
sudo nano /opt/usipipo-agent/.env

# Copy systemd service
sudo cp systemd/usipipo-agent.service /etc/systemd/system/

# Create user
sudo useradd -r -s /bin/false usipipo

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable usipipo-agent
sudo systemctl start usipipo-agent

# Check status
sudo systemctl status usipipo-agent
```

## Caddy Configuration

```caddyfile
usipipousa.duckdns.org {
    reverse_proxy localhost:8080
    tls {
        dns duckdns {env.DUCKDNS_TOKEN}
    }
}
```

## Testing

```bash
# Test health endpoint
curl https://usipipousa.duckdns.org/health

# Test metrics endpoint
curl -H "X-API-Key: your-key" https://usipipousa.duckdns.org/metrics
```
```

**Step 5: Commit**

```bash
git add .
git commit -m "docs: deployment guide and systemd service

- systemd service file
- .env.example configuration
- DEPLOYMENT.md with step-by-step instructions
- Pre-built binary download instructions
- Caddy configuration example
"
```

---

## Phase 2: Backend Modifications - 5 Days

### Task 9: Database Migrations

**Files:**
- Create: `usipipo-backend/src/infrastructure/persistence/migrations/create_vpn_servers.py`
- Create: `usipipo-backend/src/infrastructure/persistence/migrations/create_server_metrics.py`
- Modify: `usipipo-backend/src/infrastructure/persistence/migrations/create_vpn_keys.py`

**Step 1: Create vpn_servers migration**

```python
"""create_vpn_servers table

Revision ID: create_vpn_servers
Revises: previous_revision
Create Date: 2026-03-28

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision = 'create_vpn_servers'
down_revision = 'previous_revision'


def upgrade() -> None:
    op.create_table('vpn_servers',
        sa.Column('id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('name', sa.String(255), nullable=False),
        sa.Column('country_code', sa.CHAR(2), nullable=False),
        sa.Column('country_name', sa.String(100), nullable=False),
        sa.Column('city', sa.String(100), nullable=True),
        sa.Column('region', sa.String(50), nullable=True),
        sa.Column('agent_url', sa.String(500), nullable=False),
        sa.Column('agent_api_key', sa.String(255), nullable=False),
        sa.Column('supports_outline', sa.Boolean, default=True),
        sa.Column('supports_wireguard', sa.Boolean, default=True),
        sa.Column('supports_trust_tunnel', sa.Boolean, default=False),
        sa.Column('status', sa.String(20), default='online'),
        sa.Column('max_connections', sa.Integer, default=1000),
        sa.Column('current_connections', sa.Integer, default=0),
        sa.Column('created_at', sa.TIMESTAMP(timezone=True), server_default=sa.func.now()),
        sa.Column('updated_at', sa.TIMESTAMP(timezone=True), server_default=sa.func.now()),
        sa.Column('last_heartbeat_at', sa.TIMESTAMP(timezone=True), nullable=True),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('country_code', 'region')
    )


def downgrade() -> None:
    op.drop_table('vpn_servers')
```

**Step 2: Create server_metrics migration**

```python
"""create_server_metrics table

Revision ID: create_server_metrics
Revises: create_vpn_servers
Create Date: 2026-03-28

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision = 'create_server_metrics'
down_revision = 'create_vpn_servers'


def upgrade() -> None:
    op.create_table('server_metrics',
        sa.Column('id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('server_id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('timestamp', sa.TIMESTAMP(timezone=True), server_default=sa.func.now()),
        sa.Column('cpu_percent', sa.Numeric(5,2), nullable=True),
        sa.Column('memory_percent', sa.Numeric(5,2), nullable=True),
        sa.Column('disk_percent', sa.Numeric(5,2), nullable=True),
        sa.Column('network_rx_bytes', sa.BigInteger, nullable=True),
        sa.Column('network_tx_bytes', sa.BigInteger, nullable=True),
        sa.Column('outline_active_keys', sa.Integer, nullable=True),
        sa.Column('wireguard_active_peers', sa.Integer, nullable=True),
        sa.Column('total_bytes_transferred', sa.BigInteger, nullable=True),
        sa.Column('latency_avg_ms', sa.Numeric(8,2), nullable=True),
        sa.Column('latency_p95_ms', sa.Numeric(8,2), nullable=True),
        sa.Column('latency_p99_ms', sa.Numeric(8,2), nullable=True),
        sa.ForeignKeyConstraint(['server_id'], ['vpn_servers.id'], ondelete='CASCADE'),
        sa.PrimaryKeyConstraint('id')
    )
    
    # Index for efficient time-series queries
    op.create_index('idx_server_timestamp', 'server_metrics', ['server_id', sa.text('timestamp DESC')])


def downgrade() -> None:
    op.drop_index('idx_server_timestamp')
    op.drop_table('server_metrics')
```

**Step 3: Modify vpn_keys migration**

```python
"""add server_id to vpn_keys

Revision ID: add_server_id_to_vpn_keys
Revises: create_server_metrics
Create Date: 2026-03-28

"""
from alembic import op
import sqlalchemy as sa

revision = 'add_server_id_to_vpn_keys'
down_revision = 'create_server_metrics'


def upgrade() -> None:
    op.add_column('vpn_keys', sa.Column('server_id', sa.UUID(), nullable=True))
    op.add_column('vpn_keys', sa.Column('latency_ms', sa.Numeric(8,2), nullable=True))
    op.add_column('vpn_keys', sa.Column('last_latency_check', sa.TIMESTAMP(timezone=True), nullable=True))
    
    op.create_foreign_key('fk_vpn_keys_server', 'vpn_keys', 'vpn_servers', ['server_id'], ['id'])


def downgrade() -> None:
    op.drop_constraint('fk_vpn_keys_server', 'vpn_keys', type_='foreignkey')
    op.drop_column('vpn_keys', 'last_latency_check')
    op.drop_column('vpn_keys', 'latency_ms')
    op.drop_column('vpn_keys', 'server_id')
```

**Step 4: Run migrations**

```bash
cd /home/mowgli/usipipo/usipipo-backend
alembic upgrade head
```

**Step 5: Commit**

```bash
git add .
git commit -m "feat: database migrations for multi-server support

- vpn_servers table for server registry
- server_metrics table for time-series metrics
- vpn_keys modified with server_id, latency_ms
- Indexes for efficient queries
"
```

---

### Task 10: ServerRegistry Service

**Files:**
- Create: `usipipo-backend/src/core/application/services/server_registry_service.py`
- Create: `usipipo-backend/src/core/domain/entities/server.py`
- Create: `usipipo-backend/src/core/domain/enums/server_status.py`

**Step 1: Create ServerStatus enum**

```python
from enum import Enum

class ServerStatus(str, Enum):
    ONLINE = "online"
    OFFLINE = "offline"
    MAINTENANCE = "maintenance"
```

**Step 2: Create Server entity**

```python
from dataclasses import dataclass
from datetime import datetime
from uuid import UUID

@dataclass
class Server:
    id: UUID
    name: str
    country_code: str
    country_name: str
    city: str | None
    region: str | None
    agent_url: str
    agent_api_key: str
    supports_outline: bool
    supports_wireguard: bool
    supports_trust_tunnel: bool
    status: str
    max_connections: int
    current_connections: int
    created_at: datetime
    updated_at: datetime
    last_heartbeat_at: datetime | None
```

**Step 3: Create ServerRegistryService**

```python
import uuid
from datetime import datetime, timedelta
from typing import Optional

from src.core.domain.entities.server import Server
from src.core.domain.enums.server_status import ServerStatus
from src.infrastructure.persistence.repositories.server_repository import ServerRepository

class ServerRegistryService:
    def __init__(self, server_repo: ServerRepository):
        self.server_repo = server_repo
    
    async def register_server(
        self,
        name: str,
        country_code: str,
        country_name: str,
        agent_url: str,
        agent_api_key: str,
        city: Optional[str] = None,
        region: Optional[str] = None,
        protocols: list[str] = None
    ) -> Server:
        server = Server(
            id=uuid.uuid4(),
            name=name,
            country_code=country_code,
            country_name=country_name,
            city=city,
            region=region,
            agent_url=agent_url,
            agent_api_key=agent_api_key,
            supports_outline="outline" in (protocols or ["outline"]),
            supports_wireguard="wireguard" in (protocols or ["wireguard"]),
            supports_trust_tunnel="trust_tunnel" in (protocols or []),
            status=ServerStatus.ONLINE,
            max_connections=1000,
            current_connections=0,
            created_at=datetime.now(),
            updated_at=datetime.now(),
            last_heartbeat_at=None
        )
        
        return await self.server_repo.create(server)
    
    async def get_available_servers(self, country: Optional[str] = None) -> list[Server]:
        servers = await self.server_repo.find_all()
        
        # Filter by country if specified
        if country:
            servers = [s for s in servers if s.country_code == country]
        
        # Filter only online servers
        servers = [s for s in servers if s.status == ServerStatus.ONLINE]
        
        return servers
    
    async def select_best_server(
        self,
        country: str,
        protocol: str
    ) -> Optional[Server]:
        servers = await self.get_available_servers(country)
        
        # Filter by protocol support
        if protocol == "outline":
            servers = [s for s in servers if s.supports_outline]
        elif protocol == "wireguard":
            servers = [s for s in servers if s.supports_wireguard]
        
        if not servers:
            return None
        
        # Select server with lowest load
        return min(servers, key=lambda s: s.current_connections / s.max_connections)
    
    async def update_server_status(
        self,
        server_id: UUID,
        status: ServerStatus
    ) -> None:
        server = await self.server_repo.find_by_id(server_id)
        if server:
            server.status = status
            server.updated_at = datetime.now()
            await self.server_repo.update(server)
    
    async def update_heartbeat(
        self,
        server_id: UUID
    ) -> None:
        server = await self.server_repo.find_by_id(server_id)
        if server:
            server.last_heartbeat_at = datetime.now()
            await self.server_repo.update(server)
```

**Step 4: Commit**

```bash
git add .
git commit -m "feat: ServerRegistry service

- Server entity and ServerStatus enum
- register_server, get_available_servers, select_best_server
- update_server_status, update_heartbeat
- Load-based server selection
"
```

---

### Task 11: VpnAgentClient (HTTP Client for Agents)

**Files:**
- Create: `usipipo-backend/src/infrastructure/api_clients/vpn_agent_client.py`
- Create: `usipipo-backend/src/infrastructure/api_clients/schemas.py`

**Step 1: Create schemas**

```python
from pydantic import BaseModel, HttpUrl
from typing import Optional

class CreateOutlineKeyRequest(BaseModel):
    name: str

class CreateOutlineKeyResponse(BaseModel):
    id: str
    name: str
    access_url: str

class CreateWireGuardPeerRequest(BaseModel):
    name: str

class CreateWireGuardPeerResponse(BaseModel):
    public_key: str
    name: str
    ip_address: str
    config: str

class SystemMetrics(BaseModel):
    cpu_percent: float
    memory_percent: float
    disk_percent: float
    network_rx_bytes: int
    network_tx_bytes: int

class ServerMetricsRequest(BaseModel):
    server_id: str
    timestamp: datetime
    system: SystemMetrics
    # ... VPN and latency metrics
```

**Step 2: Create VpnAgentClient**

```python
import httpx
from typing import Optional

class VpnAgentClient:
    def __init__(
        self,
        base_url: str,
        api_key: str,
        timeout: float = 10.0,
        retry_count: int = 3
    ):
        self.base_url = base_url
        self.api_key = api_key
        self.timeout = timeout
        
        self.client = httpx.AsyncClient(
            headers={"X-API-Key": api_key},
            timeout=httpx.Timeout(timeout),
            transport=httpx.AsyncHTTPTransport(retries=retry_count)
        )
    
    async def create_outline_key(self, name: str) -> dict:
        response = await self.client.post(
            f"{self.base_url}/outline/keys",
            json={"name": name}
        )
        response.raise_for_status()
        return response.json()
    
    async def delete_outline_key(self, key_id: str) -> bool:
        response = await self.client.delete(
            f"{self.base_url}/outline/keys/{key_id}"
        )
        return response.status_code in [204, 404]
    
    async def create_wireguard_peer(self, name: str) -> dict:
        response = await self.client.post(
            f"{self.base_url}/wireguard/peers",
            json={"name": name}
        )
        response.raise_for_status()
        return response.json()
    
    async def delete_wireguard_peer(self, name: str) -> bool:
        response = await self.client.delete(
            f"{self.base_url}/wireguard/peers/{name}"
        )
        return response.status_code == 204
    
    async def get_status(self) -> dict:
        response = await self.client.get(f"{self.base_url}/status")
        response.raise_for_status()
        return response.json()
    
    async def get_metrics(self) -> dict:
        response = await self.client.get(f"{self.base_url}/metrics")
        response.raise_for_status()
        return response.json()
    
    async def close(self):
        await self.client.aclose()
```

**Step 3: Commit**

```bash
git add .
git commit -m "feat: VpnAgentClient for remote agent communication

- HTTP client with API Key auth
- create_outline_key, delete_outline_key
- create_wireguard_peer, delete_wireguard_peer
- get_status, get_metrics
- Retry logic with exponential backoff
"
```

---

### Task 12: Modify VpnService to Use Agents

**Files:**
- Modify: `usipipo-backend/src/core/application/services/vpn_service.py`

**Step 1: Inject ServerRegistry and VpnAgentClient**

```python
# Add imports
from src.infrastructure.api_clients.vpn_agent_client import VpnAgentClient
from src.core.application.services.server_registry_service import ServerRegistryService

# Modify __init__
class VpnService:
    def __init__(
        self,
        user_repo: IUserRepository,
        vpn_repo: IVPNRepository,
        server_registry: ServerRegistryService,
        outline_client: OutlineClient | None = None,
        wireguard_client: WireGuardClient | None = None,
    ):
        self.user_repo = user_repo
        self.vpn_repo = vpn_repo
        self.server_registry = server_registry
        self.outline_client = outline_client  # Keep for local fallback
        self.wireguard_client = wireguard_client
        self.agent_clients: dict[UUID, VpnAgentClient] = {}
    
    def _get_agent_client(self, server: Server) -> VpnAgentClient:
        if server.id not in self.agent_clients:
            self.agent_clients[server.id] = VpnAgentClient(
                base_url=server.agent_url,
                api_key=server.agent_api_key
            )
        return self.agent_clients[server.id]
    
    async def create_key(
        self,
        user_id: uuid.UUID,
        name: str,
        vpn_type: str,
        country: Optional[str] = None,
        data_limit_gb: float = 5.0,
    ) -> VpnKey:
        # ... existing user validation code ...
        
        # Select best server
        server = await self.server_registry.select_best_server(
            country=country or "US",  # Default to USA
            protocol=vpn_type
        )
        
        if not server:
            raise Exception("No available servers")
        
        # Use agent client instead of local VPN client
        agent_client = self._get_agent_client(server)
        
        if vpn_type == "outline":
            result = await agent_client.create_outline_key(name=name)
            config = result["access_url"]
            external_id = result["id"]
        elif vpn_type == "wireguard":
            result = await agent_client.create_wireguard_peer(name=name)
            config = result["config"]
            external_id = result["public_key"]
        else:
            raise InvalidVpnTypeError(f"Invalid VPN type: {vpn_type}")
        
        # ... rest of existing code ...
        
        # Create entity with server_id
        vpn_key = VpnKey(
            # ... existing fields ...
            server_id=server.id,  # NEW
            external_id=external_id,
        )
        
        return await self.vpn_repo.create(vpn_key)
```

**Step 2: Update delete_key to use agent**

```python
async def delete_key(self, user_id: uuid.UUID, key_id: uuid.UUID) -> bool:
    key = await self.vpn_repo.get_by_id(key_id)
    if not key:
        raise VpnKeyNotFoundError(f"Key {key_id} not found")
    
    # Get server for this key
    server = await self.server_registry.get_server(key.server_id)
    agent_client = self._get_agent_client(server)
    
    # Delete via agent
    if key.key_type == KeyType.OUTLINE:
        await agent_client.delete_outline_key(key.external_id or "")
    elif key.key_type == KeyType.WIREGUARD:
        await agent_client.delete_wireguard_peer(key.external_id or "")
    
    return await self.vpn_repo.delete(key_id)
```

**Step 3: Commit**

```bash
git add .
git commit -m "feat: modify VpnService to use remote agents

- Inject ServerRegistryService
- VpnAgentClient per server
- create_key uses agent instead of local VPN
- delete_key uses agent for remote deletion
- server_id stored with vpn_keys
"
```

---

### Task 13: Metrics Ingestion Endpoint

**Files:**
- Create: `usipipo-backend/src/infrastructure/api/v1/routes/metrics.py`
- Create: `usipipo-backend/src/core/application/services/metrics_service.py`

**Step 1: Create MetricsService**

```python
from datetime import datetime
from uuid import UUID

class MetricsService:
    def __init__(self, metrics_repo: ServerMetricsRepository):
        self.metrics_repo = metrics_repo
    
    async def ingest_agent_metrics(
        self,
        server_id: UUID,
        metrics: dict
    ) -> None:
        # Store metrics in database
        await self.metrics_repo.create({
            "server_id": server_id,
            **metrics
        })
        
        # Update server heartbeat
        await self.server_registry.update_heartbeat(server_id)
    
    async def get_user_key_usage(
        self,
        user_id: UUID,
        key_id: str
    ) -> dict:
        # Get key from DB
        key = await self.vpn_repo.get_by_id(key_id)
        
        # Get server for this key
        server = await self.server_registry.get_server(key.server_id)
        
        # Get usage from agent
        agent_client = self._get_agent_client(server)
        
        if key.key_type == KeyType.OUTLINE:
            bytes_used = await agent_client.get_outline_key_usage(key.external_id)
        elif key.key_type == KeyType.WIREGUARD:
            bytes_used = await agent_client.get_wireguard_peer_usage(key.external_id)
        
        return {
            "key_id": key_id,
            "bytes_used": bytes_used,
            "data_limit_bytes": key.data_limit_bytes,
            "percent_used": (bytes_used / key.data_limit_bytes) * 100
        }
```

**Step 2: Create metrics route**

```python
from fastapi import APIRouter, Depends, HTTPException
from src.core.application.services.metrics_service import MetricsService

router = APIRouter(prefix="/metrics", tags=["Metrics"])

@router.post("/agents/{server_id}")
async def ingest_agent_metrics(
    server_id: UUID,
    metrics: dict,
    metrics_service: MetricsService = Depends()
):
    await metrics_service.ingest_agent_metrics(server_id, metrics)
    return {"status": "ok"}

@router.get("/servers/{server_id}")
async def get_server_metrics(
    server_id: UUID,
    from_date: datetime | None = None,
    to_date: datetime | None = None,
    metrics_service: MetricsService = Depends()
):
    metrics = await metrics_service.get_server_metrics(server_id, from_date, to_date)
    return {"metrics": metrics}
```

**Step 3: Commit**

```bash
git add .
git commit -m "feat: metrics ingestion endpoint

- POST /api/v1/metrics/agents/{server_id}
- MetricsService for storing metrics
- Heartbeat update on metrics ingestion
- GET /api/v1/metrics/servers/{server_id} for historical data
"
```

---

## Phase 3: Testing + Deployment - 3 Days

### Task 14: Integration Testing

**Files:**
- Create: `usipipo-agent/tests/integration_test.go`
- Create: `usipipo-backend/tests/test_agent_integration.py`

**Step 1: Test agent → backend communication**

```python
async def test_metrics_ingestion():
    # Simulate agent pushing metrics
    response = await client.post(
        "/api/v1/metrics/agents/test-server-id",
        json={
            "server_id": "test-server-id",
            "timestamp": datetime.now().isoformat(),
            "system": {
                "cpu_percent": 45.2,
                "memory_percent": 62.1,
                # ...
            }
        },
        headers={"X-API-Key": "test-key"}
    )
    
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

**Step 2: Test backend → agent commands**

```go
func TestAgentEndpoints(t *testing.T) {
    // Test create outline key
    resp, _ := router.POST("/outline/keys").
        WithHeader("X-API-Key", "test-key").
        WithJSON(map[string]string{"name": "test"}).
        Expect()
    
    resp.Status(http.StatusCreated)
    resp.JSON().ContainsKeys("id", "name", "access_url")
}
```

**Step 3: Commit**

```bash
git add .
git commit -m "test: integration tests for agent-backend communication

- Agent endpoint tests
- Metrics ingestion tests
- VPN key creation/deletion tests
"
```

---

### Task 15: Deploy to 3 VPS

**Files:**
- Create: `usipipo-docs/ops/VPN-SERVER-CHECKLIST.md`

**Step 1: Deploy agent to VPS USA**

```bash
# On VPS USA
cd /home/mowgli/usipipo/usipipo-agent
GOOS=linux GOARCH=amd64 go build -o agent ./cmd/agent

scp agent user@us-vps:/opt/usipipo-agent/
scp .env.example user@us-vps:/opt/usipipo-agent/.env
scp systemd/usipipo-agent.service user@us-vps:/etc/systemd/system/

# SSH to VPS
ssh user@us-vps
sudo systemctl daemon-reload
sudo systemctl enable usipipo-agent
sudo systemctl start usipipo-agent
sudo systemctl status usipipo-agent
```

**Step 2: Configure Caddy + DuckDNS**

```bash
# On each VPS
sudo nano /etc/caddy/Caddyfile

# Add:
usipipousa.duckdns.org {
    reverse_proxy localhost:8080
    tls {
        dns duckdns YOUR_DUCKDNS_TOKEN
    }
}

sudo systemctl reload caddy
```

**Step 3: Register servers in backend**

```python
# Script to register servers
await server_registry.register_server(
    name="USA East",
    country_code="US",
    country_name="United States",
    agent_url="https://usipipousa.duckdns.org",
    agent_api_key="generated-uuid",
    city="New York",
    region="us-east-1"
)
```

**Step 4: Commit documentation**

```bash
git add .
git commit -m "docs: VPN server deployment checklist

- Step-by-step deployment guide
- Caddy + DuckDNS configuration
- Server registration script
- Testing checklist
"
```

---

## ✅ Completion Criteria

### **Agent Repository**
- [ ] Go module initialized
- [ ] HTTP server with Gin
- [ ] API Key authentication
- [ ] Outline integration
- [ ] WireGuard integration
- [ ] Metrics collector
- [ ] Metrics reporter (1-minute push)
- [ ] **GitHub Actions CI/CD configured**
- [ ] **Multi-platform builds (linux, windows, darwin)**
- [ ] **Releases con binarios pre-compilados**
- [ ] systemd service
- [ ] Deployment documentation

### **Backend Modifications**
- [ ] Database migrations (vpn_servers, server_metrics)
- [ ] ServerRegistry service
- [ ] VpnAgentClient
- [ ] VpnService modified to use agents
- [ ] Metrics ingestion endpoint
- [ ] All tests passing

### **Deployment**
- [ ] Agent deployed to 3 VPS (USA, Germany, Belgium)
- [ ] Caddy + DuckDNS configured for each
- [ ] Servers registered in backend
- [ ] End-to-end testing complete
- [ ] Metrics flowing to backend

---

## 📦 **GitHub Actions Release Process**

### **Crear un nuevo release:**

```bash
# 1. Update version in code (if applicable)
# 2. Commit changes
git add .
git commit -m "feat: add new feature"

# 3. Create and push tag
git tag -a v0.1.0 -m "Release v0.1.0 - Initial release"
git push origin v0.1.0

# 4. GitHub Actions automatically:
#    - Build for all platforms (linux, windows, darwin)
#    - Build for all architectures (amd64, arm64)
#    - Create .zip files for each
#    - Create GitHub Release with binaries attached
```

### **Binarios disponibles en cada release:**

```
usipipo-agent-linux-amd64.zip      ← VPS estándar (USA, DE, BE)
usipipo-agent-linux-arm64.zip      ← VPS ARM / Raspberry Pi
usipipo-agent-darwin-amd64.zip     ← macOS Intel
usipipo-agent-darwin-arm64.zip     ← macOS Apple Silicon
usipipo-agent-windows-amd64.zip    ← Windows 64-bit
```

### **Download URLs:**

```
Latest release:
https://github.com/uSipipo-Team/usipipo-agent/releases/latest/download/usipipo-agent-linux-amd64.zip

Specific version:
https://github.com/uSipipo-Team/usipipo-agent/releases/download/v0.1.0/usipipo-agent-linux-amd64.zip
```

---

**Plan complete!** Two execution options:

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with `executing-plans`, batch execution with checkpoints

**Which approach?**
