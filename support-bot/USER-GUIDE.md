# uSipipo Support Bot - User Guide

**Date:** 2026-03-28  
**Version:** 1.0  
**Bot Handle:** `@uSipipoSupport_Bot`  

---

## 👋 Welcome

Welcome to the uSipipo Support Bot! This bot helps you create and manage support tickets for technical issues, billing questions, service inquiries, and more.

---

## 🚀 Getting Started

### **1. Start the Bot**

1. Open Telegram
2. Search for `@uSipipoSupport_Bot`
3. Click "Start" or send `/start`

### **2. Automatic Authentication**

The bot automatically authenticates you using your Telegram account. No login required!

---

## 📋 Available Commands

| Command | Description |
|---------|-------------|
| `/start` | Start the bot and see welcome message |
| `/help` | Display help information |
| `/tickets` | View all your tickets |
| `/nuevoticket` | Create a new support ticket |

---

## 🎫 Creating a New Ticket

### **Step 1: Start Creation**

Send the command:
```
/nuevoticket
```

### **Step 2: Select Category**

The bot will show you a keyboard with categories:

```
🎫 Crear Nuevo Ticket

Por favor, seleccioná una categoría:

• 🖥️ Técnico - Problemas con VPN, conexión
• 💳 Pagos - Problemas con pagos, facturación
• 📦 Servicios - Planes, paquetes de datos
• ❓ General - Otras consultas
```

**Categories:**
- **🖥️ Técnico** - VPN connection issues, technical problems
- **💳 Pagos** - Payment issues, billing questions, refunds
- **📦 Servicios** - Plans, data packages, subscriptions
- **❓ General** - Other inquiries, general questions

### **Step 3: Ticket Created**

After selecting a category, the bot will create your ticket:

```
✅ *Ticket Creado Exitosamente!*

Número de ticket: *#TKT-12345*
Asunto: Consulta de technical

Te responderemos lo antes posible.
Usa /tickets para ver el estado.
```

---

## 📝 Viewing Your Tickets

### **List All Tickets**

Send the command:
```
/tickets
```

The bot will show your tickets:

```
🎫 *Tus Tickets*

🟢 *#TKT-12345* - VPN connection issue
Estado: OPEN
Creado: 2026-03-28

🟡 *#TKT-12344* - Billing question
Estado: RESPONDED
Creado: 2026-03-27

[🔙 Volver] button to refresh
```

**Status Indicators:**
- 🟢 **OPEN** - Ticket is open, waiting for staff response
- 🟡 **RESPONDED** - Staff has responded, check for updates
- 🔵 **RESOLVED** - Issue has been resolved
- 🔴 **CLOSED** - Ticket has been closed

### **View Ticket Details**

Click on any ticket button to see details:

```
🎫 *Ticket #TKT-12345*

📋 *Estado:* OPEN
📝 *Asunto:* VPN connection issue
📂 *Categoría:* technical
📅 *Creado:* 2026-03-28

💬 *Último mensaje:* Sin mensajes aún

[💬 Ver Mensajes] [📩 Enviar Mensaje] [✅ Cerrar Ticket] [🔙 Volver]
```

---

## 💬 Sending Messages to Tickets

### **View Message History**

1. Open a ticket detail view
2. Click "💬 Ver Mensajes"

The bot will show the conversation history.

### **Send a Message**

1. Open a ticket detail view
2. Click "📩 Enviar Mensaje"
3. Type your message
4. The bot will add it to your ticket

**Confirmation:**
```
✅ *Mensaje Enviado*

Tu mensaje ha sido agregado al ticket #TKT-12345.
Te notificaremos cuando haya una respuesta.
```

---

## ✅ Closing a Ticket

When your issue is resolved:

1. Open the ticket detail view
2. Click "✅ Cerrar Ticket"
3. Confirm closure

**Confirmation:**
```
✅ *Ticket Cerrado*

Tu ticket #TKT-12345 ha sido cerrado.
Gracias por contactar soporte.
```

---

## 🔔 Notifications

The bot will notify you when:

- ✅ A staff member responds to your ticket
- ✅ Your ticket status changes
- ✅ Your ticket is closed

**Make sure notifications are enabled for the bot!**

---

## ❓ Frequently Asked Questions

### **How long does it take to get a response?**

Our team typically responds within 2-4 hours during business hours (9 AM - 6 PM ART). For urgent issues, we respond as quickly as possible.

### **Can I create multiple tickets?**

Yes, you can create multiple tickets for different issues. However, please try to keep related issues in the same ticket.

### **What if I need to attach a file?**

Currently, the bot doesn't support file attachments. If you need to share a screenshot or file, please mention it in your ticket and a staff member will provide an alternative method.

### **Can I reopen a closed ticket?**

No, closed tickets cannot be reopened. If you have a new issue or the problem persists, please create a new ticket and reference the old ticket number.

### **Is this bot connected to the main uSipipo bot?**

This is a dedicated support bot. Your authentication is shared with the main bot (`@usipipobot`), but all support interactions happen here for better organization.

---

## 🆘 Troubleshooting

### **Bot doesn't respond**

1. Check your internet connection
2. Make sure you sent `/start` first
3. Try sending `/help`

### **"User not authenticated" error**

1. Send `/start` to re-authenticate
2. Wait a few seconds
3. Try your command again

### **Ticket not appearing**

1. Refresh by sending `/tickets` again
2. Make sure you're using the same Telegram account
3. Contact support if the issue persists

---

## 💡 Tips for Better Support

1. **Be specific:** Clearly describe your issue
2. **Include details:** Error messages, steps to reproduce
3. **One issue per ticket:** Don't mix unrelated problems
4. **Be patient:** We respond as quickly as possible
5. **Check status:** Use `/tickets` to see all your tickets

---

## 📞 Additional Support

If you need additional help:

- **Main Bot:** `@usipipobot` for account management
- **Email:** usipipo@gmail.com
- **Website:** https://usipipo.com

---

**Thank you for using uSipipo Support Bot!** 🎉

**Last Updated:** 2026-03-28  
**Version:** 1.0
