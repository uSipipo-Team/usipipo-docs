# Outline Metrics Integration Design

**Date:** 2026-04-02  
**Author:** uSipipo Team  
**Status:** Approved  
**Related Projects:** usipipo-agent, usipipo-backend  

---

## 📋 Overview

This document describes the design for integrating real-time Outline VPN server metrics collection into the uSipipo ecosystem. The implementation enables comprehensive monitoring of Outline server health, usage statistics, and performance metrics across all client platforms (Telegram Bot, Mini Web App, Android, Desktop).

---

## 🎯 Objectives

### Primary Goals
1. **Real-time Monitoring** - Collect actual Outline server status (online/offline) via API health checks
2. **Usage Analytics** - Track bandwidth consumption per access key and aggregate totals
3. **Historical Data** - Store time-series metrics for trend analysis and reporting
4. **Multi-platform Access** - Expose metrics via API for all frontend clients

### Non-Goals (Out of Scope)
- Dashboard UI implementation (frontend responsibility)
- Alert/notification system (future phase)
- Automated scaling based on metrics (future phase)

---

## 🏗️ Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────┐
│                    uSipipo Agent (Go)                        │
│  Location: /opt/usipipo-agent/                               │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Outline Metrics Collector (NEW)                     │   │
│  │  ├─ CheckStatus() → GET /server                      │   │
│  │  ├─ GetTransferMetrics() → GET /metrics/transfer     │   │
│  │  └─ GetDetailedMetrics() → GET /experimental/...     │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          ▼                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Enhanced Metrics Reporter                           │   │
│  │  POST /api/v1/metrics/agents/{server_id}             │   │
│  │  Interval: Every 1 minute (includes Outline data)    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ HTTPS POST every 1 minute
                          │ Payload: {system, vpn, outline, latency}
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                  uSipipo Backend (FastAPI)                   │
│  Location: /home/mowgli/usipipo/usipipo-backend/             │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  MetricsService.ingest_agent_metrics()               │   │
│  │  ├─ Validate metrics (range checks)                  │   │
│  │  ├─ Extract Outline data                             │   │
│  │  └─ Store in database (atomic transaction)           │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          ▼                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Database Tables                                     │   │
│  │  ├─ server_metrics (existing)                        │   │
│  │  │   └─ System metrics (CPU, memory, disk, network)  │   │
│  │  └─ outline_metrics (NEW)                            │   │
│  │      ├─ Basic metrics (every 5 min)                  │   │
│  │      └─ Detailed metrics (every 1 hour)              │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          ▼                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  REST API Endpoints (NEW)                            │   │
│  │  GET /api/v1/vpn/servers/{id}/outline                │   │
│  │  GET /api/v1/vpn/servers/{id}/outline/metrics        │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ REST API (JWT authenticated)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Frontend Clients                          │
│  - Telegram Bot (Python) - /stats command                   │
│  - Mini Web App (React/Next.js) - Dashboard                 │
│  - Android App (Kotlin) - Metrics screen                    │
│  - Desktop App (TBD)                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 Data Collection Strategy

### Hybrid Collection Schedule

| Metric Type | Endpoint | Frequency | Purpose |
|-------------|----------|-----------|---------|
| **Basic Status** | `GET /server` | Every 5 minutes | Server health, version, config |
| **Transfer Metrics** | `GET /metrics/transfer` | Every 5 minutes | Total bytes per key (30 days) |
| **Detailed Metrics** | `GET /experimental/server/metrics?since=24h` | Every 1 hour | Time-series data for charts |
| **Access Keys List** | `GET /access-keys` | Every 5 minutes | Active keys count, metadata |

### Rationale for Hybrid Approach

- **5-minute interval**: Balances freshness with API load (288 calls/day vs 1440)
- **1-hour interval**: Detailed time-series don't need minute-by-minute updates
- **Aligned with existing**: Agent already sends system metrics every 1 minute
- **Outline API friendly**: Avoids rate limiting from excessive requests

---

## 🔧 Implementation Details

### Agent Changes (usipipo-agent)

#### 1. New OutlineClient Methods

**File:** `internal/vpn/outline.go`

```go
// CheckStatus verifies Outline API connectivity and returns server info
func (c *OutlineClient) CheckStatus(ctx context.Context) (*OutlineServerInfo, error)

// GetTransferMetrics retrieves bandwidth usage per key (last 30 days)
func (c *OutlineClient) GetTransferMetrics(ctx context.Context) (*OutlineTransferMetrics, error)

// GetDetailedMetrics retrieves time-series metrics for specified period
func (c *OutlineClient) GetDetailedMetrics(ctx context.Context, since string) (*OutlineDetailedMetrics, error)

// GetActiveKeysCount returns number of active access keys
func (c *OutlineClient) GetActiveKeysCount(ctx context.Context) (int, error)
```

#### 2. New Metrics Structures

**File:** `internal/vpn/outline.go`

```go
type OutlineServerInfo struct {
    Name                      string `json:"name"`
    ServerID                  string `json:"serverId"`
    MetricsEnabled            bool   `json:"metricsEnabled"`
    Version                   string `json:"version"`
    PortForNewAccessKeys      int    `json:"portForNewAccessKeys"`
    HostnameForAccessKeys     string `json:"hostnameForAccessKeys"`
}

type OutlineTransferMetrics struct {
    BytesTransferredByUserID map[string]uint64 `json:"bytesTransferredByUserId"`
}

type OutlineDetailedMetrics struct {
    Status     string        `json:"status"`
    Data       MetricsData   `json:"data"`
    ResultType string        `json:"resultType"`
    Result     []MetricResult `json:"result"`
}

type MetricResult struct {
    Metric MetricInfo `json:"metric"`
    Values [][]interface{} `json:"values"` // [timestamp, value]
}

type MetricInfo struct {
    AccessKey string `json:"access_key"`
    Name      string `json:"__name__"`
}
```

#### 3. Enhanced Metrics Collector

**File:** `internal/metrics/collector.go`

```go
type Collector struct {
    serverID         string
    cache            *ServerMetrics
    cacheTime        time.Time
    cacheTTL         time.Duration
    outlineCache     *OutlineMetrics
    outlineCacheTime time.Time
    outlineTTL       time.Duration // 5 minutes
    detailedCache    *OutlineDetailedMetrics
    detailedCacheTime time.Time
    detailedTTL      time.Duration // 1 hour
}

// GetOutlineMetrics collects Outline-specific metrics
func (c *Collector) GetOutlineMetrics(ctx context.Context) (*OutlineMetrics, error)

// GetDetailedOutlineMetrics collects detailed time-series metrics
func (c *Collector) GetDetailedOutlineMetrics(ctx context.Context) (*OutlineDetailedMetrics, error)
```

#### 4. Updated Metrics Handler

**File:** `internal/api/handlers.go`

```go
// MetricsHandler returns detailed system metrics including Outline
func MetricsHandler(c *gin.Context) {
    // Collect system metrics (existing)
    systemMetrics, err := metricsCollector.GetMetrics(c.Request.Context())
    
    // Collect Outline metrics (NEW)
    outlineMetrics, err := metricsCollector.GetOutlineMetrics(c.Request.Context())
    
    // Collect detailed metrics if interval > 1 hour (NEW)
    var detailedMetrics *OutlineDetailedMetrics
    if time.Since(metricsCollector.lastDetailedCollection) > 1*time.Hour {
        detailedMetrics, _ = metricsCollector.GetDetailedOutlineMetrics(c.Request.Context())
        metricsCollector.lastDetailedCollection = time.Now()
    }
    
    // Combine all metrics
    response := MetricsResponse{
        ServerID:  c.serverID,
        Timestamp: time.Now(),
        System:    systemMetrics.System,
        VPN:       systemMetrics.VPN,
        Outline:   outlineMetrics,
        Detailed:  detailedMetrics,
        Latency:   systemMetrics.Latency,
    }
    
    c.JSON(http.StatusOK, response)
}
```

#### 5. Scheduler for Hybrid Collection

**File:** `internal/reporter/reporter.go`

```go
type Reporter struct {
    // ... existing fields ...
    lastOutlineCollection  time.Time
    lastDetailedCollection time.Time
}

// Start starts the metrics reporting loop
func (r *Reporter) Start() {
    ticker := time.NewTicker(r.interval)
    outlineTicker := time.NewTicker(5 * time.Minute)
    detailedTicker := time.NewTicker(1 * time.Hour)
    
    go func() {
        for {
            select {
            case <-ticker.C:
                // Send system metrics every 1 minute (existing)
                go r.sendMetrics()
                
            case <-outlineTicker.C:
                // Collect Outline basic metrics every 5 minutes (NEW)
                go r.collectOutlineMetrics()
                
            case <-detailedTicker.C:
                // Collect detailed metrics every 1 hour (NEW)
                go r.collectDetailedMetrics()
            }
        }
    }()
}
```

---

### Backend Changes (usipipo-backend)

#### 1. New Database Model

**File:** `src/infrastructure/persistence/models/outline_metric_model.py`

```python
from sqlalchemy import Column, String, BigInteger, Boolean, DateTime, ForeignKey, Index
from sqlalchemy.dialects.postgresql import JSONB
from src.infrastructure.persistence.database import Base

class OutlineMetricModel(Base):
    """Time-series metrics for Outline VPN servers.
    
    Stores both basic metrics (every 5 min) and detailed metrics (every 1 hour).
    """
    __tablename__ = "outline_metrics"
    
    id = Column(String, primary_key=True)  # UUID
    server_id = Column(String, ForeignKey("vpn_servers.id"), nullable=False, index=True)
    timestamp = Column(DateTime, nullable=False, index=True)
    
    # Basic metrics (collected every 5 minutes)
    server_status = Column(String, nullable=True)  # online, offline, error
    server_version = Column(String, nullable=True)
    server_name = Column(String, nullable=True)
    active_keys_count = Column(BigInteger, nullable=True)
    total_bytes_transferred = Column(BigInteger, nullable=True, default=0)
    port_for_new_access_keys = Column(BigInteger, nullable=True)
    hostname_for_access_keys = Column(String, nullable=True)
    
    # Detailed metrics (collected every 1 hour)
    time_series_24h = Column(JSONB, nullable=True)  # [{timestamp, key_id, bytes}]
    top_consumers = Column(JSONB, nullable=True)  # [{key_id, bytes, name}]
    
    # Health check tracking
    outline_api_reachable = Column(Boolean, nullable=True)
    last_successful_check = Column(DateTime, nullable=True)
    last_error = Column(String, nullable=True)
    consecutive_failures = Column(BigInteger, nullable=True, default=0)
    
    # Metadata
    created_at = Column(DateTime, nullable=False, default=datetime.utcnow)
    
    __table_args__ = (
        Index("idx_outline_metrics_server_timestamp", "server_id", "timestamp DESC"),
        Index("idx_outline_metrics_status", "server_status"),
    )
```

#### 2. Migration Script

**File:** `migrations/versions/2026_04_02_0000_create_outline_metrics_table.py`

```python
"""create outline_metrics table

Revision ID: 2026_04_02_0000
Revises: encrypt_existing_agent_api_keys
Create Date: 2026-04-02 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

def upgrade() -> None:
    op.create_table(
        'outline_metrics',
        sa.Column('id', sa.String(), nullable=False),
        sa.Column('server_id', sa.String(), nullable=False),
        sa.Column('timestamp', sa.DateTime(), nullable=False),
        sa.Column('server_status', sa.String(), nullable=True),
        sa.Column('server_version', sa.String(), nullable=True),
        sa.Column('server_name', sa.String(), nullable=True),
        sa.Column('active_keys_count', sa.BigInteger(), nullable=True),
        sa.Column('total_bytes_transferred', sa.BigInteger(), nullable=True),
        sa.Column('port_for_new_access_keys', sa.BigInteger(), nullable=True),
        sa.Column('hostname_for_access_keys', sa.String(), nullable=True),
        sa.Column('time_series_24h', postgresql.JSONB(astext_type=sa.Text()), nullable=True),
        sa.Column('top_consumers', postgresql.JSONB(astext_type=sa.Text()), nullable=True),
        sa.Column('outline_api_reachable', sa.Boolean(), nullable=True),
        sa.Column('last_successful_check', sa.DateTime(), nullable=True),
        sa.Column('last_error', sa.String(), nullable=True),
        sa.Column('consecutive_failures', sa.BigInteger(), nullable=True),
        sa.Column('created_at', sa.DateTime(), nullable=False),
        sa.PrimaryKeyConstraint('id')
    )
    
    # Create indexes
    op.create_index('idx_outline_metrics_server_timestamp', 'outline_metrics', ['server_id', sa.text('timestamp DESC')])
    op.create_index('idx_outline_metrics_status', 'outline_metrics', ['server_status'])

def downgrade() -> None:
    op.drop_index('idx_outline_metrics_status', table_name='outline_metrics')
    op.drop_index('idx_outline_metrics_server_timestamp', table_name='outline_metrics')
    op.drop_table('outline_metrics')
```

#### 3. Updated Metrics Service

**File:** `src/core/application/services/metrics_service.py`

```python
async def ingest_agent_metrics(
    self,
    server_id: uuid.UUID,
    metrics: dict,
) -> None:
    """Ingest metrics from a VPN agent.
    
    Now includes Outline-specific metrics in the payload.
    
    Args:
        server_id: UUID of the server sending metrics
        metrics: Metrics payload from agent
            {
                "system": {...},
                "vpn": {...},
                "outline": {
                    "server_status": "online",
                    "server_version": "1.10.0",
                    "active_keys_count": 42,
                    "total_bytes_transferred": 107374182400,
                    "time_series_24h": [...],
                    "top_consumers": [...],
                    "outline_api_reachable": true,
                    "consecutive_failures": 0
                },
                "latency_ms": {...}
            }
    """
    # ... existing system metrics validation ...
    
    # Extract and validate Outline metrics (NEW)
    outline_data = metrics.get("outline", {})
    
    outline_metric = OutlineMetricModel(
        id=str(uuid.uuid4()),
        server_id=str(server_id),
        timestamp=datetime.utcnow(),
        server_status=outline_data.get("server_status"),
        server_version=outline_data.get("server_version"),
        server_name=outline_data.get("server_name"),
        active_keys_count=outline_data.get("active_keys_count"),
        total_bytes_transferred=outline_data.get("total_bytes_transferred", 0),
        port_for_new_access_keys=outline_data.get("port_for_new_access_keys"),
        hostname_for_access_keys=outline_data.get("hostname_for_access_keys"),
        time_series_24h=outline_data.get("time_series_24h"),
        top_consumers=outline_data.get("top_consumers"),
        outline_api_reachable=outline_data.get("outline_api_reachable"),
        last_successful_check=datetime.utcnow() if outline_data.get("outline_api_reachable") else None,
        last_error=outline_data.get("last_error"),
        consecutive_failures=outline_data.get("consecutive_failures", 0),
    )
    
    self.session.add(outline_metric)
    await self.session.commit()
    
    logger.info(f"Ingested Outline metrics for server {server_id}: "
                f"status={outline_metric.server_status}, "
                f"keys={outline_metric.active_keys_count}, "
                f"bytes={outline_metric.total_bytes_transferred}")
```

#### 4. New API Endpoints

**File:** `src/infrastructure/api/v1/routes/vpn.py`

```python
@router.get("/servers/{server_id}/outline")
async def get_outline_status(
    server_id: uuid.UUID,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> OutlineStatusResponse:
    """Get current Outline server status.
    
    Returns the latest Outline metrics for a specific server.
    
    **Authentication:** Required (JWT)
    **Authorization:** User must own VPN keys on this server
    """
    # Verify user has access to this server
    user_keys = await vpn_service.get_user_keys_for_server(db, current_user.id, server_id)
    if not user_keys:
        raise HTTPException(status_code=403, detail="Access denied to this server")
    
    # Get latest Outline metrics
    query = select(OutlineMetricModel).where(
        OutlineMetricModel.server_id == str(server_id)
    ).order_by(desc(OutlineMetricModel.timestamp)).limit(1)
    
    result = await db.execute(query)
    metric = result.scalar_one_or_none()
    
    if not metric:
        raise HTTPException(status_code=404, detail="No Outline metrics found")
    
    return OutlineStatusResponse(
        server_id=server_id,
        server_status=metric.server_status,
        server_version=metric.server_version,
        server_name=metric.server_name,
        active_keys_count=metric.active_keys_count,
        total_bytes_transferred=metric.total_bytes_transferred,
        outline_api_reachable=metric.outline_api_reachable,
        last_successful_check=metric.last_successful_check,
        last_error=metric.last_error,
        consecutive_failures=metric.consecutive_failures,
        timestamp=metric.timestamp,
    )


@router.get("/servers/{server_id}/outline/metrics")
async def get_outline_metrics(
    server_id: uuid.UUID,
    since: str = Query(default="24h", description="Time range: 1h, 24h, 7d, 30d"),
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> OutlineMetricsResponse:
    """Get historical Outline metrics.
    
    Returns time-series Outline metrics for a specific time range.
    
    **Authentication:** Required (JWT)
    **Authorization:** User must own VPN keys on this server
    """
    # Parse since parameter
    time_delta = parse_time_range(since)  # e.g., "24h" -> timedelta(hours=24)
    start_time = datetime.utcnow() - time_delta
    
    # Query metrics
    query = select(OutlineMetricModel).where(
        OutlineMetricModel.server_id == str(server_id),
        OutlineMetricModel.timestamp >= start_time,
    ).order_by(desc(OutlineMetricModel.timestamp))
    
    result = await db.execute(query)
    metrics = result.scalars().all()
    
    # Transform into time-series format
    time_series = []
    for metric in metrics:
        if metric.time_series_24h:
            time_series.extend(metric.time_series_24h)
    
    return OutlineMetricsResponse(
        server_id=server_id,
        time_range=since,
        data_points=len(time_series),
        time_series=time_series,
        summary={
            "total_bytes": sum(m.total_bytes_transferred for m in metrics),
            "avg_active_keys": sum(m.active_keys_count or 0 for m in metrics) / len(metrics) if metrics else 0,
            "uptime_percent": calculate_uptime(metrics),
        }
    )
```

#### 5. New Schemas

**File:** `src/shared/schemas/vpn.py`

```python
from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any
from datetime import datetime

class OutlineStatusResponse(BaseModel):
    """Current Outline server status."""
    server_id: UUID
    server_status: str
    server_version: Optional[str] = None
    server_name: Optional[str] = None
    active_keys_count: Optional[int] = None
    total_bytes_transferred: Optional[int] = None
    outline_api_reachable: Optional[bool] = None
    last_successful_check: Optional[datetime] = None
    last_error: Optional[str] = None
    consecutive_failures: Optional[int] = None
    timestamp: datetime


class OutlineMetricsResponse(BaseModel):
    """Historical Outline metrics."""
    server_id: UUID
    time_range: str
    data_points: int
    time_series: List[Dict[str, Any]]
    summary: Dict[str, Any]
```

---

## 📡 API Payload Examples

### Agent → Backend (POST /api/v1/metrics/agents/{server_id})

```json
{
  "server_id": "1bc5c426-29de-4440-9ec6-ada7866e2c08",
  "timestamp": "2026-04-02T08:30:00Z",
  "system": {
    "cpu_percent": 45.2,
    "memory_percent": 62.5,
    "disk_percent": 38.1,
    "network_rx_bytes": 1073741824,
    "network_tx_bytes": 5368709120
  },
  "vpn": {
    "outline": {
      "active_keys": 42,
      "total_bytes_transferred": 107374182400
    },
    "wireguard": {
      "active_peers": 37,
      "total_bytes_transferred": 53687091200
    }
  },
  "outline": {
    "server_status": "online",
    "server_version": "1.10.0",
    "server_name": "USA East 1",
    "active_keys_count": 42,
    "total_bytes_transferred": 107374182400,
    "port_for_new_access_keys": 443,
    "hostname_for_access_keys": "vpn.usa.east.example.com",
    "time_series_24h": [
      {
        "metric": {"access_key": "1", "__name__": "shadowsocks_data_bytes"},
        "values": [[1704672000, "5242880"], [1704675600, "10485760"]]
      }
    ],
    "top_consumers": [
      {"key_id": "2", "bytes": 10485760000, "name": "VIP User"},
      {"key_id": "1", "bytes": 5242880000, "name": "Free User"}
    ],
    "outline_api_reachable": true,
    "last_successful_check": "2026-04-02T08:30:00Z",
    "last_error": null,
    "consecutive_failures": 0
  },
  "latency_ms": {
    "avg": 45.2,
    "p95": 78.5,
    "p99": 120.3
  }
}
```

### Backend → Frontend (GET /api/v1/vpn/servers/{id}/outline)

```json
{
  "server_id": "1bc5c426-29de-4440-9ec6-ada7866e2c08",
  "server_status": "online",
  "server_version": "1.10.0",
  "server_name": "USA East 1",
  "active_keys_count": 42,
  "total_bytes_transferred": 107374182400,
  "outline_api_reachable": true,
  "last_successful_check": "2026-04-02T08:30:00Z",
  "last_error": null,
  "consecutive_failures": 0,
  "timestamp": "2026-04-02T08:30:00Z"
}
```

---

## 🔒 Security Considerations

### Authentication
- All API endpoints require JWT authentication
- Users can only access metrics for servers where they own VPN keys

### Data Protection
- Outline API credentials stored encrypted in agent `.env`
- Metrics transmitted over HTTPS only
- No sensitive data (passwords, access URLs) exposed in metrics API

### Rate Limiting
- Agent → Backend: 1 request/minute (existing rate limit)
- Backend → Outline: 5-minute and 1-hour intervals prevent rate limiting

---

## 🧪 Testing Strategy

### Agent Tests

**File:** `internal/vpn/outline_test.go`

```go
func TestOutlineClient_CheckStatus_Success(t *testing.T) {
    // Mock Outline API response
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        assert.Equal(t, "/server", r.URL.Path)
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]interface{}{
            "name": "Test Server",
            "serverId": "test-uuid",
            "version": "1.10.0",
        })
    }))
    defer server.Close()
    
    client := NewOutlineClient(server.URL, false)
    info, err := client.CheckStatus(context.Background())
    
    assert.NoError(t, err)
    assert.Equal(t, "Test Server", info.Name)
    assert.Equal(t, "1.10.0", info.Version)
}

func TestOutlineClient_GetTransferMetrics(t *testing.T) {
    // Test implementation
}
```

### Backend Tests

**File:** `tests/unit/services/test_metrics_service_outline.py`

```python
async def test_ingest_outline_metrics_success():
    """Test successful ingestion of Outline metrics."""
    # Arrange
    server_id = uuid.uuid4()
    metrics_payload = {
        "outline": {
            "server_status": "online",
            "active_keys_count": 42,
            "total_bytes_transferred": 107374182400,
        }
    }
    
    # Act
    await metrics_service.ingest_agent_metrics(server_id, metrics_payload)
    
    # Assert
    # Verify database record created
```

**File:** `tests/integration/test_outline_metrics_api.py`

```python
async def test_get_outline_status_authenticated(client, auth_headers, test_server):
    """Test getting Outline status with valid authentication."""
    response = await client.get(
        f"/api/v1/vpn/servers/{test_server.id}/outline",
        headers=auth_headers,
    )
    assert response.status_code == 200
    assert response.json()["server_status"] == "online"
```

---

## 📈 Monitoring and Observability

### Agent Logs

```
2026-04-02 08:30:00 INFO  Outline metrics collected: status=online, keys=42, bytes=107374182400
2026-04-02 08:30:01 INFO  Metrics sent to backend successfully (200 OK)
2026-04-02 08:35:00 ERROR Outline API unreachable: connection timeout
2026-04-02 08:35:01 WARN  Consecutive failures: 1
```

### Backend Logs

```
2026-04-02 08:30:05 INFO  Ingested Outline metrics for server 1bc5c426: status=online, keys=42
2026-04-02 08:30:05 DEBUG Stored outline_metric id=abc123 server_id=1bc5c426
```

### Key Metrics to Track

1. **Collection Success Rate**: % of successful Outline API calls
2. **Data Freshness**: Time since last successful metric collection
3. **API Latency**: Response time from Outline API
4. **Storage Growth**: Database size growth rate for outline_metrics table

---

## 🚀 Deployment Plan

### Phase 1: Agent Implementation (Week 1)
- [ ] Implement OutlineClient methods (CheckStatus, GetTransferMetrics, GetDetailedMetrics)
- [ ] Update MetricsCollector to collect Outline metrics
- [ ] Implement hybrid scheduler (5-min + 1-hour intervals)
- [ ] Add unit tests for Outline metrics collection
- [ ] Deploy to staging agent for testing

### Phase 2: Backend Implementation (Week 2)
- [ ] Create outline_metrics database table (migration)
- [ ] Update MetricsService.ingest_agent_metrics() to handle Outline data
- [ ] Implement API endpoints (GET /outline, GET /outline/metrics)
- [ ] Add authentication and authorization checks
- [ ] Add unit and integration tests
- [ ] Deploy to staging backend

### Phase 3: Integration Testing (Week 3)
- [ ] End-to-end testing: Agent → Backend → API
- [ ] Load testing: Verify database performance with time-series data
- [ ] Error handling: Test Outline API failures, network issues
- [ ] Security review: Verify authentication/authorization

### Phase 4: Production Rollout (Week 4)
- [ ] Deploy agent to production (usipipo-usa.duckdns.org)
- [ ] Monitor metrics collection for 1 week
- [ ] Verify data accuracy with manual Outline API checks
- [ ] Document API for frontend teams
- [ ] Create dashboard mockups for frontend implementation

---

## 📝 Frontend Integration Guide

### For Telegram Bot Developers

```python
# Get current Outline status
async def get_outline_status(server_id: UUID, user_token: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(
            f"{BACKEND_URL}/api/v1/vpn/servers/{server_id}/outline",
            headers={"Authorization": f"Bearer {user_token}"}
        ) as response:
            return await response.json()

# Usage in bot command
@bot.message_handler(commands=['stats'])
async def handle_stats(message):
    status = await get_outline_status(SERVER_ID, user.token)
    bot.reply_to(message, f"""
📊 Outline Server Status

✅ Status: {status['server_status']}
🔑 Active Keys: {status['active_keys_count']}
📈 Data Used: {format_bytes(status['total_bytes_transferred'])}
📦 Version: {status['server_version']}
    """)
```

### For Mini Web App (React) Developers

```typescript
// Hook to fetch Outline metrics
function useOutlineMetrics(serverId: string, timeRange: string = '24h') {
  const [metrics, setMetrics] = useState<OutlineMetricsResponse | null>(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    async function fetchMetrics() {
      const response = await fetch(
        `/api/v1/vpn/servers/${serverId}/outline/metrics?since=${timeRange}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      const data = await response.json();
      setMetrics(data);
      setLoading(false);
    }
    
    fetchMetrics();
  }, [serverId, timeRange]);
  
  return { metrics, loading };
}

// Usage in component
function OutlineDashboard({ serverId }) {
  const { metrics, loading } = useOutlineMetrics(serverId, '24h');
  
  if (loading) return <Spinner />;
  
  return (
    <Card>
      <Typography variant="h6">Outline Metrics</Typography>
      <Typography>Status: {metrics.summary.total_bytes} bytes transferred</Typography>
      <LineChart data={metrics.time_series} />
    </Card>
  );
}
```

---

## 🔗 Related Documentation

- [Outline Server API Documentation](https://github.com/outlinefoundation/outline-server)
- [uSipipo Agent Auto-Registration Guide](../usipipo-agent/AUTO-REGISTRATION-GUIDE.md)
- [uSipipo Backend API Reference](../usipipo-docs/apis/backend-api-reference.md)
- [Metrics Service Architecture](./metrics-service-architecture.md)

---

## 📋 Checklist

### Pre-Implementation
- [x] Design document created
- [x] Architecture reviewed
- [x] API endpoints defined
- [x] Database schema designed
- [ ] Security review completed
- [ ] Performance impact assessed

### Post-Implementation
- [ ] All tests passing (unit + integration + E2E)
- [ ] Documentation updated (API reference, deployment guide)
- [ ] Monitoring dashboards created
- [ ] Alert rules configured (for Outline API failures)
- [ ] Frontend teams notified of new endpoints

---

**Last Updated:** 2026-04-02  
**Next Review:** After Phase 4 production rollout
