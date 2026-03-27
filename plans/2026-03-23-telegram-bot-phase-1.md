# Telegram Bot Migration - Phase 1: Setup + Basic Commands

**Fecha:** 2026-03-23  
**Duración estimada:** 3-4 días  
**Estado:** Pendiente

---

## 🎯 Objetivo

Migrar la estructura base del bot y comandos básicos desde el monorepo hacia el nuevo repositorio `usipipo-telegram-bot`, integrando con la Backend API v0.9.0.

---

## 📋 Tareas

### Task 1: Analizar estructura del bot en monorepo
**Descripción:** Explorar el directorio `/home/mowgli/usipipobot/telegram_bot/` para entender la estructura completa de archivos, handlers, keyboards y mensajes.

**Pasos:**
1. Listar todos los archivos Python en el directorio del bot
2. Leer archivos principales: `main.py`, `common/`
3. Identificar dependencias y requisitos
4. Documentar estructura encontrada

**Verificación:**
- [ ] Lista completa de archivos identificados
- [ ] Dependencias documentadas

---

### Task 2: Configurar repositorio usipipo-telegram-bot
**Descripción:** Crear la estructura de directorios y archivos base para el nuevo repositorio del bot.

**Pasos:**
1. Crear estructura de directorios:
   - `src/bot/handlers/`
   - `src/bot/keyboards/`
   - `src/bot/messages/`
   - `src/infrastructure/`
   - `tests/`
   - `docs/`
2. Crear `pyproject.toml` con dependencias (aiogram 3.x, httpx, python-dotenv)
3. Crear `.env.example` con variables requeridas
4. Crear `README.md` con instrucciones de setup
5. Crear `.gitignore` para Python

**Verificación:**
- [ ] Estructura de directorios creada
- [ ] `pyproject.toml` válido
- [ ] `.env.example` completo

---

### Task 3: Implementar configuración e infraestructura
**Descripción:** Crear módulos de configuración y cliente HTTP para conectar con el backend.

**Pasos:**
1. Crear `src/infrastructure/config.py`:
   - Cargar variables de entorno (TELEGRAM_TOKEN, ADMIN_ID, BACKEND_URL)
   - Validar configuración requerida
2. Crear `src/infrastructure/api_client.py`:
   - Cliente HTTP asíncrono (httpx.AsyncClient)
   - Métodos base: GET, POST con autenticación JWT
   - Manejo de errores y retries
3. Crear `src/bot/__init__.py` y `src/infrastructure/__init__.py`

**Verificación:**
- [ ] `mypy src/` sin errores
- [ ] `ruff check src/` sin errores
- [ ] Tests unitarios de configuración (5+ tests)

---

### Task 4: Migrar comandos básicos
**Descripción:** Migrar handlers de comandos básicos (/start, /help, /menu) desde el monorepo.

**Pasos:**
1. Leer `features/basic_commands/handlers_basic.py` del monorepo
2. Leer `features/basic_commands/messages_basic.py` del monorepo
3. Crear `src/bot/messages/basic_messages.py` con mensajes adaptados
4. Crear `src/bot/keyboards/main_menu.py` con teclado principal
5. Crear `src/bot/handlers/basic_handlers.py` con:
   - `/start` - Mensaje de bienvenida
   - `/help` - Información de ayuda
   - `/menu` - Mostrar menú principal
6. Registrar handlers en `src/bot/handlers/__init__.py`

**Verificación:**
- [ ] Handlers migrados y funcionales
- [ ] Keyboards migrados
- [ ] Mensajes adaptados al nuevo contexto

---

### Task 5: Implementar bot principal con logging
**Descripción:** Crear el punto de entrada principal del bot con configuración de logging y manejo de errores.

**Pasos:**
1. Crear `src/bot/bot.py`:
   - Inicializar aiogram Dispatcher
   - Configurar logging (formato, nivel)
   - Registrar todos los handlers
   - Configurar webhook o polling
2. Crear `src/bot/error_handler.py`:
   - Manejador global de excepciones
   - Logging de errores
   - Notificación de errores críticos a admin
3. Crear `main.py` como entry point
4. Configurar logging:
   - Formato: `[%(asctime)s] [%(levelname)s] %(name)s - %(message)s`
   - Nivel desde variable de entorno (default: INFO)

**Verificación:**
- [ ] Bot inicia sin errores
- [ ] Logging configurado correctamente
- [ ] Error handler captura excepciones

---

### Task 6: Escribir tests unitarios
**Descripción:** Crear suite de tests unitarios para la Phase 1 (mínimo 15 tests).

**Pasos:**
1. Configurar `pytest` en `pyproject.toml`
2. Crear `tests/conftest.py` con fixtures:
   - async_client
   - mock_backend_api
   - test_bot_instance
3. Crear `tests/unit/test_config.py` (5 tests)
4. Crear `tests/unit/test_api_client.py` (5 tests)
5. Crear `tests/unit/test_basic_handlers.py` (5 tests)
6. Ejecutar tests y verificar cobertura

**Verificación:**
- [ ] 15+ tests creados
- [ ] Todos los tests pasan
- [ ] Cobertura > 70%

---

### Task 7: Crear PR y merge
**Descripción:** Subir cambios a GitHub, crear Pull Request y hacer merge a main.

**Pasos:**
1. Crear branch `feature/phase-1-basic-commands`
2. Commit inicial con toda la estructura
3. Push a GitHub
4. Crear Pull Request con descripción:
   - Objetivo de Phase 1
   - Lista de archivos creados
   - Resultados de tests
5. Revisar y aprobar PR
6. Merge a main
7. Crear release tag `v0.1.0`

**Verificación:**
- [ ] PR creado
- [ ] PR aprobado y mergeado
- [ ] Release v0.1.0 publicado

---

## 📁 Estructura Final Esperada

```
usipipo-telegram-bot/
├── src/
│   ├── bot/
│   │   ├── __init__.py
│   │   ├── bot.py
│   │   ├── error_handler.py
│   │   ├── handlers/
│   │   │   ├── __init__.py
│   │   │   └── basic_handlers.py
│   │   ├── keyboards/
│   │   │   ├── __init__.py
│   │   │   └── main_menu.py
│   │   └── messages/
│   │       ├── __init__.py
│   │       └── basic_messages.py
│   └── infrastructure/
│       ├── __init__.py
│       ├── api_client.py
│       └── config.py
├── tests/
│   ├── conftest.py
│   └── unit/
│       ├── test_config.py
│       ├── test_api_client.py
│       └── test_basic_handlers.py
├── docs/
│   └── setup.md
├── .env.example
├── .gitignore
├── main.py
├── pyproject.toml
└── README.md
```

---

## 🔧 Dependencias Requeridas

```toml
[project]
dependencies = [
    "aiogram>=3.4.0",
    "httpx>=0.25.0",
    "python-dotenv>=1.0.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
    "pytest-cov>=4.1.0",
    "mypy>=1.5.0",
    "ruff>=0.1.0",
    "types-python-dotenv",
]
```

---

## ✅ Criterios de Aceptación

- [ ] Bot inicia correctamente con `python main.py`
- [ ] Comandos `/start`, `/help`, `/menu` responden correctamente
- [ ] Logging muestra información estructurada
- [ ] Tests unitarios pasan (15+ tests)
- [ ] mypy y ruff sin errores
- [ ] PR mergeado a main
- [ ] Release v0.1.0 publicado

---

**Próxima Fase:** Phase 2 - VPN Key Management
