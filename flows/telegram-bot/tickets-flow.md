# Tickets Flow

> User journey for the Support Tickets feature in @usipipobot

---

## Overview

The Tickets feature allows users to create support requests, track their status, and communicate with the support team. Users can:
- Create new tickets by category
- View their ticket history
- See ticket details and messages
- Close tickets when resolved

---

## User Journey

### **Main Flow: Creating a Ticket**

```
┌─────────────────┐
│   Usuario       │
└────────┬────────┘
         │
         │ Envía /nuevoticket
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Muestra selección de categoría
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  🎫 Crear Nuevo Ticket                  │
│                                         │
│  Selecciona la categoría:               │
│                                         │
│  🌐 Técnico (VPN, conexión)             │
│  💳 Pagos (facturación)                 │
│  📦 Servicios (planes, datos)           │
│  💬 General (otras consultas)           │
│                                         │
│  [Cancelar]                             │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario selecciona categoría
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Solicita descripción del problema
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  📝 Describe tu problema                │
│                                         │
│  Por favor describe tu problema         │
│  con el mayor detalle posible:          │
│                                         │
│  [Escribe tu mensaje...]                │
│                                         │
│  [Enviar] [Cancelar]                    │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario escribe descripción
         ▼
┌─────────────────┐
│  Backend        │
└────────┬────────┘
         │
         │ POST /api/v1/tickets
         │ Crea ticket en DB
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Muestra confirmación
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  ✅ Ticket Creado                       │
│                                         │
│  Número: TKT-1234567                    │
│  Categoría: Técnico                     │
│  Estado: Abierto                        │
│                                         │
│  Un agente te responderá pronto.        │
│                                         │
│  [📜 Ver Tickets] [↩️ Inicio]           │
│                                         │
└─────────────────────────────────────────┘
```

---

## States

### **Browsing** (Initial State)

User is viewing their list of tickets.

**Entry:** User sends `/tickets` or `/mistickets`  
**Actions Available:**
- View list of tickets
- Click on a ticket to view details
- Create new ticket
- Filter by status (open/closed)

**Exit Triggers:**
- User clicks on ticket → **Viewing** state
- User clicks "Crear Ticket" → **Creating** state

---

### **Viewing** (Viewing Ticket Details)

User is viewing a specific ticket with its messages.

**Entry:** User clicks on a ticket from the list  
**Actions Available:**
- View ticket details
- View message history
- Add a reply
- Close ticket (if open)
- Go back to list

**Flow:**
```
┌─────────────────┐
│  Viewing        │
└────────┬────────┘
         │
         │ GET /api/v1/tickets/{id}
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  🎫 Ticket: TKT-1234567                 │
│  Categoría: Técnico                     │
│  Estado: 🟢 Abierto                     │
│  Creado: 28/03/2026 10:00               │
│                                         │
│  ─────────────────────────────────      │
│                                         │
│  💬 Mensajes:                           │
│                                         │
│  👤 Tú (10:00):                         │
│  No puedo conectar mi VPN desde ayer    │
│                                         │
│  🎧 Soporte (11:30):                    │
│  Hola Juan, ¿qué error ves cuando      │
│  intentas conectar?                     │
│                                         │
│  ─────────────────────────────────      │
│                                         │
│  [📝 Responder] [❌ Cerrar Ticket]      │
│  [↩️ Volver a la lista]                 │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario selecciona acción
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Ejecuta acción
         ▼
┌─────────────────┐
│   Usuario       │
│  (Acción)       │
└─────────────────┘
```

**Reply Flow:**
```
┌─────────────────┐
│  Viewing        │
└────────┬────────┘
         │
         │ Usuario clicks "Responder"
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  📝 Responder al Ticket                 │
│                                         │
│  Ticket: TKT-1234567                    │
│                                         │
│  Escribe tu respuesta:                  │
│                                         │
│  [Escribe tu mensaje...]                │
│                                         │
│  [Enviar] [Cancelar]                    │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario envía mensaje
         ▼
┌─────────────────┐
│  Backend        │
└────────┬────────┘
         │
         │ POST /api/v1/tickets/{id}/messages
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Muestra mensaje agregado
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  ✅ Mensaje Enviado                     │
│                                         │
│  Tu respuesta ha sido agregada al      │
│  ticket. Un agente te responderá.       │
│                                         │
│  [📝 Ver Ticket] [↩️ Volver]            │
│                                         │
└─────────────────────────────────────────┘
```

**Close Flow:**
```
┌─────────────────┐
│  Viewing        │
└────────┬────────┘
         │
         │ Usuario clicks "Cerrar Ticket"
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  ❌ ¿Cerrar Ticket?                     │
│                                         │
│  Ticket: TKT-1234567                    │
│                                         │
│  ¿Estás seguro de cerrar este ticket?   │
│  Esta acción no se puede deshacer.      │
│                                         │
│  [✅ Sí, Cerrar] [↩️ Cancelar]          │
│                                         │
└─────────────────────────────────────────┘
         │
         │ Usuario confirma
         ▼
┌─────────────────┐
│  Backend        │
└────────┬────────┘
         │
         │ PATCH /api/v1/tickets/{id}/close
         ▼
┌─────────────────┐
│  Bot            │
└────────┬────────┘
         │
         │ Muestra confirmación
         ▼
┌─────────────────────────────────────────┐
│                                         │
│  ✅ Ticket Cerrado                      │
│                                         │
│  El ticket TKT-1234567 ha sido         │
│  cerrado.                               │
│                                         │
│  Gracias por usar uSipipo.              │
│                                         │
│  [📜 Ver Tickets] [↩️ Inicio]           │
│                                         │
└─────────────────────────────────────────┘
```

**Exit Triggers:**
- User adds message → Stays in **Viewing** state
- User closes ticket → **Browsing** state
- User clicks "Volver" → **Browsing** state

---

### **Creating** (Creating New Ticket)

User is in the process of creating a new ticket.

**Entry:** User sends `/nuevoticket` or clicks "Crear Ticket"  
**Actions Available:**
- Select category
- Enter description
- Cancel creation

**Exit Triggers:**
- Ticket created → **Browsing** state
- User cancels → **Browsing** state

---

## Callback Queries

### `ticket_view:{id}`

**Purpose:** User wants to view ticket details  
**Data:** `id` (string) - Ticket ID  
**Handler:** `handle_view_ticket()`

**Flow:**
1. User clicks on ticket from list
2. Bot calls `GET /api/v1/tickets/{id}`
3. Bot displays ticket details with messages
4. User can reply or close ticket

---

### `ticket_cat:{category}`

**Purpose:** User selecting ticket category  
**Data:** `category` (string) - Category slug  
**Handler:** `handle_select_category()`

**Categories:**
- `technical` - Técnico (VPN, conexión)
- `billing` - Pagos (facturación)
- `services` - Servicios (planes, datos)
- `general` - General (otras consultas)

**Flow:**
1. User clicks category button
2. Bot prompts for description
3. User types description
4. Bot creates ticket via `POST /api/v1/tickets`

---

### `ticket_close:{id}`

**Purpose:** User wants to close a ticket  
**Data:** `id` (string) - Ticket ID  
**Handler:** `handle_close_ticket()`

**Flow:**
1. User clicks "Cerrar Ticket"
2. Bot shows confirmation dialog
3. User confirms
4. Bot calls `PATCH /api/v1/tickets/{id}/close`
5. Bot shows confirmation message

---

### `tickets_back`

**Purpose:** User wants to go back to tickets list  
**Data:** None  
**Handler:** `handle_back()`

**Flow:**
1. User clicks "Volver"
2. Bot returns to tickets list

---

## Categories

| Slug | Display Name | Description | Icon |
|------|--------------|-------------|------|
| `technical` | Técnico | VPN, conexión, configuración | 🌐 |
| `billing` | Pagos | Facturación, pagos, reembolsos | 💳 |
| `services` | Servicios | Planes, datos, consumo | 📦 |
| `general` | General | Otras consultas | 💬 |

---

## Messages

### TICKETS_LIST

Shows list of user's tickets.

**Template:**
```
📜 Mis Tickets

Tienes {total} tickets ({open} abiertos, {closed} cerrados)

─────────────────────────────────

🎫 TKT-1234567
📂 Categoría: Técnico
🟢 Estado: Abierto
📅 Creado: 28/03/2026 10:00
💬 Último mensaje: 11:30

─────────────────────────────────

🎫 TKT-1234566
📂 Categoría: Pagos
🔴 Estado: Cerrado
📅 Creado: 27/03/2026 15:00
💬 Último mensaje: 27/03/2026 16:30

─────────────────────────────────

[➕ Crear Ticket] [🔄 Actualizar]
```

**Empty State:**
```
📜 Mis Tickets

No tienes tickets creados.

¿Necesitas ayuda? Crea un ticket y nuestro
equipo de soporte te responderá pronto.

[➕ Crear Ticket] [↩️ Volver]
```

---

### TICKET_DETAIL

Shows ticket details with messages.

**Template:**
```
🎫 Ticket: {ticket_id}
📂 Categoría: {category}
🟢 Estado: {status}
📅 Creado: {created_at}

─────────────────────────────────

💬 Mensajes:

👤 Tú ({time}):
{message}

🎧 {agent_name} ({time}):
{message}

─────────────────────────────────

[📝 Responder] [❌ Cerrar Ticket]
[↩️ Volver a la lista]
```

---

### CREATE_TICKET

Category selection prompt.

**Template:**
```
🎫 Crear Nuevo Ticket

Selecciona la categoría:

🌐 Técnico (VPN, conexión)
💳 Pagos (facturación)
📦 Servicios (planes, datos)
💬 General (otras consultas)

[Cancelar]
```

---

### TICKET_CREATED

Confirmation of ticket creation.

**Template:**
```
✅ Ticket Creado

Número: {ticket_id}
Categoría: {category}
Estado: Abierto

Un agente te responderá pronto.

Puedes ver el estado de tu ticket en
/tickets

[📜 Ver Tickets] [↩️ Inicio]
```

---

### TICKET_CLOSED

Closure confirmation.

**Template:**
```
✅ Ticket Cerrado

El ticket {ticket_id} ha sido cerrado.

Gracias por usar uSipipo. Si necesitas
más ayuda, crea un nuevo ticket.

[📜 Ver Tickets] [↩️ Inicio]
```

---

### NO_TICKETS

Empty state when user has no tickets.

**Template:**
```
📜 Sin Tickets

No tienes tickets creados.

¿Necesitas ayuda? Nuestro equipo de
soporte está aquí para ayudarte.

[➕ Crear Ticket] [↩️ Volver]
```

---

## Backend Integration

### API Calls

| Action | Endpoint | Method |
|--------|----------|--------|
| List tickets | `/api/v1/tickets` | GET |
| Get ticket | `/api/v1/tickets/{id}` | GET |
| Create ticket | `/api/v1/tickets` | POST |
| Add message | `/api/v1/tickets/{id}/messages` | POST |
| Close ticket | `/api/v1/tickets/{id}/close` | PATCH |

### Request/Response Examples

#### Create Ticket

**Request:**
```json
{
  "category": "technical",
  "subject": "VPN no conecta",
  "description": "No puedo conectar mi VPN desde ayer. Veo error 403."
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

#### Add Message

**Request:**
```json
{
  "message": "Veo error 403 cuando intento conectar"
}
```

**Response:**
```json
{
  "id": "MSG-003",
  "sender": "user",
  "message": "Veo error 403 cuando intento conectar",
  "created_at": "2026-03-28T12:00:00Z"
}
```

### Error Handling

| Error | Status Code | User Message |
|-------|-------------|--------------|
| Ticket not found | 404 | "Ticket no encontrado" |
| Already closed | 409 | "Este ticket ya está cerrado" |
| Invalid category | 400 | "Categoría inválida" |
| Network error | - | "Error de conexión. Intenta de nuevo." |

---

## Related Documentation

- [Phase 7 Design](../../plans/telegram-bot/2026-03-28-phase-7-referrals-tickets-design.md)
- [Referrals Flow](referrals-flow.md)
- [Telegram Bot Flow](telegram-bot-flow.md)

---

**Última actualización:** 2026-03-28
