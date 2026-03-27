# 📖 Project Overview - Visión General del Proyecto uSipipo

> Propósito, objetivos y contexto del ecosistema uSipipo VPN

---

## 🎯 ¿Qué es uSipipo?

**uSipipo** es un ecosistema de servicios VPN diseñado específicamente para la comunidad latinoamericana, proporcionando acceso seguro y privado a internet a través de una experiencia única: **todo gestionado desde Telegram**.

### **Propuesta de Valor Única**

> "VPN instantánea vía Telegram. Sin apps complejas. Sin registros. Solo privacidad."

---

## 🚀 Problema que Resolvemos

### **El Problema**

1. **VPNs tradicionales son complejas**
   - Descargar apps propietarias
   - Configurar servidores manualmente
   - Interfaces confusas
   - Setup en 5+ pasos

2. **Barreras de pago en LATAM**
   - Requieren tarjetas de crédito
   - Pocas opciones de pago local
   - Precios en USD inaccesibles
   - Sin opciones crypto

3. **Desconfianza en proveedores VPN**
   - Muchos guardan logs
   - Promesas de "anonimato 100%" falsas
   - Sin transparencia real

4. **Soporte deficiente en español**
   - Documentación solo en inglés
   - Soporte lento o inexistente
   - Timezones incompatibles

---

## 💡 Nuestra Solución

### **uSipipo hace VPN simple:**

1. **✅ Setup en 60 segundos**
   - Abres Telegram
   - Buscas @usipipobot
   - Envías `/start`
   - ¡Listo! VPN key generada

2. **✅ Pagos accesibles**
   - Crypto (USDT, USDC, BTC, ETH)
   - Telegram Stars
   - Programa de referidos
   - Precios LATAM

3. **✅ Privacidad real**
   - No guardamos logs
   - Cifrado de extremo a extremo
   - Transparencia total

4. **✅ Soporte en español**
   - Documentación completa
   - Soporte por tickets
   - Timezone America/Caracas

---

## 📊 Objetivos del Proyecto

### **Objetivos a Corto Plazo (2026 Q2)**

| Objetivo | Métrica | Estado |
|----------|---------|--------|
| Lanzar backend a producción | API estable 99.9% | ✅ Completado |
| Telegram bot funcional | 1000 usuarios activos | 🟡 En progreso |
| Landing page publicada | 5000 visitas/mes | ✅ Completado |
| Android app MVP | 500 descargas | 🟡 Fase 2/8 |

### **Objetivos a Mediano Plazo (2026 Q4)**

| Objetivo | Métrica |
|----------|---------|
| 10,000 usuarios registrados | Monthly Active Users |
| 99.95% uptime | Service Level Objective |
| 50% de usuarios pagantes | Conversion rate |
| NPS > 50 | Customer satisfaction |

### **Objetivos a Largo Plazo (2027)**

| Objetivo | Métrica |
|----------|---------|
| 100,000 usuarios | Total registered users |
| Expansión a 5 países LATAM | Market presence |
| 20+ servidores VPN | Infrastructure |
| Rentabilidad | Positive cash flow |

---

## 🎯 Público Objetivo

### **User Personas Principales**

#### **1. Carlos (28 años) - Tech-Savvy Professional**

**Perfil:**
- Desarrollador de software en Ciudad de México
- Ingresos: $2,000-3,000 USD/mes
- Usa Telegram diariamente
- Consciente de privacidad digital

**Necesidades:**
- VPN para trabajar remoto seguro
- Acceder a contenido geo-restringido
- Privacidad en redes públicas

**Frustraciones:**
- VPNs tradicionales muy complejas
- Pagos con tarjeta desde LATAM
- Soporte en inglés

**Cómo uSipipo ayuda:**
- Setup instantáneo desde Telegram
- Pagos con crypto
- Soporte en español

---

#### **2. María (35 años) - Digital Nomad**

**Perfil:**
- Diseñadora UX, viaja por LATAM
- Ingresos: $3,000-5,000 USD/mes
- Trabaja desde cafés y coworkings
- Valora simplicidad

**Necesidades:**
- Conexión segura en WiFi público
- Acceder a servicios de su país
- Múltiples dispositivos

**Frustraciones:**
- Configurar VPN en cada dispositivo
- VPNs lentas para diseño
- Precios altos

**Cómo uSipipo ayuda:**
- Keys múltiples (hasta 5 en plan premium)
- WireGuard para velocidad
- Precios accesibles

---

#### **3. Jorge (22 años) - Estudiante**

**Perfil:**
- Estudiante de ingeniería en Bogotá
- Ingresos: $200-500 USD/mes
- Budget-conscious
- Early adopter

**Necesidades:**
- VPN económico o gratis
- Para estudiar y entretenimiento
- Fácil de usar

**Frustraciones:**
- VPNs gratis venden datos
- VPNs pagos muy caros
- Configuraciones técnicas

**Cómo uSipipo ayuda:**
- Plan gratis: 5 GB, 2 claves
- Referidos: gana GB invitando
- Setup en 1 minuto

---

## 🏗️ Arquitectura del Ecosistema

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ┌──────────────┐     ┌──────────────┐     ┌────────────┐ │
│  │   Landing    │     │  Telegram    │     │  Android   │ │
│  │    Page      │     │     Bot      │     │    App     │ │
│  │  (Flask)     │     │  (PTB)       │     │  (Flutter) │ │
│  └──────┬───────┘     └──────┬───────┘     └─────┬──────┘ │
│         │                    │                     │       │
│         │                    │                     │       │
│         └────────────────────┼─────────────────────┘       │
│                              │                              │
│                              ▼                              │
│                   ┌──────────────────┐                      │
│                   │   Backend API    │                      │
│                   │    (FastAPI)     │                      │
│                   └────────┬─────────┘                      │
│                            │                                │
│         ┌──────────────────┼──────────────────┐             │
│         │                  │                  │             │
│         ▼                  ▼                  ▼             │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐        │
│  │ PostgreSQL │    │   Redis    │    │ WireGuard  │        │
│  │  (Datos)   │    │  (Cache)   │    │  (VPN)     │        │
│  └────────────┘    └────────────┘    └────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Componentes:**

1. **Frontend/Clientes:**
   - Landing Page (Flask)
   - Telegram Bot (python-telegram-bot)
   - MiniApp Web (Flask)
   - Android App (Flutter)

2. **Backend:**
   - API REST (FastAPI)
   - Base de datos (PostgreSQL)
   - Caché (Redis)
   - VPN Servers (WireGuard, Outline)

3. **Integraciones:**
   - TronDealer (pagos crypto)
   - Telegram Stars
   - Firebase (notificaciones)

---

## 💰 Modelo de Negocio

### **Fuentes de Ingreso**

1. **Suscripciones** (70% proyectado)
   - 1 mes: $7.20 USD (360 Stars)
   - 3 meses: $19.20 USD (960 Stars)
   - 6 meses: $31.20 USD (1560 Stars)

2. **Facturación por Consumo** (20% proyectado)
   - $0.50 USD/GB
   - Facturación mensual

3. **Paquetes de Datos** (10% proyectado)
   - BASIC: 5 GB - $2.50 USD
   - ESTANDAR: 15 GB - $7.50 USD
   - AVANZADO: 30 GB - $15.00 USD
   - PREMIUM: 50 GB - $25.00 USD
   - UNLIMITED: ilimitado - $50.00 USD

### **Estructura de Costos**

| Concepto | Costo Mensual | % |
|----------|---------------|---|
| Infraestructura (servidores) | $500-1,000 | 40% |
| Desarrollo (team) | $3,000-5,000 | 40% |
| Marketing | $500-1,000 | 10% |
| Operaciones | $500 | 10% |
| **Total** | **$4,500-7,500** | **100%** |

### **Proyección de Ingresos**

| Año | Usuarios | Ingreso Mensual | Ingreso Anual |
|-----|----------|-----------------|---------------|
| 2026 | 1,000 | $5,000 | $60,000 |
| 2027 | 10,000 | $50,000 | $600,000 |
| 2028 | 50,000 | $250,000 | $3,000,000 |

---

## 🌟 Diferenciadores Competitivos

### **vs VPNs Tradicionales**

| Característica | uSipipo | Competencia |
|----------------|---------|-------------|
| Setup | 60 segundos | 5-10 minutos |
| Pagos | Crypto + Stars | Solo tarjeta |
| Soporte | Español, Telegram | Inglés, email |
| Precio | Desde $0.50/GB | $5-15/mes fijo |
| Logs | No guarda | Algunos guardan |
| Plataforma | Telegram-native | Apps propietarias |

### **vs VPNs Gratis**

| Característica | uSipipo | VPNs Gratis |
|----------------|---------|-------------|
| Modelo de negocio | Pagas por servicio | Venden tus datos |
| Privacidad | No logs | Logs detallados |
| Velocidad | WireGuard rápido | Lentas intencionalmente |
| Límites | Claros y justos | Anuncios, límites ocultos |
| Seguridad | Cifrado real | Cifrado débil |

---

## 📈 Métricas de Éxito

### **Métricas de Producto**

| Métrica | Objetivo | Cómo medir |
|---------|----------|------------|
| **Usuarios activos** | 1,000 MAU | Telegram bot + app |
| **Retención D30** | > 60% | Usuarios activos a 30 días |
| **Conversión a pago** | > 20% | Free → Paid |
| **NPS** | > 50 | Encuestas post-compra |
| **Churn rate** | < 5% mensual | Cancelaciones / total |

### **Métricas Técnicas**

| Métrica | Objetivo | Cómo medir |
|---------|----------|------------|
| **Uptime** | 99.9% | Monitoring (Prometheus) |
| **Latencia API** | < 200ms p95 | APM (Jaeger) |
| **Error rate** | < 0.1% | Logs + alerting |
| **VPN success rate** | > 99% | Connection logs |

### **Métricas de Negocio**

| Métrica | Objetivo | Cómo medir |
|---------|----------|------------|
| **MRR** | $5,000/mes | Stripe + crypto |
| **CAC** | < $10 | Marketing spend / nuevos |
| **LTV** | > $50 | Ingreso promedio × retención |
| **LTV:CAC** | > 5:1 | LTV / CAC |
| **Burn rate** | < $5,000/mes | Gastos mensuales |

---

## 🗺️ Roadmap del Proyecto

### **2026 Q1 (Ene-Mar) - Cimientos**

- ✅ Backend API v0.1.0
- ✅ Commons library v0.1.0
- ✅ Telegram bot MVP
- ✅ Landing page alpha

### **2026 Q2 (Abr-Jun) - Producción**

- 🟡 Backend API v1.0.0 (producción)
- 🟡 Telegram bot v1.0.0
- 🟡 Android app beta
- 🟡 Sistema de pagos completo

### **2026 Q3 (Jul-Sep) - Crecimiento**

- ⏳ iOS app (planificado)
- ⏳ Múltiples servidores VPN
- ⏳ Programa de afiliados
- ⏳ Soporte 24/7

### **2026 Q4 (Oct-Dic) - Escalamiento**

- ⏳ Expansión regional (5 países)
- ⏳ 10,000 usuarios
- ⏳ Features enterprise
- ⏳ SOC 2 compliance (planificado)

---

## 🔐 Valores y Principios

### **Valores Centrales**

1. **Privacidad Primero**
   - No guardamos logs de actividad
   - Transparencia en manejo de datos
   - Cifrado de extremo a extremo

2. **Accesibilidad**
   - Precios en moneda local y crypto
   - Soporte en español
   - UX simple e intuitiva

3. **Simplicidad**
   - Setup en menos de 2 minutos
   - Sin configuraciones complejas
   - Documentación clara

4. **Innovación**
   - Tecnología de punta
   - Automatización total
   - Mejora continua

5. **Comunidad**
   - Enfoque LATAM
   - Programa de referidos
   - Soporte cercano

---

## 📚 Recursos Relacionados

- [Arquitectura del Ecosistema](ecosystem-architecture.md)
- [User Personas](user-personas.md)
- [Modelo de Negocio](business-model.md)
- [Getting Started](../getting-started.md)

---

**Última actualización:** 2026-03-27
