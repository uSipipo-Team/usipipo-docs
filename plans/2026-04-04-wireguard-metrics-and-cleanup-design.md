# WireGuard Metrics Collection + Metrics Cleanup - Design Doc

**Date:** 2026-04-04
**Status:** Approved
**Feature:** WireGuard peer metrics collection, server name fix, and 30-day metrics cleanup

---

## Problems Identified

| # | Problem | Root Cause |
|---|---------|------------|
| 1 | `🖥️ N/A` in WireGuard key details | Backend `VpnKeyResponse` has no `server_name` field, only `server_id` (UUID) |
| 2 | "Métricas no disponibles" for WireGuard keys | Agent only collects Outline metrics, not WireGuard peer metrics |
| 3 | Database grows indefinitely | No cleanup job for old metrics |

---

## Solution Overview

### Part 1: Fix Server Name Display (Backend)

Add `server_name: str | None` to `VpnKeyResponse` schema, populated by resolving `server_id` to server name in route handlers.

### Part 2: WireGuard Metrics Collection (Agent)

Create `WireGuardMetricsCollector` using the existing `wgctrl` library to collect:
- Peer count and details
- Bytes received/sent per peer
- Last handshake time
- Connection status (connected if handshake < 5 min ago)

### Part 3: WireGuard Metrics Storage (Backend)

New `wireguard_metrics` table + REST API endpoint for bot consumption.

### Part 4: Bot Conditional Display

Show Outline metrics for Outline keys, WireGuard peer metrics for WireGuard keys.

### Part 5: 30-Day Metrics Cleanup

Automated cleanup job deleting metrics older than 30 days from both `outline_metrics` and `wireguard_metrics` tables.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    uSipipo Agent (Go)                    │
│  ┌──────────────────┐    ┌──────────────────────────┐   │
│  │  OutlineClient   │    │ WireGuardMetricsCollector│   │
│  │  (existing)      │    │ (NEW - uses wgctrl)      │   │
│  └────────┬─────────┘    └────────────┬─────────────┘   │
│           │                           │                 │
│  ┌────────▼───────────────────────────▼──────────────┐  │
│  │           MetricsCollector                         │  │
│  │  Collects both Outline + WireGuard metrics         │  │
│  └──────────────────────┬────────────────────────────┘  │
│                         │ POST /metrics                  │
└─────────────────────────┼───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│                   uSipipo Backend                        │
│  ┌──────────────────┐    ┌──────────────────────────┐   │
│  │ outline_metrics  │    │ wireguard_metrics (NEW)  │   │
│  │ (existing table) │    │ (new table)              │   │
│  └────────┬─────────┘    └────────────┬─────────────┘   │
│           │                           │                 │
│  ┌────────▼───────────────────────────▼──────────────┐  │
│  │  GET /vpn/servers/{id}/outline/metrics            │  │
│  │  GET /vpn/servers/{id}/wireguard/metrics (NEW)    │  │
│  │  POST /admin/metrics/cleanup (NEW - 30 day)       │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│              Telegram Bot (Python)                       │
│  ┌──────────────────────────────────────────────────┐   │
│  │  if key_type == "outline":                       │   │
│  │      → fetch outline metrics                     │   │
│  │  elif key_type == "wireguard":                   │   │
│  │      → fetch wireguard peer metrics (NEW)        │   │
│  └───────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## WireGuard Metrics Data Model

### Agent Side

```go
type WireGuardPeerMetrics struct {
    PeerCount int
    Peers     []PeerDetail
}

type PeerDetail struct {
    PublicKey     string
    BytesReceived uint64    // rx bytes
    BytesSent     uint64    // tx bytes
    LastHandshake time.Time
    IsConnected   bool      // true if last handshake < 5 min ago
    AllowedIPs    []string
}
```

### Backend Side

```sql
CREATE TABLE wireguard_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    server_id UUID NOT NULL REFERENCES vpn_servers(id),
    peer_public_key TEXT NOT NULL,
    bytes_received BIGINT,
    bytes_sent BIGINT,
    last_handshake TIMESTAMPTZ,
    is_connected BOOLEAN,
    collected_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_wg_metrics_server_peer_time 
    ON wireguard_metrics(server_id, peer_public_key, collected_at DESC);
```

---

## Message Templates

### WireGuard Key Details (Enhanced)

```
💎 pigpong

📡 WIREGUARD • 🖥️ USA East 1
━━━━━━━━━━━━━

📊 Tu Consumo: 0.0/5.0GB (0%)
░░░░░░░░░░░░░░░░░░░░ 0%

🟢 Estado: Activa
📅 Expira: 2026-05-04
🕐 Última vez: Hace 2 horas

━━━━━━━━━━━━━
🌐 Estado del Servidor
🟢 Conectado • 📥 1.2GB 📤 3.4GB
🤝 Last handshake: Hace 15 minutos
━━━━━━━━━━━━━

⚡ Acciones:
```

### Outline Key Details (Unchanged)

```
💎 mykey

📡 OUTLINE • 🖥️ USA East 1
━━━━━━━━━━━━━

📊 Tu Consumo: 1.5/5.0GB (30%)
██████░░░░░░░░░░░░░░ 30%

🟢 Estado: Activa
📅 Expira: 2026-05-04
🕐 Última vez: Hace 1 hora

━━━━━━━━━━━━━
🌐 Estado del Servidor
🟢 Online • 27 claves activas
📈 18.9 GB transferidos (total)
⏱️ 99%+ uptime (30 días)
━━━━━━━━━━━━━

⚡ Acciones:
```

---

## API Endpoints

### New Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/v1/vpn/servers/{id}/wireguard/metrics` | Get WireGuard peer metrics for a server |
| POST | `/api/v1/admin/metrics/cleanup` | Delete metrics older than 30 days |

### Modified Endpoints

| Endpoint | Change |
|----------|--------|
| `GET /api/v1/vpn/keys` | Include `server_name` in response |
| `GET /api/v1/vpn/keys/{id}` | Include `server_name` in response |

---

## Error Handling

| Scenario | Behavior |
|----------|----------|
| WireGuard interface down | Show "🔴 WireGuard no disponible" |
| No peers found | Show "📭 Sin peers conectados" |
| Metrics fetch fails | Show "📡 Métricas no disponibles" |
| Cleanup job fails | Log error, don't block other operations |

---

## Testing Strategy

### Agent (Go)
- Unit tests for `WireGuardMetricsCollector`
- Mock `wgctrl` client for testing
- Test connection status logic (< 5 min threshold)

### Backend (Python)
- Unit tests for `wireguard_metrics` CRUD
- Integration tests for new endpoints
- Test cleanup job with sample data

### Bot (Python)
- Integration tests for conditional metrics display
- Test WireGuard key details message formatting
- Test Outline key details unchanged (regression)

---

## Implementation Order

1. **Backend:** Add `server_name` to `VpnKeyResponse` (fix N/A)
2. **Agent:** Create `WireGuardMetricsCollector`
3. **Agent:** Integrate into `MetricsCollector` and payload
4. **Backend:** Create `wireguard_metrics` table + endpoint
5. **Backend:** Add metrics cleanup job
6. **Bot:** Conditional metrics display (Outline vs WireGuard)
7. **Tests:** Unit + integration tests for all components
