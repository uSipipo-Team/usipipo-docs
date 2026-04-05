# VPN Key Metrics Integration - Design Doc

**Date:** 2026-04-04
**Status:** Approved
**Feature:** Enhanced VPN key messages with server metrics and professional styling

---

## Problem

Current VPN key messages in the Telegram bot show only basic per-key data (data_used_gb, data_limit_gb) without:
1. Real-time server health metrics from Outline API
2. Detailed per-key usage metrics (last_seen, connection status)
3. Professional visual styling with clear hierarchy

Users see messages like:
```
🔑 Tus claves WIREGUARD

🔑 pigpong
   📊 0.00/5.00 GB
   🟢 Activa  💎 pigpong

📡 WIREGUARD · 🖥️ N/A

📊 Consumo: 0.0/5.0GB (0%)
░░░░░░░░░░

🟢 Estado: Activa
📅 Expira: 2026-05-04

⚡ Acciones:
```

This lacks server context and professional polish.

---

## Solution

### Approach: Enhanced Key Details with Server Metrics Section

Add server-level Outline metrics to key detail and list messages with clean, professional styling.

---

## Architecture

### Data Flow

```
User views key → Bot handler calls backend
  ├─ GET /vpn/keys/{key_id} → Per-key data (existing)
  └─ GET /vpn/servers/{server_id}/outline/status → Server metrics (new)
       ↓
  Bot formats message with both data sources
       ↓
  User sees unified view with professional styling
```

### API Endpoints Used

| Endpoint | Purpose | Data |
|----------|---------|------|
| `GET /vpn/keys` | List user keys | id, name, key_type, data_used_gb, data_limit_gb, status, expires_at |
| `GET /vpn/keys/{id}` | Key details | Same + config, server info |
| `GET /vpn/servers/{id}/outline/status` | Server health | server_status, active_key_count, total_bytes_transferred, outline_api_reachable, last_successful_check |

---

## Message Templates

### Key Details Message (Enhanced)

```
💎 {name}

📡 {type} • 🖥️ {server_name}
━━━━━━━━━━━━━━━━━━━━━━

📊 Tu Consumo: {used}/{limit}GB ({pct}%)
{progress_bar}

🟢 Estado: {status}
📅 Expira: {expires}
🕐 Última vez: {last_seen}

━━━━━━━━━━━━━━━━━━━━━━
🌐 Estado del Servidor
{server_status_icon} {server_status_text} • {active_keys} claves activas
📈 {total_bandwidth} transferidos (total)
⏱️ {uptime}% uptime (30 días)
━━━━━━━━━━━━━━━━━━━━━━

⚡ Acciones:
```

### Key List Message (Enhanced)

```
🔑 Tus Claves {type} ({count})
━━━━━━━━━━━━━━━━━━━━━━

{key_list_items}

━━━━━━━━━━━━━━━━━━━━━━
🌐 Servidor: {server_status_icon} {server_status_text} • {active_keys} keys
━━━━━━━━━━━━━━━━━━━━━━

[➕ Crear Nueva] [📊 Estadísticas]
[🔙 Volver al Menú]
```

---

## Components to Modify

### 1. `src/bot/handlers/keys.py`

**Changes:**
- `show_key_details()`: Add fetch for Outline server metrics
- `show_keys_by_type()`: Add server status summary at bottom
- Helper method `_fetch_server_metrics(server_id, headers)` with error handling
- Helper method `_format_last_seen(last_seen_at)` for human-readable time

### 2. `src/bot/keyboards/messages_keys.py`

**Changes:**
- Update `KEY_DETAILS` template with server metrics section
- Update `KEYS_LIST_HEADER` with server status footer
- Add new constants for server metrics labels

### 3. `src/bot/keyboards/keys.py`

**Changes:**
- No structural changes needed (keyboard layout stays same)
- Ensure buttons have consistent spacing and emoji usage

---

## Styling Guidelines

### Visual Hierarchy

1. **Primary:** Key name and type (💎 {name}, 📡 {type})
2. **Secondary:** Usage data with progress bar
3. **Tertiary:** Server metrics section (separated by decorative line)
4. **Actions:** Keyboard buttons with clear emoji prefixes

### Emoji Usage

| Element | Emoji | Notes |
|---------|-------|-------|
| Key name | 💎 | Premium feel |
| Protocol | 📡 | Network indicator |
| Server | 🖥️ | Hardware icon |
| Usage | 📊 | Data visualization |
| Status active | 🟢 | Green circle |
| Status inactive | 🔴 | Red circle |
| Status warning | 🟡 | Yellow circle (>80% usage) |
| Server online | 🟢 | Same as key status |
| Server offline | 🔴 | Same as key status |
| Bandwidth | 📈 | Growth indicator |
| Uptime | ⏱️ | Time indicator |
| Last seen | 🕐 | Clock icon |
| Separator | ━━━━━━━━━━━━━━━━━━━━━━ | Clean divider |

### Progress Bar

```python
def _generate_progress_bar(percentage: float, width: int = 20) -> str:
    """Generate clean progress bar."""
    filled = int(width * percentage / 100)
    empty = width - filled
    return f"{'█' * filled}{'░' * empty} {percentage:.0f}%"
```

---

## Error Handling

### Server Metrics Unavailable

If Outline metrics endpoint fails:
- **Don't block** key details display
- Show fallback: "🌐 Estado del Servidor: 📡 Métricas no disponibles"
- Log error for debugging
- Continue with key data display

### Server ID Missing

If key has no server_id:
- Skip server metrics section entirely
- Show: "📡 Servidor: N/A"

### Timeout Handling

- Use existing API client retry logic (already implemented in v0.6.0)
- 3 attempts with 0.5s/1.0s backoff
- Total timeout: ~3s max for metrics fetch

---

## Testing Strategy

### Unit Tests

1. Test `_format_last_seen()` with various datetime inputs
2. Test `_generate_progress_bar()` with edge cases (0%, 100%, >100%)
3. Test message formatting with mock metrics data

### Integration Tests

1. Test `show_key_details()` with real backend response
2. Test `show_keys_by_type()` with server metrics
3. Test error handling when metrics endpoint returns 404/500

### Manual Testing

1. Create WireGuard key, view details, verify server metrics shown
2. Create Outline key, view details, verify server metrics shown
3. Test with server offline (metrics unavailable)
4. Test with key that has no server_id

---

## Implementation Order

1. **Phase 1:** Add helper methods and update message templates
2. **Phase 2:** Modify `show_key_details()` to fetch and display server metrics
3. **Phase 3:** Modify `show_keys_by_type()` to show server status summary
4. **Phase 4:** Add unit and integration tests
5. **Phase 5:** Manual testing and polish

---

## Success Criteria

- ✅ Key details message shows server metrics section
- ✅ Key list message shows server status summary
- ✅ Professional styling with consistent emoji usage and spacing
- ✅ Graceful error handling when metrics unavailable
- ✅ All existing tests pass
- ✅ New tests added for metrics integration
- ✅ Manual testing confirms correct display in Telegram

---

## Future Enhancements (Out of Scope)

- Time-series trend data ("📈 +2.3GB last 24h")
- Per-key connection status from WireGuard peer data
- Real-time bandwidth usage per key
- Server load indicators in key list (🟢 Low, 🟡 Medium, 🔴 High)
