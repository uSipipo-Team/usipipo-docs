# 📋 Changelog Unificado - Ecosistema uSipipo

> Historial de cambios de todos los proyectos del ecosistema uSipipo

---

## 📐 Formato

Este changelog sigue el formato de [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/).

**Tipos de cambios:**
- **Added** - Para nuevas funcionalidades
- **Changed** - Para cambios en funcionalidades existentes
- **Deprecated** - Para funcionalidades que serán eliminadas
- **Removed** - Para funcionalidades eliminadas
- **Fixed** - Para corrección de bugs
- **Security** - Para cambios relacionados con seguridad

---

## [Unreleased]

### usipipo-backend
- **Added**: Soporte para iOS push notifications
- **Added**: Multi-language support (i18n)
- **Changed**: Mejoras en rate limiting para webhooks

### usipipo-telegram-bot
- **Added**: Comando /newkey para generar VPN keys
- **Added**: Comando /delkey para eliminar VPN keys
- **Added**: Integración con Telegram Stars para pagos

### usipipovpnapp
- **Added**: Soporte para WireGuard en Android
- **Added**: Dashboard de estadísticas en tiempo real
- **Added**: Notificaciones de estado de conexión

---

## [2026-03-27] - Documentación Centralizada

### usipipo-docs
- **Added**: Repositorio centralizado de documentación
- **Added**: Brand Identity completa (5 documentos)
- **Added**: Contexto del Proyecto (4 documentos)
- **Added**: PRDs de los 6 repositorios
- **Added**: App Flow diagrams (5 documentos)
- **Added**: Technology Stack documentation (4 documentos)
- **Added**: API Documentation completa (3 documentos)
- **Added**: Getting Started Guide
- **Added**: Glosario de términos

---

## [2026-03-24] - uSipipo Backend v0.10.0

### usipipo-backend
- **Added**: Sistema de facturación por consumo
- **Added**: Invoices de consumo con pago crypto/stars
- **Added**: Webhook de TronDealer para confirmación de pagos
- **Added**: Endpoints de admin para gestión de usuarios
- **Changed**: Mejoras en validación de Telegram WebApp Init Data
- **Changed**: Optimización de queries de VPN keys
- **Fixed**: Bug en cálculo de data rollover
- **Security**: Implementación de JWT blacklist en Redis
- **Security**: Rate limiting mejorado con SlowAPI

### usipipo-commons
- **Added**: Entidades ConsumptionBilling y ConsumptionInvoice
- **Added**: Enums BillingStatus e InvoiceStatus
- **Changed**: Actualización a Pydantic 2.x

### usipipo-telegram-bot
- **Added**: Comando /me para ver perfil completo
- **Added**: Almacenamiento de tokens en Redis
- **Added**: Auto-refresh de tokens JWT
- **Changed**: Migración a python-telegram-bot 22.7

---

## [2026-03-22] - uSipipo Commons v0.12.0

### usipipo-commons
- **Added**: Entidades de Data Package (DataPackage, PackageType)
- **Added**: Constantes de precios de paquetes
- **Added**: Utilidades de validación de wallet addresses
- **Changed**: Mejoras en format_bytes para mostrar TB
- **Fixed**: Bug en conversión de bytes a GB

### usipipo-backend
- **Added**: Endpoints para compra de paquetes de datos
- **Added**: Integración de paquetes con sistema de balance

---

## [2026-03-20] - uSipipo Landing v0.1.0

### usipipo-landing
- **Added**: Landing page con tema Cyberpunk Neon Night
- **Added**: Página de pricing con 4 planes
- **Added**: Página de documentación
- **Added**: Página de status del servicio
- **Added**: Integración con Caddy reverse proxy
- **Added**: Deployment con systemd
- **Changed**: Optimización de assets estáticos
- **Fixed**: Responsive design en móvil

### usipipo-docs
- **Added**: Documentación de arquitectura del ecosistema
- **Added**: User personas (Carlos, María, Jorge)
- **Added**: Business Model completo

---

## [2026-03-19] - uSipipo Telegram Bot v0.3.0

### usipipo-telegram-bot
- **Added**: Autenticación invisible vía Telegram
- **Added**: Comando /start con auto-registro
- **Added**: Comando /unlink para revocar acceso
- **Added**: Comando /help con lista de comandos
- **Added**: Comando /status para health check
- **Added**: Teclados inline para navegación
- **Added**: Integración con backend API
- **Added**: 45 tests (44 passed, 1 skipped)
- **Added**: CI/CD con GitHub Actions
- **Added**: Pre-commit hooks (ruff, mypy, pytest, bandit)

### usipipo-commons
- **Added**: Entidades de Subscription (SubscriptionPlan, SubscriptionTransaction)
- **Added**: Enums PlanType y SubscriptionTransactionStatus

---

## [2026-03-18] - uSipipo Backend v0.9.0

### usipipo-backend
- **Added**: Sistema de suscripciones recurrentes
- **Added**: Planes de 1, 3 y 6 meses
- **Added**: Integración con Telegram Stars
- **Added**: Endpoints para activación de suscripciones
- **Changed**: Refactorización a Clean Architecture
- **Fixed**: Bug en generación de referral codes duplicados

### usipipo-backend
- **Added**: 29 application services
- **Added**: WireGuard provider con wg-quick
- **Added**: Outline provider con API REST
- **Added**: TronDealer gateway para pagos crypto
- **Added**: Sistema de referidos con créditos
- **Added**: Wallets y pool de saldo compartido
- **Added**: Tickets de soporte con mensajes
- **Added**: Admin panel endpoints
- **Added**: Health checks (/health, /health/ready, /health/live)
- **Added**: Métricas Prometheus
- **Changed**: Migración de pip a uv package manager
- **Security**: Implementación de CORS middleware
- **Security**: Security headers (X-Frame-Options, X-Content-Type-Options, etc.)

---

## [2026-03-17] - uSipipo VPN Android App v1.0.0+1

### usipipovpnapp
- **Added**: Project structure con Clean Architecture
- **Added**: Cyberpunk theme system
- **Added**: Domain entities (User, VpnKey, ConnectionStatus, TrafficStats)
- **Added**: API constants y app constants
- **Added**: Environment configuration (.env, .env.example)
- **Added**: Android permissions (VPN, foreground service, notifications)
- **Added**: Main entry point con ProviderScope
- **Added**: Documentación de diseño (1205 líneas)
- **Added**: Plan de implementación (3890 líneas)

---

## [2026-03-15] - uSipipo Backend v0.8.0

### usipipo-backend
- **Added**: Sistema de pagos con crypto (TronDealer)
- **Added**: Webhook handler para TronDealer
- **Added**: Endpoints de payment history
- **Added**: Sistema de reembolsos
- **Changed**: Mejoras en validación de payloads
- **Security**: HMAC SHA-256 para validación de webhooks
- **Security**: Token rotativo para webhooks

### usipipo-commons
- **Added**: Entidades de Payment (Payment, CryptoOrder, CryptoTransaction)
- **Added**: Enums PaymentMethod, PaymentStatus, CryptoOrderStatus
- **Added**: Entidad WebhookToken para validación

---

## [2026-03-10] - uSipipo Backend v0.7.0

### usipipo-backend
- **Added**: Gestión de VPN keys (CRUD)
- **Added**: Generación de keys WireGuard
- **Added**: Generación de keys Outline
- **Added**: Métricas de uso por key
- **Added**: Reset de ciclo de billing
- **Changed**: Optimización de queries con SQLAlchemy async
- **Fixed**: Bug en cálculo de used_bytes

### usipipo-commons
- **Added**: Entidad VpnKey con métodos computados
- **Added**: Enums KeyType y KeyStatus
- **Added**: Constantes FREE_GB, REFERRAL_BONUS_GB, PRICE_PER_GB

---

## [2026-03-05] - uSipipo Backend v0.6.0

### usipipo-backend
- **Added**: Autenticación JWT
- **Added**: Auto-registro vía Telegram WebApp
- **Added**: Refresh token automático
- **Added**: Endpoints de usuario (GET/PUT /users/me)
- **Added**: Sistema de referidos
- **Changed**: Mejoras en logging estructurado
- **Security**: bcrypt para password hashing
- **Security**: JWT con algoritmo HS256

### usipipo-commons
- **Added**: Entidad User completa
- **Added**: Utilidades validate_telegram_id, format_bytes, format_datetime

---

## [2026-03-01] - uSipipo Commons v0.1.0

### usipipo-commons
- **Added**: Release inicial en PyPI
- **Added**: Entidades de dominio base
- **Added**: Enums principales
- **Added**: Constantes del negocio
- **Added**: Utilidades de validación y formateo
- **Added**: Documentación completa
- **Added**: Tests unitarios

---

## [2026-02-25] - uSipipo Code Quality

### usipipo-code-quality
- **Added**: 7 code quality rules inspiradas en Linus Torvalds
- **Added**: 01-kiss-principle.md
- **Added**: 02-delete-code-fearlessly.md
- **Added**: 03-code-not-comments.md
- **Added**: 04-atomic-commits.md
- **Added**: 05-explain-simply.md
- **Added**: 06-make-it-work-first.md
- **Added**: 07-small-commits.md
- **Added**: AGENT_INSTRUCTIONS.md
- **Added**: README.md con guía de integración

---

## Versiones Anteriores

### 2026-02-15 - Inicio del Proyecto
- **Added**: Investigación de mercado
- **Added**: Diseño de arquitectura inicial
- **Added**: Setup de repositorios GitHub
- **Added**: Configuración de organizaciones

---

## 📊 Estadísticas del Proyecto

| Métrica | Valor |
|---------|-------|
| **Repositorios** | 7 |
| **Líneas de código** | ~50,000 |
| **Líneas de documentación** | ~15,000 |
| **Tests** | 200+ |
| **Cobertura** | 80%+ |
| **Contribuidores** | 4 |
| **Commits** | 500+ |

---

## 🚀 Roadmap

### Q2 2026
- [ ] iOS app (Swift/SwiftUI)
- [ ] Multi-region deployment
- [ ] Advanced analytics dashboard
- [ ] SOC 2 Type I compliance

### Q3 2026
- [ ] Machine learning para fraud detection
- [ ] WireGuard iOS SDK integration
- [ ] Auto-scaling infrastructure
- [ ] 20+ servidores VPN en LATAM

### Q4 2026
- [ ] SOC 2 Type II compliance
- [ ] Enterprise features
- [ ] API pública para partners
- [ ] 100,000 usuarios

---

## 📚 Recursos

- [GitHub Organization](https://github.com/uSipipo-Team)
- [PyPI Package](https://pypi.org/project/usipipo-commons/)
- [Telegram Bot](https://t.me/usipipobot)
- [Landing Page](https://usipipo.duckdns.org)
- [Code Quality Rules](https://github.com/uSipipo-Team/usipipo-code-quality)

---

**Última actualización:** 2026-03-27
