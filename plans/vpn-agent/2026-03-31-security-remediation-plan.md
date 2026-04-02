# Security Remediation Implementation Plan: uSipipo Agent

**Document Type:** Implementation Plan  
**Version:** 1.0  
**Created:** 2026-03-31  
**Author:** Security Architecture Specialist  
**Status:** Pending Approval  
**Related:** [Security Review Report](../specs/agent-security-review-2026-03-31.md)

---

## Executive Summary

This plan addresses **15 security vulnerabilities** identified in the usipipo-agent Go codebase during the comprehensive security review conducted on 2026-03-31. The remediation is organized into **4 phases over 6-8 weeks**, prioritizing critical risks while maintaining backward compatibility and including comprehensive testing requirements.

### Vulnerability Summary

| Severity | Count | Status |
|----------|-------|--------|
| **CRITICAL** | 4 | To be fixed in Phase 1 |
| **HIGH** | 4 | To be fixed in Phase 2 |
| **MEDIUM** | 4 | To be fixed in Phase 3 |
| **LOW** | 3 | To be fixed in Phase 4 |

### Timeline Overview

```
Phase 1: Critical Security Fixes     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ Week 1
Phase 2: High Priority Hardening     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ Week 2
Phase 3: Medium Priority Improvements ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ Weeks 3-4
Phase 4: Code Quality & Hardening    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ Month 2
Buffer & Testing                     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ Weeks 7-8
```

### Estimated Effort

- **Development:** 58 hours
- **Testing:** 16 hours
- **Total:** **74 hours** (~2 weeks full-time equivalent)

---

## Phase 1: Critical Security Fixes (Week 1)

### Task 1.1: Secure API Key Validation & Format

**Priority:** CRITICAL | **Effort:** 4 hours | **Owner:** TBD

#### Description

Replace weak string-comparison API key validation with a secure format (`agent_[32 alphanumeric chars]`) and cryptographic constant-time comparison to prevent timing attacks.

#### Files to Modify

- `internal/api/middleware.go` - Add key format validation, constant-time comparison
- `internal/config/config.go` - Add API key validation function
- `cmd/agent/main.go` - Validate key format at startup
- `internal/utils/validation/apikeys.go` - **NEW** - API key validation utilities

#### Implementation Details

```go
// New API key format: agent_[32 alphanumeric chars]
// Example: agent_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8
// Use subtle.ConstantTimeCompare for timing-safe comparison

import (
    "crypto/subtle"
    "regexp"
)

var apiKeyPattern = regexp.MustCompile(`^agent_[a-zA-Z0-9]{32}$`)

func isValidAPIKey(key string) bool {
    return apiKeyPattern.MatchString(key)
}

func compareAPIKeys(input, stored string) bool {
    if len(input) != len(stored) {
        return false
    }
    return subtle.ConstantTimeCompare([]byte(input), []byte(stored)) == 1
}
```

#### Acceptance Criteria

- [ ] API keys must match pattern `^agent_[a-zA-Z0-9]{32}$`
- [ ] Invalid format keys rejected with 401 at startup and middleware
- [ ] Constant-time comparison used (no timing attacks)
- [ ] Clear error messages without revealing key format details
- [ ] All existing tests pass

#### Testing Requirements

- **Unit tests:** Key format validation (valid/invalid patterns)
- **Unit tests:** Timing attack resistance test
- **Integration tests:** Auth failure with malformed keys
- **Manual:** Verify existing valid keys continue working

#### Backward Compatibility Notes

- Existing keys matching the pattern will continue working
- Keys not matching pattern will be rejected - requires regeneration
- Add migration note to CHANGELOG about key format requirement
- Provide tool/script to validate existing keys

---

### Task 1.2: TLS Configuration Hardening

**Priority:** CRITICAL | **Effort:** 3 hours | **Owner:** TBD

#### Description

Remove insecure default (`OutlineVerifySSL=false`). Make TLS verification mandatory by default with clear opt-out for development. Add proper TLS configuration for all HTTP clients.

#### Files to Modify

- `internal/config/config.go` - Change default to `true`, add validation
- `internal/reporter/reporter.go` - Add TLS config to resty client
- `internal/utils/geoip/geoip.go` - Add HTTPS support, timeout
- `.env.example` - Update with secure defaults

#### Implementation Details

```go
// New default: OUTLINE_VERIFY_SSL=true
// Add OUTLINE_INSECURE_SKIP_VERIFY=false for explicit opt-out
// Add TLS config with MinVersion: tls.VersionTLS12

// In config.go:
OutlineVerifySSL:   getEnv("OUTLINE_VERIFY_SSL", "true") == "true"

// In HTTP clients:
client.SetTLSClientConfig(&tls.Config{
    MinVersion: tls.VersionTLS12,
    InsecureSkipVerify: !config.OutlineVerifySSL,
})
```

#### Acceptance Criteria

- [ ] TLS verification enabled by default
- [ ] All HTTP clients use HTTPS where possible
- [ ] TLS 1.2 minimum enforced
- [ ] Clear warning logged if TLS disabled
- [ ] .env.example updated with secure defaults

#### Testing Requirements

- **Unit tests:** Config validation with TLS settings
- **Integration tests:** HTTPS endpoint connectivity
- **Manual:** Verify behavior with self-signed certs (with opt-out)

#### Backward Compatibility Notes

- Existing deployments with `OUTLINE_VERIFY_SSL=false` will continue working
- New deployments default to secure configuration
- Add security advisory note to CHANGELOG

---

### Task 1.3: Enhanced Rate Limiting Strategy

**Priority:** CRITICAL | **Effort:** 6 hours | **Owner:** TBD

#### Description

Replace simple IP-based rate limiting with hybrid approach: IP-based + API key-based rate limiting with exponential backoff for auth failures. Implement token bucket algorithm with lower limits for auth endpoints.

#### Files to Modify

- `internal/api/server.go` - Implement hybrid rate limiter
- `internal/api/middleware.go` - Add auth failure tracking, exponential backoff
- `internal/config/config.go` - Add new rate limit config options
- `internal/api/ratelimit.go` - **NEW** - Advanced rate limiting logic

#### Implementation Details

```go
// New rate limits:
// - General API: 5 RPS (reduced from 10)
// - Auth endpoints: 3 RPS with exponential backoff
// - Per-API-key limit: 100 RPS (prevent key abuse)
// - Lockout after 10 failed auth attempts in 5 minutes

// Hybrid rate limiter structure:
type HybridRateLimiter struct {
    ipLimiters     map[string]*rate.Limiter
    keyLimiters    map[string]*rate.Limiter
    authFailures   map[string]*FailureTracker
    mu             sync.RWMutex
}

type FailureTracker struct {
    count        int
    lockedUntil  time.Time
}
```

#### Acceptance Criteria

- [ ] Rate limiting by IP AND API key
- [ ] Exponential backoff for failed auth (1s, 2s, 4s, 8s, 16s, 30s)
- [ ] Temporary lockout after 10 failed attempts
- [ ] Configurable limits via environment variables
- [ ] Rate limit headers in responses (X-RateLimit-Remaining, X-RateLimit-Reset)
- [ ] Cleanup of old entries to prevent memory leaks

#### Testing Requirements

- **Unit tests:** Token bucket algorithm
- **Unit tests:** Exponential backoff calculation
- **Unit tests:** Lockout logic
- **Integration tests:** Rate limit enforcement under concurrent requests
- **Load tests:** Verify limits under sustained load
- **Manual:** Test lockout and recovery

#### Backward Compatibility Notes

- Lower default RPS may affect high-volume legitimate clients
- Document new limits in CHANGELOG with migration guide
- Provide env vars to adjust limits if needed:
  - `RATE_LIMIT_RPS` (default: 5)
  - `RATE_LIMIT_BURST` (default: 10)
  - `RATE_LIMIT_AUTH_RPS` (default: 3)
  - `RATE_LIMIT_LOCKOUT_THRESHOLD` (default: 10)

---

### Task 1.4: Security Event Logging

**Priority:** CRITICAL | **Effort:** 3 hours | **Owner:** TBD

#### Description

Implement structured JSON logging for all security events: failed auth, rate limit hits, config changes. Ensure logs don't expose sensitive data (API keys, tokens).

#### Files to Modify

- `internal/api/middleware.go` - Log auth failures
- `internal/api/server.go` - Log rate limit events
- `cmd/agent/main.go` - Initialize structured logger
- `internal/logging/security.go` - **NEW** - Security logger
- `internal/logging/sanitizer.go` - **NEW** - Log sanitization

#### Implementation Details

```go
// Log format (JSON):
// {
//   "timestamp": "2026-03-31T10:00:00Z",
//   "level": "WARN",
//   "event": "auth_failure",
//   "ip": "x.x.x.x",
//   "endpoint": "/api/v1/status",
//   "reason": "invalid_key",
//   "user_agent": "Mozilla/5.0..."
// }
//
// NEVER log: API keys, tokens, full request bodies

func logAuthFailure(ip, endpoint, reason string) {
    log.WithFields(log.Fields{
        "event": "auth_failure",
        "ip": ip,
        "endpoint": endpoint,
        "reason": reason,
    }).Warn("Authentication failed")
}

// API key masking
func maskAPIKey(key string) string {
    if len(key) < 8 {
        return "***"
    }
    return key[:4] + "..." + key[len(key)-4:]
}
```

#### Acceptance Criteria

- [ ] All auth failures logged with IP, endpoint, timestamp
- [ ] Rate limit events logged
- [ ] No sensitive data in logs (API keys masked)
- [ ] Structured JSON format for log aggregation
- [ ] Configurable log level (INFO, WARN, ERROR)
- [ ] Log rotation configured

#### Testing Requirements

- **Unit tests:** Log sanitization (no secrets leaked)
- **Unit tests:** API key masking function
- **Integration tests:** Verify security events are logged
- **Manual:** Review log output for sensitive data

---

## Phase 2: High Priority Security Hardening (Week 2)

### Task 2.1: GeoIP Service Security

**Priority:** HIGH | **Effort:** 2 hours | **Owner:** TBD

#### Description

Replace insecure HTTP GeoIP service with HTTPS endpoint. Add timeout, retry logic, and fallback behavior. Consider self-hosted GeoIP database for production.

#### Files to Modify

- `internal/utils/geoip/geoip.go` - HTTPS, timeouts, retries
- `internal/config/config.go` - Add GeoIP config options
- `internal/reporter/reporter.go` - Add timeout to resty client

#### Implementation Details

```go
// Use HTTPS: https://ip-api.com/json/
// Add 5-second timeout
// Add 3 retries with exponential backoff
// Add GEOIP_ENABLED=false option to disable

client.SetTimeout(5 * time.Second)
client.SetRetryCount(3)
client.SetRetryWaitTime(1 * time.Second)
```

#### Acceptance Criteria

- [ ] GeoIP uses HTTPS exclusively
- [ ] 5-second timeout enforced
- [ ] Graceful degradation if GeoIP fails (use defaults)
- [ ] Option to disable GeoIP entirely
- [ ] Retry logic with exponential backoff

#### Testing Requirements

- **Unit tests:** Timeout handling
- **Unit tests:** Retry logic
- **Integration tests:** HTTPS GeoIP endpoint
- **Manual:** Test behavior when GeoIP service unavailable

---

### Task 2.2: WireGuard Key Generation Validation

**Priority:** HIGH | **Effort:** 3 hours | **Owner:** TBD

#### Description

Add entropy validation for WireGuard private key generation. Verify key strength and add fallback if system RNG is weak. Log warnings for low-entropy environments.

#### Files to Modify

- `internal/vpn/wireguard.go` - Add entropy check, validation
- `internal/utils/crypto/entropy.go` - **NEW** - Entropy validation
- `internal/config/config.go` - Add WG_VALIDATE_KEYS config

#### Implementation Details

```go
// After key generation:
// - Verify key has sufficient entropy (check randomness)
// - Test multiple generations for uniqueness
// - Log warning if /dev/urandom may be compromised
// - Add WG_VALIDATE_KEYS=true config option

func validateKeyEntropy(key wgtypes.Key) error {
    // Check key randomness
    // Ensure no patterns or weak keys
    // Return error if entropy insufficient
}
```

#### Acceptance Criteria

- [ ] Private keys validated for entropy
- [ ] Warning logged in low-entropy environments
- [ ] Key generation fails safely if entropy insufficient
- [ ] Configurable validation strictness
- [ ] Unit tests for entropy validation

#### Testing Requirements

- **Unit tests:** Entropy validation logic
- **Integration tests:** Key generation on various systems
- **Manual:** Test on VM/container environments (low entropy scenarios)

---

### Task 2.3: Configuration File Security

**Priority:** HIGH | **Effort:** 2 hours | **Owner:** TBD

#### Description

Add file permission validation for config files and sensitive data. Ensure .env files have restrictive permissions (0600). Add startup check for insecure permissions.

#### Files to Modify

- `internal/config/config.go` - Add permission validation
- `cmd/agent/main.go` - Add startup security checks
- `docs/SECURITY.md` - Document permission requirements

#### Implementation Details

```go
// On startup:
// - Check .env file permissions (must be 0600 or stricter)
// - Check config directory permissions
// - Log warning (or fail) if permissions too open
// - Add CONFIG_STRICT_PERMS=true to enforce

func checkFilePermissions(path string) error {
    info, err := os.Stat(path)
    if err != nil {
        return err
    }
    
    mode := info.Mode()
    if mode.Perm()&0077 != 0 {
        return fmt.Errorf("file %s has insecure permissions: %o", path, mode.Perm())
    }
    return nil
}
```

#### Acceptance Criteria

- [ ] Startup check for .env file permissions
- [ ] Warning logged if .env readable by others
- [ ] Option to fail startup on insecure permissions (CONFIG_STRICT_PERMS=true)
- [ ] Documentation for proper permissions
- [ ] Fix permissions automatically if possible

#### Testing Requirements

- **Unit tests:** Permission check logic
- **Manual:** Test with various permission settings (0600, 0644, 0777)
- **Documentation:** Update deployment guide with permission requirements

---

### Task 2.4: Input Validation for Peer/Key Names

**Priority:** HIGH | **Effort:** 2 hours | **Owner:** TBD

#### Description

Add comprehensive input validation for user-provided names (WireGuard peers, Outline keys). Prevent injection attacks, enforce length limits, validate character sets.

#### Files to Modify

- `internal/api/handlers.go` - Add input validation
- `internal/utils/validation/input.go` - **NEW** - Validation functions
- `internal/config/config.go` - Add validation config

#### Implementation Details

```go
// Name validation rules:
// - Length: 1-64 characters
// - Characters: alphanumeric, hyphen, underscore only
// - No SQL injection patterns
// - No path traversal patterns
// - Trim whitespace

func ValidateName(name string) error {
    name = strings.TrimSpace(name)
    
    if len(name) == 0 {
        return fmt.Errorf("name cannot be empty")
    }
    if len(name) > 64 {
        return fmt.Errorf("name too long (max 64 chars)")
    }
    
    matched, _ := regexp.MatchString(`^[a-zA-Z0-9_-]+$`, name)
    if !matched {
        return fmt.Errorf("name contains invalid characters")
    }
    
    // Check for injection patterns
    if containsInjectionPatterns(name) {
        return fmt.Errorf("name contains potentially dangerous patterns")
    }
    
    return nil
}
```

#### Acceptance Criteria

- [ ] All user inputs validated before processing
- [ ] Clear error messages for invalid input
- [ ] Consistent validation across all endpoints
- [ ] Unit tests for edge cases
- [ ] SQL injection, XSS, path traversal attempts blocked

#### Testing Requirements

- **Unit tests:** Validation function (valid/invalid inputs)
- **Integration tests:** API rejects invalid names
- **Security tests:** SQL injection, XSS, path traversal attempts
- **Manual:** Test with various malicious inputs

---

## Phase 3: Medium Priority Improvements (Weeks 3-4)

### Task 3.1: Error Message Sanitization

**Priority:** MEDIUM | **Effort:** 2 hours | **Owner:** TBD

#### Description

Review and sanitize all error messages to prevent information disclosure. Replace detailed internal errors with generic messages in production, log details internally.

#### Files to Modify

- `internal/api/handlers.go` - Sanitize error responses
- `internal/api/middleware.go` - Generic auth error messages
- `internal/config/config.go` - Add DEBUG_MODE config
- `internal/logging/errors.go` - **NEW** - Error logging

#### Implementation Details

```go
// Production errors: "An error occurred processing your request"
// Debug errors (DEBUG_MODE=true): Include stack trace, internal details
// Never expose: file paths, SQL queries, internal IPs, stack traces

func handleError(c *gin.Context, err error) {
    log.Errorf("Internal error: %v", err)
    
    if config.DebugMode {
        c.JSON(http.StatusInternalServerError, gin.H{
            "error": err.Error(),
            "debug": true,
        })
    } else {
        c.JSON(http.StatusInternalServerError, gin.H{
            "error": "An internal error occurred",
            "request_id": getRequestID(c),
        })
    }
}
```

#### Acceptance Criteria

- [ ] No internal details in production error responses
- [ ] Detailed errors logged internally
- [ ] DEBUG_MODE for development debugging
- [ ] Consistent error response format
- [ ] All error paths audited

#### Testing Requirements

- **Unit tests:** Error message content
- **Manual:** Trigger various errors, verify no info leakage
- **Security review:** Audit all error paths

---

### Task 3.2: HTTP Client Timeouts

**Priority:** MEDIUM | **Effort:** 2 hours | **Owner:** TBD

#### Description

Add explicit timeouts to all HTTP clients to prevent hanging connections and resource exhaustion. Configure connection pooling appropriately.

#### Files to Modify

- `internal/reporter/reporter.go` - Add timeouts to resty client
- `internal/utils/geoip/geoip.go` - Add timeouts
- `internal/vpn/outline.go` - Add timeouts

#### Implementation Details

```go
// HTTP Client config:
// - Connection timeout: 10s
// - Read timeout: 30s
// - Write timeout: 10s
// - Idle connection timeout: 90s
// - Max idle connections: 100

client.SetTimeout(30 * time.Second)
client.SetConnectTimeout(10 * time.Second)
client.SetMaxIdleConns(100)
client.SetMaxIdleConnsPerHost(10)
```

#### Acceptance Criteria

- [ ] All HTTP clients have explicit timeouts
- [ ] Timeouts appropriate for each use case
- [ ] Connection pooling configured
- [ ] Timeout errors handled gracefully
- [ ] No resource leaks under load

#### Testing Requirements

- **Unit tests:** Timeout configuration
- **Integration tests:** Behavior under slow/unresponsive servers
- **Load tests:** Connection pool behavior

---

### Task 3.3: Metrics Endpoint Hardening

**Priority:** MEDIUM | **Effort:** 2 hours | **Owner:** TBD

#### Description

Add additional authentication layer for metrics endpoint. Consider separate admin API key or IP whitelist for metrics access.

#### Files to Modify

- `internal/api/server.go` - Add metrics-specific auth
- `internal/config/config.go` - Add METRICS_API_KEY config
- `internal/api/middleware.go` - Add metrics auth middleware

#### Implementation Details

```go
// Options:
// 1. Separate METRICS_API_KEY for /metrics endpoint
// 2. IP whitelist for metrics access (METRICS_ALLOWED_IPS)
// 3. Both (recommended for production)

// In server.go:
metricsGroup := router.Group("/metrics")
{
    if config.MetricsAPIKey != "" {
        metricsGroup.Use(APIKeyMiddleware(config.MetricsAPIKey))
    }
    if len(config.MetricsAllowedIPs) > 0 {
        metricsGroup.Use(IPWhitelistMiddleware(config.MetricsAllowedIPs))
    }
    metricsGroup.GET("", MetricsHandler)
}
```

#### Acceptance Criteria

- [ ] Metrics endpoint requires separate auth
- [ ] IP whitelist support
- [ ] Backward compatible (can use main API key)
- [ ] Documented in .env.example
- [ ] Both auth methods can be combined

#### Testing Requirements

- **Unit tests:** Metrics auth logic
- **Integration tests:** Metrics access control
- **Manual:** Test with various auth configurations

---

### Task 3.4: Configurable WireGuard IP Range

**Priority:** MEDIUM | **Effort:** 2 hours | **Owner:** TBD

#### Description

Replace hardcoded `10.0.0.0/24` range with configurable option. Support custom subnets to avoid conflicts with existing networks.

#### Files to Modify

- `internal/vpn/wireguard.go` - Use configurable IP range
- `internal/config/config.go` - Add WG_IP_RANGE config
- `.env.example` - Add new config option
- `internal/vpn/ipallocator.go` - **NEW** - IP allocation logic

#### Implementation Details

```go
// New config: WG_IP_RANGE=10.0.0.0/24 (default)
// Support: 10.x.x.x, 172.16.x.x, 192.168.x.x
// Validate CIDR format
// Ensure /24 or larger subnet

// In config.go:
WGIPRange: getEnv("WG_IP_RANGE", "10.0.0.0/24")

// In wireguard.go:
func parseIPRange(rangeStr string) (*net.IPNet, error) {
    _, ipNet, err := net.ParseCIDR(rangeStr)
    if err != nil {
        return nil, fmt.Errorf("invalid WG_IP_RANGE: %w", err)
    }
    return ipNet, nil
}
```

#### Acceptance Criteria

- [ ] Configurable IP range via environment variable
- [ ] Valid CIDR format enforced
- [ ] Backward compatible (default: 10.0.0.0/24)
- [ ] Documentation for choosing non-conflicting range
- [ ] IP allocation respects configured range

#### Testing Requirements

- **Unit tests:** CIDR validation, IP allocation
- **Integration tests:** Different IP ranges
- **Manual:** Test with various subnet configurations

---

## Phase 4: Code Quality & Hardening (Month 2)

### Task 4.1: Remove Global Variables

**Priority:** LOW | **Effort:** 4 hours | **Owner:** TBD

#### Description

Refactor global variables in handlers to use dependency injection. Improves testability and reduces potential for race conditions.

#### Files to Modify

- `internal/api/handlers.go` - Remove globals, use context
- `internal/api/server.go` - Pass dependencies to handlers
- `cmd/agent/main.go` - Wire up dependencies
- `internal/api/handlers_test.go` - **NEW** - Handler tests

#### Implementation Details

```go
// Current: var metricsCollector *metrics.Collector
// New: Pass collector via handler constructor or context

// Use handler structs with dependencies instead of package-level funcs
type Handlers struct {
    metricsCollector *metrics.Collector
    outlineClient    *vpn.OutlineClient
    wireguardClient  *vpn.WireGuardClient
}

func NewHandlers(mc *metrics.Collector, oc *vpn.OutlineClient, wc *vpn.WireGuardClient) *Handlers {
    return &Handlers{
        metricsCollector: mc,
        outlineClient:    oc,
        wireguardClient:  wc,
    }
}

func (h *Handlers) StatusHandler(c *gin.Context) {
    // Use h.metricsCollector, etc.
}
```

#### Acceptance Criteria

- [ ] No package-level mutable globals
- [ ] Dependencies injected via constructor or context
- [ ] All tests pass after refactoring
- [ ] No performance regression
- [ ] Race detector passes: `go test -race`

#### Testing Requirements

- **Unit tests:** All existing tests must pass
- **Integration tests:** Full API functionality
- **Race detector:** `go test -race` must pass

---

### Task 4.2: Remove Hardcoded DNS

**Priority:** LOW | **Effort:** 1 hour | **Owner:** TBD

#### Description

Replace hardcoded Cloudflare DNS (`1.1.1.1`) with configurable option. Support multiple DNS servers and system default.

#### Files to Modify

- `internal/vpn/wireguard.go` - Use configurable DNS
- `internal/config/config.go` - Add WG_DNS config
- `cmd/agent/main.go` - Pass DNS config

#### Implementation Details

```go
// New config: WG_DNS=1.1.1.1,1.0.0.1 (comma-separated)
// Default: 1.1.1.1 (Cloudflare)
// Support: system DNS, custom DNS, multiple servers

// In config.go:
WGDNS: getEnv("WG_DNS", "1.1.1.1")

// In wireguard.go:
dnsServers := strings.Split(config.WGDNS, ",")
```

#### Acceptance Criteria

- [ ] DNS servers configurable via environment
- [ ] Support multiple DNS servers (comma-separated)
- [ ] Validate DNS IP format
- [ ] Backward compatible
- [ ] Documentation for DNS configuration

#### Testing Requirements

- **Unit tests:** DNS parsing, validation
- **Manual:** Test with various DNS configurations

---

### Task 4.3: Security Unit Test Suite

**Priority:** LOW | **Effort:** 6 hours | **Owner:** TBD

#### Description

Create comprehensive security-focused unit tests covering auth, rate limiting, input validation, and error handling. Add security test CI job.

#### Files to Create

- `internal/api/security_test.go` - Auth & rate limit tests
- `internal/utils/validation/validation_test.go` - Input validation tests
- `.github/workflows/security-tests.yml` - **NEW** - CI job

#### Implementation Details

```go
// Test categories:
// - Authentication bypass attempts
// - Rate limit evasion attempts
// - Input validation (SQL injection, XSS, path traversal)
// - Error message information leakage
// - Timing attack resistance

// Example test:
func TestAuthFailureLogging(t *testing.T) {
    // Verify auth failures are logged
    // Verify API key is masked in logs
    // Verify IP and endpoint are captured
}
```

#### Acceptance Criteria

- [ ] 90%+ coverage on security-critical code
- [ ] All security tests pass in CI
- [ ] Security test job in GitHub Actions
- [ ] Documented test cases
- [ ] Tests run on every PR

#### Testing Requirements

- **Automated:** Run on every PR
- **Manual:** Periodic security review of test coverage

---

### Task 4.4: Security Documentation Update

**Priority:** LOW | **Effort:** 3 hours | **Owner:** TBD

#### Description

Update all security documentation with new configurations, best practices, and migration guides. Create security checklist for deployments.

#### Files to Modify

- `SECURITY.md` - Update with new features
- `README.md` - Add security section
- `.env.example` - Update with all new options
- `docs/SECURITY_CHECKLIST.md` - **NEW** - Deployment checklist
- `CHANGELOG.md` - Document all security improvements

#### Implementation Details

```markdown
# Security Checklist for Operators

## Pre-Deployment
- [ ] Generate secure API key (32+ chars)
- [ ] Review and customize rate limits
- [ ] Configure TLS certificates
- [ ] Set up firewall rules

## Post-Deployment
- [ ] Verify .env file permissions (0600)
- [ ] Test auth endpoints
- [ ] Verify logging is working
- [ ] Review security logs
```

#### Acceptance Criteria

- [ ] All new config options documented
- [ ] Migration guide for existing deployments
- [ ] Security checklist for operators
- [ ] CHANGELOG updated with security improvements
- [ ] .env.example includes all new options with secure defaults

---

## Implementation Guidelines

### Coding Standards

1. **Go Best Practices:** Follow effective Go guidelines, use `go fmt`, `go vet`
2. **Error Handling:** Always check errors, use wrapped errors with context (`fmt.Errorf("context: %w", err)`)
3. **Logging:** Use structured logging, never log secrets
4. **Testing:** All new code requires unit tests, security-critical code needs integration tests
5. **Code Review:** All changes require PR review from at least one team member

### Security Principles

1. **Defense in Depth:** Multiple layers of security controls
2. **Least Privilege:** Minimum necessary permissions
3. **Fail Secure:** Default to secure behavior on errors
4. **No Secrets in Code:** All secrets via environment variables or secret management
5. **Assume Breach:** Design with the mindset that attackers may gain partial access

### Backward Compatibility

1. **Config Changes:** New options have safe defaults, old options still work
2. **API Changes:** No breaking changes to existing API endpoints
3. **Migration Path:** Clear documentation for upgrading
4. **Deprecation:** Mark old options as deprecated, don't remove immediately
5. **Version Support:** Support at least 2 previous minor versions

---

## Risk Mitigation

| Risk | Impact | Likelihood | Mitigation Strategy |
|------|--------|------------|---------------------|
| Breaking existing deployments | High | Medium | Backward-compatible config, clear migration guide, staged rollout |
| Rate limiting too aggressive | Medium | Medium | Configurable limits, monitoring, quick rollback capability |
| TLS verification breaks connectivity | High | Low | Default secure but allow opt-out, test with common setups |
| Performance regression | Medium | Low | Load testing before release, benchmark critical paths |
| Key format change breaks clients | High | Medium | Support old format with deprecation warning, phased migration |
| Memory leak in rate limiter | Medium | Low | Implement cleanup goroutine, monitor memory usage |
| Log disk space exhaustion | Medium | Low | Implement log rotation, monitor disk usage |

---

## Rollback Plan

### Phase 1 Rollback

If critical issues are discovered in Phase 1:

- **API Key Changes:** Revert middleware to simple string comparison
  ```bash
  git revert <commit-hash> --no-edit
  ```
- **TLS Changes:** Revert default to `OutlineVerifySSL=false`
- **Rate Limiting:** Revert to 10 RPS, remove auth failure tracking
- **Logging:** Remove security logging, revert to basic logging

### Phase 2 Rollback

- **GeoIP:** Revert to HTTP endpoint
- **WireGuard:** Remove entropy validation
- **Permissions:** Remove permission checks
- **Input Validation:** Relax validation rules

### Phase 3 Rollback

- **Error Messages:** Revert to detailed errors
- **Timeouts:** Remove or increase timeouts
- **Metrics Auth:** Remove additional auth layer
- **IP Range:** Revert to hardcoded range

### Phase 4 Rollback

- **Globals:** Revert to package-level variables
- **DNS:** Revert to hardcoded Cloudflare DNS
- **Tests:** No rollback needed (additive)
- **Docs:** Revert documentation changes

### General Rollback Procedure

1. **Identify problematic change** via git blame/logs
2. **Create hotfix branch** from last known good tag
3. **Cherry-pick revert commit** or manually revert
4. **Test rollback** in staging environment
5. **Deploy rollback** via standard deployment process
6. **Create incident report** documenting issue

---

## Testing Strategy Summary

### Unit Tests (All Phases)

- Test each security function in isolation
- Mock external dependencies
- Target: 90%+ coverage on security-critical code
- Run: `go test ./... -race -coverprofile=coverage.out`

### Integration Tests (Phases 1-3)

- Test API endpoints with various attack scenarios
- Test rate limiting under concurrent load
- Test auth flows with valid/invalid credentials
- Run: `go test -tags=integration ./...`

### Security Tests (Phase 4)

- Automated security test suite in CI
- Penetration testing checklist
- Dependency vulnerability scanning: `go audit`
- Run: `gosec ./...`

### Manual Testing (All Phases)

- Verify no sensitive data in logs
- Test with various configurations
- Verify backward compatibility
- Review error messages for info leakage

### Load Testing (Phases 1, 3)

- Test rate limiting under sustained load
- Verify memory usage doesn't grow unbounded
- Test connection pool behavior
- Tool: `wrk`, `hey`, or `k6`

---

## Success Criteria

### Phase 1 (Week 1)

- [ ] All CRITICAL vulnerabilities addressed
- [ ] API key validation implemented and tested
- [ ] TLS secure by default
- [ ] Enhanced rate limiting deployed
- [ ] Security event logging operational
- [ ] All existing tests pass
- [ ] No regression in functionality

### Phase 2 (Week 2)

- [ ] All HIGH vulnerabilities addressed
- [ ] GeoIP uses HTTPS
- [ ] WireGuard key validation implemented
- [ ] Config file permission checks in place
- [ ] Input validation on all user inputs
- [ ] All tests passing

### Phase 3 (Weeks 3-4)

- [ ] All MEDIUM vulnerabilities addressed
- [ ] Error messages sanitized
- [ ] HTTP timeouts configured
- [ ] Metrics endpoint hardened
- [ ] WireGuard IP range configurable
- [ ] Load tests passing

### Phase 4 (Month 2)

- [ ] All LOW vulnerabilities addressed
- [ ] No global variables
- [ ] Configurable DNS
- [ ] Comprehensive security test suite
- [ ] Updated security documentation
- [ ] Security tests in CI

### Overall Success Metrics

- [ ] Zero CRITICAL/HIGH vulnerabilities remaining
- [ ] All security tests passing in CI
- [ ] No regression in existing functionality
- [ ] Backward compatibility maintained
- [ ] Documentation complete and accurate
- [ ] Security score improved from 5.6/10 to 9.0+/10

---

## Estimated Timeline

| Phase | Duration | Cumulative | Key Deliverables |
|-------|----------|------------|------------------|
| **Phase 1: Critical Fixes** | 1 week | Week 1 | API key validation, TLS hardening, rate limiting, security logging |
| **Phase 2: High Priority** | 1 week | Week 2 | GeoIP HTTPS, WireGuard validation, file permissions, input validation |
| **Phase 3: Medium Priority** | 2 weeks | Week 4 | Error sanitization, HTTP timeouts, metrics hardening, configurable IP range |
| **Phase 4: Code Quality** | 2 weeks | Week 6 | Remove globals, configurable DNS, security tests, documentation |
| **Buffer & Testing** | 2 weeks | Week 8 | Integration testing, load testing, security audit |

**Total Estimated Effort:** 58 hours development + 16 hours testing = **74 hours**

---

## Dependencies and Prerequisites

### Technical Dependencies

- Go 1.21+ (already in use)
- GitHub Actions for CI/CD
- Access to testing environment (staging VPS)
- Load testing tools (wrk, hey, or k6)

### Team Dependencies

- 1-2 senior Go developers for implementation
- 1 security reviewer for code review
- 1 QA engineer for testing
- DevOps support for deployment

### External Dependencies

- GeoIP service availability (ip-api.com)
- TLS certificates for testing
- WireGuard kernel module for integration tests

---

## Monitoring and Validation

### Metrics to Track

1. **Authentication:**
   - Failed auth attempts per hour
   - Unique IPs with failed auth
   - Lockout events

2. **Rate Limiting:**
   - Requests rate-limited per hour
   - Unique IPs hitting rate limits
   - Rate limit configuration changes

3. **System:**
   - Memory usage (rate limiter cleanup effectiveness)
   - Log volume
   - Error rates

4. **Security:**
   - Security events logged
   - Input validation rejections
   - TLS handshake failures

### Alerting Thresholds

- **Critical:** >100 failed auth attempts in 5 minutes from single IP
- **Warning:** >1000 rate-limited requests in 1 hour
- **Info:** Configuration changes, lockout events

---

## Next Steps

1. **Review and approve this plan** with stakeholders
   - [ ] Technical lead review
   - [ ] Security team review
   - [ ] Product owner approval

2. **Create GitHub issues** for each task with detailed acceptance criteria
   - [ ] Create project board
   - [ ] Label issues by phase and priority
   - [ ] Assign owners to tasks

3. **Set up tracking**
   - [ ] Create GitHub project board
   - [ ] Set up milestone for each phase
   - [ ] Configure automated status updates

4. **Begin Phase 1** with Task 1.1 (API Key Validation)
   - [ ] Create feature branch: `feature/security-phase1`
   - [ ] Implement Task 1.1
   - [ ] Run tests
   - [ ] Code review
   - [ ] Merge to main

5. **Schedule security review** at end of each phase
   - [ ] Phase 1 review: Week 1
   - [ ] Phase 2 review: Week 2
   - [ ] Phase 3 review: Week 4
   - [ ] Phase 4 review: Week 6
   - [ ] Final security audit: Week 8

---

## Appendix A: Configuration Changes Summary

### New Environment Variables

| Variable | Default | Description | Phase |
|----------|---------|-------------|-------|
| `RATE_LIMIT_AUTH_RPS` | 3.0 | Rate limit for auth endpoints | 1 |
| `RATE_LIMIT_LOCKOUT_THRESHOLD` | 10 | Failed attempts before lockout | 1 |
| `METRICS_API_KEY` | "" | Separate API key for /metrics | 3 |
| `METRICS_ALLOWED_IPS` | "" | Comma-separated IP whitelist for metrics | 3 |
| `WG_IP_RANGE` | 10.0.0.0/24 | WireGuard IP range | 3 |
| `WG_DNS` | 1.1.1.1 | WireGuard DNS servers | 4 |
| `WG_VALIDATE_KEYS` | false | Validate WireGuard key entropy | 2 |
| `CONFIG_STRICT_PERMS` | false | Fail startup on insecure permissions | 2 |
| `DEBUG_MODE` | false | Enable detailed error messages | 3 |
| `GEOIP_ENABLED` | true | Enable GeoIP lookups | 2 |

### Changed Defaults

| Variable | Old Default | New Default | Phase |
|----------|-------------|-------------|-------|
| `OUTLINE_VERIFY_SSL` | false | true | 1 |
| `RATE_LIMIT_RPS` | 10.0 | 5.0 | 1 |
| `RATE_LIMIT_BURST` | 20 | 10 | 1 |

---

## Appendix B: API Changes

### New Response Headers

| Header | Description | Phase |
|--------|-------------|-------|
| `X-RateLimit-Remaining` | Requests remaining in window | 1 |
| `X-RateLimit-Reset` | Unix timestamp when limit resets | 1 |
| `X-Request-ID` | Unique request identifier for tracing | 3 |

### New Error Codes

| Code | Description | Phase |
|------|-------------|-------|
| 429 | Rate limit exceeded (enhanced with retry-after) | 1 |
| 403 | Temporarily locked out (too many failures) | 1 |

---

## Appendix C: Security Testing Checklist

### Manual Penetration Testing

- [ ] Attempt brute-force API key guessing
- [ ] Test rate limit bypass via IP rotation
- [ ] Test input validation (SQL injection, XSS, path traversal)
- [ ] Test for information leakage in error messages
- [ ] Test for timing attacks on API key validation
- [ ] Test TLS configuration (use testssl.sh)
- [ ] Test file permission enforcement
- [ ] Test log sanitization (no secrets in logs)

### Automated Security Scanning

- [ ] Run `gosec ./...` for static analysis
- [ ] Run `go audit` for dependency vulnerabilities
- [ ] Run `golangci-lint run --enable-all` for linting
- [ ] Run `go test -race ./...` for race conditions
- [ ] Run OWASP ZAP against test deployment

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-31 | Security Architecture Specialist | Initial version |
| | | | |

---

**Approval Status:**

- [ ] Technical Lead: _________________ Date: _______
- [ ] Security Team: _________________ Date: _______
- [ ] Product Owner: _________________ Date: _______

---

**Related Documents:**

- [Security Review Report](../specs/agent-security-review-2026-03-31.md)
- [Security Checklist](../checklists/agent-security-checklist.md)
- [Deployment Guide](../guides/agent-deployment.md)
- [CHANGELOG](../../usipipo-agent/CHANGELOG.md)
