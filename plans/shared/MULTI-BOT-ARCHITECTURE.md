# uSipipo Multi-Bot Architecture

**Date:** 2026-03-28  
**Status:** Design Phase  

---

## 🎯 Overview

The uSipipo ecosystem uses a **multi-bot architecture** with three dedicated Telegram bots, each with clear separation of responsibilities:

| Bot | Handle | Purpose | Status |
|-----|--------|---------|--------|
| **Main Bot** | `@usipipobot` | User features (VPN, Payments, Subscriptions) | ✅ Production (v0.8.0) |
| **Support Bot** | `@uSipipoSupport_Bot` | Customer support tickets | 📋 Design Complete |
| **Staff Bot** | `@uSipipoStaff_Bot` | Administrative operations | 📋 Design Complete |

---

## 🏗️ Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                     uSipipo Ecosystem                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────┐  ┌─────────────────┐  ┌──────────────────┐ │
│  │ @usipipobot    │  │ @usipiposupport │  │ @usipipostaff    │ │
│  │ ─────────────  │  │ ─────────────── │  │ ───────────────  │ │
│  │ USER FEATURES  │  │ SUPPORT TICKETS │  │ ADMIN OPERATIONS │ │
│  │                │  │                 │  │                  │ │
│  │ • VPN Keys     │  │ • Create Ticket │  │ • User Mgmt      │ │
│  │ • Payments     │  │ • View Tickets  │  │ • VPN Keys       │ │
│  │ • Consumption  │  │ • Messages      │  │ • Tickets        │ │
│  │ • Packages     │  │ • Status        │  │ • Servers        │ │
│  │ • Profile      │  │                 │  │ • Stats          │ │
│  └────────┬───────┘  └────────┬────────┘  └─────────┬────────┘ │
│           │                    │                     │          │
│           └────────────────────┼─────────────────────┘          │
│                                │                                 │
│                   ┌────────────▼────────────┐                   │
│                   │   usipipo-backend v0.10 │                   │
│                   │   (Unified API)         │                   │
│                   └─────────────────────────┘                   │
│                                │                                 │
│                   ┌────────────▼────────────┐                   │
│                   │   PostgreSQL + Redis    │                   │
│                   └─────────────────────────┘                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📚 Design Documents

### **Support Bot**
- **Design Doc:** [`support-bot/2026-03-28-support-bot-design.md`](support-bot/2026-03-28-support-bot-design.md)
- **Purpose:** Customer support ticket management
- **Target Users:** End users creating/supporting tickets
- **Key Features:**
  - Create support tickets
  - View ticket status
  - Send messages to tickets
  - Close tickets

### **Staff Bot**
- **Design Doc:** [`staff-bot/2026-03-28-staff-bot-design.md`](staff-bot/2026-03-28-staff-bot-design.md)
- **Purpose:** Administrative operations
- **Target Users:** Authorized staff members only
- **Key Features:**
  - User management (block, unblock, delete)
  - VPN key management
  - Ticket management (assign, respond)
  - Server monitoring
  - Dashboard & analytics

---

## 🔐 Security Model

### **Access Control**

| Bot | Authentication | Authorization |
|-----|---------------|---------------|
| Main Bot | JWT (invisible) | User owns resources |
| Support Bot | JWT (invisible) | User owns tickets |
| Staff Bot | JWT + Telegram ID Whitelist | Staff roles |

### **Staff Authorization**

```bash
# .env configuration
SUPPORT_STAFF_IDS=123456789,987654321,111222333
```

Only Telegram IDs in the whitelist can access the Staff Bot.

---

## 📊 Backend Endpoints

### **Support Bot Uses:**
```
GET    /api/v1/tickets
POST   /api/v1/tickets
GET    /api/v1/tickets/{id}
PATCH  /api/v1/tickets/{id}/close
POST   /api/v1/tickets/{id}/messages
```

### **Staff Bot Uses:**
```
# Dashboard
GET    /api/v1/admin/dashboard/stats

# Users
GET    /api/v1/admin/users
POST   /api/v1/admin/users/{id}/block
POST   /api/v1/admin/users/{id}/unblock
DELETE /api/v1/admin/users/{id}

# Keys
GET    /api/v1/admin/keys
POST   /api/v1/admin/keys/{id}/toggle
DELETE /api/v1/admin/keys/{id}

# Tickets
GET    /api/v1/admin/tickets
PATCH  /api/v1/admin/tickets/{id}/assign
POST   /api/v1/admin/tickets/{id}/messages

# Servers
GET    /api/v1/admin/servers/status
GET    /api/v1/admin/servers/stats
```

---

## 🚀 Implementation Timeline

| Bot | Status | Timeline |
|-----|--------|----------|
| Main Bot | ✅ Production | v0.8.0 released |
| Support Bot | 📋 Design Complete | ~1 week to implement |
| Staff Bot | 📋 Design Complete | ~2 weeks to implement |

---

## 📝 Migration Plan

### **From Main Bot to Support Bot**

1. Extract ticket module from `usipipo-telegram-bot`
2. Create `usipipo-support-bot` repository
3. Implement support-specific features
4. Deploy support bot
5. Remove tickets from main bot (v0.9.0)

### **Staff Bot (New)**

1. Create `usipipo-staff-bot` repository
2. Implement all admin handlers
3. Configure staff whitelist
4. Deploy staff bot
5. Test all admin workflows

---

## 🎯 Benefits of Multi-Bot Architecture

### **Separation of Concerns**
- ✅ User features isolated from support
- ✅ Support isolated from admin operations
- ✅ Clear boundaries for each bot

### **Scalability**
- ✅ Independent deployment
- ✅ Independent scaling
- ✅ Fault isolation

### **Security**
- ✅ Staff access strictly controlled
- ✅ Audit logging per bot
- ✅ Different security models per use case

### **User Experience**
- ✅ Dedicated support channel
- ✅ No mixing of features and support
- ✅ Clear communication paths

### **Operational Efficiency**
- ✅ Staff can focus on support
- ✅ Admin operations streamlined
- ✅ Better metrics and monitoring

---

## 🔗 Related Documentation

- **Ecosystem Context:** `/plans/shared/ECOSYSTEM-CONTEXT.md`
- **Migration Progress:** `/plans/shared/MIGRATION-PROGRESS.md`
- **Legacy Bot Migration:** `/plans/shared/LEGACY-BOT-MIGRATION-SUMMARY.md`
- **Backend API:** http://localhost:8001/docs

---

**Last Updated:** 2026-03-28  
**Version:** 1.0  
**Status:** Design Phase
