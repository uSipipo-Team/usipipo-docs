# VPN Agent Bugs - Functional Testing Results

**Test Date:** 2026-03-29
**Agent Version:** 0.1.20
**Environment:** Production VPS (us-east-1-vps-165-140-241-96)
**Tester:** Automated Functional Test Suite

---

## Executive Summary

**Overall Status:** ❌ CRITICAL BUGS FOUND

The agent has several critical bugs that prevent it from functioning correctly in production:

1. **CRITICAL:** Backend metrics reporting broken (HTTP/HTTPS mismatch)
2. **CRITICAL:** WireGuard peer creation fails (permission denied)
3. **HIGH:** Outline API SSL verification fails (certificate SAN issue)
4. **MEDIUM:** Missing sudoers configuration for WireGuard commands
5. **LOW:** Version endpoint shows wrong version (0.1.0 vs 0.1.20)

---

## Bug #1: Backend Metrics Reporting Broken (CRITICAL)

### Description
The agent is configured with HTTPS backend URL but the backend is running on HTTP. This causes ALL metrics reporting to fail.

### Error Message
```
Failed to send metrics: Post "https://usipipo.duckdns.org:8001/api/v1/metrics/agents/us-east-1-vps-165-140-241-96": http: server gave HTTP response to HTTPS client
```

### Evidence
- Log shows: `Backend URL: https://usipipo.duckdns.org:8001`
- Backend responds to HTTP: `curl http://usipipo.duckdns.org:8001/health` → `{"status":"healthy"}`
- Backend HTTPS fails: `curl -sk https://usipipo.duckdns.org:8001/health` → (empty response)
- Error occurs every minute (metrics reporting interval)

### Impact
- **Severity:** CRITICAL
- Backend never receives metrics from this agent
- Monitoring dashboards show no data
- Cannot track server health, CPU, memory, bandwidth
- Cannot track VPN usage (Outline keys, WireGuard peers)

### Root Cause
Configuration mismatch: `.env` has `BACKEND_URL=https://usipipo.duckdns.org:8001` but backend is running on HTTP (not HTTPS).

### Fix Required
1. Change `BACKEND_URL=http://usipipo.duckdns.org:8001` (HTTP, not HTTPS)
2. OR configure backend to use HTTPS with Caddy/Nginx reverse proxy

---

## Bug #2: WireGuard Peer Creation Fails (CRITICAL)

### Description
Creating WireGuard peers via the agent API fails with "operation not permitted" error.

### Error Message
```json
{"error":"failed to configure device: operation not permitted"}
```

### Test Command
```bash
curl -s -X POST \
  -H "X-API-Key: agent_key_us_east_1_outline_wireguard" \
  -H "Content-Type: application/json" \
  -d '{"name":"test-functional-peer"}' \
  http://localhost:8080/wireguard/peers
```

### Evidence
- WireGuard interface exists and has peers: `wg show wg0` shows 16 peers
- Agent runs as `usipipo` user (not root)
- No sudoers configuration found: `/etc/sudoers.d/usipipo-agent` missing
- Agent cannot execute `wg` commands directly

### Impact
- **Severity:** CRITICAL
- Cannot create new WireGuard peers
- Cannot delete existing peers
- VPN provisioning broken for WireGuard users
- Agent cannot manage WireGuard configuration

### Root Cause
The agent process runs as `usipipo` user but WireGuard configuration requires root privileges. Missing sudoers configuration to allow passwordless `wg` commands.

### Fix Required
1. Create sudoers file: `/etc/sudoers.d/usipipo-agent`
2. Add rules for `wg genkey`, `wg pubkey`, `wg set`, `wg show`
3. Update agent to use `sudo wg` commands instead of `wg`

---

## Bug #3: Outline API SSL Verification Fails (HIGH)

### Description
Creating Outline keys fails due to SSL certificate verification error.

### Error Message
```json
{"error":"failed to create key: Post \"https://165.140.241.96:53206/H0egZ6x7_eKDOFfd3zcm0Q/access-keys\": tls: failed to verify certificate: x509: certificate relies on legacy Common Name field, use SANs instead"}
```

### Test Command
```bash
curl -s -X POST \
  -H "X-API-Key: agent_key_us_east_1_outline_wireguard" \
  -H "Content-Type: application/json" \
  -d '{"name":"test-functional-key"}' \
  http://localhost:8080/outline/keys
```

### Evidence
- Outline API is accessible: `curl -sk https://165.140.241.96:53206/.../server` works
- Outline server version: 1.12.3
- Certificate uses legacy Common Name (CN) instead of Subject Alternative Names (SANs)
- Go's HTTP client rejects certificates without SANs (strict verification)

### Impact
- **Severity:** HIGH
- Cannot create new Outline access keys
- Cannot delete existing keys
- VPN provisioning broken for Outline users
- Agent cannot manage Outline configuration

### Root Cause
Outline Manager uses self-signed certificate with legacy Common Name (CN) field. Go's `crypto/x509` package requires Subject Alternative Names (SANs) since Go 1.15.

### Fix Required
**Option A (Recommended):** Disable SSL verification for Outline API
- Set `OUTLINE_VERIFY_SSL=false` in config (already set but not respected)
- Use `InsecureSkipVerify: true` in Go HTTP client for Outline API

**Option B:** Regenerate Outline certificate with SANs
- More complex, requires Outline Manager reconfiguration

---

## Bug #4: Missing Sudoers Configuration (MEDIUM)

### Description
The `usipipo` user cannot execute WireGuard commands without password, breaking agent functionality.

### Expected Configuration
```bash
# /etc/sudoers.d/usipipo-agent
usipipo ALL=(ALL) NOPASSWD: /usr/bin/wg genkey
usipipo ALL=(ALL) NOPASSWD: /usr/bin/wg pubkey
usipipo ALL=(ALL) NOPASSWD: /usr/bin/wg set
usipipo ALL=(ALL) NOPASSWD: /usr/bin/wg show
usipipo ALL=(ALL) NOPASSWD: /sbin/wg, /usr/sbin/wg
```

### Current State
```bash
cat /etc/sudoers.d/usipipo-agent
# Output: (file not found)
```

### Impact
- **Severity:** MEDIUM
- Agent cannot manage WireGuard peers
- Manual WireGuard management requires root access
- Breaks automation and orchestration

### Root Cause
Installation script did not create sudoers configuration file.

### Fix Required
1. Create `/etc/sudoers.d/usipipo-agent` with proper rules
2. Validate with `visudo -c -f /etc/sudoers.d/usipipo-agent`
3. Test: `sudo -u usipipo wg genkey` should work without password

---

## Bug #5: Version Endpoint Shows Wrong Version (LOW)

### Description
The `/status` endpoint reports version `0.1.0` instead of `0.1.20`.

### Test Command
```bash
curl -s -H "X-API-Key: agent_key_us_east_1_outline_wireguard" http://localhost:8080/status
# Response: {"status":"online","version":"0.1.0"}
```

### Expected Response
```json
{"status":"online","version":"0.1.20"}
```

### Impact
- **Severity:** LOW
- Monitoring systems show wrong version
- Hard to track which servers have which version
- Debugging becomes harder

### Root Cause
Version is hardcoded in the binary at build time. The binary was built from an older version or version string not updated.

### Fix Required
1. Update version in `cmd/agent/main.go` or use build-time ldflags
2. Rebuild binary with correct version: `go build -ldflags "-X main.Version=0.1.20"`
3. Or use GitHub Actions to auto-set version from git tag

---

## Test Results Summary

### Passing Tests ✅
- Health endpoint: `GET /health` → 200 OK
- Metrics endpoint: `GET /metrics` → 200 OK (with auth)
- Status endpoint: `GET /status` → 200 OK
- API key authentication: 401/403 for invalid keys
- Rate limiting: Working (config: 10 RPS, burst 20)
- WireGuard interface: Exists with 16 active peers
- Outline API: Accessible (SSL issues aside)

### Failing Tests ❌
- Backend metrics reporting: HTTP/HTTPS mismatch
- WireGuard peer creation: Permission denied
- Outline key creation: SSL certificate error
- Sudoers configuration: Missing
- Version reporting: Incorrect version

---

## Recommended Fix Priority

### Immediate (Critical)
1. **Fix Backend URL** - Change to HTTP or configure HTTPS
2. **Add Sudoers Configuration** - Enable WireGuard management

### Short-term (High)
3. **Fix Outline SSL** - Disable verification or regenerate cert

### Medium-term (Low)
4. **Fix Version String** - Update build process

---

## Next Steps

1. **Invoke `systematic-debugging` skill** - Deep dive into each bug
2. **Invoke `brainstorming` skill** - Design robust fixes
3. **Create implementation plan** - Fix each bug systematically
4. **Test fixes** - Re-run functional tests
5. **Deploy updated agent** - Release v0.1.21 with fixes

---

## Appendix: Test Commands Used

```bash
# Health check
curl -s http://localhost:8080/health

# Metrics (with auth)
curl -s -H "X-API-Key: agent_key_us_east_1_outline_wireguard" http://localhost:8080/metrics

# Status
curl -s -H "X-API-Key: agent_key_us_east_1_outline_wireguard" http://localhost:8080/status

# Create Outline key
curl -s -X POST -H "X-API-Key: ..." -H "Content-Type: application/json" \
  -d '{"name":"test-key"}' http://localhost:8080/outline/keys

# Create WireGuard peer
curl -s -X POST -H "X-API-Key: ..." -H "Content-Type: application/json" \
  -d '{"name":"test-peer"}' http://localhost:8080/wireguard/peers

# Check logs
sudo journalctl -u usipipo-agent -n 100 --no-pager

# Check WireGuard
sudo wg show wg0

# Check sudoers
cat /etc/sudoers.d/usipipo-agent

# Test backend connectivity
curl -s http://usipipo.duckdns.org:8001/health
```

---

**Report Generated:** 2026-03-29T18:15:00-04:00
**Status:** Ready for systematic-debugging + brainstorming
