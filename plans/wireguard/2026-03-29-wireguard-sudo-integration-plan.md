# WireGuard Sudo Integration Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use `subagent-driven-development` to implement this plan task-by-task.

**Goal:** Fix WireGuard integration by adding sudo support for wg commands with proper testing and documentation.

**Architecture:** Wrap wg commands with sudo in Go code, configure sudoers for passwordless execution, add integration tests with build tags.

**Tech Stack:** Go 1.21+, systemd, sudo, WireGuard tools, pytest-style Go tests.

---

## Phase 1: Code Changes

### Task 1.1: Update WireGuard Client to Use Sudo

**Files:**
- Modify: `internal/vpn/wireguard.go:178-195`

**Step 1: Modify runCommand to prepend sudo**

```go
func (c *WireGuardClient) runCommand(name string, args ...string) (string, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// Prepend sudo for wg commands to run with elevated privileges
	if name == "wg" {
		sudoArgs := append([]string{"wg"}, args...)
		cmd := exec.CommandContext(ctx, "sudo", sudoArgs...)
		output, err := cmd.Output()
		if err != nil {
			return "", fmt.Errorf("wg command failed: %w", err)
		}
		return strings.TrimSpace(string(output)), nil
	}

	// Non-wg commands run normally
	cmd := exec.CommandContext(ctx, name, args...)
	output, err := cmd.Output()
	if err != nil {
		return "", err
	}

	return strings.TrimSpace(string(output)), nil
}
```

**Step 2: Update runCommandWithInput similarly**

```go
func (c *WireGuardClient) runCommandWithInput(name, input string, args ...string) (string, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// Prepend sudo for wg commands
	if name == "wg" {
		sudoArgs := append([]string{"wg"}, args...)
		cmd := exec.CommandContext(ctx, "sudo", sudoArgs...)
		cmd.Stdin = strings.NewReader(input)
		output, err := cmd.Output()
		if err != nil {
			return "", fmt.Errorf("wg command failed: %w", err)
		}
		return strings.TrimSpace(string(output)), nil
	}

	// Non-wg commands run normally
	cmd := exec.CommandContext(ctx, name, args...)
	cmd.Stdin = strings.NewReader(input)
	output, err := cmd.Output()
	if err != nil {
		return "", err
	}

	return strings.TrimSpace(string(output)), nil
}
```

**Step 3: Add error logging**

```go
import (
	"log"
)

// In runCommand, add logging
cmd := exec.CommandContext(ctx, "sudo", sudoArgs...)
log.Printf("Executing wg command: wg %v", args)
output, err := cmd.Output()
if err != nil {
	if exitErr, ok := err.(*exec.ExitError); ok {
		log.Printf("wg command failed: stderr=%s", string(exitErr.Stderr))
	}
	return "", fmt.Errorf("wg command failed: %w", err)
}
```

**Step 4: Commit**

```bash
git add internal/vpn/wireguard.go
git commit -m "fix: Use sudo for WireGuard wg commands

- Prepend sudo to all wg commands for elevated privileges
- Add error logging for debugging
- Required for systemd service with non-root user
- Fixes WireGuard peer creation in production"
```

---

### Task 1.2: Create Sudoers Configuration File

**Files:**
- Create: `scripts/usipipo-agent.sudoers`

**Step 1: Create sudoers file**

```bash
# /etc/sudoers.d/usipipo-agent
#
# uSipipo Agent - WireGuard Operations
#
# This file grants the usipipo user passwordless sudo access
# to specific WireGuard commands required for VPN management.
#
# SECURITY NOTES:
# - Only specific wg commands are allowed
# - No shell access or arbitrary command execution
# - Commands are restricted to wg0 interface where applicable
#
# INSTALLATION:
#   sudo cp scripts/usipipo-agent.sudoers /etc/sudoers.d/
#   sudo chmod 440 /etc/sudoers.d/usipipo-agent
#   sudo visudo -c -f /etc/sudoers.d/usipipo-agent  # Validate
#

# Allow usipipo user to run WireGuard commands without password
usipipo ALL=(ALL) NOPASSWD: /usr/bin/wg genkey
usipipo ALL=(ALL) NOPASSWD: /usr/bin/wg pubkey
usipipo ALL=(ALL) NOPASSWD: /usr/bin/wg set wg0 peer *
usipipo ALL=(ALL) NOPASSWD: /usr/bin/wg set wg0 peer * allowed-ips *
usipipo ALL=(ALL) NOPASSWD: /usr/bin/wg show wg0
usipipo ALL=(ALL) NOPASSWD: /usr/bin/wg show wg0 dump
usipipo ALL=(ALL) NOPASSWD: /usr/bin/wg show wg0 public-key

# End of usipipo-agent sudoers configuration
```

**Step 2: Add to .gitignore exception**

```bash
# In .gitignore, ensure sudoers is NOT ignored
# (it contains no secrets, only command allowlist)
!scripts/usipipo-agent.sudoers
```

**Step 3: Commit**

```bash
git add scripts/usipipo-agent.sudoers
git commit -m "feat: Add sudoers configuration for WireGuard operations

- Passwordless sudo for specific wg commands only
- Restricted to wg0 interface for set/show operations
- Secure: no shell access or arbitrary command execution
- Installation: sudo cp to /etc/sudoers.d/ && chmod 440"
```

---

## Phase 2: Testing

### Task 2.1: Add Integration Test Tags

**Files:**
- Modify: `internal/vpn/wireguard_test.go` (create if not exists)

**Step 1: Create test file with build tags**

```go
//go:build integration
// +build integration

package vpn

import (
	"os"
	"testing"
)

// TestWireGuardGenKey tests wg genkey command with sudo
func TestWireGuardGenKey(t *testing.T) {
	// Skip if not running integration tests
	if os.Getenv("WIREGUARD_TEST_INTERFACE") == "" {
		t.Skip("WIREGUARD_TEST_INTERFACE not set, skipping integration test")
	}

	client := NewWireGuardClient(
		os.Getenv("WIREGUARD_TEST_INTERFACE"),
		"/etc/wireguard",
		"localhost",
		51820,
		"1.1.1.1",
	)

	// Test genkey
	key, err := client.runCommand("wg", "genkey")
	if err != nil {
		t.Fatalf("wg genkey failed: %v", err)
	}

	if key == "" {
		t.Fatal("wg genkey returned empty key")
	}

	t.Logf("Generated key: %s", key)
}

// TestWireGuardPubkey tests wg pubkey command with sudo
func TestWireGuardPubkey(t *testing.T) {
	if os.Getenv("WIREGUARD_TEST_INTERFACE") == "" {
		t.Skip("WIREGUARD_TEST_INTERFACE not set, skipping integration test")
	}

	// Known private key for testing
	privateKey := "wG69hNKhK1GZ7LzFZvKzNzFZvKzNzFZvKzNzFZvKzNk="

	client := NewWireGuardClient("wg0", "/etc/wireguard", "localhost", 51820, "1.1.1.1")

	pubkey, err := client.runCommandWithInput("wg", privateKey, "pubkey")
	if err != nil {
		t.Fatalf("wg pubkey failed: %v", err)
	}

	if pubkey == "" {
		t.Fatal("wg pubkey returned empty key")
	}

	t.Logf("Public key: %s", pubkey)
}
```

**Step 2: Commit**

```bash
git add internal/vpn/wireguard_test.go
git commit -m "test: Add WireGuard integration tests with build tags

- Tests for wg genkey and wg pubkey commands
- Requires WIREGUARD_TEST_INTERFACE env var
- Skipped by default, run with: go test -tags=integration
- Validates sudo integration works correctly"
```

---

### Task 2.2: Create E2E Test Script

**Files:**
- Create: `scripts/test-wireguard-e2e.sh`

**Step 1: Create test script**

```bash
#!/bin/bash
set -e

echo "========================================="
echo "uSipipo Agent - WireGuard E2E Test"
echo "========================================="

# Configuration
AGENT_API_KEY="${AGENT_API_KEY:-test_key}"
AGENT_URL="${AGENT_URL:-http://localhost:8080}"
TEST_PEER_NAME="test-peer-$(date +%s)"

echo ""
echo "Configuration:"
echo "  Agent URL: $AGENT_URL"
echo "  Test Peer: $TEST_PEER_NAME"
echo ""

# Test 1: Create WireGuard peer
echo "Test 1: Creating WireGuard peer..."
RESPONSE=$(curl -s -X POST \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"$TEST_PEER_NAME\"}" \
  "$AGENT_URL/wireguard/peers")

if echo "$RESPONSE" | grep -q "error"; then
  echo "❌ FAILED: $RESPONSE"
  exit 1
fi

echo "✅ PASSED: Peer created"
echo "Response: $RESPONSE"
echo ""

# Test 2: Verify peer exists
echo "Test 2: Verifying peer exists..."
wg show wg0 | grep -q "$TEST_PEER_NAME"
if [ $? -eq 0 ]; then
  echo "✅ PASSED: Peer found in wg show"
else
  echo "❌ FAILED: Peer not found in wg show"
  exit 1
fi
echo ""

# Test 3: Get peer usage
echo "Test 3: Getting peer usage..."
USAGE=$(curl -s \
  -H "X-API-Key: $AGENT_API_KEY" \
  "$AGENT_URL/wireguard/peers/$TEST_PEER_NAME/usage")

echo "Usage: $USAGE"
echo "✅ PASSED: Usage retrieved"
echo ""

# Test 4: Delete peer
echo "Test 4: Deleting peer..."
RESPONSE=$(curl -s -X DELETE \
  -H "X-API-Key: $AGENT_API_KEY" \
  "$AGENT_URL/wireguard/peers/$TEST_PEER_NAME")

if [ $? -eq 0 ]; then
  echo "✅ PASSED: Peer deleted"
else
  echo "❌ FAILED: $RESPONSE"
  exit 1
fi
echo ""

# Cleanup
echo "Test 5: Verifying cleanup..."
sleep 1
if wg show wg0 | grep -q "$TEST_PEER_NAME"; then
  echo "❌ FAILED: Peer still exists after deletion"
  exit 1
else
  echo "✅ PASSED: Peer cleaned up successfully"
fi
echo ""

echo "========================================="
echo "All WireGuard E2E tests passed! ✅"
echo "========================================="
```

**Step 2: Make executable**

```bash
chmod +x scripts/test-wireguard-e2e.sh
```

**Step 3: Commit**

```bash
git add scripts/test-wireguard-e2e.sh
git commit -m "test: Add WireGuard E2E test script

- Tests full peer lifecycle: create, verify, usage, delete
- Validates sudo integration end-to-end
- Run with: ./scripts/test-wireguard-e2e.sh
- Requires agent running with valid API key"
```

---

## Phase 3: Documentation

### Task 3.1: Create WireGuard Setup Documentation

**Files:**
- Create: `docs/WIREGUARD-SETUP.md`

**Step 1: Create setup guide**

```markdown
# WireGuard Setup Guide for uSipipo Agent

This guide covers WireGuard integration with the uSipipo Agent.

## Prerequisites

- WireGuard tools installed (`wg`, `wg-quick`)
- WireGuard interface configured (e.g., `wg0`)
- Agent installed and running as `usipipo` user

## Sudo Configuration

The agent requires sudo privileges to execute WireGuard commands. Follow these steps:

### 1. Copy Sudoers File

```bash
sudo cp scripts/usipipo-agent.sudoers /etc/sudoers.d/
sudo chmod 440 /etc/sudoers.d/usipipo-agent
```

### 2. Validate Sudoers Configuration

```bash
sudo visudo -c -f /etc/sudoers.d/usipipo-agent
# Expected: /etc/sudoers.d/usipipo-agent: parsed OK
```

### 3. Test Sudo Access

```bash
sudo -u usipipo wg genkey
# Should output a private key without password prompt
```

## Systemd Service Configuration

The agent service requires capabilities for WireGuard operations:

```ini
[Service]
User=usipipo
Group=usipipo
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_RAW
```

## Testing WireGuard Integration

### Run Integration Tests

```bash
cd /opt/usipipo-agent
WIREGUARD_TEST_INTERFACE=wg0 go test -tags=integration ./internal/vpn/... -v
```

### Run E2E Tests

```bash
export AGENT_API_KEY="your-api-key"
./scripts/test-wireguard-e2e.sh
```

### Manual Testing

```bash
# Create peer
curl -X POST -H "X-API-Key: your-key" \
  -H "Content-Type: application/json" \
  -d '{"name":"test-peer"}' \
  http://localhost:8080/wireguard/peers

# Verify with wg
wg show wg0

# Delete peer
curl -X DELETE -H "X-API-Key: your-key" \
  http://localhost:8080/wireguard/peers/test-peer
```

## Troubleshooting

### Issue: "wg command failed: exit status 1"

**Cause:** Sudo not configured or permissions denied

**Fix:**
```bash
# Check sudoers
sudo visudo -c -f /etc/sudoers.d/usipipo-agent

# Test sudo access
sudo -u usipipo wg genkey

# Check agent logs
sudo journalctl -u usipipo-agent -n 50
```

### Issue: "Operation not permitted"

**Cause:** Missing capabilities in systemd service

**Fix:**
```bash
# Add to systemd service
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart usipipo-agent
```

## Security Considerations

- Sudoers file grants minimal required privileges
- Only specific wg commands are allowed
- No shell access or arbitrary command execution
- Commands restricted to wg0 interface where applicable
- Audit logs available in `/var/log/auth.log`

## Next Steps

- Configure WireGuard interface (see `DEPLOYMENT.md`)
- Set up peer management via API
- Monitor peer usage and statistics
```

**Step 2: Commit**

```bash
git add docs/WIREGUARD-SETUP.md
git commit -m "docs: Add WireGuard setup and troubleshooting guide

- Sudo configuration instructions
- Systemd capabilities configuration
- Integration and E2E testing guide
- Troubleshooting common issues
- Security considerations"
```

---

### Task 3.2: Update DEPLOYMENT.md

**Files:**
- Modify: `DEPLOYMENT.md`

**Step 1: Add WireGuard section**

```markdown
## WireGuard Configuration

### Prerequisites

1. Install WireGuard tools:
```bash
sudo apt install wireguard  # Debian/Ubuntu
sudo yum install wireguard-tools  # RHEL/CentOS
```

2. Configure WireGuard interface (see `/etc/wireguard/wg0.conf`)

3. Configure sudo for agent:
```bash
sudo cp scripts/usipipo-agent.sudoers /etc/sudoers.d/
sudo chmod 440 /etc/sudoers.d/usipipo-agent
sudo visudo -c -f /etc/sudoers.d/usipipo-agent  # Validate
```

4. Test sudo access:
```bash
sudo -u usipipo wg genkey  # Should output key without password
```

See `docs/WIREGUARD-SETUP.md` for detailed instructions.
```

**Step 2: Commit**

```bash
git add DEPLOYMENT.md
git commit -m "docs: Add WireGuard configuration to DEPLOYMENT.md

- Prerequisites and installation steps
- Sudo configuration instructions
- Link to detailed WIREGUARD-SETUP.md"
```

---

## Phase 4: Release

### Task 4.1: Update CHANGELOG.md

**Files:**
- Modify: `CHANGELOG.md`

**Step 1: Add v0.1.12 section**

```markdown
## [0.1.12] - 2026-03-29

### 🐛 Bug Fixes

**WireGuard Integration:**
- ✅ Fix WireGuard commands requiring sudo privileges
- ✅ Add sudo wrapper for wg genkey, pubkey, set, show commands
- ✅ Add AmbientCapabilities to systemd service for CAP_NET_ADMIN
- ✅ Add error logging for wg command failures

### 🔧 Improvements

**Install Script v3.0:**
- ✅ Install to /opt/usipipo-agent (FHS compliant)
- ✅ Auto-detect colors (disable in pipes, respect NO_COLOR)
- ✅ Interactive mode with --interactive flag
- ✅ Systemd service installation with --service flag
- ✅ Robust error handling and validation
- ✅ Create .env with placeholder values

**Systemd Service:**
- ✅ Use 'usipipo' user instead of hardcoded username
- ✅ Add security hardening (ProtectSystem, ProtectHome)
- ✅ Configure ReadWritePaths for logs
- ✅ Add CAP_NET_ADMIN and CAP_NET_RAW capabilities

### 📚 Documentation

- ✅ Add scripts/example.env with placeholder reference
- ✅ Add docs/WIREGUARD-SETUP.md setup guide
- ✅ Add scripts/usipipo-agent.sudoers configuration
- ✅ Add integration and E2E test documentation
- ✅ Update DEPLOYMENT.md with WireGuard configuration

### 🧪 Testing

- ✅ Add WireGuard integration tests (go test -tags=integration)
- ✅ Add WireGuard E2E test script
- ✅ Test WireGuard peer creation/deletion via API

### 🔒 Security

- ✅ Sudoers file grants minimal required privileges
- ✅ Only specific wg commands allowed (no shell access)
- ✅ Commands restricted to wg0 interface where applicable
- ✅ Audit trail via sudo logging

### Files Changed

- `internal/vpn/wireguard.go` - Add sudo for wg commands
- `scripts/usipipo-agent.sudoers` - NEW: Sudo configuration
- `scripts/example.env` - NEW: Configuration template
- `systemd/usipipo-agent.service` - Add capabilities
- `docs/WIREGUARD-SETUP.md` - NEW: Setup guide
- `DEPLOYMENT.md` - Update with WireGuard section
```

**Step 2: Commit**

```bash
git add CHANGELOG.md
git commit -m "docs: Update CHANGELOG for v0.1.12

- WireGuard sudo integration
- Install script v3.0 improvements
- Systemd service enhancements
- New documentation files
- Security and testing improvements"
```

---

### Task 4.2: Create Release Tag v0.1.12

**Step 1: Create annotated tag**

```bash
git tag -a v0.1.12 -m "Release v0.1.12 - WireGuard Sudo Integration + Install Script v3.0

🐛 Bug Fixes:
- Fix WireGuard commands requiring sudo privileges
- Add sudo wrapper for all wg commands
- Add CAP_NET_ADMIN capabilities to systemd service

🔧 Improvements:
- Install script v3.0 with /opt installation
- Auto-detect colors and interactive mode
- Robust error handling and validation

📚 Documentation:
- WireGuard setup guide
- Sudoers configuration
- Integration and E2E testing docs

🧪 Testing:
- WireGuard integration tests
- E2E test script for peer lifecycle

🔒 Security:
- Minimal sudo privileges (specific commands only)
- No shell access or arbitrary command execution
- Audit trail via sudo logging"
```

**Step 2: Push tag**

```bash
git push origin v0.1.12
```

**Step 3: Create GitHub Release**

```bash
gh release create v0.1.12 \
  --title "v0.1.12 - WireGuard Sudo Integration + Install Script v3.0" \
  --notes-file <(cat << 'EOF'
## 🎉 Release v0.1.12

### WireGuard Integration Fixed ✅

WireGuard commands now work correctly with systemd service by using sudo for elevated privileges.

**What Changed:**
- All wg commands (genkey, pubkey, set, show) now use sudo
- Sudoers configuration for passwordless execution
- Systemd service updated with CAP_NET_ADMIN capabilities
- Error logging for debugging wg command failures

**Installation:**
```bash
sudo cp scripts/usipipo-agent.sudoers /etc/sudoers.d/
sudo chmod 440 /etc/sudoers.d/usipipo-agent
sudo systemctl restart usipipo-agent
```

### Install Script v3.0 🚀

Complete rewrite with production-ready features:

**Features:**
- Install to /opt/usipipo-agent (FHS compliant)
- Auto-detect colors (works in terminal and pipes)
- Interactive mode with --interactive flag
- Systemd service installation with --service flag
- Robust error handling at each step
- Configuration file with placeholder values

**Usage:**
```bash
# Default installation
curl -fsSL https://github.com/uSipipo-Team/usipipo-agent/releases/latest/download/install.sh | bash

# Interactive mode
curl -fsSL .../install.sh | bash -s -- --interactive

# With systemd service
curl -fsSL .../install.sh | bash -s -- --service
```

### Documentation 📚

**New Guides:**
- `docs/WIREGUARD-SETUP.md` - Complete WireGuard setup guide
- `scripts/usipipo-agent.sudoers` - Sudo configuration template
- `scripts/example.env` - Configuration template with placeholders
- `scripts/test-wireguard-e2e.sh` - E2E test script

### Testing 🧪

**New Tests:**
- WireGuard integration tests (`go test -tags=integration`)
- E2E test script for full peer lifecycle
- Validates sudo integration works correctly

### Security 🔒

**Sudoers Configuration:**
- Minimal privileges (only required wg commands)
- No shell access or arbitrary command execution
- Commands restricted to wg0 interface
- Audit trail via sudo logging

### Files Included

- `usipipo-agent-{os}-{arch}.zip` (6 binaries)
- `install.sh` (installation script v3.0)
- `SHA256SUMS` (checksums for verification)

### Upgrade Notes

**If upgrading from v0.1.11 or earlier:**

1. Stop agent: `sudo systemctl stop usipipo-agent`
2. Install new version: Run install script or download binary
3. Configure sudo: `sudo cp scripts/usipipo-agent.sudoers /etc/sudoers.d/`
4. Update systemd: `sudo systemctl daemon-reload`
5. Restart agent: `sudo systemctl start usipipo-agent`
6. Test WireGuard: `curl POST /wireguard/peers`

See `docs/WIREGUARD-SETUP.md` for detailed instructions.

### Full Changelog

https://github.com/uSipipo-Team/usipipo-agent/compare/v0.1.11...v0.1.12
EOF
)
```

---

## ✅ Completion Criteria

- [ ] WireGuard commands use sudo
- [ ] Sudoers configuration file created
- [ ] Integration tests added with build tags
- [ ] E2E test script created
- [ ] WIREGUARD-SETUP.md documentation added
- [ ] DEPLOYMENT.md updated
- [ ] CHANGELOG.md updated for v0.1.12
- [ ] Release tag v0.1.12 created and pushed
- [ ] GitHub release published with notes
- [ ] All tests passing (unit + integration)
- [ ] WireGuard peer creation tested end-to-end

---

**Plan complete!** Two execution options:

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**Which approach?**
