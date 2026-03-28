# Phase 7: Referrals + Tickets Design

**Date:** 2026-03-28  
**Status:** Complete  
**Version:** v0.8.0  
**Branch:** `feature/phase-7-referrals-tickets`

---

## Overview

Migration of Referrals and Tickets features from legacy monorepo to new hexagonal architecture.

This phase completes the user engagement features:
- **Referrals:** Users can invite friends, track referrals, and redeem rewards
- **Tickets:** Users can create support tickets, view history, and communicate with support

---

## Architecture

### Bot Architecture (Hexagonal)

```
┌─────────────────────────────────────────┐
│           Telegram Bot                  │
├─────────────────────────────────────────┤
│  Handlers (Application Layer)           │
│  ├── ReferralsHandler                   │
│  └── TicketsHandler                     │
├─────────────────────────────────────────┤
│  Keyboards (UI Layer)                   │
│  ├── referrals.py                       │
│  ├── messages_referrals.py              │
│  ├── tickets.py                         │
│  └── messages_tickets.py                │
├─────────────────────────────────────────┤
│  APIClient (Infrastructure Layer)       │
│  └── httpx async client                 │
└─────────────────────────────────────────┘
```

### Backend Integration

- **APIClient:** `httpx` async client for FastAPI endpoints
- **Authentication:** JWT token in Authorization header
- **Error Handling:** Graceful degradation with user-friendly messages

---

## Commands

### Referrals Commands

| Command | Description | Handler |
|---------|-------------|---------|
| `/referidos` | View referral stats and menu | `ReferralsHandler.show_stats()` |
| `/invitar` | Get referral link | `ReferralsHandler.get_invite_link()` |

### Tickets Commands

| Command | Description | Handler |
|---------|-------------|---------|
| `/tickets` | List all tickets | `TicketsHandler.list_tickets()` |
| `/nuevoticket` | Create new ticket | `TicketsHandler.create_ticket()` |
| `/mistickets` | View my tickets | `TicketsHandler.my_tickets()` |

---

## Backend Endpoints

### Referrals Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/referrals/me` | Get current user's referral stats |
| `POST` | `/api/v1/referrals/apply` | Apply referral code |
| `POST` | `/api/v1/referrals/redeem` | Redeem referral credits |

#### GET /api/v1/referrals/me

**Response:**
```json
{
  "code": "JUAN123",
  "link": "https://t.me/usipipobot?start=JUAN123",
  "total_referrals": 5,
  "active_referrals": 3,
  "credits_earned": 25,
  "credits_redeemed": 10,
  "credits_available": 15
}
```

#### POST /api/v1/referrals/apply

**Request:**
```json
{
  "referral_code": "PEDRO456"
}
```

**Response:**
```json
{
  "success": true,
  "bonus": 5,
  "message": "Código aplicado exitosamente"
}
```

#### POST /api/v1/referrals/redeem

**Request:**
```json
{
  "credits": 10
}
```

**Response:**
```json
{
  "success": true,
  "data_awarded": 5.0,
  "remaining_credits": 5
}
```

---

### Tickets Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/tickets` | Create new ticket |
| `GET` | `/api/v1/tickets` | List all tickets (paginated) |
| `GET` | `/api/v1/tickets/{id}` | Get ticket details |
| `POST` | `/api/v1/tickets/{id}/messages` | Add message to ticket |
| `PATCH` | `/api/v1/tickets/{id}/close` | Close ticket |

#### POST /api/v1/tickets

**Request:**
```json
{
  "category": "technical",
  "subject": "VPN no conecta",
  "description": "No puedo conectar mi VPN desde ayer"
}
```

**Response:**
```json
{
  "id": "TKT-1234567",
  "category": "technical",
  "status": "open",
  "created_at": "2026-03-28T10:00:00Z"
}
```

#### GET /api/v1/tickets

**Query Params:**
- `page` (int): Page number (default: 1)
- `limit` (int): Items per page (default: 10)
- `status` (str): Filter by status (open, closed, all)

**Response:**
```json
{
  "items": [
    {
      "id": "TKT-1234567",
      "category": "technical",
      "status": "open",
      "created_at": "2026-03-28T10:00:00Z",
      "last_message_at": "2026-03-28T11:30:00Z"
    }
  ],
  "total": 5,
  "page": 1,
  "pages": 1
}
```

#### GET /api/v1/tickets/{id}

**Response:**
```json
{
  "id": "TKT-1234567",
  "category": "technical",
  "status": "open",
  "subject": "VPN no conecta",
  "description": "No puedo conectar mi VPN desde ayer",
  "messages": [
    {
      "id": "MSG-001",
      "sender": "user",
      "message": "No puedo conectar",
      "created_at": "2026-03-28T10:00:00Z"
    },
    {
      "id": "MSG-002",
      "sender": "support",
      "message": "¿Qué error ves?",
      "created_at": "2026-03-28T11:30:00Z"
    }
  ],
  "created_at": "2026-03-28T10:00:00Z"
}
```

#### POST /api/v1/tickets/{id}/messages

**Request:**
```json
{
  "message": "Veo error 403"
}
```

**Response:**
```json
{
  "id": "MSG-003",
  "sender": "user",
  "message": "Veo error 403",
  "created_at": "2026-03-28T12:00:00Z"
}
```

#### PATCH /api/v1/tickets/{id}/close

**Response:**
```json
{
  "id": "TKT-1234567",
  "status": "closed",
  "closed_at": "2026-03-28T12:30:00Z"
}
```

---

## Files Created

### Referrals Module

| File | Purpose |
|------|---------|
| `src/bot/handlers/referrals.py` | Command and callback handlers |
| `src/bot/keyboards/referrals.py` | Inline keyboard layouts |
| `src/bot/keyboards/messages_referrals.py` | Message constants and templates |

### Tickets Module

| File | Purpose |
|------|---------|
| `src/bot/handlers/tickets.py` | Command and callback handlers |
| `src/bot/keyboards/tickets.py` | Inline keyboard layouts |
| `src/bot/keyboards/messages_tickets.py` | Message constants and templates |

---

## Tests

### Test Coverage

| Module | Unit Tests | Integration Tests | Coverage |
|--------|-----------|-------------------|----------|
| Referrals | 13 | 2 | 95% |
| Tickets | 15 | 2 | 96% |
| **Total** | **28** | **4** | **95.5%** |

### Test Results

```
============================= test session starts ==============================
platform linux -- Python 3.13, pytest-8.4.1
collected 32 items

tests/unit/bot/handlers/test_referrals.py .............                  [ 40%]
tests/unit/bot/handlers/test_tickets.py ...............                  [ 87%]
tests/integration/bot/test_referrals_integration.py ..                   [ 93%]
tests/integration/bot/test_tickets_integration.py ..                     [100%]

========================= 32 passed in 2.45s =============================
```

### Test Categories

#### Referrals Tests
- ✅ Show referral stats
- ✅ Get invite link
- ✅ Apply referral code (success)
- ✅ Apply referral code (invalid)
- ✅ Apply referral code (self)
- ✅ Redeem credits (success)
- ✅ Redeem credits (insufficient)
- ✅ Callback query handling
- ✅ Error handling

#### Tickets Tests
- ✅ Create ticket (all categories)
- ✅ List tickets (empty)
- ✅ List tickets (with data)
- ✅ View ticket details
- ✅ View ticket (not found)
- ✅ Close ticket (success)
- ✅ Close ticket (already closed)
- ✅ Add message to ticket
- ✅ Pagination handling
- ✅ Error handling

---

## State Machines

### Referrals States

```
┌─────────────────┐
│  Viewing Stats  │ ← Initial state
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌─────────┐ ┌──────────┐
│Redeeming│ │ Applying │
└─────────┘ └──────────┘
```

**States:**
- **Viewing Stats:** User viewing referral statistics and menu
- **Redeeming:** User selecting credits to redeem
- **Applying:** User entering referral code

### Tickets States

```
┌───────────┐
│ Browsing  │ ← Initial state
└─────┬─────┘
      │
   ┌──┴──┐
   │     │
   ▼     ▼
┌─────────┐ ┌──────────┐
│ Viewing │ │ Creating │
└─────────┘ └──────────┘
```

**States:**
- **Browsing:** User viewing tickets list
- **Viewing:** User viewing ticket details with messages
- **Creating:** User creating new ticket (selecting category)

---

## Callback Query Patterns

### Referrals Callbacks

| Pattern | Data | Handler |
|---------|------|---------|
| `referral_redeem_confirm:{credits}` | credits (int) | `handle_redeem_confirm()` |
| `referral_apply` | - | `handle_apply_code()` |
| `referral_back` | - | `handle_back()` |

### Tickets Callbacks

| Pattern | Data | Handler |
|---------|------|---------|
| `ticket_view:{id}` | ticket_id (str) | `handle_view_ticket()` |
| `ticket_cat:{category}` | category (str) | `handle_select_category()` |
| `ticket_close:{id}` | ticket_id (str) | `handle_close_ticket()` |
| `tickets_back` | - | `handle_back()` |

---

## Message Templates

### Referrals Messages

| Constant | Purpose |
|----------|---------|
| `REFERRAL_STATS` | Shows stats with menu |
| `INVITE_LINK` | Shows referral link |
| `REDEEM_CONFIRMATION` | Confirms redemption |
| `APPLY_SUCCESS` | Code applied successfully |
| `APPLY_ERROR` | Invalid or expired code |

### Tickets Messages

| Constant | Purpose |
|----------|---------|
| `TICKETS_LIST` | Shows list of tickets |
| `TICKET_DETAIL` | Shows ticket details |
| `CREATE_TICKET` | Category selection prompt |
| `TICKET_CREATED` | Confirmation of creation |
| `TICKET_CLOSED` | Closure confirmation |
| `NO_TICKETS` | Empty state message |

---

## Error Handling

### API Errors

| Status Code | User Message |
|-------------|--------------|
| 400 | "Solicitud inválida. Por favor intenta de nuevo." |
| 401 | "Sesión expirada. Por favor inicia de nuevo." |
| 404 | "No encontrado. El recurso no existe." |
| 409 | "Conflicto. Esta acción ya fue realizada." |
| 500 | "Error interno. Nuestro equipo fue notificado." |

### Network Errors

- **Timeout:** "La solicitud tomó demasiado tiempo. Intenta de nuevo."
- **Connection Error:** "Error de conexión. Verifica tu internet."

---

## Release Information

| Field | Value |
|-------|-------|
| **Version** | v0.8.0 |
| **Branch** | `feature/phase-7-referrals-tickets` |
| **Status** | Ready for merge |
| **PR** | Pending |
| **Tests** | 32 passed (100%) |
| **Coverage** | 95.5% |

---

## Related Documentation

- [Referrals Flow](../../flows/telegram-bot/referrals-flow.md)
- [Tickets Flow](../../flows/telegram-bot/tickets-flow.md)
- [Backend API Flow](../../flows/backend-api-flow.md)
- [Hexagonal Architecture Migration](2026-03-24-hexagonal-architecture-migration.md)

---

**Última actualización:** 2026-03-28
