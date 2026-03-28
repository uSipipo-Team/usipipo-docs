# Semana 5: MiniApp Web + Landing Page

**Fecha:** 15-21 Abril 2026  
**Objetivo:** Refactorizar MiniApp para consumir APIs del backend y crear Landing Page desde cero

**Dependencias:** Semana 4 completada ✅ (Bot refactorizado, backend estable)

---

## 📦 Entregables de la Semana

- [ ] `usipipo-miniapp-web` refactorizado para consumir APIs del backend
- [ ] Auth con Telegram WebApp initData
- [ ] Todas las features de MiniApp funcionando con backend
- [ ] `usipipo-landing` creado desde cero
- [ ] Landing page con: Home, Pricing, Docs, Status
- [ ] Docker compose funcional para ambos servicios

---

## 🎯 Tareas Detalladas

### DÍA 1-2: MiniApp - Estructura + Auth

#### 1.1 Clonar y estructurar MiniApp

```bash
cd /home/mowgli
git clone git@github.com:usipipo/usipipo-miniapp-web.git
cd usipipo-miniapp-web

# Inicializar con uv
uv init --name usipipo-miniapp-web

# Crear estructura basada en features
mkdir -p src/core/domain/entities
mkdir -p src/features/dashboard/{routes,templates,services}
mkdir -p src/features/keys/{routes,templates,services}
mkdir -p src/features/payments/{routes,templates,services}
mkdir -p src/features/profile/{routes,templates,services}
mkdir -p src/features/auth/{routes,middleware,services}
mkdir -p src/infrastructure/api/{endpoints,client}
mkdir -p src/infrastructure/web/{templates,static}
mkdir -p src/shared
mkdir -p tests
```

#### 1.2 Migrar código desde monorepo

```bash
# Copiar templates existentes
cp -r /home/mowgli/usipipobot/miniapp/templates/* src/infrastructure/web/templates/

# Copiar static existente
cp -r /home/mowgli/usipipobot/miniapp/static/* src/infrastructure/web/static/

# Copiar servicios existentes (ajustar imports)
cp /home/mowgli/usipipobot/miniapp/services/*.py src/features/auth/services/
```

#### 1.3 Crear cliente HTTP hacia backend

**Archivo:** `src/infrastructure/api/client.py`
```python
from typing import Optional
from uuid import UUID

import httpx

from ...core.config import settings


class BackendClient:
    """Cliente HTTP para comunicar con el backend desde MiniApp."""
    
    def __init__(self, base_url: str, jwt_token: str):
        self.base_url = base_url
        self.http = httpx.AsyncClient(
            base_url=base_url,
            headers={
                "Authorization": f"Bearer {jwt_token}",
                "Content-Type": "application/json",
            },
            timeout=30.0,
        )
    
    async def close(self):
        """Cierra el cliente HTTP."""
        await self.http.aclose()
```

**Archivo:** `src/infrastructure/api/endpoints/auth.py`
```python
from typing import Optional, Dict, Any


class AuthEndpoint:
    """Endpoints de autenticación del backend."""
    
    def __init__(self, client: "BackendClient"):
        self.client = client
    
    async def authenticate_telegram(self, init_data: str) -> Optional[Dict[str, Any]]:
        """
        Autentica con Telegram initData.
        
        Retorna: {"access_token": "...", "user_id": "..."}
        """
        response = await self.client.http.post(
            "/api/v1/auth/telegram",
            json={"init_data": init_data},
        )
        if response.status_code == 200:
            return response.json()
        return None
```

**Archivo:** `src/infrastructure/api/endpoints/users.py`
```python
from typing import Optional, Dict, Any


class UsersEndpoint:
    """Endpoints de usuarios del backend."""
    
    def __init__(self, client: "BackendClient"):
        self.client = client
    
    async def get_me(self) -> Optional[Dict[str, Any]]:
        """Obtiene perfil del usuario."""
        response = await self.client.http.get("/api/v1/users/me")
        if response.status_code == 200:
            return response.json()
        return None
    
    async def update_me(self, data: Dict[str, Any]) -> Optional[Dict[str, Any]]:
        """Actualiza perfil del usuario."""
        response = await self.client.http.put(
            "/api/v1/users/me",
            json=data,
        )
        if response.status_code == 200:
            return response.json()
        return None
```

**Archivo:** `src/infrastructure/api/endpoints/vpn.py`
```python
from typing import List, Optional, Dict, Any
from uuid import UUID


class VpnEndpoint:
    """Endpoints de VPN del backend."""
    
    def __init__(self, client: "BackendClient"):
        self.client = client
    
    async def list_keys(self) -> List[Dict[str, Any]]:
        """Lista claves VPN del usuario."""
        response = await self.client.http.get("/api/v1/vpn/keys")
        if response.status_code == 200:
            return response.json()
        return []
    
    async def create_key(
        self,
        name: str,
        vpn_type: str,
        data_limit_gb: float = 5.0,
    ) -> Optional[Dict[str, Any]]:
        """Crea nueva clave VPN."""
        response = await self.client.http.post(
            "/api/v1/vpn/keys",
            json={
                "name": name,
                "vpn_type": vpn_type,
                "data_limit_gb": data_limit_gb,
            },
        )
        if response.status_code == 201:
            return response.json()
        return None
    
    async def delete_key(self, key_id: UUID) -> bool:
        """Elimina clave VPN."""
        response = await self.client.http.delete(f"/api/v1/vpn/keys/{key_id}")
        return response.status_code == 204
    
    async def get_key_config(self, key_id: UUID) -> Optional[str]:
        """Obtiene configuración de clave."""
        response = await self.client.http.get(f"/api/v1/vpn/keys/{key_id}/config")
        if response.status_code == 200:
            data = response.json()
            return data.get("config")
        return None
```

**Archivo:** `src/infrastructure/api/endpoints/payments.py`
```python
from typing import List, Optional, Dict, Any
from uuid import UUID


class PaymentsEndpoint:
    """Endpoints de pagos del backend."""
    
    def __init__(self, client: "BackendClient"):
        self.client = client
    
    async def create_crypto_payment(
        self,
        amount_usd: float,
        gb_purchased: float,
        network: str = "BSC",
    ) -> Optional[Dict[str, Any]]:
        """Crea pago con criptomoneda."""
        response = await self.client.http.post(
            "/api/v1/payments/crypto",
            json={
                "amount_usd": amount_usd,
                "gb_purchased": gb_purchased,
                "network": network,
            },
        )
        if response.status_code == 201:
            return response.json()
        return None
    
    async def create_stars_payment(
        self,
        amount_usd: float,
        gb_purchased: float,
    ) -> Optional[Dict[str, Any]]:
        """Crea pago con Telegram Stars."""
        response = await self.client.http.post(
            "/api/v1/payments/stars",
            json={
                "amount_usd": amount_usd,
                "gb_purchased": gb_purchased,
            },
        )
        if response.status_code == 201:
            return response.json()
        return None
    
    async def get_history(self) -> List[Dict[str, Any]]:
        """Obtiene historial de pagos."""
        response = await self.client.http.get("/api/v1/payments/history")
        if response.status_code == 200:
            return response.json()
        return []
```

**Archivo:** `src/infrastructure/api/endpoints/billing.py`
```python
from typing import Optional, Dict, Any
from uuid import UUID


class BillingEndpoint:
    """Endpoints de billing del backend."""
    
    def __init__(self, client: "BackendClient"):
        self.client = client
    
    async def get_usage(self) -> Optional[Dict[str, Any]]:
        """Obtiene consumo de datos."""
        response = await self.client.http.get("/api/v1/billing/usage")
        if response.status_code == 200:
            return response.json()
        return None
```

#### 1.4 Unificar endpoints en cliente

**Archivo:** `src/infrastructure/api/client.py` (actualizar)
```python
from .endpoints.auth import AuthEndpoint
from .endpoints.users import UsersEndpoint
from .endpoints.vpn import VpnEndpoint
from .endpoints.payments import PaymentsEndpoint
from .endpoints.billing import BillingEndpoint


class BackendClient:
    """Cliente HTTP para comunicar con el backend."""
    
    def __init__(self, base_url: str, jwt_token: str):
        self.base_url = base_url
        self.http = httpx.AsyncClient(
            base_url=base_url,
            headers={
                "Authorization": f"Bearer {jwt_token}",
                "Content-Type": "application/json",
            },
            timeout=30.0,
        )
        
        # Endpoints
        self.auth = AuthEndpoint(self)
        self.users = UsersEndpoint(self)
        self.vpn = VpnEndpoint(self)
        self.payments = PaymentsEndpoint(self)
        self.billing = BillingEndpoint(self)
    
    async def close(self):
        await self.http.aclose()
```

#### 1.5 Middleware de autenticación

**Archivo:** `src/features/auth/middleware.py`
```python
from functools import wraps
from typing import Optional

from fastapi import Request, HTTPException, status
from fastapi.responses import RedirectResponse
from fastapi.templating import Jinja2Templates

from ...infrastructure.api.client import BackendClient
from ...core.config import settings

templates = Jinja2Templates(directory="src/infrastructure/web/templates")


def require_auth(func):
    """Decorator para requerir autenticación en routes."""
    
    @wraps(func)
    async def wrapper(request: Request, *args, **kwargs):
        # Obtuir token de cookie o header
        jwt_token = request.cookies.get("jwt_token")
        
        if not jwt_token:
            # Redirigir a login
            return RedirectResponse(url="/auth/login", status_code=303)
        
        # Verificar token con backend
        async with BackendClient(settings.BACKEND_URL, jwt_token) as backend:
            user = await backend.users.get_me()
            
            if not user:
                # Token inválido
                response = RedirectResponse(url="/auth/login", status_code=303)
                response.delete_cookie("jwt_token")
                return response
            
            # Agregar usuario al contexto
            request.state.user = user
            request.state.backend = backend
            
            return await func(request, *args, **kwargs)
    
    return wrapper
```

#### 1.6 Route de login con Telegram

**Archivo:** `src/features/auth/routes.py`
```python
from fastapi import APIRouter, Request, Form, Response
from fastapi.responses import RedirectResponse, HTMLResponse
from fastapi.templating import Jinja2Templates

from ...infrastructure.api.client import BackendClient
from ...core.config import settings

router = APIRouter(tags=["Authentication"])
templates = Jinja2Templates(directory="src/infrastructure/web/templates")


@router.get("/auth/login", response_class=HTMLResponse)
async def login_page(request: Request):
    """Página de login con Telegram."""
    return templates.TemplateResponse(
        "auth/login.html",
        {"request": request},
    )


@router.post("/auth/telegram")
async def authenticate_telegram(
    request: Request,
    init_data: str = Form(...),
):
    """
    Autentica con Telegram initData.
    
    Recibe initData desde Telegram WebApp y obtiene JWT del backend.
    """
    async with BackendClient(settings.BACKEND_URL, "") as backend:
        # Autenticar con backend
        result = await backend.auth.authenticate_telegram(init_data)
        
        if not result:
            return {"error": "Authentication failed"}
        
        access_token = result["access_token"]
        
        # Redirigir a dashboard con JWT en cookie
        response = RedirectResponse(url="/dashboard", status_code=303)
        response.set_cookie(
            key="jwt_token",
            value=access_token,
            httponly=True,
            secure=True,  # Solo HTTPS en producción
            samesite="lax",
            max_age=86400,  # 24 horas
        )
        
        return response
```

---

### DÍA 3-4: MiniApp - Features

#### 2.1 Feature de Dashboard

**Archivo:** `src/features/dashboard/routes.py`
```python
from fastapi import APIRouter, Request
from fastapi.responses import HTMLResponse
from fastapi.templating import Jinja2Templates

from ..auth.middleware import require_auth

router = APIRouter(tags=["Dashboard"])
templates = Jinja2Templates(directory="src/infrastructure/web/templates")


@router.get("/dashboard", response_class=HTMLResponse)
@require_auth
async def dashboard(request: Request):
    """Dashboard principal de la MiniApp."""
    user = request.state.user
    backend = request.state.backend
    
    # Obtener consumo
    usage = await backend.billing.get_usage()
    
    # Obtener claves
    keys = await backend.vpn.list_keys()
    
    return templates.TemplateResponse(
        "dashboard/index.html",
        {
            "request": request,
            "user": user,
            "usage": usage,
            "keys": keys,
        },
    )
```

**Archivo:** `src/features/dashboard/templates/dashboard/index.html`
```html
{% extends "base.html" %}

{% block title %}Dashboard - uSipipo VPN{% endblock %}

{% block content %}
<div class="dashboard">
    <h1>👋 Hola, {{ user.first_name }}</h1>
    
    <!-- Stats Cards -->
    <div class="stats-grid">
        <div class="stat-card">
            <div class="stat-icon">💰</div>
            <div class="stat-value">{{ usage.balance_gb }} GB</div>
            <div class="stat-label">Saldo Disponible</div>
        </div>
        
        <div class="stat-card">
            <div class="stat-icon">📊</div>
            <div class="stat-value">{{ usage.usage_percentage }}%</div>
            <div class="stat-label">Consumo</div>
        </div>
        
        <div class="stat-card">
            <div class="stat-icon">🔑</div>
            <div class="stat-value">{{ keys|length }}</div>
            <div class="stat-label">Claves Activas</div>
        </div>
    </div>
    
    <!-- Keys List -->
    <section class="keys-section">
        <h2>Mis Claves VPN</h2>
        {% if keys %}
            <div class="keys-list">
                {% for key in keys %}
                <div class="key-card">
                    <div class="key-header">
                        <span class="key-name">{{ key.name }}</span>
                        <span class="key-type {{ key.vpn_type }}">{{ key.vpn_type }}</span>
                    </div>
                    <div class="key-stats">
                        <span>📊 {{ key.data_used_gb }}/{{ key.data_limit_gb }} GB</span>
                        <span>🟢 {{ key.status }}</span>
                    </div>
                    <div class="key-actions">
                        <a href="/keys/{{ key.id }}" class="btn btn-sm">Ver</a>
                        <button class="btn btn-sm btn-danger" data-key-id="{{ key.id }}">Eliminar</button>
                    </div>
                </div>
                {% endfor %}
            </div>
        {% else %}
            <p class="empty-state">No tienes claves VPN. <a href="/keys/new">Crear primera clave</a></p>
        {% endif %}
    </section>
</div>
{% endblock %}
```

#### 2.2 Feature de Keys

**Archivo:** `src/features/keys/routes.py`
```python
from fastapi import APIRouter, Request, Form, HTTPException
from fastapi.responses import HTMLResponse, RedirectResponse
from fastapi.templating import Jinja2Templates
from uuid import UUID

from ..auth.middleware import require_auth

router = APIRouter(tags=["Keys"])
templates = Jinja2Templates(directory="src/infrastructure/web/templates")


@router.get("/keys/new", response_class=HTMLResponse)
@require_auth
async def new_key_page(request: Request):
    """Página para crear nueva clave."""
    return templates.TemplateResponse(
        "keys/new.html",
        {"request": request},
    )


@router.post("/keys")
@require_auth
async def create_key(
    request: Request,
    name: str = Form(...),
    vpn_type: str = Form(...),
    data_limit_gb: float = Form(default=5.0),
):
    """Crea nueva clave VPN."""
    backend = request.state.backend
    
    key = await backend.vpn.create_key(
        name=name,
        vpn_type=vpn_type,
        data_limit_gb=data_limit_gb,
    )
    
    if not key:
        raise HTTPException(status_code=400, detail="Failed to create key")
    
    return RedirectResponse(url="/dashboard", status_code=303)


@router.get("/keys/{key_id}", response_class=HTMLResponse)
@require_auth
async def key_detail(request: Request, key_id: UUID):
    """Detalles de una clave."""
    backend = request.state.backend
    
    key = await backend.vpn.get_key(key_id)
    config = await backend.vpn.get_key_config(key_id)
    
    if not key:
        raise HTTPException(status_code=404, detail="Key not found")
    
    return templates.TemplateResponse(
        "keys/detail.html",
        {
            "request": request,
            "key": key,
            "config": config,
        },
    )


@router.delete("/keys/{key_id}")
@require_auth
async def delete_key(request: Request, key_id: UUID):
    """Elimina una clave."""
    backend = request.state.backend
    
    success = await backend.vpn.delete_key(key_id)
    
    if not success:
        raise HTTPException(status_code=400, detail="Failed to delete key")
    
    return RedirectResponse(url="/dashboard", status_code=303)
```

#### 2.3 Feature de Payments

**Archivo:** `src/features/payments/routes.py`
```python
from fastapi import APIRouter, Request, Form, HTTPException
from fastapi.responses import HTMLResponse, RedirectResponse
from fastapi.templating import Jinja2Templates

from ..auth.middleware import require_auth

router = APIRouter(tags=["Payments"])
templates = Jinja2Templates(directory="src/infrastructure/web/templates")


@router.get("/buy", response_class=HTMLResponse)
@require_auth
async def buy_gb_page(request: Request):
    """Página para comprar GB."""
    backend = request.state.backend
    usage = await backend.billing.get_usage()
    
    return templates.TemplateResponse(
        "payments/buy.html",
        {
            "request": request,
            "usage": usage,
        },
    )


@router.post("/payments/crypto")
@require_auth
async def create_crypto_payment(
    request: Request,
    amount_usd: float = Form(...),
    gb_purchased: float = Form(...),
):
    """Crea pago con criptomoneda."""
    backend = request.state.backend
    
    payment = await backend.payments.create_crypto_payment(
        amount_usd=amount_usd,
        gb_purchased=gb_purchased,
        network="BSC",
    )
    
    if not payment:
        raise HTTPException(status_code=400, detail="Failed to create payment")
    
    # Redirigir a página de pago con QR
    return RedirectResponse(
        url=f"/payments/crypto/{payment['id']}",
        status_code=303,
    )
```

---

### DÍA 5-6: Landing Page

#### 3.1 Clonar y estructurar Landing

```bash
cd /home/mowgli
git clone git@github.com:usipipo/usipipo-landing.git
cd usipipo-landing

# Inicializar con uv
uv init --name usipipo-landing

# Crear estructura
mkdir -p src/features/home/{routes,templates}
mkdir -p src/features/pricing/{routes,templates}
mkdir -p src/features/docs/{routes,templates}
mkdir -p src/features/status/{routes,templates}
mkdir -p src/infrastructure/web/{templates,static}
mkdir -p src/shared
```

#### 3.2 Crear app FastAPI base

**Archivo:** `src/infrastructure/web/app.py`
```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

from ...features.home.routes import router as home_router
from ...features.pricing.routes import router as pricing_router
from ...features.docs.routes import router as docs_router
from ...features.status.routes import router as status_router
from ...core.config import settings


def create_app() -> FastAPI:
    """Factory para crear aplicación FastAPI."""
    app = FastAPI(
        title="uSipipo VPN",
        description="VPN Manager con Telegram Bot y Mini App",
        version="1.0.0",
    )
    
    # Templates
    templates = Jinja2Templates(directory="src/infrastructure/web/templates")
    app.state.templates = templates
    
    # Static files
    app.mount("/static", StaticFiles(directory="src/infrastructure/web/static"), name="static")
    
    # Routes
    app.include_router(home_router)
    app.include_router(pricing_router)
    app.include_router(docs_router)
    app.include_router(status_router)
    
    return app


app = create_app()
```

#### 3.3 Feature Home

**Archivo:** `src/features/home/routes.py`
```python
from fastapi import APIRouter, Request
from fastapi.responses import HTMLResponse

router = APIRouter(tags=["Home"])


@router.get("/", response_class=HTMLResponse)
async def home(request: Request):
    """Página de inicio."""
    templates = request.app.state.templates
    return templates.TemplateResponse(
        "home/index.html",
        {"request": request},
    )
```

**Archivo:** `src/features/home/templates/home/index.html`
```html
{% extends "base.html" %}

{% block title %}uSipipo VPN - VPN Manager con Telegram{% endblock %}

{% block content %}
<!-- Hero Section -->
<section class="hero">
    <div class="container">
        <h1>VPN Privada y Segura</h1>
        <p class="lead">
            Gestiona tu VPN personal directamente desde Telegram.
            WireGuard y Outline con pagos en crypto y Telegram Stars.
        </p>
        <div class="hero-cta">
            <a href="https://t.me/usipipo_bot" class="btn btn-primary btn-lg">
                🚀 Abrir en Telegram
            </a>
            <a href="/pricing" class="btn btn-outline btn-lg">
                💰 Ver Precios
            </a>
        </div>
    </div>
</section>

<!-- Features -->
<section class="features">
    <div class="container">
        <h2>¿Por qué uSipipo?</h2>
        <div class="features-grid">
            <div class="feature-card">
                <div class="feature-icon">🔐</div>
                <h3>Privacidad Total</h3>
                <p>Sin logs, sin registros. Tu tráfico es completamente privado.</p>
            </div>
            
            <div class="feature-card">
                <div class="feature-icon">⚡</div>
                <h3>Alta Velocidad</h3>
                <p>Servidores optimizados para máximo rendimiento con WireGuard.</p>
            </div>
            
            <div class="feature-card">
                <div class="feature-icon">💎</div>
                <h3>Fácil de Usar</h3>
                <p>Todo desde Telegram. Sin aplicaciones complicadas.</p>
            </div>
            
            <div class="feature-card">
                <div class="feature-icon">🌍</div>
                <h3>Global</h3>
                <p>Accede a contenido de todo el mundo sin restricciones.</p>
            </div>
        </div>
    </div>
</section>

<!-- How It Works -->
<section class="how-it-works">
    <div class="container">
        <h2>¿Cómo Funciona?</h2>
        <div class="steps">
            <div class="step">
                <div class="step-number">1</div>
                <h3>Abre el Bot</h3>
                <p>Inicia @usipipo_bot en Telegram</p>
            </div>
            
            <div class="step">
                <div class="step-number">2</div>
                <h3>Crea tu Clave</h3>
                <p>Genera una clave WireGuard o Outline</p>
            </div>
            
            <div class="step">
                <div class="step-number">3</div>
                <h3>Conéctate</h3>
                <p>Importa la config en tu cliente VPN</p>
            </div>
            
            <div class="step">
                <div class="step-number">4</div>
                <h3>Disfruta</h3>
                <p>Navega de forma privada y segura</p>
            </div>
        </div>
    </div>
</section>
{% endblock %}
```

#### 3.4 Feature Pricing

**Archivo:** `src/features/pricing/routes.py`
```python
from fastapi import APIRouter, Request
from fastapi.responses import HTMLResponse

router = APIRouter(tags=["Pricing"])


@router.get("/pricing", response_class=HTMLResponse)
async def pricing(request: Request):
    """Página de precios."""
    templates = request.app.state.templates
    return templates.TemplateResponse(
        "pricing/index.html",
        {"request": request},
    )
```

**Archivo:** `src/features/pricing/templates/pricing/index.html`
```html
{% extends "base.html" %}

{% block title %}Precios - uSipipo VPN{% endblock %}

{% block content %}
<section class="pricing">
    <div class="container">
        <h2>Planes Simples y Transparentes</h2>
        
        <!-- Free Tier -->
        <div class="pricing-card">
            <div class="pricing-header">
                <h3>Gratis</h3>
                <div class="price">$0</div>
            </div>
            <ul class="pricing-features">
                <li>✅ 5 GB de datos</li>
                <li>✅ 2 claves VPN</li>
                <li>✅ WireGuard y Outline</li>
                <li>❌ Soporte prioritario</li>
            </ul>
            <a href="https://t.me/usipipo_bot" class="btn btn-block">Comenzar Gratis</a>
        </div>
        
        <!-- Paid Packages -->
        <div class="pricing-grid">
            <div class="pricing-card">
                <div class="pricing-header">
                    <h3>10 GB</h3>
                    <div class="price">$5</div>
                </div>
                <ul class="pricing-features">
                    <li>✅ 10 GB adicionales</li>
                    <li>✅ Válido por 30 días</li>
                    <li>✅ Pago con crypto o Stars</li>
                </ul>
                <a href="https://t.me/usipipo_bot" class="btn btn-block">Comprar</a>
            </div>
            
            <div class="pricing-card featured">
                <div class="pricing-badge">Más Popular</div>
                <div class="pricing-header">
                    <h3>25 GB</h3>
                    <div class="price">$10</div>
                </div>
                <ul class="pricing-features">
                    <li>✅ 25 GB adicionales</li>
                    <li>✅ Válido por 30 días</li>
                    <li>✅ Soporte prioritario</li>
                </ul>
                <a href="https://t.me/usipipo_bot" class="btn btn-block">Comprar</a>
            </div>
            
            <div class="pricing-card">
                <div class="pricing-header">
                    <h3>50 GB</h3>
                    <div class="price">$18</div>
                </div>
                <ul class="pricing-features">
                    <li>✅ 50 GB adicionales</li>
                    <li>✅ Válido por 30 días</li>
                    <li>✅ Descuento del 10%</li>
                </ul>
                <a href="https://t.me/usipipo_bot" class="btn btn-block">Comprar</a>
            </div>
        </div>
    </div>
</section>
{% endblock %}
```

#### 3.5 Feature Docs

**Archivo:** `src/features/docs/routes.py`
```python
from fastapi import APIRouter, Request
from fastapi.responses import HTMLResponse

router = APIRouter(tags=["Documentation"])


@router.get("/docs", response_class=HTMLResponse)
async def docs(request: Request):
    """Página de documentación."""
    templates = request.app.state.templates
    return templates.TemplateResponse(
        "docs/index.html",
        {"request": request},
    )
```

#### 3.6 Feature Status

**Archivo:** `src/features/status/routes.py`
```python
from fastapi import APIRouter, Request
from fastapi.responses import HTMLResponse

router = APIRouter(tags=["Status"])


@router.get("/status", response_class=HTMLResponse)
async def status(request: Request):
    """Página de status del servicio."""
    templates = request.app.state.templates
    return templates.TemplateResponse(
        "status/index.html",
        {"request": request},
    )
```

---

### DÍA 7: Docker + Tests

#### 4.1 Docker para MiniApp

**Archivo:** `usipipo-miniapp-web/docker-compose.yml`
```yaml
version: '3.8'

services:
  miniapp:
    build: .
    ports:
      - "8001:8001"
    environment:
      - BACKEND_URL=http://backend:8000
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - backend
    restart: unless-stopped
```

#### 4.2 Docker para Landing

**Archivo:** `usipipo-landing/docker-compose.yml`
```yaml
version: '3.8'

services:
  landing:
    build: .
    ports:
      - "8002:8002"
    restart: unless-stopped
```

---

## ✅ Criterios de Aceptación

- [ ] MiniApp consume APIs del backend (no lógica embebida)
- [ ] Auth con Telegram WebApp funciona
- [ ] Features principales de MiniApp funcionando:
  - [ ] Dashboard
  - [ ] Gestión de claves
  - [ ] Pagos
- [ ] Landing page publicada con:
  - [ ] Home
  - [ ] Pricing
  - [ ] Docs
  - [ ] Status
- [ ] Docker compose funcional para ambos servicios
- [ ] Tests básicos pasando

---

## 📚 Recursos

- [FastAPI + Jinja2](https://fastapi.tiangolo.com/advanced/templates/)
- [Telegram WebApp](https://core.telegram.org/bots/webapps)
- [httpx async](https://www.python-httpx.org/async/)

---

## 🔄 Dependencias para Semana 6

La Semana 6 necesita:
- ✅ Todos los servicios funcionando independientemente
- ✅ Backend estable en producción
- ✅ Bot y MiniApp consumiendo APIs correctamente

---

## 📝 Notas

- Mantener consistencia visual entre MiniApp y Landing (mismo CSS base)
- MiniApp debe funcionar dentro de Telegram WebApp
- Landing page debe ser SEO-friendly (meta tags, structured data)
