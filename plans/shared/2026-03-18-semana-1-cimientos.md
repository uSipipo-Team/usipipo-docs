# Semana 1: Cimientos del Ecosistema uSipipo

**Fecha:** 18-24 Marzo 2026
**Estado:** ✅ **COMPLETADO**
**Objetivo:** Crear los 6 repositorios base y establecer la fundación del ecosistema

---

## 📦 Entregables de la Semana

- [x] 6 repositorios creados en GitHub (vacíos o con estructura base)
- [x] `usipipo-commons` funcional y publicado ✅ **EN PYPI**
- [x] Documentación base en cada repositorio
- [x] Estructura de directorios consistente en todos los repos

---

## 🎯 Tareas Detalladas

### ✅ Día 1-2: Crear Repositorios en GitHub - **COMPLETADO**

#### 1.1 Crear repositorios ✅

**Organización:** `uSipipo-Team` (anteriormente `usipipo`)

| # | Repositorio | Visibilidad | Estado | URL |
|---|-------------|-------------|--------|-----|
| 1 | `usipipo-backend` | Privado | ✅ Creado | https://github.com/uSipipo-Team/usipipo-backend |
| 2 | `usipipo-telegram-bot` | Público | ✅ Creado | https://github.com/uSipipo-Team/usipipo-telegram-bot |
| 3 | `usipipo-miniapp-web` | Público | ✅ Creado | https://github.com/uSipipo-Team/usipipo-miniapp-web |
| 4 | `usipipo-vpn-android` | Público | ✅ Creado | https://github.com/uSipipo-Team/usipipo-vpn-android |
| 5 | `usipipo-landing` | Público | ✅ Creado | https://github.com/uSipipo-Team/usipipo-landing |
| 6 | `usipipo-commons` | Privado | ✅ Creado | https://github.com/uSipipo-Team/usipipo-commons |

#### 1.2 Configurar branch protection en todos los repos ✅ **COMPLETADO**

**Completado:** 2026-03-18 con script `scripts/set-branch-protection.sh`

Para cada repositorio (Settings → Branches → Add branch protection rule):
- Branch name pattern: `main`
- [x] Require a pull request before merging
- [x] Require status checks to pass before merging (pytest, build, lint)
- [x] Require branches to be up to date before merging
- [x] Enforce admins enabled

**Notas:**
- Repos `usipipo-backend` y `usipipo-commons` cambiados de privado a público (GitHub Pro no requerido)
- Script guardado en: `/home/mowgli/usipipo/scripts/set-branch-protection.sh`

---

### ✅ Día 2-4: usipipo-commons (Librería Compartida) - **COMPLETADO**

#### 2.1-2.10 ✅ **TODAS COMPLETADAS**

**Logros:**
- ✅ Estructura completa creada (entities, schemas, constants, utils)
- ✅ 3 entidades del dominio: `User`, `VpnKey`, `Payment`
- ✅ 4 enums: `VpnType`, `KeyStatus`, `PaymentStatus`, `PaymentMethod`
- ✅ 6 schemas Pydantic para validación
- ✅ Constantes compartidas (planes, bonos, errores)
- ✅ Utilitarios (validadores, formateadores)
- ✅ 33 tests pasando
- ✅ Workflow CI/CD configurado
- ✅ **Publicado en PyPI.org** 🎉

**Enlaces:**
- PyPI: https://pypi.org/project/usipipo-commons/
- GitHub: https://github.com/uSipipo-Team/usipipo-commons
- Release: https://github.com/uSipipo-Team/usipipo-commons/releases/tag/v0.2.0

**Instalación:**
```bash
pip install usipipo-commons
```

---

### ✅ Día 5-6: Estructura Base para Todos los Repos - **COMPLETADO**

#### 3.1 Crear estructura base en cada repositorio ✅ **COMPLETADO**

**Repositorios:**
- [x] `usipipo-backend` ✅ **COMPLETADO** - https://github.com/uSipipo-Team/usipipo-backend
- [x] `usipipo-telegram-bot` ✅ **COMPLETADO** - https://github.com/uSipipo-Team/usipipo-telegram-bot
- [x] `usipipo-miniapp-web` ✅ **COMPLETADO** - https://github.com/uSipipo-Team/usipipo-miniapp-web
- [x] `usipipo-vpn-android` ✅ **COMPLETADO** - https://github.com/uSipipo-Team/usipipo-vpn-android (Flutter existente)
- [x] `usipipo-landing` ✅ **COMPLETADO** - https://github.com/uSipipo-Team/usipipo-landing

**Archivos creados en cada repo:**
1. **`README.md`** ✅ (template base)
2. **`.gitignore`** ✅ (template Python/FastAPI)
3. **`pyproject.toml`** ✅ (template base)
4. **`example.env`** ✅
5. **`Dockerfile`** ✅ (template Python)

**Estructura de directorios:**
- `src/` ✅ (código fuente)
- `tests/` ✅ (tests unitarios)
- `docs/` ✅ (documentación: ARCHITECTURE.md, API.md, DEPLOYMENT.md)

---

## ✅ Criterios de Aceptación

| Criterio | Estado | Notas |
|----------|--------|-------|
| Los 6 repositorios existen en GitHub | ✅ Completado | Organización: uSipipo-Team |
| `usipipo-commons` tiene estructura completa | ✅ Completado | Entities, schemas, constants, utils |
| `usipipo-commons` está publicado | ✅ Completado | PyPI.org (v0.2.0) |
| `usipipo-backend` tiene estructura base | ✅ Completado | FastAPI, tests, docs |
| `usipipo-telegram-bot` tiene estructura base | ✅ Completado | python-telegram-bot, tests, docs |
| `usipipo-miniapp-web` tiene estructura base | ✅ Completado | Flask, templates, tests, docs |
| `usipipo-vpn-android` tiene estructura base | ✅ Completado | Flutter existente, remote actualizado |
| `usipipo-landing` tiene estructura base | ✅ Completado | Flask, templates, tests, docs |
| Branch protection activado | ✅ Completado | 6/6 repos con PR + status checks |
| Tests básicos pasan en usipipo-commons | ✅ Completado | 33 tests pasando |

---

## 📊 Resumen de Progreso

```
Semana 1: Cimientos
├── Fase 1: Crear Repositorios      ✅ 100%
├── Fase 2: usipipo-commons         ✅ 100%
├── Fase 3: Estructura Base         ✅ 100%
└── Fase 4: Branch Protection       ✅ 100%
```

**Logros adicionales:**
- ✅ Script automatizado para branch protection (`scripts/set-branch-protection.sh`)
- ✅ Todos los repos cambiados a públicos (no requiere GitHub Pro)

---

## ✅ SEMANA 1: COMPLETADA

**Fecha de finalización:** 2026-03-18

**Próximo:** Continuar con [Semana 2: Backend Auth VPN](./2026-03-25-semana-2-backend-auth-vpn.md)

---

## 📋 Templates para Tarea 3.1 (Referencia)

### README.md Template
```markdown
# {Nombre del Servicio}

> {Descripción breve}

## Estado

- [x] En desarrollo
- [ ] Alpha
- [ ] Beta
- [ ] Producción

## Documentación

- [Architecture](docs/ARCHITECTURE.md)
- [API](docs/API.md) (si aplica)
- [Deployment](docs/DEPLOYMENT.md)

## Desarrollo

```bash
# Clonar
git clone https://github.com/uSipipo-Team/{repo}.git
cd {repo}

# Instalar dependencias
uv sync --dev

# Configurar entorno
cp example.env .env

# Ejecutar tests
uv run pytest

# Ejecutar servicio
uv run python -m src
```

## Docker

```bash
# Build
docker build -t {image-name} .

# Ejecutar
docker run --env-file .env {image-name}
```

## License

MIT © uSipipo
```

### .gitignore Template
```
# Byte-compiled / optimized / DLL files
__pycache__/
*.py[cod]
*$py.class

# Virtual environments
.venv/
venv/
ENV/

# IDE
.idea/
.vscode/
*.swp
*.swo

# Testing
.pytest_cache/
.coverage
htmlcov/
*.egg-info/

# Environment
.env
.env.local

# Logs
logs/
*.log

# Temp
temp/
tmp/
.backups/

# Build
dist/
build/
```

### pyproject.toml Template (Python services)
```toml
[project]
name = "{package-name}"
version = "0.1.0"
description = "{description}"
requires-python = ">=3.13"
license = {text = "MIT"}
authors = [{name = "uSipipo Team", email = "dev@usipipo.com"}]

dependencies = [
    "usipipo-commons>=0.1.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.0.0",
    "mypy>=1.0.0",
    "ruff>=0.1.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
line-length = 100
target-version = "py313"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

### example.env Template
```bash
# Copiar a .env y configurar
SECRET_KEY=change_me_in_production
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/dbname
```

### Dockerfile Template (Python)
```dockerfile
FROM python:3.13-slim

WORKDIR /app

# Instalar uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

# Copiar dependencias
COPY pyproject.toml uv.lock ./

# Instalar dependencias
RUN uv sync --frozen --no-dev

# Copiar código
COPY src/ ./src/

# Exponer puerto (ajustar por servicio)
EXPOSE 8000

# Comando (ajustar por servicio)
CMD ["uv", "run", "python", "-m", "src"]
```

---

## 📚 Recursos

- [uv documentation](https://docs.astral.sh/uv/)
- [Pydantic v2 docs](https://docs.pydantic.dev/latest/)
- [PyPI publishing](https://pypi.org/help/)

---

## 🔄 Dependencias para Semana 2

La Semana 2 necesita:
- ✅ `usipipo-commons` publicado y accesible desde PyPI
- ⏳ Estructura base de `usipipo-backend` lista (Tarea 3.1)
- ✅ Tokens de PyPI configurados

---

## 📝 Notas

- ✅ Organización cambiada de `usipipo` a `uSipipo-Team`
- ✅ `usipipo-commons` publicado en PyPI.org (no GitHub Packages)
- ✅ Workflow CI/CD funcionando (tests + build + publish)
- Mantener consistencia en naming conventions entre todos los repos
- Usar siempre `uv` como package manager
- Todos los paquetes deben ser instalables vía `uv sync`

---

## 📖 Historial de Versiones

| Versión | Fecha | Cambios |
|---------|-------|---------|
| v0.2.0 | 2026-03-18 | Primera release en PyPI |
| v0.1.5 | 2026-03-18 | Fix publish con uv |
| v0.1.0 | 2026-03-18 | Release inicial |

---

**PRÓXIMA SESIÓN:** Continuar con Tarea 3.1 - Crear estructura base en los 5 repositorios restantes.
