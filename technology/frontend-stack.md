# 🖥️ Frontend Stack - Tecnologías del Frontend

> Documento detallado de las tecnologías frontend del ecosistema uSipipo

---

## 📊 Visión General

El ecosistema uSipipo utiliza múltiples interfaces frontend para diferentes plataformas y casos de uso.

| Plataforma | Tecnología | Estado | Propósito |
|------------|------------|--------|-----------|
| **Landing Page** | Flask + Jinja2 | ✅ Producción | Marketing y conversión |
| **Telegram Bot** | python-telegram-bot | ✅ Producción | Interface principal |
| **MiniApp Web** | Flask + Telegram WebApp | 🟡 Desarrollo | Web app embebida |
| **Android App** | Flutter 3.16 | 🟡 Desarrollo | Cliente VPN nativo |

---

## 🏠 Landing Page (Flask)

### **Arquitectura**

```
src/
├── core/
│   └── config/
│       └── settings.py      # Configuración
│
├── features/
│   ├── home/
│   │   ├── routes.py        # Endpoints home
│   │   └── templates/
│   │       └── index.html   # Home page
│   │
│   ├── pricing/
│   │   ├── routes.py        # Endpoints pricing
│   │   └── templates/
│   │       └── index.html   # Pricing page
│   │
│   ├── docs/
│   │   ├── routes.py        # Endpoints docs
│   │   └── templates/
│   │       └── index.html   # Documentation
│   │
│   └── status/
│       ├── routes.py        # Endpoints status
│       └── templates/
│           └── index.html   # Status page
│
└── infrastructure/
    └── web/
        ├── app.py           # App factory
        ├── templates/
        │   └── base.html    # Base template
        └── static/
            ├── css/
            │   └── style.css  # Cyberpunk styles
            └── js/
                └── main.js    # Scripts
```

---

### **App Factory**

```python
# src/infrastructure/web/app.py
from flask import Flask
from flask_caching import Cache
from flask_talisman import Talisman
from src.core.config import settings

cache = Cache()
talisman = Talisman()

def create_app(config_object: str = "src.core.config.settings") -> Flask:
    app = Flask(__name__)
    
    # Configuration
    app.config.from_object(config_object)
    
    # Initialize extensions
    cache.init_app(app, config={
        'CACHE_TYPE': 'redis',
        'CACHE_REDIS_URL': settings.REDIS_URL,
    })
    
    # Security headers (production only)
    if settings.APP_ENV == "production":
        talisman.init_app(
            app,
            content_security_policy={
                'default-src': "'self'",
                'script-src': ["'self'", "'unsafe-inline'"],
                'style-src': ["'self'", "'unsafe-inline'", "fonts.googleapis.com"],
                'font-src': ["'self'", "fonts.gstatic.com"],
            },
            force_https=False,  # Caddy handles HTTPS
        )
    
    # Register blueprints
    from src.features.home.routes import home_bp
    from src.features.pricing.routes import pricing_bp
    from src.features.docs.routes import docs_bp
    from src.features.status.routes import status_bp
    
    app.register_blueprint(home_bp)
    app.register_blueprint(pricing_bp, url_prefix="/pricing")
    app.register_blueprint(docs_bp, url_prefix="/docs")
    app.register_blueprint(status_bp, url_prefix="/status")
    
    return app
```

---

### **Base Template**

```html+jinja
<!-- src/infrastructure/web/templates/base.html -->
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="{% block meta_description %}uSipipo VPN - Privacidad instantánea vía Telegram{% endblock %}">
    
    <title>{% block title %}uSipipo VPN{% endblock %}</title>
    
    <!-- Fonts -->
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;500;600;700&family=Rajdhani:wght@300;400;500;600;700&display=swap" rel="stylesheet">
    
    <!-- CSS -->
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
    
    {% block extra_css %}{% endblock %}
</head>
<body class="cyberpunk-theme">
    <!-- Navigation -->
    <nav class="navbar">
        <div class="container">
            <a href="/" class="logo">
                <span class="logo-icon">🛡️⚡</span>
                <span class="logo-text">uSipipo</span>
            </a>
            
            <ul class="nav-links">
                <li><a href="/">Inicio</a></li>
                <li><a href="/pricing">Precios</a></li>
                <li><a href="/docs">Documentación</a></li>
                <li><a href="/status">Estado</a></li>
            </ul>
            
            <a href="https://t.me/usipipobot" class="btn btn-primary" target="_blank">
                Abrir Bot
            </a>
        </div>
    </nav>
    
    <!-- Main Content -->
    <main class="main-content">
        {% block content %}{% endblock %}
    </main>
    
    <!-- Footer -->
    <footer class="footer">
        <div class="container">
            <div class="footer-content">
                <div class="footer-section">
                    <h4>uSipipo VPN</h4>
                    <p>Privacidad instantánea vía Telegram</p>
                </div>
                
                <div class="footer-section">
                    <h4>Enlaces</h4>
                    <ul>
                        <li><a href="/pricing">Precios</a></li>
                        <li><a href="/docs">Documentación</a></li>
                        <li><a href="/status">Estado</a></li>
                    </ul>
                </div>
                
                <div class="footer-section">
                    <h4>Legal</h4>
                    <ul>
                        <li><a href="/privacy">Privacidad</a></li>
                        <li><a href="/terms">Términos</a></li>
                    </ul>
                </div>
                
                <div class="footer-section">
                    <h4>Contacto</h4>
                    <ul>
                        <li><a href="mailto:dev@usipipo.com">dev@usipipo.com</a></li>
                        <li><a href="https://t.me/usipipobot">@usipipobot</a></li>
                    </ul>
                </div>
            </div>
            
            <div class="footer-bottom">
                <p>&copy; 2026 uSipipo. Todos los derechos reservados.</p>
            </div>
        </div>
    </footer>
    
    <!-- Scripts -->
    <script src="{{ url_for('static', filename='js/main.js') }}"></script>
    {% block extra_js %}{% endblock %}
</body>
</html>
```

---

### **Cyberpunk CSS Theme**

```css
/* src/infrastructure/web/static/css/style.css */

:root {
    /* Colores principales */
    --color-primary: #00F0FF;
    --color-secondary: #FF00AA;
    --color-accent: #00FF41;
    
    /* Fondos */
    --bg-void: #0A0A0F;
    --bg-card: #12121A;
    --bg-elevated: #1A1A24;
    
    /* Texto */
    --text-primary: #E0E0E0;
    --text-secondary: #888888;
    --text-muted: #555555;
    
    /* Estados */
    --color-success: #00FF41;
    --color-warning: #FFAA00;
    --color-error: #FF0044;
    --color-info: #00F0FF;
    
    /* Tipografía */
    --font-mono: 'Orbitron', monospace;
    --font-sans: 'Rajdhani', sans-serif;
    
    /* Efectos */
    --glow-cyan: 0 0 10px rgba(0, 240, 255, 0.8),
                 0 0 20px rgba(0, 240, 255, 0.6);
    --glow-magenta: 0 0 10px rgba(255, 0, 170, 0.8),
                    0 0 20px rgba(255, 0, 170, 0.6);
}

/* Reset y base */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: var(--font-sans);
    background-color: var(--bg-void);
    color: var(--text-primary);
    line-height: 1.6;
}

/* Navbar */
.navbar {
    background-color: var(--bg-card);
    border-bottom: 1px solid rgba(0, 240, 255, 0.2);
    padding: 1rem 0;
    position: sticky;
    top: 0;
    z-index: 1000;
}

.navbar .container {
    display: flex;
    justify-content: space-between;
    align-items: center;
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 2rem;
}

.logo {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    text-decoration: none;
    color: var(--text-primary);
    font-family: var(--font-mono);
    font-size: 1.5rem;
}

.logo-icon {
    filter: drop-shadow(var(--glow-cyan));
}

/* Botones */
.btn {
    display: inline-block;
    padding: 0.75rem 1.5rem;
    border-radius: 4px;
    font-family: var(--font-mono);
    font-weight: 600;
    text-decoration: none;
    cursor: pointer;
    transition: all 0.3s ease;
}

.btn-primary {
    background: linear-gradient(135deg, var(--color-primary), var(--color-secondary));
    color: var(--bg-void);
    border: none;
}

.btn-primary:hover {
    transform: translateY(-2px);
    box-shadow: var(--glow-cyan);
}

.btn-secondary {
    background: transparent;
    color: var(--color-primary);
    border: 1px solid var(--color-primary);
}

.btn-secondary:hover {
    background: rgba(0, 240, 255, 0.1);
}

/* Hero Section */
.hero {
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
    background: 
        radial-gradient(ellipse at center, rgba(0, 240, 255, 0.1) 0%, transparent 70%),
        var(--bg-void);
    position: relative;
    overflow: hidden;
}

.hero::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: 
        linear-gradient(rgba(0, 240, 255, 0.03) 1px, transparent 1px),
        linear-gradient(90deg, rgba(0, 240, 255, 0.03) 1px, transparent 1px);
    background-size: 50px 50px;
    pointer-events: none;
}

.hero-content {
    position: relative;
    z-index: 1;
    max-width: 800px;
    padding: 2rem;
}

.hero h1 {
    font-family: var(--font-mono);
    font-size: 3.5rem;
    font-weight: 700;
    margin-bottom: 1.5rem;
    background: linear-gradient(135deg, var(--color-primary), var(--color-secondary));
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
}

.hero p {
    font-size: 1.25rem;
    color: var(--text-secondary);
    margin-bottom: 2rem;
}

/* Features Grid */
.features {
    padding: 5rem 2rem;
    max-width: 1200px;
    margin: 0 auto;
}

.features-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
    gap: 2rem;
    margin-top: 3rem;
}

.feature-card {
    background: var(--bg-card);
    border: 1px solid rgba(0, 240, 255, 0.1);
    border-radius: 8px;
    padding: 2rem;
    transition: all 0.3s ease;
}

.feature-card:hover {
    transform: translateY(-5px);
    border-color: var(--color-primary);
    box-shadow: 0 10px 30px rgba(0, 240, 255, 0.2);
}

.feature-icon {
    font-size: 3rem;
    margin-bottom: 1rem;
}

.feature-card h3 {
    font-family: var(--font-mono);
    font-size: 1.25rem;
    margin-bottom: 0.75rem;
    color: var(--color-primary);
}

.feature-card p {
    color: var(--text-secondary);
    line-height: 1.7;
}

/* Pricing Cards */
.pricing {
    padding: 5rem 2rem;
    max-width: 1200px;
    margin: 0 auto;
}

.pricing-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
    gap: 2rem;
    margin-top: 3rem;
}

.pricing-card {
    background: var(--bg-card);
    border: 1px solid rgba(0, 240, 255, 0.1);
    border-radius: 8px;
    padding: 2rem;
    text-align: center;
    position: relative;
    overflow: hidden;
}

.pricing-card.featured {
    border-color: var(--color-primary);
    box-shadow: 0 0 30px rgba(0, 240, 255, 0.2);
}

.pricing-card.featured::before {
    content: 'POPULAR';
    position: absolute;
    top: 1rem;
    right: -2rem;
    background: var(--color-primary);
    color: var(--bg-void);
    padding: 0.25rem 2rem;
    font-size: 0.75rem;
    font-weight: 700;
    transform: rotate(45deg);
}

.pricing-card h3 {
    font-family: var(--font-mono);
    font-size: 1.5rem;
    margin-bottom: 1rem;
}

.pricing-price {
    font-size: 3rem;
    font-weight: 700;
    color: var(--color-primary);
    margin-bottom: 0.5rem;
}

.pricing-price span {
    font-size: 1rem;
    color: var(--text-secondary);
}

.pricing-features {
    list-style: none;
    margin: 2rem 0;
    text-align: left;
}

.pricing-features li {
    padding: 0.5rem 0;
    color: var(--text-secondary);
    display: flex;
    align-items: center;
    gap: 0.5rem;
}

.pricing-features li::before {
    content: '✓';
    color: var(--color-success);
    font-weight: 700;
}

/* Footer */
.footer {
    background: var(--bg-elevated);
    border-top: 1px solid rgba(0, 240, 255, 0.1);
    padding: 3rem 2rem 1rem;
    margin-top: 5rem;
}

.footer-content {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 2rem;
    max-width: 1200px;
    margin: 0 auto 2rem;
}

.footer-section h4 {
    font-family: var(--font-mono);
    color: var(--color-primary);
    margin-bottom: 1rem;
}

.footer-section ul {
    list-style: none;
}

.footer-section li {
    margin-bottom: 0.5rem;
}

.footer-section a {
    color: var(--text-secondary);
    text-decoration: none;
    transition: color 0.3s ease;
}

.footer-section a:hover {
    color: var(--color-primary);
}

.footer-bottom {
    text-align: center;
    padding-top: 2rem;
    border-top: 1px solid rgba(255, 255, 255, 0.1);
    color: var(--text-muted);
}

/* Responsive */
@media (max-width: 768px) {
    .hero h1 {
        font-size: 2rem;
    }
    
    .navbar .container {
        flex-wrap: wrap;
        gap: 1rem;
    }
    
    .nav-links {
        display: none;
    }
}
```

---

## 🤖 Telegram Bot (python-telegram-bot)

### **Arquitectura**

```
src/
├── bot/
│   ├── handlers/
│   │   ├── auth.py          # /start, /me, /unlink
│   │   ├── basic.py         # /help, /status
│   │   └── payments.py      # /pago (futuro)
│   │
│   └── keyboards/
│       ├── main.py          # Main keyboard
│       ├── auth.py          # Auth keyboards
│       └── payments.py      # Payment keyboards
│
├── application/
│   ├── use_cases/
│   │   ├── authenticate.py
│   │   ├── get_user_profile.py
│   │   └── refresh_tokens.py
│   │
│   └── ports/
│       └── api_client.py    # HTTP client interface
│
└── infrastructure/
    ├── api_client.py        # HTTP client implementation
    ├── redis.py             # Redis connection
    ├── token_storage.py     # JWT storage
    ├── config.py            # Settings
    ├── logger.py            # Logging
    └── error_handler.py     # Error handling
```

---

### **Handler de Autenticación**

```python
# src/bot/handlers/auth.py
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ContextTypes, CommandHandler, CallbackQueryHandler
from src.infrastructure.api_client import APIClient
from src.infrastructure.token_storage import TokenStorage
from src.core.config import settings

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Manejador del comando /start"""
    # Obtener Telegram Init Data
    init_data = update.message.web_app_data.data if update.message.web_app_data else None
    
    if not init_data:
        # Fallback sin WebApp
        keyboard = [
            [InlineKeyboardButton("🔐 Autenticar con Telegram", callback_data="auth_telegram")]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await update.message.reply_text(
            "👋 ¡Bienvenido a uSipipo VPN!\n\n"
            "Para comenzar, necesitas autenticarte.\n"
            "Usa el botón de abajo para autenticarte con Telegram.",
            reply_markup=reply_markup,
        )
        return
    
    # Autenticar con backend
    async with APIClient() as api:
        response = await api.post(
            "/auth/telegram/auto-register",
            headers={"Telegram-Init-Data": init_data},
        )
    
    if response["status"] == "success":
        user = response["data"]["user"]
        tokens = response["data"]["tokens"]
        
        # Guardar tokens en Redis
        await TokenStorage.store(
            telegram_id=user["telegram_id"],
            access_token=tokens["access_token"],
            refresh_token=tokens["refresh_token"],
        )
        
        # Mensaje de bienvenida
        keyboard = [
            [InlineKeyboardButton("🔑 Mis Keys", callback_data="my_keys")],
            [InlineKeyboardButton("📊 Consumo", callback_data="consumption")],
            [InlineKeyboardButton("💳 Pagos", callback_data="payments")],
            [InlineKeyboardButton("🎫 Soporte", callback_data="support")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await update.message.reply_text(
            f"✅ ¡Autenticación exitosa!\n\n"
            f"👤 Bienvenido, {user['username'] or 'Usuario'}\n"
            f"📊 Plan: {user.get('plan_type', 'Free')}\n"
            f"💾 Balance: {user['balance_gb']} GB\n\n"
            f"¿Qué necesitas?",
            reply_markup=reply_markup,
        )
    else:
        await update.message.reply_text(
            "❌ Error de autenticación. Por favor intenta de nuevo."
        )

async def me(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Manejador del comando /me"""
    telegram_id = update.effective_user.id
    
    # Obtener tokens
    tokens = await TokenStorage.get(telegram_id)
    
    if not tokens:
        await update.message.reply_text(
            "❌ No estás autenticado. Usa /start para comenzar."
        )
        return
    
    # Obtener perfil del backend
    async with APIClient() as api:
        response = await api.get(
            "/users/me",
            headers={"Authorization": f"Bearer {tokens['access_token']}"},
        )
    
    if response["status"] == "success":
        user = response["data"]
        
        await update.message.reply_text(
            f"👤 {user.get('username') or 'Usuario'}\n"
            f"📊 Plan: {user.get('plan_type', 'Free')}\n"
            f"💾 Balance: {user['balance_gb']} GB\n"
            f"🔑 Keys activas: {user.get('active_keys', 0)}\n"
            f"📅 Registro: {user['created_at'][:10]}"
        )
    else:
        # Token expirado, intentar refresh
        new_tokens = await _refresh_tokens(telegram_id, tokens["refresh_token"])
        
        if new_tokens:
            await me(update, context)  # Reintentar
        else:
            await update.message.reply_text(
                "❌ Sesión expirada. Usa /start para autenticarte de nuevo."
            )

async def _refresh_tokens(telegram_id: int, refresh_token: str) -> bool:
    """Refresh automático de tokens"""
    async with APIClient() as api:
        response = await api.post(
            "/auth/refresh",
            headers={"Authorization": f"Bearer {refresh_token}"},
        )
    
    if response["status"] == "success":
        await TokenStorage.store(
            telegram_id=telegram_id,
            access_token=response["data"]["access_token"],
            refresh_token=refresh_token,  # Mismo refresh token
        )
        return True
    
    return False
```

---

### **API Client**

```python
# src/infrastructure/api_client.py
import httpx
from typing import Optional, Dict, Any
from src.core.config import settings

class APIClient:
    def __init__(self):
        self.base_url = settings.BACKEND_URL
        self.api_prefix = "/api/v1"
        self.client: Optional[httpx.AsyncClient] = None
    
    async def __aenter__(self):
        self.client = httpx.AsyncClient(
            base_url=self.base_url,
            timeout=httpx.Timeout(30.0),
            headers={
                "Content-Type": "application/json",
                "User-Agent": "uSipipo-Bot/0.3.0",
            },
        )
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.client:
            await self.client.aclose()
    
    async def get(self, path: str, headers: Optional[Dict[str, str]] = None) -> Dict[str, Any]:
        response = await self.client.get(f"{self.api_prefix}{path}", headers=headers)
        return response.json()
    
    async def post(
        self,
        path: str,
        json: Optional[Dict[str, Any]] = None,
        headers: Optional[Dict[str, str]] = None,
    ) -> Dict[str, Any]:
        response = await self.client.post(
            f"{self.api_prefix}{path}",
            json=json,
            headers=headers,
        )
        return response.json()
    
    async def put(
        self,
        path: str,
        json: Optional[Dict[str, Any]] = None,
        headers: Optional[Dict[str, str]] = None,
    ) -> Dict[str, Any]:
        response = await self.client.put(
            f"{self.api_prefix}{path}",
            json=json,
            headers=headers,
        )
        return response.json()
    
    async def delete(self, path: str, headers: Optional[Dict[str, str]] = None) -> Dict[str, Any]:
        response = await self.client.delete(f"{self.api_prefix}{path}", headers=headers)
        return response.json()
```

---

### **Token Storage (Redis)**

```python
# src/infrastructure/token_storage.py
import redis.asyncio as redis
import json
from typing import Optional, Dict
from datetime import datetime, timedelta
from src.core.config import settings

class TokenStorage:
    TOKEN_PREFIX = "usipipo:bot:tokens:"
    ACCESS_TOKEN_EXPIRY = timedelta(minutes=30)
    REFRESH_TOKEN_EXPIRY = timedelta(days=30)
    
    @classmethod
    async def store(
        cls,
        telegram_id: int,
        access_token: str,
        refresh_token: str,
    ):
        """Guardar tokens en Redis"""
        client = await cls._get_client()
        key = f"{cls.TOKEN_PREFIX}{telegram_id}"
        
        data = {
            "access_token": access_token,
            "refresh_token": refresh_token,
            "stored_at": datetime.now().isoformat(),
        }
        
        await client.hset(key, mapping={
            "access_token": access_token,
            "refresh_token": refresh_token,
        })
        await client.expire(key, int(cls.REFRESH_TOKEN_EXPIRY.total_seconds()))
    
    @classmethod
    async def get(cls, telegram_id: int) -> Optional[Dict[str, str]]:
        """Obtener tokens de Redis"""
        client = await cls._get_client()
        key = f"{cls.TOKEN_PREFIX}{telegram_id}"
        
        data = await client.hgetall(key)
        return data if data else None
    
    @classmethod
    async def delete(cls, telegram_id: int):
        """Eliminar tokens de Redis"""
        client = await cls._get_client()
        key = f"{cls.TOKEN_PREFIX}{telegram_id}"
        await client.delete(key)
    
    @classmethod
    async def is_authenticated(cls, telegram_id: int) -> bool:
        """Verificar si el usuario está autenticado"""
        tokens = await cls.get(telegram_id)
        return tokens is not None
    
    @classmethod
    async def needs_refresh(cls, telegram_id: int) -> bool:
        """Verificar si los tokens necesitan refresh (5 min antes de expirar)"""
        tokens = await cls.get(telegram_id)
        
        if not tokens:
            return True
        
        stored_at = datetime.fromisoformat(tokens.get("stored_at", ""))
        elapsed = datetime.now() - stored_at
        
        # Refresh 5 minutos antes de expirar
        refresh_threshold = cls.ACCESS_TOKEN_EXPIRY - timedelta(minutes=5)
        return elapsed > refresh_threshold
    
    @classmethod
    async def _get_client(cls) -> redis.Redis:
        return redis.from_url(
            settings.REDIS_URL,
            encoding="utf-8",
            decode_responses=True,
        )
```

---

## 📱 Android App (Flutter)

### **Arquitectura**

```
lib/
├── main.dart                  # Entry point
│
├── core/
│   ├── constants/
│   │   ├── api_constants.dart   # Endpoints
│   │   └── app_constants.dart   # Constantes
│   │
│   └── theme/
│       ├── cyberpunk_colors.dart  # Colores
│       └── cyberpunk_theme.dart   # ThemeData
│
├── domain/
│   └── entities/
│       ├── user.dart
│       ├── vpn_key.dart
│       ├── connection_status.dart
│       └── traffic_stats.dart
│
├── data/
│   ├── providers/
│   │   ├── api_provider.dart
│   │   └── vpn_provider.dart
│   │
│   ├── repositories/
│   │   ├── auth_repository.dart
│   │   └── vpn_repository.dart
│   │
│   └── models/
│       ├── user_model.dart
│       └── vpn_key_model.dart
│
└── presentation/
    ├── providers/
    │   ├── auth_provider.dart
    │   └── vpn_provider.dart
    │
    ├── screens/
    │   ├── login/
    │   │   └── login_screen.dart
    │   ├── home/
    │   │   └── home_screen.dart
    │   ├── keys/
    │   │   └── keys_screen.dart
    │   └── settings/
    │       └── settings_screen.dart
    │
    └── widgets/
        ├── cyberpunk_button.dart
        ├── vpn_status_card.dart
        └── traffic_stats.dart
```

---

### **Entry Point**

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'core/theme/cyberpunk_theme.dart';
import 'presentation/screens/login/login_screen.dart';
import 'presentation/screens/home/home_screen.dart';
import 'presentation/screens/keys/keys_screen.dart';
import 'presentation/screens/settings/settings_screen.dart';

final GoRouter router = GoRouter(
  initialLocation: '/login',
  routes: [
    GoRoute(path: '/login', builder: (_, __) => LoginScreen()),
    GoRoute(path: '/home', builder: (_, __) => HomeScreen()),
    GoRoute(path: '/keys', builder: (_, __) => KeysScreen()),
    GoRoute(path: '/settings', builder: (_, __) => SettingsScreen()),
  ],
);

void main() {
  runApp(
    ProviderScope(
      child: uSipipoApp(),
    ),
  );
}

class uSipipoApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      title: 'uSipipo VPN',
      theme: CyberpunkTheme.darkTheme,
      routerConfig: router,
      debugShowCheckedModeBanner: false,
    );
  }
}
```

---

### **Cyberpunk Theme**

```dart
// lib/core/theme/cyberpunk_theme.dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import 'cyberpunk_colors.dart';

class CyberpunkTheme {
  static ThemeData get darkTheme {
    return ThemeData(
      useMaterial3: true,
      brightness: Brightness.dark,
      
      // Colores
      primaryColor: CyberpunkColors.primary,
      scaffoldBackgroundColor: CyberpunkColors.bgVoid,
      cardColor: CyberpunkColors.bgCard,
      
      // Tipografía
      textTheme: GoogleFonts.rajdhaniTextTheme(
        ThemeData.dark().textTheme,
      ).apply(
        bodyColor: CyberpunkColors.textPrimary,
        displayColor: CyberpunkColors.textPrimary,
      ).copyWith(
        displayLarge: GoogleFonts.jetBrainsMono(
          fontSize: 48,
          fontWeight: FontWeight.bold,
          color: CyberpunkColors.textPrimary,
        ),
        displayMedium: GoogleFonts.jetBrainsMono(
          fontSize: 36,
          fontWeight: FontWeight.w600,
          color: CyberpunkColors.textPrimary,
        ),
        headlineLarge: GoogleFonts.jetBrainsMono(
          fontSize: 28,
          fontWeight: FontWeight.w600,
          color: CyberpunkColors.textPrimary,
        ),
        titleLarge: GoogleFonts.jetBrainsMono(
          fontSize: 20,
          fontWeight: FontWeight.w500,
          color: CyberpunkColors.textPrimary,
        ),
        bodyLarge: GoogleFonts.rajdhani(
          fontSize: 16,
          fontWeight: FontWeight.normal,
          color: CyberpunkColors.textPrimary,
        ),
        bodyMedium: GoogleFonts.rajdhani(
          fontSize: 14,
          fontWeight: FontWeight.normal,
          color: CyberpunkColors.textSecondary,
        ),
      ),
      
      // Color scheme
      colorScheme: ColorScheme.dark(
        primary: CyberpunkColors.primary,
        secondary: CyberpunkColors.secondary,
        tertiary: CyberpunkColors.accent,
        surface: CyberpunkColors.bgCard,
        error: CyberpunkColors.error,
      ),
      
      // Botones
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: CyberpunkColors.primary,
          foregroundColor: CyberpunkColors.bgVoid,
          padding: EdgeInsets.symmetric(horizontal: 24, vertical: 12),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(4),
          ),
        ),
      ),
      
      // Cards
      cardTheme: CardTheme(
        color: CyberpunkColors.bgCard,
        elevation: 0,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(8),
          side: BorderSide(
            color: CyberpunkColors.primary.withOpacity(0.2),
          ),
        ),
      ),
    );
  }
}
```

---

### **Cyberpunk Colors**

```dart
// lib/core/theme/cyberpunk_colors.dart
import 'package:flutter/material.dart';

class CyberpunkColors {
  // Colores principales
  static const Color primary = Color(0xFF00F0FF);      // Cyan neón
  static const Color secondary = Color(0xFFFF00AA);    // Magenta neón
  static const Color accent = Color(0xFF00FF41);       // Verde terminal
  
  // Fondos
  static const Color bgVoid = Color(0xFF0A0A0F);       // Void dark
  static const Color bgCard = Color(0xFF12121A);       // Card background
  static const Color bgElevated = Color(0xFF1A1A24);   // Elevated
  
  // Texto
  static const Color textPrimary = Color(0xFFE0E0E0);
  static const Color textSecondary = Color(0xFF888888);
  static const Color textMuted = Color(0xFF555555);
  
  // Estados
  static const Color success = Color(0xFF00FF41);
  static const Color warning = Color(0xFFFFAA00);
  static const Color error = Color(0xFFFF0044);
  static const Color info = Color(0xFF00F0FF);
  
  // Gradientes
  static const LinearGradient cyberpunkGradient = LinearGradient(
    colors: [primary, secondary],
    begin: Alignment.topLeft,
    end: Alignment.bottomRight,
  );
  
  // Efectos de glow
  static List<BoxShadow> cyanGlow = [
    BoxShadow(
      color: primary.withOpacity(0.8),
      blurRadius: 10,
      spreadRadius: 2,
    ),
    BoxShadow(
      color: primary.withOpacity(0.6),
      blurRadius: 20,
      spreadRadius: 4,
    ),
  ];
}
```

---

## 📦 Dependencias

### **Landing Page (Python)**

```toml
# pyproject.toml
dependencies = [
    "flask>=3.1.0",
    "flask-caching>=2.1.0",
    "flask-talisman>=1.1.0",
    "redis>=5.0.0",
    "python-dotenv>=1.0.0",
    "pydantic-settings>=2.1.0",
]
```

---

### **Telegram Bot (Python)**

```toml
dependencies = [
    "python-telegram-bot>=22.7",
    "httpx>=0.26.0",
    "redis>=5.0.0",
    "pydantic-settings>=2.1.0",
]
```

---

### **Android App (Flutter)**

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  
  # State management
  flutter_riverpod: ^2.4.0
  
  # Navigation
  go_router: ^12.0.0
  
  # HTTP
  dio: ^5.3.0
  
  # Storage
  flutter_secure_storage: ^9.0.0
  shared_preferences: ^2.2.0
  
  # Firebase
  firebase_core: ^2.24.0
  firebase_messaging: ^14.7.0
  
  # Notifications
  flutter_local_notifications: ^16.0.0
  
  # UI
  google_fonts: ^6.0.0
  flutter_svg: ^2.0.0
  
  # Utils
  connectivity_plus: ^5.0.0
  intl: ^0.18.0
```

---

## 🚀 Deployment

### **Landing Page (systemd)**

```ini
# /etc/systemd/system/usipipo-landing.service
[Unit]
Description=uSipipo Landing Page
After=network.target

[Service]
Type=notify
User=usipipo
WorkingDirectory=/opt/usipipo/usipipo-landing
EnvironmentFile=/opt/usipipo/.env
ExecStart=/opt/usipipo/.venv/bin/python -m src
Restart=always

[Install]
WantedBy=multi-user.target
```

---

### **Telegram Bot (systemd)**

```ini
# /etc/systemd/system/usipipo-bot.service
[Unit]
Description=uSipipo Telegram Bot
After=network.target redis.service

[Service]
Type=simple
User=usipipo
WorkingDirectory=/opt/usipipo/usipipo-telegram-bot
EnvironmentFile=/opt/usipipo/.env
ExecStart=/opt/usipipo/.venv/bin/python -m src
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

## 📚 Recursos Relacionados

- [Stack Overview](stack-overview.md)
- [Brand Identity](../brand/identity.md)
- [PRD Landing](../prds/prd-usipipo-landing.md)
- [PRD Bot](../prds/prd-usipipo-telegram-bot.md)

---

**Última actualización:** 2026-03-27
