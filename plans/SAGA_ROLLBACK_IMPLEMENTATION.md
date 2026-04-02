# VPN Key Saga Pattern Rollback Implementation

## Summary

Implemented a saga pattern with compensating transactions to handle partial failures in VPN key operations. This prevents orphaned resources on VPN servers when database operations fail.

## Problem Statement

**Issue**: If the VPN agent succeeds but the database fails, resources are orphaned on the VPN server with no cleanup mechanism.

**Example scenarios**:
1. **Create**: WireGuard peer created on server → DB insert fails → Peer orphaned
2. **Delete**: WireGuard peer deleted from server → DB delete fails → DB record remains but peer is gone

## Solution: Saga Pattern

The saga pattern implements distributed transactions through a sequence of local transactions with compensating actions for rollback.

### Create Key Flow

```
Step 1: Create on VPN Agent (WireGuard/Outline)
   ↓
Step 2: Create in Database
   ↓
Success
```

**Rollback (if Step 2 fails)**:
```
Step 2 Failed → Compensating: Delete from Agent
   ↓
If rollback succeeds → Re-raise original error
   ↓
If rollback fails → Raise VpnKeyRollbackError
```

### Delete Key Flow

```
Step 1: Delete from VPN Agent
   ↓
Step 2: Delete from Database
   ↓
Success
```

**Partial Failure (if Step 2 fails)**:
```
Step 2 Failed → Cannot rollback (already deleted from agent)
   ↓
Raise VpnKeyRollbackError with cleanup_status="partial"
   ↓
Manual intervention required
```

## Files Modified

### 1. `usipipo-backend/src/core/application/exceptions/domain_exceptions.py`

Added `VpnKeyRollbackError` exception with full context:

```python
class VpnKeyRollbackError(DomainException):
    """Exception when VPN key rollback fails after partial failure."""

    Attributes:
        operation: "create" or "delete"
        key_name: Name of VPN key (for create)
        key_id: ID of VPN key (for delete)
        external_id: External ID on VPN server
        server_name: VPN server name
        cleanup_status: "failed" or "partial"
        original_error: Original exception that triggered rollback
        cleanup_error: Exception from cleanup operation
```

### 2. `usipipo-backend/src/core/application/services/vpn_service.py`

**create_key()** - Wrapped in try/except with compensating transaction:

```python
try:
    # Step 1: Create on agent
    result = await agent_client.create_wireguard_peer(name=name)
    external_id = result["public_key"]

    # Step 2: Create in database
    vpn_key = VpnKey(...)
    created_key = await self.vpn_repo.create(vpn_key)

    return created_key

except Exception as e:
    # Compensating transaction
    if external_id:
        try:
            await agent_client.delete_wireguard_peer(external_id)
            logger.warning(f"Rollback successful: Deleted WireGuard peer...")
            raise e  # Re-raise original error
        except Exception as cleanup_error:
            logger.error(f"ROLLBACK FAILED: ...")
            raise VpnKeyRollbackError(...) from cleanup_error
```

**delete_key()** - Wrapped with partial failure handling:

```python
try:
    # Step 1: Delete from agent
    await agent_client.delete_wireguard_peer(external_id)

    # Step 2: Delete from database
    await self.vpn_repo.delete(key_id)

except Exception as db_error:
    # Agent succeeded but DB failed - can't rollback
    logger.error(f"DATABASE DELETE FAILED: Key deleted from agent but not DB...")
    raise VpnKeyRollbackError(
        operation="delete",
        cleanup_status="partial",
        original_error=db_error,
    ) from db_error
```

### 3. `usipipo-backend/src/infrastructure/api/v1/routes/vpn.py`

Added exception handler for `VpnKeyRollbackError`:

```python
@router.post("/keys")
async def create_vpn_key(...):
    try:
        key = await vpn_service.create_key(...)
        return VpnKeyResponse(...)
    except VpnKeyRollbackError as e:
        logger.error(f"VPN KEY ROLLBACK FAILED (CREATE): {e.message}...")
        raise HTTPException(
            status_code=500,
            detail={
                "error": "rollback_failed",
                "message": "Failed to create VPN key and cleanup failed...",
                "operation": e.operation,
                "cleanup_status": e.cleanup_status,
            },
        )
```

### 4. `usipipo-commons/usipipo_commons/schemas/vpn.py`

Added tracking fields to response schema:

```python
class VpnKeyResponse(BaseModel):
    id: UUID
    user_id: UUID
    name: str
    key_type: KeyType
    status: KeyStatus
    config: Optional[str] = None
    external_id: Optional[str] = None  # NEW: For tracking
    server_id: Optional[UUID] = None   # NEW: For tracking
    server_name: Optional[str] = None  # NEW: For tracking
    created_at: datetime
    expires_at: Optional[datetime] = None
    last_used_at: Optional[datetime] = None
    data_used_gb: float
    data_limit_gb: float
```

### 5. `usipipo-backend/src/core/application/exceptions/__init__.py`

Exported new exception:

```python
from .domain_exceptions import (
    ...
    VpnKeyRollbackError,
)

__all__ = [
    ...
    "VpnKeyRollbackError",
]
```

### 6. `usipipo-backend/tests/unit/services/test_vpn_service_saga_rollback.py`

Created comprehensive test suite with 16 test cases covering:
- Successful operations (no rollback needed)
- DB failure triggers rollback
- Rollback failure raises VpnKeyRollbackError
- Agent failure (no rollback needed)
- Both WireGuard and Outline types
- Logging verification
- Edge cases (server selection, fallback clients)

## Logging Output Examples

### Successful Rollback (Create)

```
2026-04-01 21:56:02.221 | INFO  | Step 1 complete: Created VPN key on agent for 'Test Key'
    (external_id: wg-pub-key-123... on server Test Server US)
2026-04-01 21:56:02.222 | ERROR | Database operation failed for VPN key 'Test Key':
    Database connection timeout. Initiating rollback on agent...
2026-04-01 21:56:02.222 | WARNING | Rollback successful: Deleted WireGuard peer wg-pub-key-123...
    from server Test Server US after DB failure
```

### Failed Rollback (Create) - CRITICAL

```
2026-04-01 21:56:02.222 | ERROR | ROLLBACK FAILED: Could not delete VPN key from agent after
    DB failure. Key 'Test Key' (external_id: wg-pub-key-123) may be orphaned on server
    Test Server US. Manual cleanup required.
    Original error: Database connection timeout, Cleanup error: Agent connection refused
```

### Partial Failure (Delete) - CRITICAL

```
2026-04-01 22:10:15.445 | ERROR | DATABASE DELETE FAILED: Key 550e8400-e29b-41d4-a716-446655440000
    deleted from agent but NOT from database. Key exists on VPN server but database record
    remains. Manual cleanup may be required to remove database record.
    Original error: Database deadlock
```

## Test Scenarios

### Create Key Tests

| Test | Scenario | Expected Result |
|------|----------|-----------------|
| `test_create_key_success_no_rollback` | Agent + DB succeed | Key created, no rollback |
| `test_create_key_db_failure_triggers_rollback` | Agent succeeds, DB fails | Rollback executed, original error raised |
| `test_create_key_rollback_failure_raises_vpnkeyrollbackerror` | Agent succeeds, DB fails, rollback fails | VpnKeyRollbackError raised |
| `test_create_key_outline_rollback` | Outline key with DB failure | Outline-specific rollback |
| `test_create_key_agent_failure_no_rollback_needed` | Agent fails | No rollback (nothing to clean) |

### Delete Key Tests

| Test | Scenario | Expected Result |
|------|----------|-----------------|
| `test_delete_key_success_no_rollback` | Agent + DB succeed | Key deleted, no rollback |
| `test_delete_key_db_failure_raises_rollback_error` | Agent succeeds, DB fails | VpnKeyRollbackError (partial) |
| `test_delete_key_agent_failure_prevents_db_delete` | Agent fails | DB not touched (no partial state) |
| `test_delete_key_outline_type` | Outline key deletion | Outline-specific delete |

### Exception Tests

| Test | Scenario | Expected Result |
|------|----------|-----------------|
| `test_rollback_error_create_operation_message` | Create rollback error | Correct message format |
| `test_rollback_error_delete_operation_message` | Delete rollback error | Correct message format |
| `test_rollback_error_with_original_and_cleanup_errors` | Both errors present | Errors preserved |

## Benefits

1. **No Orphaned Resources**: Automatic cleanup when DB fails after agent success
2. **Clear Error Messages**: VpnKeyRollbackError provides full context for debugging
3. **Monitoring Ready**: Structured logging enables alerting on rollback failures
4. **Manual Intervention Path**: When automatic rollback fails, operators have all context needed
5. **Type Safety**: Full type hints and exception chaining for proper error handling

## Monitoring Recommendations

### Alerts to Configure

1. **Critical Alert**: `VpnKeyRollbackError` with `cleanup_status="failed"`
   - Indicates orphaned resources on VPN server
   - Requires immediate manual cleanup

2. **Warning Alert**: `VpnKeyRollbackError` with `cleanup_status="partial"`
   - Database record inconsistent with VPN server state
   - Schedule cleanup job

3. **Metric**: Rollback success rate
   - Track `rollback_success / (rollback_success + rollback_failed)`
   - Alert if rate drops below 99%

### Log Aggregation

Configure log aggregation to capture:
- `ROLLBACK FAILED` messages → PagerDuty
- `Rollback successful` messages → Metrics
- `DATABASE DELETE FAILED` messages → Slack alert

## Future Improvements

1. **Automated Cleanup Job**: Periodic scan for orphaned resources
2. **Two-Phase Commit**: If VPN agent supports transactional operations
3. **Retry Logic**: Exponential backoff for transient rollback failures
4. **Circuit Breaker**: Prevent cascading failures when agent is unhealthy
