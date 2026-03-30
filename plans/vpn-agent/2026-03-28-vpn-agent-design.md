# VPN Agent Architecture Design

**Date:** 2026-03-28
**Status:** Approved ✅
**Repository:** `usipipo-agent` (Go)
**Related:** Backend (`usipipo-backend`), Multi-Server Orchestration

---

## 🎯 Objective

Build a scalable multi-country VPN infrastructure with centralized orchestration. Enable uSipipo to deploy VPN servers across 200+ countries while maintaining a single backend that manages all servers through lightweight agents.

**Current State:** 1 VPS (USA) with Outline, WireGuard, Trust Tunnel
**Target State:** 3 VPS (USA, Germany, Belgium) → Scale to 200+ countries

---

## 🏗️ Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                  BACKEND CENTRAL (Orchestrator)                  │
│  - Business logic (limits, payments, users)                      │
│  - ServerRegistry (server metadata + status)                     │
│  - Metrics Storage (historical data)                             │
│  - API for frontends (Telegram Bot, Web, Mobile, Desktop)        │
└──────────────────────────────────────────────────────────────────┘
                               │
                               │ HTTPS + API Key (every 1 minute)
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
┌────────▼────────┐   ┌────────▼────────┐   ┌────────▼────────┐
│  VPS USA        │   │  VPS Germany    │   │  VPS Belgium    │
│ ┌─────────────┐ │   │ ┌─────────────┐ │   │ ┌─────────────┐ │
│ │ VPN Agent   │ │   │ │ VPN Agent   │ │   │ │ VPN Agent   │ │
│ │ (Go)        │ │   │ │ (Go)        │ │   │ │ (Go)        │ │
│ │ :8080       │ │   │ │ :8080       │ │   │ │ :8080       │ │
│ └──────┬──────┘ │   │ └──────┬──────┘ │   │ └──────┬──────┘ │
│        │       │   │        │       │   │        │       │
│        ▼       │   │        ▼       │   │        ▼       │
│ ┌─────────────┐ │   │ ┌─────────────┐ │   │ ┌─────────────┐ │
│ │ Outline     │ │   │ │ Outline     │ │   │ │ Outline     │ │
│ │ WireGuard   │ │   │ │ WireGuard   │   │ │ WireGuard   │ │
│ │ Trust Tunnel│ │   │ │ Trust Tunnel│ │   │ │ Trust Tunnel│ │
│ └─────────────┘ │   │ └─────────────┘ │   │ └─────────────┘ │
│                  │   │                  │   │                  │
│ Caddy           │   │ Caddy           │   │ Caddy           │
│ usipipousa.     │   │ usipipode.      │   │ usipipobe.      │
│ duckdns.org     │   │ duckdns.org     │   │ duckdns.org     │
└─────────────────┘   └─────────────────┘   └─────────────────┘
```

---

## 📦 Components

### **1. VPN Agent (Go)**

**Repository:** `usipipo-agent`
**Language:** Go (static binary, low memory footprint)
**Deployment:** One instance per VPS (alongside VPN servers)

#### **Project Structure**
```
usipipo-agent/
├── cmd/
│   └── agent/
│       └── main.go              # Entry point
├── internal/
│   ├── api/
│   │   ├── handlers.go          # HTTP handlers
│   │   ├── middleware.go        # API Key auth
│   │   └── server.go            # HTTP server setup
│   ├── vpn/
│   │   ├── outline.go           # Outline API client
│   │   ├── wireguard.go         # WireGuard wrapper (wg commands)
│   │   └── trusttunnel.go       # Trust Tunnel (AdGuard) wrapper
│   ├── metrics/
│   │   └── collector.go         # System + VPN metrics collector
│   ├── reporter/
│   │   └── reporter.go          # Push metrics to backend every 1 min
│   └── config/
│       └── config.go            # Configuration (env vars)
├── go.mod
├── go.sum
├── Dockerfile
├── systemd/
│   └── usipipo-agent.service
└── README.md
```

#### **Agent API Endpoints**

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `POST` | `/outline/keys` | Create Outline key | API Key |
| `DELETE` | `/outline/keys/:id` | Delete Outline key | API Key |
| `GET` | `/outline/keys/:id/usage` | Get usage for specific key | API Key |
| `POST` | `/wireguard/peers` | Create WireGuard peer | API Key |
| `DELETE` | `/wireguard/peers/:name` | Delete WireGuard peer | API Key |
| `GET` | `/wireguard/peers/:name/usage` | Get usage for specific peer | API Key |
| `GET` | `/status` | Server health status | API Key |
| `GET` | `/metrics` | Detailed metrics (system + VPN) | API Key |

#### **Agent Responsibilities**

1. **VPN Management**
   - Create/delete Outline keys (via Outline Manager API)
   - Create/delete WireGuard peers (via `wg` commands)
   - Support Trust Tunnel (AdGuard) when available

2. **Metrics Collection** (every 10 seconds, cached)
   ```json
   {
     "system": {
       "cpu_percent": 45.2,
       "memory_percent": 62.1,
       "disk_percent": 38.5,
       "network_rx_bytes": 1234567890,
       "network_tx_bytes": 9876543210
     },
     "vpn": {
       "outline": {
         "active_keys": 42,
         "total_bytes_transferred": 5000000000
       },
       "wireguard": {
         "active_peers": 38,
         "total_bytes_transferred": 4500000000
       }
     },
     "latency_ms": {
       "avg": 12.5,
       "p95": 25.3,
       "p99": 45.8
     }
   }
   ```

3. **Auto-Reporting to Backend** (every 1 minute)
   - Push metrics to backend: `POST /api/v1/metrics/agents/{server_id}`
   - Include server health status
   - Retry with exponential backoff on failure
   - Buffer metrics locally if backend is unreachable

---

### **2. Backend Central (Modifications)**

#### **New Services**

**a) ServerRegistry Service**
```python
# src/core/application/services/server_registry_service.py

class ServerRegistryService:
    """Management of VPN servers in the ecosystem."""
    
    async def register_server(
        self,
        name: str,
        country_code: str,
        country_name: str,
        city: str | None = None,
        region: str | None = None,
        agent_url: str = None,
        agent_api_key: str = None,
        protocols: list[str] = ["outline", "wireguard"]
    ) -> Server:
        """Register new VPN server"""
    
    async def get_available_servers(self, country: str | None = None) -> list[Server]:
        """Get available servers (optionally by country)"""
    
    async def select_best_server(
        self,
        country: str,
        protocol: VpnType
    ) -> Server:
        """Select best server by country + protocol + load"""
    
    async def update_server_status(
        self,
        server_id: str,
        status: ServerStatus
    ):
        """Update server status (online/offline/maintenance)"""
    
    async def get_server_metrics(
        self,
        server_id: str,
        from_date: datetime | None = None,
        to_date: datetime | None = None
    ) -> ServerMetrics:
        """Get historical metrics for a server"""
```

**b) VpnAgentClient**
```python
# src/infrastructure/api_clients/vpn_agent_client.py

class VpnAgentClient:
    """HTTP client to communicate with remote VPN agents."""
    
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
        self.session = httpx.AsyncClient(
            headers={"X-API-Key": api_key},
            timeout=timeout,
            transport=httpx.AsyncHTTPTransport(retries=retry_count)
        )
    
    async def create_outline_key(self, name: str) -> OutlineKeyResponse:
        """Create Outline key on remote server"""
    
    async def delete_outline_key(self, key_id: str) -> bool:
        """Delete Outline key"""
    
    async def create_wireguard_peer(self, name: str) -> WireGuardPeerResponse:
        """Create WireGuard peer"""
    
    async def delete_wireguard_peer(self, name: str) -> bool:
        """Delete WireGuard peer"""
    
    async def get_status(self) -> ServerStatusResponse:
        """Get server health status"""
    
    async def get_metrics(self) -> ServerMetricsResponse:
        """Get detailed metrics"""
```

**c) Metrics Ingestion Service**
```python
# src/core/application/services/metrics_service.py

class MetricsService:
    """Ingest and store metrics from VPN agents."""
    
    async def ingest_agent_metrics(
        self,
        server_id: str,
        metrics: AgentMetricsRequest
    ):
        """Store metrics from agent (called every 1 minute per server)"""
    
    async def get_user_key_usage(
        self,
        user_id: uuid.UUID,
        key_id: str
    ) -> KeyUsage:
        """Get usage for specific user key (for user dashboard)"""
    
    async def get_country_latency_comparison(
        self,
        user_location: str
    ) -> list[CountryLatency]:
        """Get latency comparison across countries (for user recommendations)"""
```

#### **New API Endpoints**

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/metrics/agents/{server_id}` | Agent metrics ingestion (API Key auth) |
| `GET` | `/api/v1/servers` | List available servers (admin) |
| `GET` | `/api/v1/servers/{server_id}/metrics` | Get server metrics (admin) |
| `GET` | `/api/v1/users/me/keys/{key_id}/usage` | Get key usage (user) |
| `GET` | `/api/v1/vpn/compare?from=US&to=DE,BE` | Compare latency across countries |

---

### **3. Database Schema**

#### **New Tables**

```sql
-- VPN Servers registry
CREATE TABLE vpn_servers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    country_code CHAR(2) NOT NULL,
    country_name VARCHAR(100) NOT NULL,
    city VARCHAR(100),
    region VARCHAR(50), -- us-east-1, eu-central-1, etc.
    
    -- Agent configuration
    agent_url VARCHAR(500) NOT NULL, -- https://usipipousa.duckdns.org
    agent_api_key VARCHAR(255) NOT NULL,
    
    -- Supported protocols
    supports_outline BOOLEAN DEFAULT TRUE,
    supports_wireguard BOOLEAN DEFAULT TRUE,
    supports_trust_tunnel BOOLEAN DEFAULT FALSE,
    
    -- Status and capacity
    status VARCHAR(20) DEFAULT 'online', -- online, offline, maintenance
    max_connections INT DEFAULT 1000,
    current_connections INT DEFAULT 0,
    
    -- Metadata
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    last_heartbeat_at TIMESTAMPTZ,
    
    UNIQUE(country_code, region)
);

-- Historical metrics
CREATE TABLE server_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    server_id UUID REFERENCES vpn_servers(id) ON DELETE CASCADE,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    
    -- System metrics
    cpu_percent DECIMAL(5,2),
    memory_percent DECIMAL(5,2),
    disk_percent DECIMAL(5,2),
    
    -- Network metrics
    network_rx_bytes BIGINT,
    network_tx_bytes BIGINT,
    
    -- VPN metrics
    outline_active_keys INT,
    wireguard_active_peers INT,
    total_bytes_transferred BIGINT,
    
    -- Latency metrics
    latency_avg_ms DECIMAL(8,2),
    latency_p95_ms DECIMAL(8,2),
    latency_p99_ms DECIMAL(8,2),
    
    INDEX idx_server_timestamp (server_id, timestamp DESC)
);

-- Modify existing vpn_keys table
ALTER TABLE vpn_keys ADD COLUMN server_id UUID REFERENCES vpn_servers(id);
ALTER TABLE vpn_keys ADD COLUMN latency_ms DECIMAL(8,2);
ALTER TABLE vpn_keys ADD COLUMN last_latency_check TIMESTAMPTZ;
```

---

## 🔐 Security

### **API Key Authentication**

**Backend → Agent:**
- Each server has a unique API key (UUID v4)
- API key stored in backend database (`vpn_servers.agent_api_key`)
- API key rotated every 90 days
- API key sent via `X-API-Key` header

**Agent → Backend (metrics push):**
- Same API key used for mutual authentication
- Backend validates API key on metrics ingestion endpoint

### **HTTPS with Caddy + DuckDNS**

Each VPS runs Caddy as reverse proxy:

```caddyfile
# VPS USA
usipipousa.duckdns.org {
    reverse_proxy localhost:8080
    tls {
        dns duckdns {env.DUCKDNS_TOKEN}
    }
}

# VPS Germany
usipipode.duckdns.org {
    reverse_proxy localhost:8080
    tls {
        dns duckdns {env.DUCKDNS_TOKEN}
    }
}

# VPS Belgium
usipipobe.duckdns.org {
    reverse_proxy localhost:8080
    tls {
        dns duckdns {env.DUCKDNS_TOKEN}
    }
}
```

**Benefits:**
- Automatic HTTPS with Let's Encrypt
- DNS challenge via DuckDNS (no port 80 required)
- Auto-renewal handled by Caddy
- Each server has unique domain

### **Agent Middleware (Go)**

```go
// internal/api/middleware.go

func APIKeyMiddleware(validKeys map[string]bool) gin.HandlerFunc {
    return func(c *gin.Context) {
        apiKey := c.GetHeader("X-API-Key")
        if apiKey == "" {
            c.AbortWithStatusJSON(401, gin.H{"error": "Missing API key"})
            return
        }
        
        if !validKeys[apiKey] {
            c.AbortWithStatusJSON(401, gin.H{"error": "Invalid API key"})
            return
        }
        
        c.Next()
    }
}
```

---

## 📊 Metrics & Monitoring

### **Agent Auto-Reporting (Push Model)**

**Interval:** Every 1 minute
**Endpoint:** `POST /api/v1/metrics/agents/{server_id}`
**Payload:**
```json
{
  "server_id": "us-east-1",
  "timestamp": "2026-03-28T10:00:00Z",
  "system": {
    "cpu_percent": 45.2,
    "memory_percent": 62.1,
    "disk_percent": 38.5,
    "network_rx_bytes": 1234567890,
    "network_tx_bytes": 9876543210
  },
  "vpn": {
    "outline": {
      "active_keys": 42,
      "total_bytes_transferred": 5000000000
    },
    "wireguard": {
      "active_peers": 38,
      "total_bytes_transferred": 4500000000
    }
  },
  "latency_ms": {
    "avg": 12.5,
    "p95": 25.3,
    "p99": 45.8
  }
}
```

**Retry Logic:**
- First retry: 5 seconds
- Second retry: 15 seconds
- Third retry: 60 seconds
- After 3 failures: Buffer metrics locally, retry on next cycle

---

### **Admin Dashboard Data**

**Server Overview:**
- World map with server locations (green/red status)
- Total active connections per server
- CPU/RAM/Disk usage per server
- Total bandwidth (GB) per server (last 24h)
- Average latency per server

**Alerts:**
- Server offline (no heartbeat > 2 minutes)
- High CPU usage (> 80%)
- High memory usage (> 85%)
- High connection count (> 90% capacity)

---

### **User Dashboard Data**

**Per-Key Usage:**
```
Your VPN Key - United States:
  📊 Usage: 5.2 GB / 10 GB
  ⚡ Latency: 45ms (from your location)
  📈 Speed: 125 Mbps download
  📅 Expires: 2026-04-28
```

**Country Comparison:**
```
Available Countries (from your location):
  🇺🇸 USA:      45ms ⭐⭐⭐⭐⭐ (Recommended)
  🇩🇪 Germany:  89ms ⭐⭐⭐⭐
  🇧🇪 Belgium:  92ms ⭐⭐⭐⭐
```

**Recommendations:**
- Auto-suggest best country based on latency
- Show server load (low/medium/high)
- Allow manual country selection

---

## 🚀 Implementation Plan

### **Phase 1: VPN Agent (Go)** - 5 days

**Day 1-2: Core Agent**
- [ ] Project scaffolding (Go modules, structure)
- [ ] HTTP server setup (Gin or Echo framework)
- [ ] API Key middleware
- [ ] Configuration (env vars: API_KEY, BACKEND_URL, SERVER_ID)
- [ ] Health endpoint (`GET /status`)

**Day 3-4: VPN Integrations**
- [ ] Outline client (call local Outline Manager API)
  - `POST /outline/keys` → Create key
  - `DELETE /outline/keys/:id` → Delete key
- [ ] WireGuard wrapper (execute `wg` commands)
  - `POST /wireguard/peers` → Create peer
  - `DELETE /wireguard/peers/:name` → Delete peer
- [ ] Metrics collector (CPU, RAM, disk, network)
- [ ] VPN-specific metrics (active keys/peers, bandwidth)

**Day 5: Reporting + Deployment**
- [ ] Metrics reporter (push to backend every 1 min)
- [ ] Retry logic with exponential backoff
- [ ] systemd service file
- [ ] Dockerfile (optional)
- [ ] README with deployment instructions

---

### **Phase 2: Backend Modifications** - 5 days

**Day 1-2: Database + Models**
- [ ] Create `vpn_servers` table migration
- [ ] Create `server_metrics` table migration
- [ ] Modify `vpn_keys` table (add `server_id`, `latency_ms`)
- [ ] Create SQLAlchemy models
- [ ] Create Pydantic schemas

**Day 3-4: Services**
- [ ] ServerRegistryService (CRUD for servers)
- [ ] VpnAgentClient (HTTP client for agents)
- [ ] MetricsService (ingest + query metrics)
- [ ] Modify VpnService to use agents (instead of direct VPN calls)

**Day 5: API Endpoints**
- [ ] `POST /api/v1/metrics/agents/{server_id}` (metrics ingestion)
- [ ] `GET /api/v1/servers` (list servers - admin)
- [ ] `GET /api/v1/servers/{server_id}/metrics` (server metrics - admin)
- [ ] `GET /api/v1/users/me/keys/{key_id}/usage` (key usage - user)
- [ ] `GET /api/v1/vpn/compare` (latency comparison - user)

---

### **Phase 3: Testing + Deployment** - 3 days

**Day 1: Integration Testing**
- [ ] Test agent → backend communication
- [ ] Test backend → agent commands (create/delete keys)
- [ ] Test metrics ingestion
- [ ] Test failover (agent offline scenario)

**Day 2: Deploy to VPS**
- [ ] Deploy agent to VPS USA (local test)
- [ ] Configure Caddy + DuckDNS for USA
- [ ] Register server in backend database
- [ ] Verify metrics reporting

**Day 3: Multi-Country Deploy**
- [ ] Deploy agent to VPS Germany
- [ ] Deploy agent to VPS Belgium
- [ ] Configure Caddy + DuckDNS for each
- [ ] Register servers in backend
- [ ] End-to-end testing (create keys in all 3 countries)
- [ ] Load testing (simulate 100 concurrent users)

---

## 📁 Documentation Deliverables

1. **Design Document** (this file)
   - `usipipo-docs/plans/vpn-agent/2026-03-28-vpn-agent-design.md`

2. **Agent Documentation**
   - `usipipo-docs/agent/ARCHITECTURE.md`
   - `usipipo-docs/agent/DEPLOYMENT.md`
   - `usipipo-docs/agent/AGENT-API.md`
   - `usipipo-docs/agent/METRICS.md`

3. **Backend Documentation**
   - `usipipo-docs/backend/MULTI-SERVER-SETUP.md`
   - `usipipo-docs/backend/SERVER-REGISTRY.md`
   - `usipipo-docs/backend/METRICS-INGESTION.md`

4. **Operations**
   - `usipipo-docs/ops/VPN-SERVER-CHECKLIST.md`
   - `usipipo-docs/ops/CADDY-DUCKDNS-SETUP.md`
   - `usipipo-docs/ops/MONITORING-ALERTS.md`

---

## 🎯 Success Criteria

### **Functional**
- ✅ Agent deployed on 3 VPS (USA, Germany, Belgium)
- ✅ Backend can create/delete keys on remote servers
- ✅ Metrics reported every 1 minute from each agent
- ✅ Admin dashboard shows server status + metrics
- ✅ User can see key usage + country comparison

### **Non-Functional**
- ✅ Agent memory usage < 50MB per instance
- ✅ Metrics push latency < 500ms
- ✅ Backend API response time < 200ms (p95)
- ✅ System supports 200+ servers without architecture changes
- ✅ Zero-downtime deployment for agent updates

### **Security**
- ✅ All agent communication over HTTPS
- ✅ API Key authentication enforced
- ✅ API keys rotated every 90 days
- ✅ Caddy auto-renews Let's Encrypt certificates

---

## 📈 Scaling Roadmap

### **Phase 1: Initial Launch (3 countries)**
- USA (existing VPS)
- Germany (new VPS)
- Belgium (new VPS)

### **Phase 2: European Expansion (10 countries)**
- Add: France, Spain, Italy, UK, Netherlands, Poland, Sweden
- Implement load balancing (round-robin within country)

### **Phase 3: Global Coverage (50 countries)**
- Add: Asia (Japan, Singapore, India, etc.)
- Add: South America (Brazil, Argentina, Chile)
- Add: Oceania (Australia, New Zealand)

### **Phase 4: Full Scale (200+ countries)**
- Implement region-based routing (auto-select nearest)
- Add CDN-like caching for configuration
- Implement auto-failover (if server down, switch to nearest)

---

## 🔧 Technology Stack

| Component | Technology | Rationale |
|-----------|------------|-----------|
| **Agent Language** | Go | Static binary, low memory, concurrent |
| **Agent Framework** | Gin or Echo | Lightweight, fast, mature |
| **Backend** | Python (FastAPI) | Existing codebase |
| **Database** | PostgreSQL | Existing + TimescaleDB for metrics |
| **Reverse Proxy** | Caddy | Auto HTTPS, DuckDNS integration |
| **DNS** | DuckDNS | Free, supports DNS challenge |
| **Deployment** | systemd | Simple, reliable, no orchestration needed |
| **Monitoring** | Custom (DB queries) | Simple for MVP, can add Grafana later |

---

## 🚨 Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Agent offline | Users can't create keys in that country | Backend marks server offline, routes to nearest alternative |
| Backend offline | No new keys, but existing keys work | Agents continue serving existing keys, queue metrics locally |
| Certificate expiry | HTTPS breaks | Caddy auto-renews, monitor cert expiry with alerts |
| API key leak | Attacker can create/delete keys | Rotate keys immediately, implement IP whitelisting |
| High latency | Poor user experience | Show latency to users, recommend nearest country |

---

## 📝 Glossary

| Term | Definition |
|------|------------|
| **Agent** | Go service running on each VPS, manages local VPN servers |
| **Backend** | Central orchestrator (FastAPI), manages business logic |
| **Server** | A VPS in a specific country with VPN services |
| **ServerRegistry** | Backend service that tracks all servers |
| **Metrics Push** | Agent sends metrics to backend every 1 minute |
| **Caddy** | Reverse proxy with auto HTTPS on each VPS |

---

**Last Updated:** 2026-03-28
**Status:** Approved ✅
**Next Step:** Invoke `writing-plans` skill to create implementation plan
