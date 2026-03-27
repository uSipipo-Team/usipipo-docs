# 🚀 Getting Started - Guía de Inicio para Desarrolladores

> Configura tu entorno de desarrollo para contribuir al ecosistema uSipipo

---

## 📋 Prerrequisitos

### Software Requerido

| Herramienta | Versión Mínima | Propósito |
|-------------|----------------|-----------|
| **Python** | 3.13+ | Backend y servicios |
| **uv** | 0.5+ | Package manager (recomendado) |
| **pip** | 24.0+ | Package manager (alternativo) |
| **Git** | 2.40+ | Control de versiones |
| **Docker** | 24.0+ | Contenerización |
| **PostgreSQL** | 15+ | Base de datos |
| **Redis** | 7.0+ | Caché y colas |
| **Node.js** | 20+ | Frontend (opcional) |
| **Flutter** | 3.16+ | App móvil (opcional) |

### Instalación de uv (Recomendado)

```bash
# Linux/macOS
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# Verificar instalación
uv --version
```

---

## 🏗️ Configuración del Ecosistema

### 1. Clonar Repositorios

```bash
# Directorio base
mkdir ~/usipipo && cd ~/usipipo

# Clonar todos los repositorios
git clone https://github.com/uSipipo-Team/usipipo-backend.git
git clone https://github.com/uSipipo-Team/usipipo-commons.git
git clone https://github.com/uSipipo-Team/usipipo-telegram-bot.git
git clone https://github.com/uSipipo-Team/usipipo-landing.git
git clone https://github.com/uSipipo-Team/usipipo-miniapp-web.git
git clone https://github.com/uSipipo-Team/usipipovpnapp.git
git clone https://github.com/uSipipo-Team/usipipo-code-quality.git
```

### 2. Configurar Variables de Entorno

Cada repositorio tiene un archivo `example.env`. Copia y configura:

```bash
# Ejemplo para backend
cd usipipo-backend
cp example.env .env
# Editar .env con tus credenciales
```

**Variables críticas comunes:**
- `DATABASE_URL` - Conexión a PostgreSQL
- `REDIS_URL` - Conexión a Redis
- `TELEGRAM_BOT_TOKEN` - Token del bot
- `SECRET_KEY` - Clave secreta para JWT
- `BACKEND_API_URL` - URL de la API backend

---

## 💻 Configuración por Proyecto

### usipipo-backend

```bash
cd usipipo-backend

# Instalar dependencias
uv sync --dev

# Configurar base de datos
uv run alembic upgrade head

# Ejecutar tests
uv run pytest

# Iniciar servidor de desarrollo
uv run python -m src
```

**Endpoints de desarrollo:**
- API: http://localhost:8000
- Docs (Swagger): http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc

### usipipo-commons

```bash
cd usipipo-commons

# Instalar dependencias
uv sync --dev

# Instalar localmente para desarrollo
uv pip install -e .

# Ejecutar tests
uv run pytest

# Build para PyPI
uv build
```

### usipipo-telegram-bot

```bash
cd usipipo-telegram-bot

# Instalar dependencias
uv sync --dev

# Iniciar bot
uv run python -m src

# Ejecutar tests
uv run pytest
```

### usipipo-landing

```bash
cd usipipo-landing

# Instalar dependencias
uv sync --dev

# Iniciar servidor Flask
uv run python -m src

# Acceso: http://localhost:5000
```

### usipipo-miniapp-web

```bash
cd usipipo-miniapp-web

# Instalar dependencias
uv sync --dev

# Iniciar MiniApp
uv run python -m src

# Acceso: http://localhost:5000
```

### usipipovpnapp (Flutter)

```bash
cd usipipovpnapp

# Instalar dependencias
flutter pub get

# Ejecutar en emulador
flutter run

# Build APK
flutter build apk --release
```

---

## 🧪 Ejecución de Tests

### Backend

```bash
cd usipipo-backend

# Todos los tests
uv run pytest

# Con coverage
uv run pytest --cov=src --cov-report=html

# Tests específicos
uv run pytest tests/unit/test_auth.py -v
```

### Commons

```bash
cd usipipo-commons

# Todos los tests
uv run pytest

# Con coverage
uv run pytest --cov=usipipo_commons
```

### Telegram Bot

```bash
cd usipipo-telegram-bot

# Todos los tests
uv run pytest

# Tests de integración
uv run pytest tests/integration/ -v
```

---

## 🔍 Code Quality

### Pre-commit Hooks

Todos los repositorios usan pre-commit hooks. Configúralos:

```bash
# Instalar pre-commit
uv pip install pre-commit

# Instalar hooks en cada repositorio
cd usipipo-backend
pre-commit install

# Ejecutar hooks manualmente
pre-commit run --all-files
```

### Herramientas de Calidad

| Herramienta | Propósito | Comando |
|-------------|-----------|---------|
| **ruff** | Linting + formato | `uv run ruff check . --fix` |
| **mypy** | Type checking | `uv run mypy .` |
| **pytest** | Tests | `uv run pytest` |
| **bandit** | Security scan | `uv run bandit -r src/` |

### Code Quality Rules

Consulta las 7 reglas de calidad de código en [usipipo-code-quality](https://github.com/uSipipo-Team/usipipo-code-quality):

1. KISS Principle
2. Delete Code Fearlessly
3. Code Not Comments
4. Atomic Commits
5. Explain Simply
6. Make It Work First
7. Small Commits

---

## 🐳 Docker

### Backend

```bash
cd usipipo-backend

# Build
docker build -t usipipo-backend .

# Ejecutar
docker run -p 8000:8000 --env-file .env usipipo-backend

# Docker Compose (con PostgreSQL y Redis)
docker-compose up -d
```

### Telegram Bot

```bash
cd usipipo-telegram-bot

# Build
docker build -t usipipo-bot .

# Ejecutar
docker run --env-file .env usipipo-bot
```

### Landing

```bash
cd usipipo-landing

# Build
docker build -t usipipo-landing .

# Ejecutar
docker run -p 5000:5000 usipipo-landing
```

---

## 🗄️ Base de Datos

### PostgreSQL Setup

```bash
# Docker (recomendado para desarrollo)
docker run -d \
  --name usipipo-postgres \
  -e POSTGRES_USER=usipipo \
  -e POSTGRES_PASSWORD=devpassword \
  -e POSTGRES_DB=usipipo_dev \
  -p 5432:5432 \
  postgres:15

# DATABASE_URL=postgresql://usipipo:devpassword@localhost:5432/usipipo_dev
```

### Redis Setup

```bash
# Docker
docker run -d \
  --name usipipo-redis \
  -p 6379:6379 \
  redis:7-alpine

# REDIS_URL=redis://localhost:6379
```

### Migraciones (Backend)

```bash
cd usipipo-backend

# Crear nueva migración
uv run alembic revision --autogenerate -m "Descripción del cambio"

# Aplicar migraciones
uv run alembic upgrade head

# Revertir última migración
uv run alembic downgrade -1

# Ver estado actual
uv run alembic current
```

---

## 🔐 Configuración de Seguridad

### JWT Keys

```bash
# Generar SECRET_KEY segura
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

### Telegram Bot Token

1. Abre Telegram y busca @BotFather
2. Envía `/newbot`
3. Sigue las instrucciones
4. Copia el token generado

### Base de Datos

**Nunca uses credenciales de producción en desarrollo.**

Crea usuarios separados:
```sql
-- Usuario de desarrollo
CREATE USER usipipo_dev WITH PASSWORD 'dev_password';
CREATE DATABASE usipipo_dev OWNER usipipo_dev;

-- Usuario de producción (solo en prod)
CREATE USER usipipo_prod WITH PASSWORD 'STRONG_PASSWORD';
CREATE DATABASE usipipo_prod OWNER usipipo_prod;
```

---

## 📝 Convenciones de Commits

Usa [Conventional Commits](https://www.conventionalcommits.org/):

```bash
# Nuevas features
git commit -m "feat: agregar sistema de referidos"

# Bug fixes
git commit -m "fix: corregir cálculo de consumo de datos"

# Refactors
git commit -m "refactor: simplificar lógica de autenticación"

# Docs
git commit -m "docs: actualizar README con ejemplos de API"

# Tests
git commit -m "test: agregar tests para payment flow"

# Chore
git commit -m "chore: actualizar dependencias de pytest"
```

---

## 🚨 Solución de Problemas Comunes

### Error: "ModuleNotFoundError: No module named 'usipipo_commons'"

```bash
# Instalar commons localmente
cd usipipo-commons
uv pip install -e .
```

### Error: "Database connection failed"

```bash
# Verificar PostgreSQL está corriendo
docker ps | grep postgres

# Verificar DATABASE_URL en .env
echo $DATABASE_URL

# Testear conexión
psql $DATABASE_URL -c "SELECT 1"
```

### Error: "Redis connection refused"

```bash
# Verificar Redis está corriendo
docker ps | grep redis

# Testear conexión
redis-cli ping  # Debe responder "PONG"
```

### Error: "Port already in use"

```bash
# Ver qué proceso usa el puerto
lsof -i :8000

# Matar proceso
kill -9 <PID>

# O cambiar puerto en .env
PORT=8001
```

---

## 📚 Recursos Adicionales

| Recurso | Enlace |
|---------|--------|
| **Documentación de APIs** | [apis/backend-api-reference.md](apis/backend-api-reference.md) |
| **Arquitectura del Sistema** | [context/ecosystem-architecture.md](context/ecosystem-architecture.md) |
| **Guías de Estilo** | [brand/identity.md](brand/identity.md) |
| **Code Quality Rules** | https://github.com/uSipipo-Team/usipipo-code-quality |

---

## 🆘 ¿Necesitas Ayuda?

- **Documentación:** Revisa este repositorio de docs
- **Issues:** Reporta bugs en GitHub
- **Email:** dev@usipipo.com
- **Telegram:** @usipipobot (soporte)

---

**Última actualización:** 2026-03-27
