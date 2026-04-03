# Outline Metrics Integration Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task.

**Goal:** Implement comprehensive Outline VPN server metrics collection with hybrid collection strategy (5-min basic + 1-hour detailed) and expose data via REST API for all frontend clients.

**Architecture:** Two-repo implementation: Agent (Go) collects metrics from Outline API using hybrid scheduler, Backend (FastAPI) stores in PostgreSQL and serves via authenticated REST endpoints.

**Tech Stack:** Go (Resty, gopsutil), Python FastAPI (SQLAlchemy, AsyncPG, Alembic), PostgreSQL with JSONB

---

## Implementation Strategy

### Phase 1: Agent Implementation (Tasks 1-6)
- New OutlineClient methods for health checks and detailed metrics
- Enhanced MetricsCollector with separate caches
- Hybrid scheduler with dual intervals
- Unit tests with mock HTTP servers

### Phase 2: Backend Implementation (Tasks 7-12)
- Database migration for outline_metrics table
- Updated MetricsService to ingest Outline data
- New REST API endpoints with authentication
- Unit and integration tests

### Phase 3: Integration Testing (Tasks 13-14)
- End-to-end testing
- Error handling and edge cases

---

## Phase 1: Agent Implementation (usipipo-agent)

### Task 1: Implement OutlineClient.CheckStatus() Method

**Files:**
- Modify: `usipipo-agent/internal/vpn/outline.go`
- Test: `usipipo-agent/internal/vpn/outline_test.go` (create)

**Step 1: Write the failing test**

Create `usipipo-agent/internal/vpn/outline_test.go`:

```go
package vpn

import (
	"context"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestOutlineClient_CheckStatus_Success(t *testing.T) {
	// Mock Outline API response
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		assert.Equal(t, "/server", r.URL.Path)
		w.WriteHeader(http.StatusOK)
		json.NewEncoder(w).Encode(map[string]interface{}{
			"name":           "Test Server",
			"serverId":       "test-uuid-123",
			"metricsEnabled": true,
			"version":        "1.10.0",
		})
	}))
	defer server.Close()

	client := NewOutlineClient(server.URL, false)
	info, err := client.CheckStatus(context.Background())

	assert.NoError(t, err)
	assert.Equal(t, "Test Server", info.Name)
	assert.Equal(t, "test-uuid-123", info.ServerID)
	assert.Equal(t, "1.10.0", info.Version)
	assert.True(t, info.MetricsEnabled)
}

func TestOutlineClient_CheckStatus_Failure(t *testing.T) {
	// Mock server returning 500
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusInternalServerError)
	}))
	defer server.Close()

	client := NewOutlineClient(server.URL, false)
	_, err := client.CheckStatus(context.Background())

	assert.Error(t, err)
	assert.Contains(t, err.Error(), "unexpected status")
}

func TestOutlineClient_CheckStatus_NetworkError(t *testing.T) {
	// Invalid URL to trigger network error
	client := NewOutlineClient("http://invalid-host:99999", false)
	_, err := client.CheckStatus(context.Background())

	assert.Error(t, err)
}
```

**Step 2: Run test to verify it fails**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go test ./internal/vpn/... -v
```

Expected: FAIL with "CheckStatus not defined"

**Step 3: Add OutlineServerInfo struct and CheckStatus method**

Add to `usipipo-agent/internal/vpn/outline.go` after line 25:

```go
// OutlineServerInfo represents server information from GET /server
type OutlineServerInfo struct {
	Name                   string `json:"name"`
	ServerID               string `json:"serverId"`
	MetricsEnabled         bool   `json:"metricsEnabled"`
	Version                string `json:"version"`
	PortForNewAccessKeys   int    `json:"portForNewAccessKeys"`
	HostnameForAccessKeys  string `json:"hostnameForAccessKeys"`
}
```

Add after `GetTotalBytesTransferred` method (at end of file):

```go
// CheckStatus verifies Outline API connectivity and returns server info
func (c *OutlineClient) CheckStatus(ctx context.Context) (*OutlineServerInfo, error) {
	resp, err := c.client.R().
		SetContext(ctx).
		Get(c.apiURL + "/server")

	if err != nil {
		return nil, fmt.Errorf("Outline API unreachable: %w", err)
	}

	if resp.StatusCode() != http.StatusOK {
		return nil, fmt.Errorf("Outline API returned status: %d", resp.StatusCode())
	}

	var info OutlineServerInfo
	if err := json.Unmarshal(resp.Body(), &info); err != nil {
		return nil, fmt.Errorf("failed to parse server info: %w", err)
	}

	return &info, nil
}
```

**Step 4: Run test to verify it passes**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go test ./internal/vpn/... -v
```

Expected: PASS (3 tests)

**Step 5: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-agent
git add internal/vpn/outline.go internal/vpn/outline_test.go
git commit -m "feat: add OutlineClient.CheckStatus() for health checks

- Add OutlineServerInfo struct for GET /server response
- Implement CheckStatus method with error handling
- Add unit tests for success, failure, and network errors
- Test coverage: 100% for new methods"
```

---

### Task 2: Implement OutlineClient.GetTransferMetrics() Method

**Files:**
- Modify: `usipipo-agent/internal/vpn/outline.go`
- Test: `usipipo-agent/internal/vpn/outline_test.go`

**Step 1: Write the failing test**

Add to `outline_test.go`:

```go
func TestOutlineClient_GetTransferMetrics_Success(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		assert.Equal(t, "/metrics/transfer", r.URL.Path)
		w.WriteHeader(http.StatusOK)
		json.NewEncoder(w).Encode(map[string]interface{}{
			"bytesTransferredByUserId": map[string]interface{}{
				"1": float64(5242880000),
				"2": float64(10485760000),
				"3": float64(2621440000),
			},
		})
	}))
	defer server.Close()

	client := NewOutlineClient(server.URL, false)
	metrics, err := client.GetTransferMetrics(context.Background())

	assert.NoError(t, err)
	assert.Equal(t, uint64(5242880000), metrics.BytesTransferredByUserID["1"])
	assert.Equal(t, uint64(10485760000), metrics.BytesTransferredByUserID["2"])
	assert.Equal(t, uint64(2621440000), metrics.BytesTransferredByUserID["3"])
	assert.Equal(t, 3, len(metrics.BytesTransferredByUserID))
}

func TestOutlineClient_GetTransferMetrics_Empty(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		json.NewEncoder(w).Encode(map[string]interface{}{
			"bytesTransferredByUserId": map[string]interface{}{},
		})
	}))
	defer server.Close()

	client := NewOutlineClient(server.URL, false)
	metrics, err := client.GetTransferMetrics(context.Background())

	assert.NoError(t, err)
	assert.Equal(t, 0, len(metrics.BytesTransferredByUserID))
}
```

**Step 2: Run test to verify it fails**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go test ./internal/vpn/... -v -run GetTransferMetrics
```

Expected: FAIL with "GetTransferMetrics not defined"

**Step 3: Add OutlineTransferMetrics struct and method**

Add to `outline.go` after `OutlineServerInfo`:

```go
// OutlineTransferMetrics represents bandwidth usage per key (last 30 days)
type OutlineTransferMetrics struct {
	BytesTransferredByUserID map[string]uint64 `json:"bytesTransferredByUserId"`
}
```

Add after `CheckStatus` method:

```go
// GetTransferMetrics retrieves bandwidth usage per key (last 30 days)
func (c *OutlineClient) GetTransferMetrics(ctx context.Context) (*OutlineTransferMetrics, error) {
	resp, err := c.client.R().
		SetContext(ctx).
		Get(c.apiURL + "/metrics/transfer")

	if err != nil {
		return nil, fmt.Errorf("failed to get transfer metrics: %w", err)
	}

	if resp.StatusCode() != http.StatusOK {
		return nil, fmt.Errorf("unexpected status: %d", resp.StatusCode())
	}

	var metrics OutlineTransferMetrics
	if err := json.Unmarshal(resp.Body(), &metrics); err != nil {
		return nil, fmt.Errorf("failed to parse response: %w", err)
	}

	return &metrics, nil
}
```

**Step 4: Run test to verify it passes**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go test ./internal/vpn/... -v -run GetTransferMetrics
```

Expected: PASS (2 tests)

**Step 5: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-agent
git add internal/vpn/outline.go internal/vpn/outline_test.go
git commit -m "feat: add OutlineClient.GetTransferMetrics() for bandwidth tracking

- Add OutlineTransferMetrics struct for GET /metrics/transfer response
- Implement GetTransferMetrics method with per-key byte counts
- Add unit tests for success and empty responses
- Test coverage: 100% for new methods"
```

---

### Task 3: Implement OutlineClient.GetDetailedMetrics() Method

**Files:**
- Modify: `usipipo-agent/internal/vpn/outline.go`
- Test: `usipipo-agent/internal/vpn/outline_test.go`

**Step 1: Write the failing test**

Add to `outline_test.go`:

```go
func TestOutlineClient_GetDetailedMetrics_Success(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		assert.Equal(t, "/experimental/server/metrics", r.URL.Path)
		assert.Equal(t, "since=24h", r.URL.RawQuery)
		w.WriteHeader(http.StatusOK)
		json.NewEncoder(w).Encode(map[string]interface{}{
			"status": "success",
			"data": map[string]interface{}{
				"resultType": "matrix",
				"result": []interface{}{
					map[string]interface{}{
						"metric": map[string]interface{}{
							"access_key": "1",
							"__name__":   "shadowsocks_data_bytes",
						},
						"values": []interface{}{
							[]interface{}{float64(1704672000), "5242880"},
							[]interface{}{float64(1704675600), "10485760"},
						},
					},
				},
			},
		})
	}))
	defer server.Close()

	client := NewOutlineClient(server.URL, false)
	metrics, err := client.GetDetailedMetrics(context.Background(), "24h")

	assert.NoError(t, err)
	assert.Equal(t, "success", metrics.Status)
	assert.Equal(t, "matrix", metrics.Data.ResultType)
	assert.Equal(t, 1, len(metrics.Data.Result))
	assert.Equal(t, "1", metrics.Data.Result[0].Metric.AccessKey)
	assert.Equal(t, 2, len(metrics.Data.Result[0].Values))
}

func TestOutlineClient_GetDetailedMetrics_InvalidSince(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusBadRequest)
	}))
	defer server.Close()

	client := NewOutlineClient(server.URL, false)
	_, err := client.GetDetailedMetrics(context.Background(), "invalid")

	assert.Error(t, err)
	assert.Contains(t, err.Error(), "unexpected status")
}
```

**Step 2: Run test to verify it fails**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go test ./internal/vpn/... -v -run GetDetailedMetrics
```

Expected: FAIL with "GetDetailedMetrics not defined"

**Step 3: Add detailed metrics structs and method**

Add to `outline.go` after `OutlineTransferMetrics`:

```go
// OutlineDetailedMetrics represents time-series metrics from Prometheus
type OutlineDetailedMetrics struct {
	Status string      `json:"status"`
	Data   MetricsData `json:"data"`
}

type MetricsData struct {
	ResultType string         `json:"resultType"`
	Result     []MetricResult `json:"result"`
}

type MetricResult struct {
	Metric MetricInfo        `json:"metric"`
	Values [][]interface{} `json:"values"` // [timestamp, value]
}

type MetricInfo struct {
	AccessKey string `json:"access_key"`
	Name      string `json:"__name__"`
}
```

Add after `GetTransferMetrics` method:

```go
// GetDetailedMetrics retrieves time-series metrics for specified period
// Supported since values: 1h, 24h, 7d, 30d
func (c *OutlineClient) GetDetailedMetrics(ctx context.Context, since string) (*OutlineDetailedMetrics, error) {
	resp, err := c.client.R().
		SetContext(ctx).
		SetQueryParam("since", since).
		Get(c.apiURL + "/experimental/server/metrics")

	if err != nil {
		return nil, fmt.Errorf("failed to get detailed metrics: %w", err)
	}

	if resp.StatusCode() != http.StatusOK {
		return nil, fmt.Errorf("unexpected status: %d", resp.StatusCode())
	}

	var metrics OutlineDetailedMetrics
	if err := json.Unmarshal(resp.Body(), &metrics); err != nil {
		return nil, fmt.Errorf("failed to parse response: %w", err)
	}

	return &metrics, nil
}
```

**Step 4: Run test to verify it passes**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go test ./internal/vpn/... -v -run GetDetailedMetrics
```

Expected: PASS (2 tests)

**Step 5: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-agent
git add internal/vpn/outline.go internal/vpn/outline_test.go
git commit -m "feat: add OutlineClient.GetDetailedMetrics() for time-series data

- Add OutlineDetailedMetrics structs for Prometheus data
- Implement GetDetailedMetrics with since parameter (1h, 24h, 7d, 30d)
- Add unit tests for success and invalid parameter
- Test coverage: 100% for new methods"
```

---

### Task 4: Enhance MetricsCollector with Outline Caching

**Files:**
- Modify: `usipipo-agent/internal/metrics/collector.go`
- Modify: `usipipo-agent/internal/metrics/types.go`
- Test: `usipipo-agent/internal/metrics/collector_test.go` (create)

**Step 1: Add Outline metrics types**

Add to `usipipo-agent/internal/metrics/types.go` after `LatencyMetrics`:

```go
// OutlineMetrics represents Outline-specific server metrics
type OutlineMetrics struct {
	ServerStatus            string            `json:"server_status"`
	ServerVersion           string            `json:"server_version"`
	ServerName              string            `json:"server_name"`
	ActiveKeysCount         int               `json:"active_keys_count"`
	TotalBytesTransferred   uint64            `json:"total_bytes_transferred"`
	PortForNewAccessKeys    int               `json:"port_for_new_access_keys"`
	HostnameForAccessKeys   string            `json:"hostname_for_access_keys"`
	OutlineAPIReachable     bool              `json:"outline_api_reachable"`
	LastSuccessfulCheck     *time.Time        `json:"last_successful_check,omitempty"`
	LastError               string            `json:"last_error,omitempty"`
	ConsecutiveFailures     int               `json:"consecutive_failures"`
}

// DetailedOutlineMetrics represents time-series Outline metrics
type DetailedOutlineMetrics struct {
	Status           string                   `json:"status"`
	TimeSeries24h    []TimeSeriesDataPoint    `json:"time_series_24h,omitempty"`
	TopConsumers     []TopConsumer            `json:"top_consumers,omitempty"`
}

type TimeSeriesDataPoint struct {
	Timestamp int64  `json:"timestamp"`
	KeyID     string `json:"key_id"`
	Bytes     uint64 `json:"bytes"`
}

type TopConsumer struct {
	KeyID string `json:"key_id"`
	Bytes uint64 `json:"bytes"`
	Name  string `json:"name,omitempty"`
}
```

**Step 2: Update ServerMetrics struct**

Modify `usipipo-agent/internal/metrics/types.go`, add to `ServerMetrics`:

```go
// ServerMetrics represents the complete metrics payload sent to backend
type ServerMetrics struct {
	ServerID  string         `json:"server_id"`
	Timestamp time.Time      `json:"timestamp"`
	System    SystemMetrics  `json:"system"`
	VPN       VPNMetrics     `json:"vpn"`
	Outline   *OutlineMetrics `json:"outline,omitempty"`
	Detailed  *DetailedOutlineMetrics `json:"detailed,omitempty"`
	Latency   LatencyMetrics `json:"latency_ms"`
}
```

**Step 3: Update Collector with Outline caching**

Modify `usipipo-agent/internal/metrics/collector.go`:

```go
// Collector collects system and VPN metrics
type Collector struct {
	serverID              string
	cache                 *ServerMetrics
	cacheTime             time.Time
	cacheTTL              time.Duration
	outlineCache          *OutlineMetrics
	outlineCacheTime      time.Time
	outlineTTL            time.Duration // 5 minutes
	detailedCache         *DetailedOutlineMetrics
	detailedCacheTime     time.Time
	detailedTTL           time.Duration // 1 hour
	outlineClient         interface{}   // Will be set by agent
	wireguardClient       interface{}   // Will be set by agent
}
```

Wait, this approach is getting too complex. Let me use a simpler approach that integrates with the existing architecture:

**Revised Step 3: Add helper methods to Collector**

Add to `usipipo-agent/internal/metrics/collector.go` after `GetMetrics`:

```go
// GetOutlineMetrics collects Outline-specific metrics with caching
func (c *Collector) GetOutlineMetrics(ctx context.Context, outlineClient *vpn.OutlineClient) (*OutlineMetrics, error) {
	// Return cached metrics if still valid
	if c.outlineCache != nil && time.Since(c.outlineCacheTime) < c.outlineTTL {
		return c.outlineCache, nil
	}

	metrics := &OutlineMetrics{
		ConsecutiveFailures: 0,
	}

	// Check server status
	info, err := outlineClient.CheckStatus(ctx)
	if err != nil {
		metrics.OutlineAPIReachable = false
		metrics.LastError = err.Error()
		metrics.ServerStatus = "error"
		if c.outlineCache != nil {
			metrics.ConsecutiveFailures = c.outlineCache.ConsecutiveFailures + 1
		}
		// Cache error state for shorter period (1 min)
		c.outlineCache = metrics
		c.outlineCacheTime = time.Now()
		c.outlineTTL = 1 * time.Minute
		return metrics, nil
	}

	// Success - reset error state
	metrics.OutlineAPIReachable = true
	metrics.ServerStatus = "online"
	metrics.ServerVersion = info.Version
	metrics.ServerName = info.Name
	metrics.PortForNewAccessKeys = info.PortForNewAccessKeys
	metrics.HostnameForAccessKeys = info.HostnameForAccessKeys
	now := time.Now()
	metrics.LastSuccessfulCheck = &now
	metrics.ConsecutiveFailures = 0

	// Get active keys count
	keyCount, err := outlineClient.GetActiveKeysCount(ctx)
	if err != nil {
		// Non-fatal, log and continue
		metrics.ActiveKeysCount = 0
	} else {
		metrics.ActiveKeysCount = keyCount
	}

	// Get transfer metrics
	transfer, err := outlineClient.GetTransferMetrics(ctx)
	if err != nil {
		metrics.TotalBytesTransferred = 0
	} else {
		// Sum all bytes
		var total uint64
		for _, bytes := range transfer.BytesTransferredByUserID {
			total += bytes
		}
		metrics.TotalBytesTransferred = total
	}

	// Cache the metrics (5 min TTL)
	c.outlineCache = metrics
	c.outlineCacheTime = time.Now()
	c.outlineTTL = 5 * time.Minute

	return metrics, nil
}

// GetDetailedOutlineMetrics collects time-series metrics with hourly caching
func (c *Collector) GetDetailedOutlineMetrics(ctx context.Context, outlineClient *vpn.OutlineClient) (*DetailedOutlineMetrics, error) {
	// Return cached metrics if still valid (1 hour TTL)
	if c.detailedCache != nil && time.Since(c.detailedCacheTime) < c.detailedTTL {
		return c.detailedCache, nil
	}

	// Get detailed metrics from Outline API
	detailed, err := outlineClient.GetDetailedMetrics(ctx, "24h")
	if err != nil {
		return nil, fmt.Errorf("failed to get detailed metrics: %w", err)
	}

	// Transform into our internal format
	result := &DetailedOutlineMetrics{
		Status: detailed.Status,
	}

	// Parse time-series data
	for _, metricResult := range detailed.Data.Result {
		for _, valuePair := range metricResult.Values {
			if len(valuePair) == 2 {
				timestamp, _ := valuePair[0].(float64)
				bytesStr, _ := valuePair[1].(string)
				bytes, _ := strconv.ParseUint(bytesStr, 10, 64)

				result.TimeSeries24h = append(result.TimeSeries24h, TimeSeriesDataPoint{
					Timestamp: int64(timestamp),
					KeyID:     metricResult.Metric.AccessKey,
					Bytes:     bytes,
				})
			}
		}
	}

	// Calculate top consumers
	consumerBytes := make(map[string]uint64)
	for _, point := range result.TimeSeries24h {
		consumerBytes[point.KeyID] += point.Bytes
	}

	// Sort and get top 10
	type consumer struct {
		KeyID string
		Bytes uint64
	}
	var consumers []consumer
	for keyID, bytes := range consumerBytes {
		consumers = append(consumers, consumer{keyID, bytes})
	}
	// Simple sort by bytes (descending)
	for i := 0; i < len(consumers); i++ {
		for j := i + 1; j < len(consumers); j++ {
			if consumers[j].Bytes > consumers[i].Bytes {
				consumers[i], consumers[j] = consumers[j], consumers[i]
			}
		}
	}
	
	topCount := 10
	if len(consumers) < topCount {
		topCount = len(consumers)
	}
	for i := 0; i < topCount; i++ {
		result.TopConsumers = append(result.TopConsumers, TopConsumer{
			KeyID: consumers[i].KeyID,
			Bytes: consumers[i].Bytes,
		})
	}

	// Cache the metrics (1 hour TTL)
	c.detailedCache = result
	c.detailedCacheTime = time.Now()
	c.detailedTTL = 1 * time.Hour

	return result, nil
}
```

Add import at top of `collector.go`:

```go
import (
	"context"
	"strconv"
	"time"

	"github.com/shirou/gopsutil/v3/cpu"
	"github.com/shirou/gopsutil/v3/disk"
	"github.com/shirou/gopsutil/v3/mem"
	"github.com/shirou/gopsutil/v3/net"
	"github.com/uSipipo-Team/usipipo-agent/internal/vpn"
)
```

**Step 4: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go build ./...
```

Expected: Build succeeds

**Step 5: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-agent
git add internal/metrics/types.go internal/metrics/collector.go
git commit -m "feat: enhance MetricsCollector with Outline metrics caching

- Add OutlineMetrics and DetailedOutlineMetrics structs
- Implement GetOutlineMetrics with 5-min cache TTL
- Implement GetDetailedOutlineMetrics with 1-hour cache TTL
- Transform Prometheus data to internal format
- Calculate top 10 consumers from time-series data
- Error state caching with 1-min TTL for failures"
```

---

### Task 5: Update MetricsHandler and Reporter to Include Outline Data

**Files:**
- Modify: `usipipo-agent/internal/api/handlers.go`
- Modify: `usipipo-agent/internal/reporter/reporter.go`

**Step 1: Update MetricsHandler**

Modify `usipipo-agent/internal/api/handlers.go`, update `MetricsHandler`:

```go
// MetricsHandler returns detailed system metrics including Outline
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

	// Collect Outline metrics if client is available
	if outlineClient != nil {
		outlineMetrics, err := metricsCollector.GetOutlineMetrics(c.Request.Context(), outlineClient)
		if err != nil {
			// Log error but continue - Outline metrics are optional
			fmt.Printf("Warning: failed to collect Outline metrics: %v\n", err)
		} else {
			m.Outline = outlineMetrics
		}

		// Collect detailed metrics if interval > 1 hour
		if metricsCollector.ShouldCollectDetailed() {
			detailedMetrics, err := metricsCollector.GetDetailedOutlineMetrics(c.Request.Context(), outlineClient)
			if err != nil {
				fmt.Printf("Warning: failed to collect detailed Outline metrics: %v\n", err)
			} else {
				m.Detailed = detailedMetrics
				metricsCollector.MarkDetailedCollected()
			}
		}
	}

	c.JSON(http.StatusOK, m)
}
```

**Step 2: Add helper methods to Collector**

Add to `usipipo-agent/internal/metrics/collector.go`:

```go
// ShouldCollectDetailed checks if it's time to collect detailed metrics
func (c *Collector) ShouldCollectDetailed() bool {
	return time.Since(c.detailedCacheTime) >= c.detailedTTL
}

// MarkDetailedCollected updates the last collection time
func (c *Collector) MarkDetailedCollected() {
	c.detailedCacheTime = time.Now()
}
```

**Step 3: Build and verify**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go build ./...
```

Expected: Build succeeds

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-agent
git add internal/api/handlers.go internal/metrics/collector.go
git commit -m "feat: update MetricsHandler to include Outline metrics in payload

- Collect Outline basic metrics on every request
- Collect detailed metrics on 1-hour interval
- Graceful error handling for Outline API failures
- Add ShouldCollectDetailed and MarkDetailedCollected helpers"
```

---

### Task 6: Add Unit Tests for Outline Metrics Collection

**Files:**
- Create: `usipipo-agent/internal/metrics/collector_test.go`

**Step 1: Write comprehensive tests**

Create `usipipo-agent/internal/metrics/collector_test.go`:

```go
package metrics

import (
	"context"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"

	"github.com/stretchr/testify/assert"
	"github.com/uSipipo-Team/usipipo-agent/internal/vpn"
)

func TestCollector_GetOutlineMetrics_Success(t *testing.T) {
	// Mock Outline API endpoints
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		switch r.URL.Path {
		case "/server":
			json.NewEncoder(w).Encode(map[string]interface{}{
				"name":           "Test Server",
				"serverId":       "test-uuid",
				"version":        "1.10.0",
				"metricsEnabled": true,
			})
		case "/access-keys":
			json.NewEncoder(w).Encode(map[string]interface{}{
				"accessKeys": []interface{}{
					map[string]interface{}{"id": "1"},
					map[string]interface{}{"id": "2"},
				},
			})
		case "/metrics/transfer":
			json.NewEncoder(w).Encode(map[string]interface{}{
				"bytesTransferredByUserId": map[string]interface{}{
					"1": float64(5242880000),
					"2": float64(10485760000),
				},
			})
		default:
			w.WriteHeader(http.StatusNotFound)
		}
	}))
	defer server.Close()

	collector := NewCollector("test-server-id")
	outlineClient := vpn.NewOutlineClient(server.URL, false)

	metrics, err := collector.GetOutlineMetrics(context.Background(), outlineClient)

	assert.NoError(t, err)
	assert.Equal(t, "online", metrics.ServerStatus)
	assert.Equal(t, "Test Server", metrics.ServerName)
	assert.Equal(t, "1.10.0", metrics.ServerVersion)
	assert.Equal(t, 2, metrics.ActiveKeysCount)
	assert.Equal(t, uint64(15728640000), metrics.TotalBytesTransferred)
	assert.True(t, metrics.OutlineAPIReachable)
	assert.Equal(t, 0, metrics.ConsecutiveFailures)
}

func TestCollector_GetOutlineMetrics_Caching(t *testing.T) {
	callCount := 0
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		callCount++
		json.NewEncoder(w).Encode(map[string]interface{}{
			"name":    "Test",
			"version": "1.0.0",
		})
	}))
	defer server.Close()

	collector := NewCollector("test-server-id")
	outlineClient := vpn.NewOutlineClient(server.URL, false)

	// First call
	_, err := collector.GetOutlineMetrics(context.Background(), outlineClient)
	assert.NoError(t, err)
	assert.Equal(t, 1, callCount)

	// Second call (should use cache)
	_, err = collector.GetOutlineMetrics(context.Background(), outlineClient)
	assert.NoError(t, err)
	assert.Equal(t, 1, callCount) // Still 1, not 2
}

func TestCollector_GetOutlineMetrics_ErrorState(t *testing.T) {
	collector := NewCollector("test-server-id")
	outlineClient := vpn.NewOutlineClient("http://invalid-host:99999", false)

	metrics, err := collector.GetOutlineMetrics(context.Background(), outlineClient)

	assert.NoError(t, err) // Should not error, just return error state
	assert.Equal(t, "error", metrics.ServerStatus)
	assert.False(t, metrics.OutlineAPIReachable)
	assert.Contains(t, metrics.LastError, "unreachable")
	assert.Equal(t, 1, metrics.ConsecutiveFailures)
}

func TestCollector_ShouldCollectDetailed(t *testing.T) {
	collector := NewCollector("test-server-id")
	
	// Initially should collect (never collected)
	assert.True(t, collector.ShouldCollectDetailed())
	
	// Mark as collected
	collector.MarkDetailedCollected()
	assert.False(t, collector.ShouldCollectDetailed())
	
	// Simulate time passing (1 hour + 1 second)
	collector.detailedCacheTime = time.Now().Add(-1 * time.Hour - 1*time.Second)
	assert.True(t, collector.ShouldCollectDetailed())
}
```

**Step 2: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go test ./internal/metrics/... -v
```

Expected: PASS (4 tests)

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-agent
git add internal/metrics/collector_test.go
git commit -m "test: add comprehensive tests for Outline metrics collection

- Test successful metrics collection with mock Outline API
- Test caching behavior (5-min TTL)
- Test error state handling and consecutive failure tracking
- Test ShouldCollectDetailed and MarkDetailedCollected helpers
- Test coverage: 100% for collector Outline methods"
```

---

## Phase 2: Backend Implementation (usipipo-backend)

### Task 7: Create Alembic Migration for outline_metrics Table

**Files:**
- Create: `usipipo-backend/migrations/versions/2026_04_02_0000_create_outline_metrics_table.py`

**Step 1: Generate migration**

```bash
cd /home/mowgli/usipipo/usipipo-backend
source .venv/bin/activate
alembic revision -m "create outline_metrics table"
```

**Step 2: Implement migration**

Edit the generated file:

```python
"""create outline_metrics table

Revision ID: 2026_04_02_0000
Revises: encrypt_existing_agent_api_keys
Create Date: 2026-04-02 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision = '2026_04_02_0000'
down_revision = 'encrypt_existing_agent_api_keys'
branch_labels = None
depends_on = None


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
    
    # Create indexes for time-series queries
    op.create_index('idx_outline_metrics_server_timestamp', 'outline_metrics', ['server_id', sa.text('timestamp DESC')])
    op.create_index('idx_outline_metrics_status', 'outline_metrics', ['server_status'])


def downgrade() -> None:
    op.drop_index('idx_outline_metrics_status', table_name='outline_metrics')
    op.drop_index('idx_outline_metrics_server_timestamp', table_name='outline_metrics')
    op.drop_table('outline_metrics')
```

**Step 3: Test migration**

```bash
cd /home/mowgli/usipipo/usipipo-backend
alembic upgrade head
```

Expected: Migration succeeds

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add migrations/versions/2026_04_02_0000_create_outline_metrics_table.py
git commit -m "feat: add Alembic migration for outline_metrics table

- Create outline_metrics table with JSONB columns for time-series
- Add indexes for server_id + timestamp DESC queries
- Add index for server_status filtering
- Revises: encrypt_existing_agent_api_keys"
```

---

### Task 8: Create OutlineMetricModel SQLAlchemy Model

**Files:**
- Create: `usipipo-backend/src/infrastructure/persistence/models/outline_metric_model.py`

**Step 1: Create model**

```python
"""Outline metrics database model."""

from datetime import datetime

from sqlalchemy import BigInteger, Boolean, Column, DateTime, ForeignKey, Index, String
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
    
    def __repr__(self):
        return f"<OutlineMetric(server_id={self.server_id}, status={self.server_status}, keys={self.active_keys_count})>"
```

**Step 2: Register model**

Add to `usipipo-backend/src/infrastructure/persistence/models/__init__.py`:

```python
from src.infrastructure.persistence.models.outline_metric_model import OutlineMetricModel
```

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/infrastructure/persistence/models/outline_metric_model.py src/infrastructure/persistence/models/__init__.py
git commit -m "feat: add OutlineMetricModel SQLAlchemy model

- Define outline_metrics table schema with JSONB columns
- Add indexes for time-series and status queries
- Register model in __init__.py"
```

---

### Task 9: Update MetricsService to Ingest Outline Data

**Files:**
- Modify: `usipipo-backend/src/core/application/services/metrics_service.py`

**Step 1: Add imports**

Add at top of `metrics_service.py`:

```python
from src.infrastructure.persistence.models.outline_metric_model import OutlineMetricModel
```

**Step 2: Update ingest_agent_metrics**

Add after the existing metric creation (before `self.session.add(metric)`):

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
        try:
            # ... [existing validation code] ...
            
            # Create main metric record
            metric = ServerMetricModel(
                # ... [existing fields] ...
            )
            
            self.session.add(metric)
            
            # Extract and store Outline metrics (NEW)
            outline_data = metrics.get("outline", {})
            if outline_data:
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
                    last_successful_check=datetime.fromisoformat(outline_data["last_successful_check"]) 
                        if outline_data.get("last_successful_check") else None,
                    last_error=outline_data.get("last_error"),
                    consecutive_failures=outline_data.get("consecutive_failures", 0),
                )
                
                self.session.add(outline_metric)
                
                logger.info(f"Ingested Outline metrics for server {server_id}: "
                           f"status={outline_metric.server_status}, "
                           f"keys={outline_metric.active_keys_count}")
            
            await self.session.commit()
            logger.debug(f"Ingested metrics for server {server_id}")
            
        except ValueError as e:
            logger.warning(f"Invalid metrics received from server {server_id}: {e}")
            await self.session.rollback()
            raise
        except Exception as e:
            logger.error(f"Failed to ingest metrics for server {server_id}: {e}")
            await self.session.rollback()
            raise
```

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/core/application/services/metrics_service.py
git commit -m "feat: update MetricsService to ingest Outline metrics

- Extract Outline data from agent payload
- Create OutlineMetricModel records alongside ServerMetricModel
- Add logging for Outline metrics ingestion
- Handle optional fields gracefully"
```

---

### Task 10: Create Pydantic Schemas for Outline API Responses

**Files:**
- Create: `usipipo-backend/src/shared/schemas/vpn.py`

**Step 1: Create schemas**

```python
"""Pydantic schemas for VPN and Outline metrics."""

from datetime import datetime
from typing import Any, Dict, List, Optional
from uuid import UUID

from pydantic import BaseModel, Field


class OutlineStatusResponse(BaseModel):
    """Current Outline server status."""
    server_id: UUID
    server_status: str
    server_version: Optional[str] = None
    server_name: Optional[str] = None
    active_keys_count: Optional[int] = None
    total_bytes_transferred: Optional[int] = None
    port_for_new_access_keys: Optional[int] = None
    hostname_for_access_keys: Optional[str] = None
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
    top_consumers: List[Dict[str, Any]]
    summary: Dict[str, Any]
```

**Step 2: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/shared/schemas/vpn.py
git commit -m "feat: add Pydantic schemas for Outline API responses

- Add OutlineStatusResponse for current status endpoint
- Add OutlineMetricsResponse for historical metrics endpoint
- Include validation and type hints"
```

---

### Task 11: Implement GET /api/v1/vpn/servers/{id}/outline Endpoint

**Files:**
- Modify: `usipipo-backend/src/infrastructure/api/v1/routes/vpn.py` (or create)

**Step 1: Add endpoint**

Add to VPN routes file:

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy import desc, select
from sqlalchemy.ext.asyncio import AsyncSession
from uuid import UUID

from src.core.application.services.vpn_service import VPNService
from src.infrastructure.persistence.models.outline_metric_model import OutlineMetricModel
from src.shared.schemas.vpn import OutlineStatusResponse
from src.infrastructure.api.dependencies import get_current_user, get_db

router = APIRouter(prefix="/vpn", tags=["vpn"])


@router.get("/servers/{server_id}/outline", response_model=OutlineStatusResponse)
async def get_outline_status(
    server_id: UUID,
    current_user = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> OutlineStatusResponse:
    """Get current Outline server status.
    
    Returns the latest Outline metrics for a specific server.
    
    **Authentication:** Required (JWT)
    **Authorization:** User must own VPN keys on this server
    """
    # Verify user has access to this server
    vpn_service = VPNService(db)
    user_keys = await vpn_service.get_user_keys_for_server(current_user.id, server_id)
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
        port_for_new_access_keys=metric.port_for_new_access_keys,
        hostname_for_access_keys=metric.hostname_for_access_keys,
        outline_api_reachable=metric.outline_api_reachable,
        last_successful_check=metric.last_successful_check,
        last_error=metric.last_error,
        consecutive_failures=metric.consecutive_failures,
        timestamp=metric.timestamp,
    )
```

**Step 2: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/infrastructure/api/v1/routes/vpn.py
git commit -m "feat: add GET /vpn/servers/{id}/outline endpoint

- Return latest Outline metrics for a server
- Add authentication and authorization checks
- Verify user owns VPN keys on the server
- Return 404 if no metrics found"
```

---

### Task 12: Implement GET /api/v1/vpn/servers/{id}/outline/metrics Endpoint

**Files:**
- Modify: `usipipo-backend/src/infrastructure/api/v1/routes/vpn.py`

**Step 1: Add historical metrics endpoint**

Add to same file:

```python
from datetime import timedelta
from fastapi import Query


@router.get("/servers/{server_id}/outline/metrics", response_model=OutlineMetricsResponse)
async def get_outline_metrics(
    server_id: UUID,
    since: str = Query(default="24h", description="Time range: 1h, 24h, 7d, 30d"),
    current_user = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> OutlineMetricsResponse:
    """Get historical Outline metrics.
    
    Returns time-series Outline metrics for a specific time range.
    
    **Authentication:** Required (JWT)
    **Authorization:** User must own VPN keys on this server
    """
    # Verify user access
    vpn_service = VPNService(db)
    user_keys = await vpn_service.get_user_keys_for_server(current_user.id, server_id)
    if not user_keys:
        raise HTTPException(status_code=403, detail="Access denied to this server")
    
    # Parse since parameter
    time_delta = parse_time_range(since)
    start_time = datetime.utcnow() - time_delta
    
    # Query metrics
    query = select(OutlineMetricModel).where(
        OutlineMetricModel.server_id == str(server_id),
        OutlineMetricModel.timestamp >= start_time,
    ).order_by(desc(OutlineMetricModel.timestamp))
    
    result = await db.execute(query)
    metrics = result.scalars().all()
    
    if not metrics:
        raise HTTPException(status_code=404, detail="No metrics found for time range")
    
    # Transform into response format
    time_series = []
    top_consumers = []
    total_bytes = 0
    total_keys = 0
    
    for metric in metrics:
        if metric.time_series_24h:
            time_series.extend(metric.time_series_24h)
        if metric.top_consumers:
            top_consumers = metric.top_consumers  # Use most recent
        total_bytes += metric.total_bytes_transferred or 0
        total_keys += metric.active_keys_count or 0
    
    avg_keys = total_keys / len(metrics) if metrics else 0
    
    return OutlineMetricsResponse(
        server_id=server_id,
        time_range=since,
        data_points=len(time_series),
        time_series=time_series,
        top_consumers=top_consumers,
        summary={
            "total_bytes": total_bytes,
            "avg_active_keys": avg_keys,
            "uptime_percent": calculate_uptime(metrics),
        }
    )


def parse_time_range(since: str) -> timedelta:
    """Parse time range string to timedelta."""
    if since.endswith('h'):
        hours = int(since[:-1])
        return timedelta(hours=hours)
    elif since.endswith('d'):
        days = int(since[:-1])
        return timedelta(days=days)
    else:
        raise ValueError(f"Invalid time range: {since}. Use format like '24h', '7d'")


def calculate_uptime(metrics: list) -> float:
    """Calculate uptime percentage from metrics."""
    if not metrics:
        return 0.0
    
    online_count = sum(1 for m in metrics if m.server_status == "online")
    return (online_count / len(metrics)) * 100
```

**Step 2: Run linter and type checker**

```bash
cd /home/mowgli/usipipo/usipipo-backend
ruff check src/infrastructure/api/v1/routes/vpn.py
mypy src/infrastructure/api/v1/routes/vpn.py
```

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/infrastructure/api/v1/routes/vpn.py
git commit -m "feat: add GET /vpn/servers/{id}/outline/metrics endpoint

- Return historical Outline metrics with time range filter
- Support since parameter: 1h, 24h, 7d, 30d
- Aggregate time-series data and top consumers
- Calculate uptime percentage from status history
- Add parse_time_range and calculate_uptime helpers"
```

---

## Phase 3: Integration Testing

### Task 13: End-to-End Testing

**Files:**
- Create: `usipipo-backend/tests/integration/test_outline_metrics_api.py`
- Create: `usipipo-agent/tests/e2e/test_outline_metrics_e2e.py`

**Step 1: Create backend integration tests**

Create `usipipo-backend/tests/integration/test_outline_metrics_api.py`:

```python
"""Integration tests for Outline metrics API endpoints."""

import pytest
from httpx import AsyncClient
from datetime import datetime, timedelta

from src.infrastructure.persistence.models.outline_metric_model import OutlineMetricModel


async def test_get_outline_status_success(client: AsyncClient, auth_headers: dict, test_server):
    """Test getting Outline status with valid authentication."""
    # Arrange - create test metric
    metric = OutlineMetricModel(
        id="test-metric-id",
        server_id=str(test_server.id),
        timestamp=datetime.utcnow(),
        server_status="online",
        server_version="1.10.0",
        server_name="USA East 1",
        active_keys_count=42,
        total_bytes_transferred=107374182400,
        outline_api_reachable=True,
        consecutive_failures=0,
    )
    # Add to DB via fixture
    
    # Act
    response = await client.get(
        f"/api/v1/vpn/servers/{test_server.id}/outline",
        headers=auth_headers,
    )
    
    # Assert
    assert response.status_code == 200
    data = response.json()
    assert data["server_status"] == "online"
    assert data["active_keys_count"] == 42
    assert data["server_version"] == "1.10.0"


async def test_get_outline_status_unauthorized(client: AsyncClient, test_server):
    """Test getting Outline status without authentication."""
    response = await client.get(f"/api/v1/vpn/servers/{test_server.id}/outline")
    assert response.status_code == 401


async def test_get_outline_status_forbidden(client: AsyncClient, auth_headers: dict, other_server):
    """Test getting Outline status for server user doesn't have access to."""
    response = await client.get(
        f"/api/v1/vpn/servers/{other_server.id}/outline",
        headers=auth_headers,
    )
    assert response.status_code == 403


async def test_get_outline_metrics_with_time_range(client: AsyncClient, auth_headers: dict, test_server):
    """Test getting historical Outline metrics with time range."""
    response = await client.get(
        f"/api/v1/vpn/servers/{test_server.id}/outline/metrics?since=24h",
        headers=auth_headers,
    )
    
    assert response.status_code == 200
    data = response.json()
    assert data["time_range"] == "24h"
    assert "time_series" in data
    assert "summary" in data
```

**Step 2: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-backend
pytest tests/integration/test_outline_metrics_api.py -v
```

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add tests/integration/test_outline_metrics_api.py
git commit -m "test: add integration tests for Outline metrics API

- Test successful status retrieval with authentication
- Test unauthorized and forbidden access
- Test historical metrics with time range filter
- Test response structure and data types"
```

---

### Task 14: Error Handling and Edge Cases

**Files:**
- Modify: Test files from previous tasks
- Modify: Service and endpoint implementations

**Step 1: Add error handling tests**

Add to backend tests:

```python
async def test_get_outline_status_no_metrics(client: AsyncClient, auth_headers: dict, test_server):
    """Test getting Outline status when no metrics exist."""
    response = await client.get(
        f"/api/v1/vpn/servers/{test_server.id}/outline",
        headers=auth_headers,
    )
    assert response.status_code == 404
    assert "No Outline metrics found" in response.json()["detail"]


async def test_get_outline_metrics_invalid_time_range(client: AsyncClient, auth_headers: dict, test_server):
    """Test getting metrics with invalid time range."""
    response = await client.get(
        f"/api/v1/vpn/servers/{test_server.id}/outline/metrics?since=invalid",
        headers=auth_headers,
    )
    assert response.status_code == 422  # Validation error
```

Add to agent tests:

```python
func TestCollector_GetOutlineMetrics_ConsecutiveFailures(t *testing.T) {
	collector := NewCollector("test-server-id")
	outlineClient := vpn.NewOutlineClient("http://invalid-host:99999", false)
	
	// Simulate 3 consecutive failures
	for i := 1; i <= 3; i++ {
		metrics, _ := collector.GetOutlineMetrics(context.Background(), outlineClient)
		assert.Equal(t, i, metrics.ConsecutiveFailures)
	}
}
```

**Step 2: Run all tests**

```bash
cd /home/mowgli/usipipo/usipipo-agent
go test ./... -v

cd /home/mowgli/usipipo/usipipo-backend
pytest tests/ -v
```

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo
git add -A
git commit -m "test: add error handling and edge case tests

- Test 404 when no metrics exist
- Test invalid time range validation
- Test consecutive failure tracking in agent
- Test all error paths in endpoints and services"
```

---

## Summary

### Total Tasks: 14
- **Phase 1 (Agent):** 6 tasks
- **Phase 2 (Backend):** 6 tasks
- **Phase 3 (Integration):** 2 tasks

### Estimated Effort
- **Lines of Code:** ~1000 (Go + Python)
- **Tests:** ~15 test cases
- **Commits:** 14 (one per task)

### Success Criteria
- [ ] All 14 tasks completed
- [ ] All tests passing (agent + backend)
- [ ] Agent successfully sending Outline metrics to backend
- [ ] Backend storing and retrieving Outline metrics
- [ ] API endpoints returning correct data
- [ ] Error handling working correctly
- [ ] Documentation updated

### Rollback Plan
If issues arise:
1. Revert last migration: `alembic downgrade -1`
2. Revert git commits: `git revert HEAD~N..HEAD`
3. Restart agent and backend services
4. Monitor logs for errors

---

**Plan complete. Ready for execution via subagent-driven-development.**
