# VPN Agent Auto-Registration Design

**Date:** 2026-03-29
**Status:** Approved for Implementation
**Architecture:** Auto-registration on first metrics send with API Key validation

---

## 🎯 Goal

Implement automatic server registration flow where VPN agents can register themselves with the backend on first metrics send, using pre-generated API keys for authentication.

---

## 🏗️ Architecture Overview

### Current Flow (Manual - BROKEN)
```
1. Admin installs agent → .env with SERVER_ID
2. Admin manually inserts in DB (SQL) ← 🚨 MANUAL STEP
3. Agent sends metrics → Backend accepts
```

### New Flow (Auto-Registration)
```
1. Admin generates API key in backend (admin endpoint)
2. Admin installs agent → .env with AGENT_API_KEY + metadata
3. Agent starts → Sends first metrics → Backend auto-registers
4. Backend creates server record → Returns UUID
5. Agent saves UUID to .env → Future metrics use UUID
```

---

## 📋 Requirements

### Backend Requirements
1. **Endpoint:** `POST /api/v1/servers/register-agent` (explicit registration)
2. **Enhanced Metrics Endpoint:** `POST /api/v1/metrics/agents/{server_id}` (auto-register if not exists)
3. **Admin Endpoint:** `POST /api/v1/admin/agent-api-keys` (generate API keys)
4. **Database:** Store agent API keys with status (active/revoked)
5. **Validation:** Verify API key before registration
6. **Idempotency:** Same API key can register only once

### Agent Requirements
1. **Configuration:** `.env` includes `AGENT_API_KEY` (not `SERVER_ID`)
2. **First Run:** Send registration request with metadata
3. **Store UUID:** Save returned `server_id` to `.env`
4. **Subsequent Runs:** Use saved UUID for metrics
5. **Metadata Collection:** Gather system info (hostname, IP, country, version, etc.)

---

## 🔐 Security Design

### API Key Generation
```python
# Backend generates secure API keys
agent_api_key = f"agent_{uuid.uuid4().hex}"
# Example: agent_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

### API Key Storage
```sql
CREATE TABLE agent_api_keys (
    id UUID PRIMARY KEY,
    api_key_hash VARCHAR(255) NOT NULL UNIQUE,  -- Hashed, not plain text
    status VARCHAR(20) DEFAULT 'active',         -- active, revoked, used
    server_id UUID REFERENCES vpn_servers(id),   -- Linked after registration
    created_at TIMESTAMP DEFAULT NOW(),
    used_at TIMESTAMP,                           -- When first used
    expires_at TIMESTAMP,                        -- Optional expiration
    description VARCHAR(255)                     -- Admin notes
);
```

### Registration Flow
```
┌─────────────┐                    ┌─────────────┐                    ┌─────────────┐
│   Agent     │                    │   Backend   │                    │  Database   │
└──────┬──────┘                    └──────┬──────┘                    └──────┬──────┘
       │                                  │                                  │
       │  POST /servers/register-agent    │                                  │
       │  Headers: X-API-Key: agent_...   │                                  │
       │  Body: {                         │                                  │
       │    "hostname": "vps-123",        │                                  │
       │    "ip": "1.2.3.4",              │                                  │
       │    "country": "US",              │                                  │
       │    "version": "0.1.20"           │                                  │
       │  }                               │                                  │
       │─────────────────────────────────>│                                  │
       │                                  │                                  │
       │                                  │  1. Validate API key hash       │
       │                                  │  2. Check not used/revoked      │
       │                                  │─────────────────────────────────>│
       │                                  │                                  │
       │                                  │  3. Create vpn_servers record   │
       │                                  │  4. Mark API key as used        │
       │                                  │<─────────────────────────────────│
       │                                  │                                  │
       │  Response: {                     │                                  │
       │    "server_id": "uuid-...",      │                                  │
       │    "status": "registered"        │                                  │
       │  }                               │                                  │
       │<─────────────────────────────────│                                  │
       │                                  │                                  │
       │  Save server_id to .env          │                                  │
       │                                  │                                  │
```

---

## 🗂️ Data Model

### Agent Registration Metadata

| Field | Type | Source | Example |
|-------|------|--------|---------|
| `hostname` | string | `os.hostname()` | `us-east-1-vps-165-140-241-96` |
| `ip_address` | string | External API | `165.140.241.96` |
| `country_code` | string | GeoIP API | `US` |
| `country_name` | string | GeoIP API | `United States` |
| `region` | string | GeoIP API | `Virginia` |
| `city` | string | GeoIP API | `Ashburn` |
| `agent_version` | string | Build constant | `0.1.20` |
| `os_type` | string | `runtime.GOOS` | `linux` |
| `os_arch` | string | `runtime.GOARCH` | `amd64` |
| `supports_outline` | bool | Config | `true` |
| `supports_wireguard` | bool | Config | `true` |
| `agent_url` | string | Config | `http://usipipousa.duckdns.org:8080` |

### Backend Schema Changes

```sql
-- New table: agent_api_keys
CREATE TABLE agent_api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_key_hash VARCHAR(255) NOT NULL UNIQUE,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    server_id UUID REFERENCES vpn_servers(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    used_at TIMESTAMP WITH TIME ZONE,
    expires_at TIMESTAMP WITH TIME ZONE,
    description VARCHAR(255),
    created_by UUID REFERENCES admin_users(id),
    CONSTRAINT chk_status CHECK (status IN ('active', 'used', 'revoked', 'expired'))
);

CREATE INDEX idx_agent_api_keys_hash ON agent_api_keys(api_key_hash);
CREATE INDEX idx_agent_api_keys_status ON agent_api_keys(status);

-- Modify vpn_servers: add agent metadata columns
ALTER TABLE vpn_servers ADD COLUMN IF NOT EXISTS agent_version VARCHAR(20);
ALTER TABLE vpn_servers ADD COLUMN IF NOT EXISTS os_type VARCHAR(50);
ALTER TABLE vpn_servers ADD COLUMN IF NOT EXISTS os_arch VARCHAR(20);
ALTER TABLE vpn_servers ADD COLUMN IF NOT EXISTS last_registration_ip INET;
```

---

## 🔌 API Endpoints

### 1. Register Agent (Explicit)

**Endpoint:** `POST /api/v1/servers/register-agent`

**Authentication:** `X-API-Key: agent_<key>`

**Request Body:**
```json
{
  "hostname": "us-east-1-vps-165-140-241-96",
  "ip_address": "165.140.241.96",
  "country_code": "US",
  "country_name": "United States",
  "region": "Virginia",
  "city": "Ashburn",
  "agent_version": "0.1.20",
  "os_type": "linux",
  "os_arch": "amd64",
  "supports_outline": true,
  "supports_wireguard": true,
  "agent_url": "http://usipipousa.duckdns.org:8080",
  "agent_api_key": "agent_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"
}
```

**Success Response (201):**
```json
{
  "server_id": "1bc5c426-29de-4440-9ec6-ada7866e2c08",
  "status": "registered",
  "message": "Server registered successfully"
}
```

**Error Responses:**
```json
// 401 Unauthorized - Invalid API key
{"detail": "Invalid agent API key"}

// 409 Conflict - API key already used
{"detail": "Agent API key already used for registration"}

// 400 Bad Request - Invalid metadata
{"detail": "Invalid metadata: country_code must be 2 characters"}
```

---

### 2. Ingest Metrics (Auto-Register)

**Endpoint:** `POST /api/v1/metrics/agents/{server_id}`

**Enhanced Logic:**
```python
async def ingest_agent_metrics(server_id: str, metrics: dict, api_key: str):
    # Case 1: server_id is valid UUID
    if is_valid_uuid(server_id):
        server = await get_server(server_id)
        if not server:
            # Auto-register with provided UUID
            server = await auto_register_server(server_id, metrics, api_key)
    
    # Case 2: server_id is not UUID (hostname or placeholder)
    else:
        # Check if API key already registered
        existing = await get_server_by_api_key(api_key)
        if existing:
            server_id = existing.id
        else:
            # Auto-register and get new UUID
            server = await auto_register_server(None, metrics, api_key)
            server_id = server.id
    
    # Save metrics
    await save_metrics(server_id, metrics)
    return {"status": "ok"}
```

---

### 3. Generate Agent API Key (Admin)

**Endpoint:** `POST /api/v1/admin/agent-api-keys`

**Authentication:** `Authorization: Bearer <admin_jwt>`

**Request Body:**
```json
{
  "description": "USA East VPS #1",
  "expires_at": "2026-12-31T23:59:59Z",  // Optional
  "metadata": {
    "expected_country": "US",
    "expected_ip": "165.140.241.96"
  }
}
```

**Response:**
```json
{
  "id": "uuid-of-key-record",
  "api_key": "agent_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "status": "active",
  "created_at": "2026-03-29T19:30:00Z",
  "expires_at": "2026-12-31T23:59:59Z"
}
```

---

### 4. List Agent API Keys (Admin)

**Endpoint:** `GET /api/v1/admin/agent-api-keys`

**Query Parameters:**
- `status` (optional): `active`, `used`, `revoked`, `expired`
- `limit` (optional): Default 50, max 100

**Response:**
```json
{
  "keys": [
    {
      "id": "uuid-1",
      "status": "active",
      "description": "USA East VPS #1",
      "created_at": "2026-03-29T19:30:00Z",
      "server_id": null  // Not registered yet
    },
    {
      "id": "uuid-2",
      "status": "used",
      "description": "Germany VPS #1",
      "created_at": "2026-03-28T10:00:00Z",
      "used_at": "2026-03-28T10:05:00Z",
      "server_id": "uuid-of-german-server"
    }
  ],
  "total": 2
}
```

---

## 🤖 Agent Implementation

### Configuration Changes

**Old `.env`:**
```env
SERVER_ID=us-east-1-vps-165-140-241-96
AGENT_API_KEY=agent_key_us_east_1_outline_wireguard
```

**New `.env`:**
```env
# Pre-generated API key (from backend admin)
AGENT_API_KEY=agent_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6

# Server ID (auto-filled after registration, empty initially)
SERVER_ID=

# Agent metadata
AGENT_HOSTNAME=us-east-1-vps-165-140-241-96
AGENT_COUNTRY_CODE=US
AGENT_REGION=Virginia
AGENT_CITY=Ashburn
AGENT_URL=http://usipipousa.duckdns.org:8080

# Feature flags
SUPPORTS_OUTLINE=true
SUPPORTS_WIREGUARD=true
```

### Registration Logic (Go)

```go
// In reporter.go or new registrar.go

type Registrar struct {
    backendURL string
    apiKey     string
    serverID   string
    client     *resty.Client
}

func (r *Registrar) RegisterOrGetServerID() (string, error) {
    // If SERVER_ID already set and valid UUID, use it
    if r.serverID != "" && isValidUUID(r.serverID) {
        return r.serverID, nil
    }
    
    // Collect metadata
    metadata := r.collectMetadata()
    
    // Send registration request
    endpoint := fmt.Sprintf("%s/api/v1/servers/register-agent", r.backendURL)
    
    resp, err := r.client.R().
        SetHeader("X-API-Key", r.apiKey).
        SetBody(metadata).
        Post(endpoint)
    
    if err != nil {
        return "", fmt.Errorf("registration failed: %w", err)
    }
    
    if resp.StatusCode() != 201 {
        return "", fmt.Errorf("registration failed with status: %d", resp.StatusCode())
    }
    
    // Parse response
    var result struct {
        ServerID string `json:"server_id"`
        Status   string `json:"status"`
    }
    json.Unmarshal(resp.Body(), &result)
    
    // Save SERVER_ID to .env
    if err := saveServerIDToEnv(result.ServerID); err != nil {
        log.Printf("Warning: Could not save SERVER_ID to .env: %v", err)
    }
    
    return result.ServerID, nil
}

func (r *Registrar) collectMetadata() map[string]interface{} {
    hostname, _ := os.Hostname()
    
    // Get public IP and geo data
    ip, country, region, city := r.getGeoData()
    
    return map[string]interface{}{
        "hostname":           hostname,
        "ip_address":         ip,
        "country_code":       country,
        "country_name":       getCountryName(country),
        "region":             region,
        "city":               city,
        "agent_version":      Version,  // Build constant
        "os_type":            runtime.GOOS,
        "os_arch":            runtime.GOARCH,
        "supports_outline":   os.Getenv("SUPPORTS_OUTLINE") == "true",
        "supports_wireguard": os.Getenv("SUPPORTS_WIREGUARD") == "true",
        "agent_url":          os.Getenv("AGENT_URL"),
        "agent_api_key":      r.apiKey,
    }
}

func (r *Registrar) getGeoData() (ip, country, region, city string) {
    // Use free GeoIP API (e.g., ipapi.co, ip-api.com)
    resp, err := r.client.R().Get("http://ip-api.com/json/")
    if err != nil {
        return "unknown", "XX", "Unknown", "Unknown"
    }
    
    var geo struct {
        Query       string `json:"query"`
        CountryCode string `json:"countryCode"`
        CountryName string `json:"countryName"`
        RegionName  string `json:"regionName"`
        City        string `json:"city"`
    }
    json.Unmarshal(resp.Body(), &geo)
    
    return geo.Query, geo.CountryCode, geo.RegionName, geo.City
}
```

### Metrics Sending Logic

```go
// Modified sendMetrics in reporter.go

func (r *Reporter) sendMetrics() {
    // Ensure we have a valid server_id
    if r.serverID == "" || !isValidUUID(r.serverID) {
        log.Println("Server ID not set, attempting registration...")
        
        registrar := NewRegistrar(r.backendURL, r.apiKey, r.serverID)
        serverID, err := registrar.RegisterOrGetServerID()
        if err != nil {
            log.Printf("Failed to register: %v", err)
            return
        }
        
        r.serverID = serverID
        log.Printf("Registered with server_id: %s", serverID)
    }
    
    // Now send metrics as usual
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

---

## 📝 Implementation Tasks

### Backend Tasks

1. **Create migration:** `alembic revision --autogenerate -m "Add agent_api_keys table"`
2. **Create model:** `src/infrastructure/persistence/models/agent_api_key_model.py`
3. **Create repository:** `src/infrastructure/persistence/repositories/agent_api_key_repository.py`
4. **Create service:** `src/core/application/services/agent_registration_service.py`
5. **Create schema:** `src/shared/schemas/agent_registration.py`
6. **Create routes:** `src/infrastructure/api/v1/routes/agent_registration.py`
7. **Update metrics route:** Modify to support auto-registration
8. **Admin routes:** Add endpoints for key generation and listing
9. **Tests:** Unit + integration tests for registration flow

### Agent Tasks

1. **Create registrar:** `internal/registrar/registrar.go`
2. **Add GeoIP:** `internal/utils/geoip.go`
3. **Update config:** Load new env vars in `internal/config/config.go`
4. **Update reporter:** Integrate registration in `internal/reporter/reporter.go`
5. **Env file writer:** `internal/config/env_writer.go` to save SERVER_ID
6. **Version constant:** Add `Version` in `cmd/agent/main.go`
7. **Tests:** Unit tests for registration logic

### Documentation Tasks

1. **Admin guide:** How to generate API keys
2. **Deployment guide:** Updated installation steps
3. **API docs:** OpenAPI spec for new endpoints
4. **Agent docs:** Configuration reference

---

## 🧪 Testing Plan

### Backend Tests
```python
# Test agent registration
async def test_register_agent_with_valid_key():
    api_key = await generate_agent_api_key()
    response = await client.post(
        "/api/v1/servers/register-agent",
        headers={"X-API-Key": api_key},
        json={...metadata...}
    )
    assert response.status_code == 201
    assert "server_id" in response.json()

# Test duplicate registration
async def test_register_agent_with_used_key():
    api_key = await generate_agent_api_key()
    # First registration
    await register_agent(api_key)
    # Second registration should fail
    response = await register_agent(api_key)
    assert response.status_code == 409

# Test metrics with auto-registration
async def test_metrics_auto_registers_unknown_server():
    api_key = await generate_agent_api_key()
    response = await client.post(
        "/api/v1/metrics/agents/placeholder",
        headers={"X-API-Key": api_key},
        json={...metrics...}
    )
    assert response.status_code == 200
    # Verify server was created
    server = await get_server_by_api_key(api_key)
    assert server is not None
```

### Agent Tests
```go
func TestRegistrar_RegistrationSuccess(t *testing.T) {
    registrar := NewRegistrar(backendURL, apiKey, "")
    serverID, err := registrar.RegisterOrGetServerID()
    
    assert.NoError(t, err)
    assert.NotEmpty(t, serverID)
    assert.True(t, isValidUUID(serverID))
}

func TestRegistrar_SaveServerIDToEnv(t *testing.T) {
    err := saveServerIDToEnv("test-uuid")
    assert.NoError(t, err)
    
    // Verify .env was updated
    env, _ := config.Load()
    assert.Equal(t, "test-uuid", env.ServerID)
}
```

---

## 🚀 Deployment Plan

### Phase 1: Backend Deployment
1. Deploy database migration
2. Deploy backend code
3. Generate API keys for existing servers
4. Test registration endpoint

### Phase 2: Agent Update
1. Build agent v0.2.0 with registration support
2. Update `.env` on test server:
   - Remove `SERVER_ID` (or leave empty)
   - Add `AGENT_API_KEY=<newly-generated>`
3. Restart agent
4. Verify registration in DB
5. Verify metrics flowing

### Phase 3: Rollout
1. Generate API keys for all existing VPS
2. Deploy updated agent to all VPS
3. Monitor registration logs
4. Clean up manual SERVER_ID entries

---

## 🔒 Security Considerations

1. **API Key Hashing:** Store bcrypt hash, not plain text
2. **Rate Limiting:** Limit registration attempts per IP
3. **IP Validation:** Optionally validate expected IP matches actual IP
4. **Key Expiration:** Support expiration for temporary deployments
5. **Audit Log:** Log all registration attempts (success/failure)
6. **Admin Only:** Only admins can generate API keys

---

## 📊 Success Metrics

- [ ] Agent can register with valid API key
- [ ] Duplicate registration rejected (409)
- [ ] Invalid API key rejected (401)
- [ ] Metrics endpoint auto-registers unknown servers
- [ ] SERVER_ID saved to .env after registration
- [ ] Admin can list generated keys
- [ ] Admin can revoke keys
- [ ] Existing servers can migrate to new flow

---

**Design approved for implementation.** Next step: Invoke `writing-plans` skill for detailed implementation plan.
