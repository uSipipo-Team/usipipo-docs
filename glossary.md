# 📖 Glosario de Términos uSipipo

> Diccionario de términos técnicos y del dominio del ecosistema uSipipo

---

## A

### **Access Token**
Token JWT de corta duración (30 minutos) que autentica las solicitudes del usuario a la API. Se refresca automáticamente usando el Refresh Token.

### **Admin Panel**
Interfaz administrativa para gestionar usuarios, VPN keys, pagos y tickets de soporte.

### **Alembic**
Herramienta de migración de base de datos usada en el backend para gestionar cambios en el schema de PostgreSQL.

### **API Gateway**
Punto de entrada único para todas las solicitudes API. En uSipipo, el backend FastAPI actúa como gateway.

---

## B

### **Backend**
Servicio central de la API construido con FastAPI que maneja autenticación, VPN keys, pagos, suscripciones y facturación.

### **Balance GB**
Cantidad de datos (en GB) disponible para que un usuario genere VPN keys. Se incrementa con compras y referidos.

### **Billing Cycle**
Período de 30 días para facturación por consumo. Se resetea automáticamente al finalizar el período.

### **Bot Father**
Bot de Telegram oficial para crear y gestionar bots. Usado para obtener el `TELEGRAM_BOT_TOKEN`.

---

## C

### **Clean Architecture**
Patrón de arquitectura usado en todos los repositorios uSipipo, separando dominio, aplicación e infraestructura.

### **Commons**
Librería compartida (`usipipo-commons`) publicada en PyPI con entidades de dominio, enums y utilidades comunes.

### **Consumption Billing**
Sistema de facturación basado en uso real de datos ($0.50/GB), alternativo a las suscripciones fijas.

### **Crypto Order**
Orden de pago en criptomonedas (USDT/USDC) generada a través de TronDealer.

### **Cyberpunk Neon Night**
Tema visual del ecosistema uSipipo: colores neón (cyan, magenta, verde), fondos oscuros y estética futurista.

---

## D

### **Data Limit**
Límite de datos asignado a una VPN key (ej: 5GB, 10GB). Cuando se excede, la key se desactiva hasta el próximo ciclo.

### **Data Package**
Paquete de datos prepagos comprables con Telegram Stars (BASIC: 5GB, ESTANDAR: 15GB, AVANZADO: 30GB, PREMIUM: 50GB, UNLIMITED: ilimitado).

### **Deep Link**
Enlace que abre directamente la app de Telegram. Usado en el flujo de autenticación híbrida.

---

## E

### **Entity**
Objeto de dominio definido en `usipipo-commons` que representa conceptos del negocio (User, VpnKey, Payment, etc.).

### **Enum**
Tipo de dato que define un conjunto de valores constantes (ej: `KeyType`, `PaymentStatus`, `PlanType`).

### **Expires At**
Fecha de expiración de una VPN key, suscripción o invoice. Después de esta fecha, el recurso se vuelve inválido.

---

## F

### **Facturación por Consumo**
Modelo de pricing donde el usuario paga solo por los GB consumidos en el ciclo, en lugar de una suscripción fija.

### **FastAPI**
Framework web asíncrono de Python usado para construir la API backend.

### **FCM Token**
Token de Firebase Cloud Messaging usado para enviar notificaciones push a la app Android.

### **Foreground Service**
Servicio de Android que permite que la VPN se mantenga activa en segundo plano.

---

## G

### **GB (Gigabyte)**
Unidad de medida de datos. 1 GB = 1024 MB = 1,073,741,824 bytes.

### **Go Router**
Paquete de Flutter para navegación y enrutamiento en la app móvil.

---

## H

### **Hexagonal Architecture**
Ver **Clean Architecture**. Patrón que aísla la lógica de negocio de dependencias externas.

---

## I

### **Invoice**
Factura generada para pagos de consumo. Puede ser pagada con Stars o Crypto.

### **mTLS (mutual TLS)**
Protocolo de seguridad para comunicación service-to-service con certificados mutuos.

---

## J

### **JWT (JSON Web Token)**
Estándar para tokens de autenticación. uSipipo usa JWT con algoritmo HS256 para access y refresh tokens.

---

## K

### **Key Data**
Datos de conexión de una VPN key: URL `ss://` para Outline o configuración completa para WireGuard.

### **Key Status**
Estado de una VPN key: `ACTIVE` (activa), `EXPIRED` (expirada), `REVOKED` (revocada), `PENDING` (pendiente).

### **KeyType**
Tipo de protocolo VPN: `OUTLINE` (Shadowsocks) o `WIREGUARD` (WireGuard).

---

## L

### **Landing Page**
Sitio web de marketing en Flask que presenta los servicios de uSipipo VPN.

### **Last Seen**
Timestamp de la última actividad de una VPN key. Usado para detectar uso y calcular billing.

---

## M

### **Material Design 3**
Sistema de diseño de Google usado en la app Android con personalización cyberpunk.

### **MiniApp**
Aplicación web embebida en Telegram que permite gestionar VPN keys sin salir del chat.

### **Mypy**
Herramienta de type checking estático para Python, usada en todos los repositorios.

---

## O

### **Outline**
Protocolo VPN basado en Shadowsocks, desarrollado por Jigsaw (Google). Uno de los dos protocolos soportados por uSipipo.

### **Outline API**
API REST del servidor Outline para gestionar keys y métricas.

---

## P

### **Payment Method**
Método de pago: `CRYPTO_USDT`, `CRYPTO_USDC`, `TELEGRAM_STARS`.

### **Payment Status**
Estado de un pago: `PENDING`, `COMPLETED`, `FAILED`, `EXPIRED`.

### **PlanType**
Tipo de suscripción: `FREE`, `ONE_MONTH`, `THREE_MONTHS`, `SIX_MONTHS`, `CONSUMPTION`.

### **PostgreSQL**
Base de datos relacional usada para almacenar usuarios, keys, pagos, etc.

### **Pre-mature Optimization**
Anti-patrón de optimizar código antes de que sea necesario. Evitado siguiendo el principio "Make It Work First".

---

## R

### **Redis**
Base de datos en memoria usada para caché, rate limiting y blacklist de tokens JWT.

### **Referral Code**
Código único de referido que los usuarios comparten para ganar créditos (5 GB por referido que compra).

### **Referral Credits**
Créditos ganados por referir usuarios. Canjeables por GB de VPN.

### **Refresh Token**
Token JWT de larga duración (30 días) usado para obtener nuevos Access Tokens sin re-autenticar.

### **Riverpod**
Librería de state management para Flutter usada en la app Android.

---

## S

### **Shadowsocks**
Protocolo de proxy SOCKS5 cifrado. Outline está basado en Shadowsocks.

### **SLO (Service Level Objective)**
Objetivo de nivel de servicio para métricas de confiabilidad (ej: 99.9% uptime).

### **SQLAlchemy**
ORM asíncrono usado en el backend para interactuar con PostgreSQL.

### **Stars**
Moneda virtual de Telegram usada para pagos dentro del ecosistema Telegram. 120 Stars ≈ $1 USD.

### **Subscription**
Suscripción recurrente a un plan de VPN (1, 3 o 6 meses).

---

## T

### **Telegram WebApp**
API de Telegram para construir Mini Apps que se ejecutan dentro del cliente de Telegram.

### **Ticket**
Ticket de soporte creado por usuarios para reportar problemas. Categorías: `VPN_FAIL`, `PAYMENT`, `ACCOUNT`, `OTHER`.

### **TronDealer**
Gateway de pagos crypto usado para procesar USDT/USDC en múltiples redes (BSC, Ethereum, Polygon, TRON).

---

## U

### **User Persona**
Representación de usuarios objetivo:
- **Carlos (28 años)** - Usuario técnico que busca privacidad
- **María (35 años)** - Nómada digital que necesita acceso seguro
- **Jorge (22 años)** - Estudiante que busca VPN económico

### **UUID**
Identificador único universal usado como primary key en todas las tablas de la base de datos.

### **uv**
Package manager moderno para Python (de Astral), usado en todos los proyectos uSipipo.

---

## V

### **VPN (Virtual Private Network)**
Red privada que cifra el tráfico de internet. uSipipo provee keys VPN vía WireGuard y Outline.

---

## W

### **WebApp Init Data**
Datos de inicialización enviados por Telegram WebApp, usados para validar autenticidad del usuario.

### **Webhook**
Endpoint que recibe notificaciones de eventos externos (TronDealer, Telegram, FCM).

### **WireGuard**
Protocolo VPN moderno de alto rendimiento. Uno de los dos protocolos soportados por uSipipo.

### **wg-quick**
Comando de WireGuard para establecer túneles VPN. Usado en la app Android.

---

## Símbolos y Formatos

### **Formato de Fechas**
- ISO 8601: `2026-03-27T10:30:00Z`
- Legible: `2026-03-27 10:30:00 UTC`

### **Formato de Datos**
- Bytes: `5,368,709,120 bytes`
- KB: `5,242,880 KB`
- MB: `5,120 MB`
- GB: `5 GB`

### **Formato de Moneda**
- USD: `$5.00 USD`
- Stars: `600 Stars`
- Crypto: `5.00 USDT`

---

## Acrónimos

| Acrónimo | Significado |
|----------|-------------|
| **API** | Application Programming Interface |
| **CLI** | Command Line Interface |
| **CORS** | Cross-Origin Resource Sharing |
| **CI/CD** | Continuous Integration / Continuous Deployment |
| **DDoS** | Distributed Denial of Service |
| **DNS** | Domain Name System |
| **FCM** | Firebase Cloud Messaging |
| **GB** | Gigabyte |
| **HTTP** | Hypertext Transfer Protocol |
| **IP** | Internet Protocol |
| **JWT** | JSON Web Token |
| **KB** | Kilobyte |
| **LATAM** | Latinoamérica |
| **MB** | Megabyte |
| **mTLS** | mutual Transport Layer Security |
| **ORM** | Object-Relational Mapping |
| **PDF** | Portable Document Format |
| **PRD** | Product Requirements Document |
| **REST** | Representational State Transfer |
| **SLO** | Service Level Objective |
| **SLI** | Service Level Indicator |
| **SSH** | Secure Shell |
| **SSL/TLS** | Secure Sockets Layer / Transport Layer Security |
| **TCP** | Transmission Control Protocol |
| **UDP** | User Datagram Protocol |
| **UI** | User Interface |
| **URL** | Uniform Resource Locator |
| **USD** | US Dollar |
| **USDT/USDC** | USD Tether / USD Coin (stablecoins) |
| **UUID** | Universally Unique Identifier |
| **UX** | User Experience |
| **VPN** | Virtual Private Network |

---

**Última actualización:** 2026-03-27
