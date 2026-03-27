# 📚 uSipipo Docs - Documentación Centralizada

> Documentación oficial del ecosistema uSipipo VPN

[![Licencia](https://img.shields.io/badge/licencia-MIT-blue.svg)](LICENSE)
[![Versión](https://img.shields.io/badge/versión-1.0.0-green.svg)](./changelog.md)
[![Estado](https://img.shields.io/badge/estado-producción-success.svg)]()

---

## 🎯 ¿Qué es uSipipo?

**uSipipo** es un ecosistema de servicios VPN dirigido a la comunidad LATAM, proporcionando acceso seguro y privado a internet a través de múltiples protocolos (WireGuard, Outline/Shadowsocks) con autenticación vía Telegram y pagos en criptomonedas + Telegram Stars.

### 🌟 Características Principales

- ✅ **VPN Keys instantáneas** - Generación automática vía Telegram
- ✅ **Múltiples protocolos** - WireGuard y Outline (Shadowsocks)
- ✅ **Pagos flexibles** - Crypto (USDT/USDC) y Telegram Stars
- ✅ **Facturación por consumo** - Paga solo por lo que usas
- ✅ **Programa de referidos** - Gana créditos invitando amigos
- ✅ **Sin logs** - Tu privacidad es nuestra prioridad

---

## 📖 Estructura de la Documentación

### 🎨 Brand Identity

Documentación de marca y diseño visual del ecosistema.

| Documento | Descripción |
|-----------|-------------|
| [Identidad](brand/identity.md) | Visión, misión y valores de marca |
| [Paleta de Colores](brand/color-palette.md) | Sistema de colores Cyberpunk Neon Night |
| [Tipografía](brand/typography.md) | Fuentes y jerarquías tipográficas |
| [Logo Assets](brand/logo-assets.md) | Logotipos y variaciones |
| [Voz y Tono](brand/voice-tone.md) | Guía de comunicación y estilo |

### 📖 Contexto del Proyecto

Información general sobre el ecosistema y su arquitectura.

| Documento | Descripción |
|-----------|-------------|
| [Visión General](context/project-overview.md) | Propósito y objetivos del proyecto |
| [Arquitectura del Ecosistema](context/ecosystem-architecture.md) | Diagrama de componentes y relaciones |
| [User Personas](context/user-personas.md) | Perfiles de usuarios objetivo |
| [Modelo de Negocio](context/business-model.md) | Estrategia de monetización y pricing |

### 📋 PRDs (Product Requirements Documents)

Especificaciones detalladas de cada producto del ecosistema.

| Repositorio | PRD | Estado | Versión |
|-------------|-----|--------|---------|
| **usipipo-backend** | [PRD Backend](prds/prd-usipipo-backend.md) | ✅ Producción | v0.10.0 |
| **usipipo-commons** | [PRD Commons](prds/prd-usipipo-commons.md) | ✅ Producción | v0.12.0 |
| **usipipo-telegram-bot** | [PRD Bot](prds/prd-usipipo-telegram-bot.md) | ✅ Producción | v0.3.0 |
| **usipipo-landing** | [PRD Landing](prds/prd-usipipo-landing.md) | ✅ Producción | v0.1.0 |
| **usipipo-miniapp-web** | [PRD MiniApp](prds/prd-usipipo-miniapp-web.md) | 🟡 Desarrollo | v0.1.0 |
| **usipipovpnapp** | [PRD Android](prds/prd-usipipovpnapp.md) | 🟡 Desarrollo | v1.0.0 |

### 🔄 App Flows

Diagramas de flujo de usuario y procesos del sistema.

| Flujo | Descripción |
|-------|-------------|
| [Backend API Flow](flows/backend-api-flow.md) | Flujo de autenticación y gestión de VPN keys |
| [Telegram Bot Flow](flows/telegram-bot-flow.md) | Journey del usuario en el bot de Telegram |
| [Mobile App Flow](flows/mobile-app-flow.md) | Flujo de conexión VPN en la app Android |
| [Payment Flow](flows/payment-flow.md) | Proceso de pagos (Crypto + Stars) |
| [VPN Connection Flow](flows/vpn-connection-flow.md) | Establecimiento de conexión VPN |

### 🛠️ Technology Stack

Documentación técnica de tecnologías utilizadas.

| Documento | Descripción |
|-----------|-------------|
| [Stack Overview](technology/stack-overview.md) | Visión general de todas las tecnologías |
| [Backend Stack](technology/backend-stack.md) | FastAPI, PostgreSQL, Redis, SQLAlchemy |
| [Frontend Stack](technology/frontend-stack.md) | Flask, React Native, Flutter |
| [Infrastructure Stack](technology/infrastructure-stack.md) | Docker, Kubernetes, CI/CD, Cloud |

### 🔌 API Documentation

Referencia completa de APIs y protocolos.

| Documento | Descripción |
|-----------|-------------|
| [Backend API Reference](apis/backend-api-reference.md) | Referencia completa de endpoints (50+) |
| [Webhooks Integration](apis/webhooks-integration.md) | TronDealer, Telegram, payment webhooks |
| [VPN Protocols](apis/vpn-protocols.md) | WireGuard y Outline/Shadowsocks |

---

## 🚀 Inicio Rápido

### Para Nuevos Desarrolladores

1. **Lee la visión general** → [Project Overview](context/project-overview.md)
2. **Entiende la arquitectura** → [Ecosystem Architecture](context/ecosystem-architecture.md)
3. **Revisa el stack tecnológico** → [Stack Overview](technology/stack-overview.md)
4. **Explora los PRDs** de tu área de interés

### Para Contribuidores

1. **Configura tu entorno** → [Getting Started](getting-started.md)
2. **Revisa estándares de código** → [Code Quality Rules](https://github.com/uSipipo-Team/usipipo-code-quality)
3. **Entiende los flujos** → [App Flows](#-app-flows)

### Para Usuarios de la API

1. **Autenticación** → [Backend API Reference](apis/backend-api-reference.md#autenticación)
2. **Endpoints principales** → [Backend API Reference](apis/backend-api-reference.md#endpoints)
3. **Webhooks** → [Webhooks Integration](apis/webhooks-integration.md)

---

## 📊 Estado del Ecosistema

| Componente | Estado | Versión | Última Actualización |
|------------|--------|---------|---------------------|
| Backend API | ✅ Producción | v0.10.0 | 2026-03-24 |
| Commons Library | ✅ Producción | v0.12.0 | 2026-03-22 |
| Telegram Bot | ✅ Producción | v0.3.0 | 2026-03-24 |
| Landing Page | ✅ Producción | v0.1.0 | 2026-03-19 |
| MiniApp Web | 🟡 Desarrollo | v0.1.0 | En progreso |
| Android App | 🟡 Desarrollo | v1.0.0 | Fase 2/8 |

---

## 🔗 Enlaces Externos

| Recurso | URL |
|---------|-----|
| **Backend (GitHub)** | https://github.com/uSipipo-Team/usipipo-backend |
| **Commons (PyPI)** | https://pypi.org/project/usipipo-commons/ |
| **Telegram Bot** | https://t.me/usipipobot |
| **Landing Page** | https://usipipo.duckdns.org |
| **Code Quality Rules** | https://github.com/uSipipo-Team/usipipo-code-quality |

---

## 📝 Glosario

Consulta términos técnicos y del dominio en el [Glosario](glossary.md).

---

## 📄 Licencia

MIT License © 2026 uSipipo Team

---

## 📞 Contacto

- **Email:** dev@usipipo.com
- **Telegram:** @usipipobot
- **Soporte:** tickets@usipipo.com

---

**Última actualización:** 2026-03-27
