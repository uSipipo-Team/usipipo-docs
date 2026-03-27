# 📋 PRD - uSipipo Landing Page

> Product Requirements Document de la Landing Page

---

## 📊 Información del Producto

| Campo | Valor |
|-------|-------|
| **Nombre** | uSipipo Landing Page |
| **Repositorio** | https://github.com/uSipipo-Team/usipipo-landing |
| **URL** | https://usipipo.duckdns.org |
| **Estado** | ✅ Producción |
| **Versión Actual** | v0.1.0 |
| **Tech Stack** | Flask 3.1+, Jinja2, CSS |

---

## 🎯 Propósito

Landing page de marketing que presenta los servicios de uSipipo VPN, muestra pricing, documentación y dirige usuarios al bot de Telegram para onboarding.

---

## 🎨 Diseño

**Tema:** Cyberpunk Neon Night

**Colores:**
- Primary: `#00F0FF` (Cyan)
- Secondary: `#FF00AA` (Magenta)
- Background: `#0A0A0F` (Void Dark)

**Tipografía:**
- Headings: Orbitron
- Body: Rajdhani

---

## 📄 Páginas

### **1. Home (`/`)**

**Secciones:**
- Hero (headline + CTA)
- Features (4 cards)
- Protocols (WireGuard, Outline)
- How It Works (3 pasos)
- FAQ (5 preguntas)
- Footer

**CTA Principal:**
```
[Comenzar Gratis] → https://t.me/usipipobot
```

---

### **2. Pricing (`/pricing`)**

**Planes:**

| Plan | Precio | Data | Claves |
|------|--------|------|--------|
| **Free** | $0 | 5 GB | 2 |
| **1 Mes** | $7.20 | 10 GB | 5 |
| **3 Meses** | $19.20 | 30 GB | 5 |
| **6 Meses** | $31.20 | 60 GB | 5 |

**CTA:** Cada plan → Telegram bot

---

### **3. Docs (`/docs`)**

**Tarjetas:**
- Setup de WireGuard
- Setup de Outline
- Trust Tunnel (futuro)
- FAQ técnico

---

### **4. Status (`/status`)**

**Indicadores:**
- API: 🟢 Operational
- VPN Servers: 🟢 Operational
- Payments: 🟢 Operational
- Uptime: 99.9%

---

## 📈 Métricas

| KPI | Objetivo | Actual |
|-----|----------|--------|
| Visitas/mes | 5,000 | 1,200 |
| CTR a Telegram | 15% | 8% |
| Bounce rate | < 40% | 52% |
| Load time | < 2s | 1.8s |

---

## 🚀 Deployment

**Stack:**
- Flask (Gunicorn)
- Caddy reverse proxy
- TLS automático
- systemd service

**Puerto:** 5000

---

**Última actualización:** 2026-03-27
