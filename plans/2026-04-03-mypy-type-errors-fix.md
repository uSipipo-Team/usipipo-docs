# Mypy Type Errors Fix - Commons + Backend Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task.

**Goal:** Fix all remaining mypy type errors (97 errors across 17 files) by adding missing attributes/methods to usipipo-commons entities and fixing type mismatches in usipipo-backend.

**Architecture:** Two-phase approach:
1. **Phase 1 (Commons):** Add missing attributes/methods to User, Payment, ServerStatus entities in usipipo-commons
2. **Phase 2 (Backend):** Fix type mismatches, remove obsolete VpnKey constructor calls, fix arg types

**Tech Stack:** Python 3.13, mypy, usipipo-commons (PyPI), usipipo-backend (FastAPI)

---

## Error Analysis Summary

| Category | Errors | Files | Fix Location |
|----------|--------|-------|--------------|
| User entity missing attrs | 10 | consumption_billing_*.py, user_service.py | **commons** |
| Payment entity issues | 2 | payment_service.py, admin_stats_service.py | **commons** |
| ServerStatus call-overload | 4 | admin_server_service.py | **commons** |
| VpnKey constructor mismatches | 11 | vpn_service.py, usage_sync_job.py | **backend** |
| Type arg mismatches (int vs UUID) | 6 | consumption_*.py | **backend** |
| Unused type:ignore | 1 | subscriptions.py | **backend** |
| Other | 3 | crypto_transaction, agent_registration | **backend** |

---

## Phase 1: usipipo-commons Changes

### Task 1: Add missing methods to User entity

**Files:**
- Modify: `usipipo-commons/usipipo_commons/domain/entities/user.py`
- Test: `usipipo-commons/tests/domain/entities/test_user.py`

**Step 1: Read current User entity**

```bash
cat /home/mowgli/usipipo/usipipo-commons/usipipo_commons/domain/entities/user.py
```

**Step 2: Add missing attributes and methods**

Add to User dataclass:

```python
# New fields
current_billing_id: Optional[UUID] = None
has_pending_debt: bool = False
consumption_mode_enabled: bool = False

# New methods
def mark_as_has_debt(self) -> None:
    """Mark user as having pending debt."""
    self.has_pending_debt = True

def clear_debt(self) -> None:
    """Clear user debt."""
    self.has_pending_debt = False

def activate_consumption_mode(self) -> None:
    """Enable consumption mode."""
    self.consumption_mode_enabled = True

def deactivate_consumption_mode(self) -> None:
    """Disable consumption mode."""
    self.consumption_mode_enabled = False
```

**Step 3: Add tests**

```python
# tests/domain/entities/test_user.py
class TestUserConsumptionMethods:
    def test_mark_as_has_debt(self, user):
        user.mark_as_has_debt()
        assert user.has_pending_debt is True

    def test_clear_debt(self, user):
        user.mark_as_has_debt()
        user.clear_debt()
        assert user.has_pending_debt is False

    def test_activate_consumption_mode(self, user):
        user.activate_consumption_mode()
        assert user.consumption_mode_enabled is True

    def test_deactivate_consumption_mode(self, user):
        user.deactivate_consumption_mode()
        assert user.consumption_mode_enabled is False
```

**Step 4: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-commons
uv run pytest tests/domain/entities/test_user.py -v
```

**Step 5: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-commons
git add usipipo_commons/domain/entities/user.py tests/domain/entities/test_user.py
git commit -m "feat: add consumption mode and debt methods to User entity"
```

---

### Task 2: Add missing attributes to Payment entity

**Files:**
- Modify: `usipipo-commons/usipipo_commons/domain/entities/payment.py`
- Test: `usipipo-commons/tests/domain/entities/test_payment.py`

**Step 1: Read current Payment entity**

```bash
cat /home/mowgli/usipipo/usipipo-commons/usipipo_commons/domain/entities/payment.py
```

**Step 2: Add missing `amount` property**

The error is `"Payment" has no attribute "amount"`. Add:

```python
@property
def amount(self) -> float:
    """Return payment amount."""
    return self.amount_cents / 100.0 if hasattr(self, 'amount_cents') else 0.0
```

Or if the field exists with different name, add alias.

**Step 3: Fix Payment.create → Payment constructor**

The error `"type[Payment]" has no attribute "create"` means backend calls `Payment.create()` but Payment is a dataclass. Either:
- Add a `@classmethod create(...)` to Payment, OR
- Fix backend to use `Payment(...)` constructor

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-commons
git add usipipo_commons/domain/entities/payment.py
git commit -m "feat: add amount property and create classmethod to Payment"
```

---

### Task 3: Fix ServerStatus entity

**Files:**
- Modify: `usipipo-commons/usipipo_commons/domain/entities/server_status.py`

**Step 1: Read current ServerStatus**

```bash
cat /home/mowgli/usipipo/usipipo-commons/usipipo_commons/domain/entities/server_status.py
```

**Step 2: The error is `No overload variant of "ServerStatus" matches argument types`**

ServerStatus is likely a dataclass. The backend passes wrong argument types. Either:
- Make fields Optional where None is passed
- Add `__init__` overload for different signatures

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-commons
git add usipipo_commons/domain/entities/server_status.py
git commit -m "fix: make ServerStatus fields Optional for None compatibility"
```

---

## Phase 2: usipipo-backend Changes

### Task 4: Fix VpnKey constructor calls in vpn_service.py

**Files:**
- Modify: `usipipo-backend/src/core/application/services/vpn_service.py`

**Step 1: Read lines 350-420**

```bash
sed -n '350,420p' /home/mowgli/usipipo/usipipo-backend/src/core/application/services/vpn_service.py
```

**Step 2: Fix all VpnKey constructor calls**

Replace obsolete fields:
- `vpn_type` → `key_type`
- `config` → `key_data`
- `last_used_at` → `last_seen_at`
- `data_used_gb` → `used_bytes` (convert: `gb * 1024**3`)
- `data_limit_gb` → `data_limit_bytes` (convert: `gb * 1024**3`)
- `is_active` → `status`

**Step 3: Fix line 161 - UUID type**

```python
# Change from:
VpnKey(id=str(uuid), ...)
# To:
VpnKey(id=uuid.UUID(str(uuid)), ...)
```

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/core/application/services/vpn_service.py
git commit -m "fix: update VpnKey constructor calls to match entity fields"
```

---

### Task 5: Fix type arg mismatches in consumption services

**Files:**
- Modify: `usipipo-backend/src/core/application/services/consumption_billing_cycle.py`
- Modify: `usipipo-backend/src/core/application/services/consumption_billing_activation.py`
- Modify: `usipipo-backend/src/core/application/services/consumption_invoice_service.py`

**Step 1: Fix int → UUID for user_id**

Wherever `get_by_id(user_id)` receives `int`, convert:
```python
from uuid import UUID
user_id_uuid = UUID(int=user_id) if isinstance(user_id, int) else user_id
```

Or better, fix the repository interface to accept `int | UUID`.

**Step 2: Remove unused type:ignore in subscriptions.py**

```bash
sed -i 's/  # type: ignore\[arg-type\]//' /home/mowgli/usipipo/usipipo-backend/src/infrastructure/api/v1/routes/subscriptions.py
```

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/core/application/services/consumption_*.py src/infrastructure/api/v1/routes/subscriptions.py
git commit -m "fix: resolve type arg mismatches in consumption services"
```

---

### Task 6: Fix remaining errors

**Files:**
- Modify: `usipipo-backend/src/core/application/services/agent_registration_service.py`
- Modify: `usipipo-backend/src/core/application/services/admin_server_service.py`
- Modify: `usipipo-backend/src/core/application/services/admin_key_service.py`
- Modify: `usipipo-backend/src/core/application/services/admin_stats_service.py`
- Modify: `usipipo-backend/src/core/application/services/user_service.py`
- Modify: `usipipo-backend/src/infrastructure/jobs/usage_sync_job.py`
- Modify: `usipipo-backend/src/infrastructure/persistence/models/crypto_transaction_model.py`
- Modify: `usipipo-backend/src/infrastructure/api/v1/routes/consumption_invoices.py`

**Step 1: Fix agent_registration_service.py return types**

Lines 142, 147, 173: Wrong return types. Fix function signatures.

**Step 2: Fix admin_server_service.py ServerStatus calls**

Use correct field names from commons ServerStatus entity.

**Step 3: Fix admin_key_service.py and admin_stats_service.py**

Replace `data_used_bytes` → `used_bytes`.
Fix `is_active` property assignment (it's read-only).

**Step 4: Fix user_service.py line 86**

```python
# Change from:
User(telegram_id=None, ...)
# To:
User(telegram_id=0, ...)  # or make Optional in commons
```

**Step 5: Fix usage_sync_job.py line 57**

`data_used_gb` is not defined. Use `used_bytes` instead.

**Step 6: Fix crypto_transaction_model.py**

Ensure `raw_payload` is always `dict`, not `str | dict`.

**Step 7: Fix consumption_invoices.py**

Fix int → UUID for user_id arguments.

**Step 8: Commit all**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add -A
git commit -m "fix: resolve remaining mypy type errors across 8 files"
```

---

## Phase 3: Verification

### Task 7: Run mypy and verify zero errors

**Step 1: Run mypy**

```bash
cd /home/mowgli/usipipo/usipipo-backend
uv run mypy src/ --config-file mypy.ini --ignore-missing-imports
```

Expected: `Success: no issues found`

**Step 2: Run pre-commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add -A && git commit -m "chore: final mypy verification"
```

Expected: All hooks pass

**Step 3: Push and verify CI**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git push origin fix/mypy-type-errors
# Wait for CI, then:
gh pr checks 29
```

Expected: All checks pass ✓

---

## Execution Order

1. Task 1 (Commons: User entity) → commit → publish
2. Task 2 (Commons: Payment entity) → commit → publish
3. Task 3 (Commons: ServerStatus) → commit → publish
4. Release commons v0.18.0
5. Task 4 (Backend: vpn_service.py)
6. Task 5 (Backend: consumption services)
7. Task 6 (Backend: remaining files)
8. Task 7 (Verification)
9. Create PR #29, merge to main
