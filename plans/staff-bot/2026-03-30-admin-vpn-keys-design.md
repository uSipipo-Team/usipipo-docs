# Admin VPN Keys CRUD - Design Document

**Date:** 2026-03-30
**Version:** 1.0
**Status:** Approved
**Scope:** Backend + Agent (Staff Bot pending)

---

## 📋 Overview

### **Purpose**
Complete CRUD API for VPN key management designed for staff administrative operations. This provides the backend foundation for the future Staff Bot (`@uSipipoStaff_Bot`).

### **Business Goals**
- ✅ **Complete Administrative Control** - Full CRUD operations for VPN keys
- ✅ **Professional Filtering** - Pagination, search, sorting, multi-criteria filters
- ✅ **Role-Based Access** - Support (read + toggle) vs Admin (all operations)
- ✅ **Audit Trail** - All operations logged with full context
- ✅ **Future-Ready** - API designed for Staff Bot integration

### **Scope (Current Implementation)**
- ✅ Backend API endpoints (`/api/v1/admin/vpn-keys`)
- ✅ Admin service layer (`AdminVpnKeyService`)
- ✅ Database migrations (audit logs, staff roles)
- ✅ Agent API verification/enhancement
- ❌ Staff Bot (pending repository creation)

---

## 🎯 Features

### **Backend API Endpoints**

| Method | Endpoint | Description | Role Required |
|--------|----------|-------------|---------------|
| `GET` | `/admin/vpn-keys` | List all keys (paginated + filters) | Support |
| `GET` | `/admin/vpn-keys/{key_id}` | Get key details | Support |
| `GET` | `/admin/users/{telegram_id}/vpn-keys` | List keys by user Telegram ID | Support |
| `POST` | `/admin/vpn-keys` | Create key manually | Admin |
| `PATCH` | `/admin/vpn-keys/{key_id}/toggle` | Toggle key status | Support |
| `PATCH` | `/admin/vpn-keys/{key_id}/data-limit` | Update data limit | Admin |
| `PATCH` | `/admin/vpn-keys/{key_id}/reset-usage` | Reset data usage | Admin |
| `POST` | `/admin/vpn-keys/{key_id}/regenerate` | Regenerate config | Admin |
| `DELETE` | `/admin/vpn-keys/{key_id}` | Delete key permanently | Admin |

### **Filters (GET /admin/vpn-keys)**

```python
# Pagination
page: int = 1                    # Current page
page_size: int = 20              # Items per page (max 100)

# Filters
user_telegram_id: int | None     # Filter by user
vpn_type: str | None             # "outline" or "wireguard"
status: str | None               # "active", "inactive", "revoked"
server_id: UUID | None           # Filter by server
country: str | None              # Filter by country

# Search
search: str | None               # Search by key name

# Sorting
sort_by: str = "created_at"      # "created_at", "last_used", "data_used"
sort_order: str = "desc"         # "asc" or "desc"

# Date range
created_after: datetime | None   # Created after date
created_before: datetime | None  # Created before date
```

### **Role Permissions Matrix**

| Action | Support | Admin |
|--------|---------|-------|
| List keys | ✅ | ✅ |
| View details | ✅ | ✅ |
| List by user | ✅ | ✅ |
| Toggle status | ✅ | ✅ |
| Create key | ❌ | ✅ |
| Update data limit | ❌ | ✅ |
| Reset usage | ❌ | ✅ |
| Regenerate config | ❌ | ✅ |
| Delete key | ❌ | ✅ |

---

## 🏗️ Architecture

### **High-Level Architecture**

```
┌─────────────────┐
│  Staff Bot      │
│  (Future)       │
└────────┬────────┘
         │
         │ HTTP/REST
         │ Authorization: Bearer <JWT>
         ▼
┌─────────────────┐
│  Backend API    │
│  /api/v1/admin/ │
│  - Role check   │
│  - Rate limit   │
│  - Audit log    │
└────────┬────────┘
         │
         │ VpnAgentClient
         │ X-API-Key: ****
         ▼
┌─────────────────┐
│  VPN Agent      │
│  (Go + wgctrl)  │
└─────────────────┘
```

### **Service Layer**

```
┌─────────────────────────────────────────┐
│  AdminVpnKeyService                     │
├─────────────────────────────────────────┤
│  + list_keys(filters)                   │
│  + get_key_detail(key_id)               │
│  + create_key(request)                  │
│  + toggle_key(key_id)                   │
│  + update_data_limit(key_id, request)   │
│  + reset_usage(key_id)                  │
│  + regenerate_config(key_id)            │
│  + delete_key(key_id)                   │
│  + log_audit(operation, details)        │
└─────────────────────────────────────────┘
```

---

## 🔌 Backend Implementation

### **Project Structure**

```
usipipo-backend/
├── src/
│   ├── core/
│   │   ├── domain/
│   │   │   └── interfaces/
│   │   │       └── i_admin_vpn_key_service.py    # NEW
│   │   └── application/
│   │       └── services/
│   │           └── admin_vpn_key_service.py      # NEW
│   ├── infrastructure/
│   │   └── api/
│   │       └── v1/
│   │           └── routes/
│   │               └── admin_vpn_keys.py         # NEW
│   └── shared/
│       └── schemas/
│           └── admin_vpn_keys.py                 # NEW
├── migrations/
│   └── versions/
│       ├── 2026_03_30_0001_admin_audit_logs.py   # NEW
│       └── 2026_03_30_0002_staff_roles.py        # NEW
└── tests/
    └── integration/
        └── test_admin_vpn_keys.py                # NEW
```

### **Database Schema**

#### **Table: `admin_audit_logs`**

```sql
CREATE TABLE admin_audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    admin_telegram_id BIGINT NOT NULL,
    admin_username VARCHAR(255),
    operation VARCHAR(50) NOT NULL,
    target_type VARCHAR(50) NOT NULL,
    target_id UUID NOT NULL,
    target_user_telegram_id BIGINT,
    details JSONB,
    ip_address INET,
    success BOOLEAN NOT NULL,
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_audit_logs_timestamp ON admin_audit_logs(timestamp DESC);
CREATE INDEX idx_audit_logs_admin ON admin_audit_logs(admin_telegram_id);
CREATE INDEX idx_audit_logs_operation ON admin_audit_logs(operation);
CREATE INDEX idx_audit_logs_target ON admin_audit_logs(target_type, target_id);
```

#### **Table: `staff_roles`**

```sql
CREATE TABLE staff_roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    telegram_id BIGINT UNIQUE NOT NULL,
    username VARCHAR(255),
    role VARCHAR(20) NOT NULL DEFAULT 'support',  -- 'support' or 'admin'
    granted_by BIGINT,
    granted_at TIMESTAMPTZ DEFAULT NOW(),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### **Schemas (Pydantic)**

#### **Request Schemas**

```python
class VpnKeyListFilters(BaseModel):
    page: int = Field(1, ge=1)
    page_size: int = Field(20, ge=1, le=100)
    user_telegram_id: int | None = None
    vpn_type: Literal["outline", "wireguard"] | None = None
    status: Literal["active", "inactive", "revoked"] | None = None
    server_id: UUID | None = None
    country: str | None = None
    search: str | None = None
    sort_by: Literal["created_at", "last_used", "data_used"] = "created_at"
    sort_order: Literal["asc", "desc"] = "desc"
    created_after: datetime | None = None
    created_before: datetime | None = None

class CreateVpnKeyAdminRequest(BaseModel):
    user_telegram_id: int
    name: str = Field(..., min_length=3, max_length=50)
    vpn_type: Literal["outline", "wireguard"]
    data_limit_gb: float = Field(5.0, ge=1, le=1000)
    country: str = "US"
    expires_in_days: int = Field(30, ge=1, le=365)

class UpdateDataLimitRequest(BaseModel):
    data_limit_gb: float = Field(..., ge=1, le=1000)
    reason: str | None = None

class ResetUsageRequest(BaseModel):
    reason: str | None = None

class RegenerateConfigRequest(BaseModel):
    notify_user: bool = False
```

#### **Response Schemas**

```python
class VpnKeyListItemResponse(BaseModel):
    id: UUID
    user_telegram_id: int
    user_name: str
    name: str
    vpn_type: str
    status: str
    country_code: str
    created_at: datetime
    last_used_at: datetime | None
    data_used_gb: float
    data_limit_gb: float
    usage_percentage: float
    is_active: bool

class VpnKeyDetailResponse(BaseModel):
    id: UUID
    user_id: UUID
    user_telegram_id: int
    user_name: str
    name: str
    vpn_type: str
    status: str
    server_id: UUID
    server_name: str
    country_code: str
    config: str
    created_at: datetime
    expires_at: datetime
    last_used_at: datetime | None
    data_used_gb: float
    data_limit_gb: float
    usage_percentage: float
    is_active: bool

class VpnKeyListResponse(BaseModel):
    keys: list[VpnKeyListItemResponse]
    total: int
    page: int
    page_size: int
    total_pages: int
    has_next: bool
    has_previous: bool

class AdminOperationResult(BaseModel):
    success: bool
    operation: str
    key_id: UUID
    message: str
    timestamp: datetime
    admin_telegram_id: int
```

---

## 🔐 Security & Access Control

### **Authentication Flow**

```
1. Staff member authenticates via Telegram bot
2. Backend issues JWT with role claim (support/admin)
3. All admin endpoints require JWT with appropriate role
4. Backend verifies role on each request
5. All operations logged to audit trail
```

### **Role Verification**

```python
# Dependency for route protection
async def require_admin(
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
) -> User:
    """Verify user has admin role."""
    repo = StaffRoleRepository(db)
    staff_role = await repo.get_by_telegram_id(current_user.telegram_id)
    
    if not staff_role or staff_role.role != "admin":
        raise HTTPException(
            status_code=403,
            detail="Admin role required for this action"
        )
    
    return current_user

async def require_support(
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
) -> User:
    """Verify user has at least support role."""
    repo = StaffRoleRepository(db)
    staff_role = await repo.get_by_telegram_id(current_user.telegram_id)
    
    if not staff_role or staff_role.role not in ["support", "admin"]:
        raise HTTPException(
            status_code=403,
            detail="Staff role required for this action"
        )
    
    return current_user
```

### **Audit Logging**

```python
class AuditLogService:
    """Service for logging admin operations."""

    async def log_operation(
        self,
        operation: str,
        target_type: str,
        target_id: UUID,
        admin_telegram_id: int,
        admin_username: str,
        success: bool,
        details: dict | None = None,
        error_message: str | None = None,
        ip_address: str | None = None
    ):
        """Log admin operation to audit trail."""
        log_entry = AuditLogEntry(
            operation=operation,
            target_type=target_type,
            target_id=target_id,
            admin_telegram_id=admin_telegram_id,
            admin_username=admin_username,
            target_user_telegram_id=details.get("user_telegram_id"),
            success=success,
            details=details or {},
            error_message=error_message,
            ip_address=ip_address,
        )
        
        await self.audit_repo.create(log_entry)
```

---

## 🔧 Agent Enhancements

### **Current Agent Endpoints**

| Endpoint | Status | Notes |
|----------|--------|-------|
| `POST /outline/keys` | ✅ Existing | Create Outline key |
| `DELETE /outline/keys/{id}` | ✅ Existing | Delete Outline key |
| `POST /wireguard/peers` | ✅ Existing | Create WireGuard peer |
| `DELETE /wireguard/peers/{name}` | ✅ Fixed (v0.3.1) | Delete WireGuard peer (idempotent) |
| `GET /health` | ✅ Existing | Health check |
| `GET /metrics` | ✅ Existing | Server metrics |

### **Required Agent Enhancements**

| Enhancement | Priority | Description |
|-------------|----------|-------------|
| **Regenerate Config** | Medium | `POST /wireguard/peers/{name}/regenerate` - Delete and recreate peer |
| **Get Peer Details** | Low | `GET /wireguard/peers/{name}` - Get peer configuration |
| **Error Handling** | High | Improve error messages for better debugging |

---

## 🧪 Testing Strategy

### **Test Categories**

| Type | Count | Description |
|------|-------|-------------|
| Unit Tests | ~50 | Service logic, schemas, validation |
| Integration Tests | ~20 | API endpoints, database |
| Role-Based Tests | ~10 | Support vs Admin permissions |
| Audit Log Tests | ~10 | Verify all operations logged |
| **Total** | **~90** | **All automated** |

### **Test Coverage Goals**

- ✅ **Services:** 95%+ coverage
- ✅ **Routes:** 90%+ coverage
- ✅ **Audit Logging:** 100% coverage
- ✅ **Role Checks:** 100% coverage

---

## 📅 Implementation Timeline

| Phase | Duration | Tasks |
|-------|----------|-------|
| **Phase 1: Backend - Core** | 2 days | Interface, Service, Repositories |
| **Phase 2: Backend - API** | 1.5 days | Routes, Schemas, Validation |
| **Phase 3: Backend - DB** | 0.5 days | Migrations, Seed data |
| **Phase 4: Agent Enhancements** | 1 day | Regenerate endpoint, error handling |
| **Phase 5: Testing** | 1.5 days | Unit, Integration, E2E |
| **Phase 6: Deploy** | 1 day | Docs, Deploy, Monitoring |
| **Total** | **7.5 days** | **~1.5 weeks** |

---

## 📝 Related Documentation

- **Staff Bot Design:** `/home/mowgli/usipipo/usipipo-docs/plans/staff-bot/2026-03-28-staff-bot-design.md`
- **Backend Admin API:** `src/infrastructure/api/v1/routes/admin.py`
- **Agent API:** `/home/mowgli/usipipo/usipipo-agent/internal/api/handlers.go`

---

## 📋 Approval

**Design Author:** uSipipo Development Team
**Review Status:** ✅ Approved
**Approved By:** mowgliph
**Approval Date:** 2026-03-30

---

**Last Updated:** 2026-03-30
**Version:** 1.0
**Status:** Ready for Implementation
