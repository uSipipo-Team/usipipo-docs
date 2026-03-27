# Semana 6: CI/CD + Deploy a Producción

**Fecha:** 22-28 Abril 2026  
**Objetivo:** Configurar CI/CD para todos los repositorios y deployar a producción en VPS con Docker

**Dependencias:** Semanas 1-5 completadas ✅ (Todos los servicios funcionando)

---

## 📦 Entregables de la Semana

- [ ] GitHub Actions configurado para los 6 repositorios
- [ ] Pipelines de CI: tests, linting, build
- [ ] Pipelines de CD: build de Docker images, deploy a VPS
- [ ] Docker registry configurado (GitHub Container Registry)
- [ ] VPS configurado con Docker + Docker Compose
- [ ] Todos los servicios corriendo en producción
- [ ] Smoke tests pasando
- [ ] Documentación de deployment actualizada

---

## 🎯 Tareas Detalladas

### DÍA 1-2: CI para Todos los Repos

#### 1.1 GitHub Actions para usipipo-commons

**Archivo:** `usipipo-commons/.github/workflows/ci.yml`
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          version: "latest"
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      
      - name: Install dependencies
        run: uv sync --dev
      
      - name: Run tests
        run: uv run pytest -v --cov=usipipo_commons --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
          flags: unittests

  lint:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      
      - name: Install dependencies
        run: uv sync --dev
      
      - name: Run ruff
        run: uv run ruff check usipipo_commons
      
      - name: Run mypy
        run: uv run mypy usipipo_commons

  build:
    runs-on: ubuntu-latest
    needs: [test, lint]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      
      - name: Build package
        run: uv build
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
```

**Archivo:** `usipipo-commons/.github/workflows/publish.yml`
```yaml
name: Publish to GitHub Packages

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      
      - name: Build package
        run: uv build
      
      - name: Publish to GitHub Packages
        run: uv publish --publish-url https://pypi.pkg.github.com
        env:
          UV_PUBLISH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

#### 1.2 GitHub Actions para usipipo-backend

**Archivo:** `usipipo-backend/.github/workflows/ci.yml`
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: usipipo_test
          POSTGRES_USER: usipipo
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      
      - name: Install dependencies
        run: uv sync --dev
      
      - name: Run migrations
        run: uv run alembic upgrade head
        env:
          DATABASE_URL: postgresql+asyncpg://usipipo:testpass@localhost:5432/usipipo_test
      
      - name: Run tests
        run: uv run pytest -v --cov=src --cov-report=xml
        env:
          DATABASE_URL: postgresql+asyncpg://usipipo:testpass@localhost:5432/usipipo_test
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test_secret
          TELEGRAM_TOKEN: ${{ secrets.TELEGRAM_TOKEN }}
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4

  lint:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      
      - name: Install dependencies
        run: uv sync --dev
      
      - name: Run ruff
        run: uv run ruff check src
      
      - name: Run mypy
        run: uv run mypy src

  security:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      
      - name: Install dependencies
        run: uv sync --dev
      
      - name: Run Bandit
        run: uv run bandit -r src -f json -o bandit-report.json
      
      - name: Upload security report
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: bandit-report.json
```

**Archivo:** `usipipo-backend/.github/workflows/docker.yml`
```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags:
      - 'v*'
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

#### 1.3 GitHub Actions para usipipo-telegram-bot

**Archivo:** `usipipo-telegram-bot/.github/workflows/ci.yml`
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      
      - name: Install dependencies
        run: uv sync --dev
      
      - name: Run tests
        run: uv run pytest -v --cov=src --cov-report=xml
        env:
          TELEGRAM_TOKEN: ${{ secrets.TELEGRAM_TOKEN }}
          BACKEND_URL: http://localhost:8000
          JWT_SECRET: test_secret
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4

  lint:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      
      - name: Install dependencies
        run: uv sync --dev
      
      - name: Run ruff
        run: uv run ruff check src

  build-docker:
    runs-on: ubuntu-latest
    needs: [test, lint]
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

#### 1.4 GitHub Actions para usipipo-miniapp-web

**Archivo:** `usipipo-miniapp-web/.github/workflows/ci.yml`
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      
      - name: Install dependencies
        run: uv sync --dev
      
      - name: Run tests
        run: uv run pytest -v --cov=src --cov-report=xml
        env:
          BACKEND_URL: http://localhost:8000
          JWT_SECRET: test_secret
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4

  lint:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      
      - name: Install dependencies
        run: uv sync --dev
      
      - name: Run ruff
        run: uv run ruff check src

  build-docker:
    runs-on: ubuntu-latest
    needs: [test, lint]
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

#### 1.5 GitHub Actions para usipipo-landing

**Archivo:** `usipipo-landing/.github/workflows/ci.yml`
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v3
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      
      - name: Install dependencies
        run: uv sync --dev
      
      - name: Run ruff
        run: uv run ruff check src

  build-docker:
    runs-on: ubuntu-latest
    needs: [lint]
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

#### 1.6 GitHub Actions para usipipo-vpn-flutter

**Archivo:** `usipipo-vpn-flutter/.github/workflows/ci.yml`
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.x'
      
      - name: Install dependencies
        run: flutter pub get
      
      - name: Run tests
        run: flutter test
      
      - name: Run analyzer
        run: flutter analyze
      
      - name: Build APK (debug)
        run: flutter build apk --debug

  build-web:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.x'
      
      - name: Install dependencies
        run: flutter pub get
      
      - name: Build Web
        run: flutter build web
```

---

### DÍA 3-4: Configurar VPS

#### 2.1 Preparar VPS

```bash
# Conectar al VPS
ssh user@your-vps-ip

# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Instalar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verificar instalación
docker --version
docker-compose --version

# Configurar firewall
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 8000/tcp  # Backend API
sudo ufw allow 8001/tcp  # MiniApp
sudo ufw allow 8002/tcp  # Landing
sudo ufw enable
```

#### 2.2 Configurar Docker Compose en Producción

```bash
# Crear directorio de producción
sudo mkdir -p /opt/usipipo
cd /opt/usipipo

# Crear estructura de directorios
sudo mkdir -p backend miniapp landing landing-bot
```

**Archivo:** `/opt/usipipo/docker-compose.yml`
```yaml
version: '3.8'

services:
  # ========================
  # Backend
  # ========================
  backend:
    image: ghcr.io/usipipo/usipipo-backend:latest
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://usipipo:${DB_PASSWORD}@db:5432/usipipo
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
      - TELEGRAM_TOKEN=${TELEGRAM_TOKEN}
      - TRON_DEALER_API_KEY=${TRON_DEALER_API_KEY}
      - TRON_DEALER_WEBHOOK_SECRET=${TRON_DEALER_WEBHOOK_SECRET}
      - SECRET_KEY=${SECRET_KEY}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - usipipo-network
    volumes:
      - backend-logs:/app/logs

  # ========================
  # Database
  # ========================
  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_DB=usipipo
      - POSTGRES_USER=usipipo
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - usipipo-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U usipipo"]
      interval: 5s
      timeout: 5s
      retries: 5

  # ========================
  # Redis
  # ========================
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data
    networks:
      - usipipo-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  # ========================
  # Telegram Bot
  # ========================
  bot:
    image: ghcr.io/usipipo/usipipo-telegram-bot:latest
    restart: unless-stopped
    environment:
      - TELEGRAM_TOKEN=${TELEGRAM_TOKEN}
      - BACKEND_URL=http://backend:8000
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - backend
    networks:
      - usipipo-network
    volumes:
      - bot-logs:/app/logs

  # ========================
  # MiniApp Web
  # ========================
  miniapp:
    image: ghcr.io/usipipo/usipipo-miniapp-web:latest
    restart: unless-stopped
    ports:
      - "8001:8001"
    environment:
      - BACKEND_URL=http://backend:8000
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - backend
    networks:
      - usipipo-network
    volumes:
      - miniapp-logs:/app/logs

  # ========================
  # Landing Page
  # ========================
  landing:
    image: ghcr.io/usipipo/usipipo-landing:latest
    restart: unless-stopped
    ports:
      - "8002:8002"
    networks:
      - usipipo-network
    volumes:
      - landing-logs:/app/logs

networks:
  usipipo-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  backend-logs:
  bot-logs:
  miniapp-logs:
  landing-logs:
```

#### 2.3 Crear script de deploy

**Archivo:** `/opt/usipipo/deploy.sh`
```bash
#!/bin/bash

set -e

echo "🚀 Deploying uSipipo Ecosystem..."

# Variables de entorno
export $(cat .env | xargs)

# Pull latest images
echo "📦 Pulling latest Docker images..."
docker-compose pull

# Run migrations
echo "🗄️  Running database migrations..."
docker-compose run --rm backend uv run alembic upgrade head

# Restart services
echo "🔄 Restarting services..."
docker-compose up -d

# Wait for services to be healthy
echo "⏳ Waiting for services to be healthy..."
sleep 10

# Check health
echo "✅ Checking service health..."
docker-compose ps

# Show logs
echo "📋 Showing recent logs..."
docker-compose logs --tail=20

echo "🎉 Deployment complete!"
echo ""
echo "Services:"
echo "  - Backend API:  http://$(hostname -I | awk '{print $1}'):8000"
echo "  - MiniApp Web:  http://$(hostname -I | awk '{print $1}'):8001"
echo "  - Landing Page: http://$(hostname -I | awk '{print $1}'):8002"
echo "  - Bot:          Running (check logs with: docker-compose logs bot)"
```

```bash
# Hacer ejecutable
sudo chmod +x /opt/usipipo/deploy.sh
```

#### 2.4 Crear archivo .env de producción

**Archivo:** `/opt/usipipo/.env`
```bash
# ========================
# Database
# ========================
DB_PASSWORD=super_secure_password_change_me

# ========================
# JWT
# ========================
JWT_SECRET=openssl_rand_hex_32_generate_me

# ========================
# Telegram
# ========================
TELEGRAM_TOKEN=bot_token_from_botfather

# ========================
# Backend
# ========================
SECRET_KEY=openssl_rand_hex_32_generate_me

# ========================
# Crypto Payments
# ========================
TRON_DEALER_API_KEY=td_xxx_change_me
TRON_DEALER_WEBHOOK_SECRET=openssl_rand_hex_32_generate_me
TRON_DEALER_SWEEP_WALLET=your_bsc_wallet_address

# ========================
# Server
# ========================
SERVER_IP=your_vps_public_ip
```

```bash
# Generar secrets
openssl rand -hex 32  # Para JWT_SECRET
openssl rand -hex 32  # Para SECRET_KEY
openssl rand -hex 32  # Para TRON_DEALER_WEBHOOK_SECRET
```

---

### DÍA 5: Reverse Proxy con Nginx (Opcional pero recomendado)

#### 3.1 Instalar Nginx

```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

#### 3.2 Configurar Nginx

**Archivo:** `/etc/nginx/sites-available/usipipo`
```nginx
# Backend API
server {
    listen 80;
    server_name api.usipipo.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# MiniApp Web
server {
    listen 80;
    server_name app.usipipo.com;

    location / {
        proxy_pass http://localhost:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Landing Page
server {
    listen 80;
    server_name usipipo.com www.usipipo.com;

    location / {
        proxy_pass http://localhost:8002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
# Habilitar sitio
sudo ln -s /etc/nginx/sites-available/usipipo /etc/nginx/sites-enabled/

# Testear configuración
sudo nginx -t

# Recargar Nginx
sudo systemctl reload nginx
```

---

### DÍA 6: Deploy + Smoke Tests

#### 4.1 Deploy inicial

```bash
cd /opt/usipipo

# Copiar .env
sudo nano .env  # Pegar variables de entorno

# Ejecutar deploy
sudo ./deploy.sh
```

#### 4.2 Smoke tests

**Archivo:** `tests/smoke/test_api.py`
```python
import requests

BACKEND_URL = "http://localhost:8000"


def test_health_check():
    """Test de health check del backend."""
    response = requests.get(f"{BACKEND_URL}/health")
    assert response.status_code == 200
    assert response.json() == {"status": "healthy"}


def test_openapi_docs():
    """Test de documentación OpenAPI."""
    response = requests.get(f"{BACKEND_URL}/docs")
    assert response.status_code == 200


def test_auth_endpoint():
    """Test de endpoint de auth (debe fallar sin datos válidos)."""
    response = requests.post(
        f"{BACKEND_URL}/api/v1/auth/telegram",
        json={"init_data": "invalid"},
    )
    assert response.status_code == 401
```

**Archivo:** `tests/smoke/test_bot.py`
```python
"""
Tests para el bot de Telegram.

NOTA: Estos tests requieren un bot de Telegram real.
Ejecutar manualmente o con variables de entorno configuradas.
"""
import os
from telegram import Bot


def test_bot_info():
    """Test de información del bot."""
    bot_token = os.getenv("TELEGRAM_TOKEN")
    if not bot_token:
        print("⚠️  TELEGRAM_TOKEN not set, skipping test")
        return
    
    bot = Bot(token=bot_token)
    info = bot.get_me()
    assert info.username is not None
    print(f"✅ Bot @{info.username} is running")
```

**Archivo:** `tests/smoke/test_web.py`
```python
import requests


def test_miniapp_health():
    """Test de health check de MiniApp."""
    response = requests.get("http://localhost:8001")
    assert response.status_code == 200


def test_landing_health():
    """Test de health check de Landing."""
    response = requests.get("http://localhost:8002")
    assert response.status_code == 200
```

Ejecutar smoke tests:

```bash
cd /opt/usipipo

# Instalar dependencias para tests
pip install requests python-telegram-bot

# Ejecutar tests
pytest tests/smoke/ -v
```

---

### DÍA 7: Documentación + Monitoreo

#### 5.1 Crear documentación de deployment

**Archivo:** `docs/DEPLOYMENT.md` (en cada repositorio)
```markdown
# Deployment Guide

## Requisitos

- Docker 24+
- Docker Compose 2.24+
- VPS con Ubuntu 22.04+

## Variables de Entorno

Copiar `.env.example` a `.env` y configurar:

```bash
# Ver example.env para todas las variables requeridas
```

## Deploy Local

```bash
docker-compose up -d
```

## Deploy a Producción

Ver [Production Deployment](../../usipipo-infrastructure/docs/production-deployment.md)
```

#### 5.2 Configurar monitoreo básico

**Archivo:** `/opt/usipipo/docker-compose.monitoring.yml`
```yaml
version: '3.8'

services:
  # Prometheus (métricas)
  prometheus:
    image: prom/prometheus:latest
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - usipipo-network

  # Grafana (visualización)
  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin_change_me
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - usipipo-network

  # Loki (logs)
  loki:
    image: grafana/loki:latest
    restart: unless-stopped
    ports:
      - "3100:3100"
    networks:
      - usipipo-network

volumes:
  prometheus_data:
  grafana_data:
```

---

## ✅ Criterios de Aceptación

- [ ] GitHub Actions configurado para los 6 repositorios
- [ ] CI pasa en todos los repos (tests + linting)
- [ ] Docker images se build y publican a GHCR
- [ ] VPS configurado con Docker + Docker Compose
- [ ] Todos los servicios corriendo en producción:
  - [ ] Backend API (puerto 8000)
  - [ ] PostgreSQL + Redis
  - [ ] Telegram Bot
  - [ ] MiniApp Web (puerto 8001)
  - [ ] Landing Page (puerto 8002)
- [ ] Smoke tests pasando
- [ ] Documentación de deployment actualizada
- [ ] Monitoreo básico configurado (opcional)

---

## 📚 Recursos

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Docker Compose Reference](https://docs.docker.com/compose/)
- [Nginx Configuration Guide](https://nginx.org/en/docs/)

---

## 🎉 ¡Migración Completada!

Al finalizar esta semana, tendrás:

✅ 6 repositorios independientes  
✅ Cada servicio con su propio CI/CD  
✅ Deploy automático a producción  
✅ Arquitectura escalable y mantenible  
✅ Documentación completa  

---

## 📝 Notas

- Mantener secrets seguros (usar GitHub Secrets)
- Backup automático de PostgreSQL configurado
- Rotar secrets periódicamente
- Monitorear logs y métricas regularmente
