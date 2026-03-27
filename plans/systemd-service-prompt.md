# Systemd Service Profesional para Proyectos Python

> Prompt template para crear servicios systemd de nivel production-ready siguiendo el estándar establecido en los proyectos usipipo-backend, usipipo-telegram-bot y usipipo-landing.

---

## Prompt

```markdown
Crea un servicio systemd de nivel profesional para el proyecto `<NOMBRE_PROYECTO>` siguiendo el estándar establecido en los proyectos usipipo-backend, usipipo-telegram-bot y usipipo-landing.

## Contexto del Proyecto

- **Nombre:** `<NOMBRE_PROYECTO>`
- **Tipo de aplicación:** `<Flask/FastAPI/Bot de Telegram/CLI>`
- **Método de ejecución:** `<python -m src / flask run / uvicorn / custom>`
- **Puerto:** `<PUERTO>`
- **Dependencias de sistema:** `<postgresql/redis/none>`
- **Usuario del sistema:** `<USUARIO>` (ej: mowgli)
- **Ruta del proyecto:** `<RUTA_COMPLETA>` (ej: /home/mowgli/usipipo/<proyecto>)

## Requisitos del Servicio

### 1. Archivo de Servicio (`deploy/<nombre>.service`)

Crea un archivo con:
- ✅ Header documentado con instrucciones de instalación
- ✅ Placeholders marcados con `<PLACEHOLDER>` para configuración específica
- ✅ Tipo `exec` (no `simple`)
- ✅ Dependencias explícitas (`After=`, `Wants=`)
- ✅ Environment variables (`PATH`, `VIRTUAL_ENV`, `EnvironmentFile`)
- ✅ Graceful shutdown (`TimeoutStopSec=30`, `KillMode=mixed`, `KillSignal=SIGTERM`)
- ✅ Security hardening (`ProtectSystem=strict`, `PrivateTmp=true`, `NoNewPrivileges=true`, `ProtectHome=false`)
- ✅ Resource limits (`LimitNOFILE=65536`, `LimitNPROC=4096`)
- ✅ Logging centralizado (`StandardOutput=journal`, `StandardError=journal`, `SyslogIdentifier=<nombre>`)
- ✅ Restart policy (`Restart=always`, `RestartSec=10`)

### 2. README de Deploy (`deploy/README.md`)

Crea una guía completa con:
- 📋 Prerequisites
- 🚀 Quick Start (pasos numerados con comandos)
- ⚙️ Configuration (tabla de placeholders)
- 📖 Service Management Commands (start/stop/restart/status)
- 🔒 Security Hardening (tabla explicativa)
- 🔧 Troubleshooting (common issues + soluciones)
- 📊 Monitoring (comandos de verificación)
- 🔄 Updates and Maintenance (deploy/rollback)
- 📝 Architecture Notes (process model, graceful shutdown)
- 📚 References (enlaces a documentación oficial)

### 3. Estructura de Directorios

```
<NOMBRE_PROYECTO>/
├── deploy/
│   ├── <nombre>.service          # Template con placeholders
│   └── README.md                  # Guía completa
├── logs/                          # Crear si no existe
└── .env                           # Variables de entorno
```

## Ejemplo de Comando ExecStart

### Para Flask (python -m src):
```ini
ExecStart=<PATH>/.venv/bin/python -m src
```

### Para FastAPI (uvicorn):
```ini
ExecStart=<PATH>/.venv/bin/python -m uvicorn src.main:app \
    --host 0.0.0.0 \
    --port <PUERTO> \
    --workers 2 \
    --access-log
```

### Para Flask (flask run):
```ini
# Requiere FLASK_APP en .env o inline
Environment="FLASK_APP=src.infrastructure.web.app:app"
ExecStart=<PATH>/.venv/bin/flask run --host=0.0.0.0 --port=<PUERTO>
```

### Para producción con Gunicorn:
```ini
ExecStart=<PATH>/.venv/bin/gunicorn \
    --bind 0.0.0.0:<PUERTO> \
    --workers 2 \
    --worker-class sync \
    --timeout 30 \
    src.infrastructure.web.app:app
```

## Comandos de Instalación (documentar en README)

```bash
# 1. Copiar servicio
sudo cp deploy/<nombre>.service /etc/systemd/system/<nombre>.service

# 2. Editar placeholders
sudo nano /etc/systemd/system/<nombre>.service

# 3. Crear directorio logs
sudo mkdir -p <PATH>/logs
sudo chown <USER>:<GROUP> <PATH>/logs

# 4. Recargar e iniciar
sudo systemctl daemon-reload
sudo systemctl enable <nombre>
sudo systemctl start <nombre>

# 5. Verificar
sudo systemctl status <nombre>
```

## Verificación Final

El servicio debe:
- ✅ Estar activo y ejecutándose (`active (running)`)
- ✅ Habilitado para inicio automático (`enabled`)
- ✅ Logs accesibles vía journalctl
- ✅ Reiniciarse automáticamente en caso de fallo
- ✅ Shutdown graceful (sin conexiones dropped)

## Notas Adicionales

- No incluir credenciales hardcodeadas en el servicio (usar `.env`)
- El archivo `.service` en el repo debe tener placeholders
- El archivo instalado en `/etc/systemd/system/` tiene rutas reales
- Mantener consistencia con los otros servicios del ecosistema
```

---

## 🎯 Uso Rápido

Copia y pega el prompt anterior, reemplazando solo:

| Variable | Ejemplo |
|----------|---------|
| `<NOMBRE_PROYECTO>` | `usipipo-worker` |
| `<Tipo de aplicación>` | `FastAPI Background Worker` |
| `<Método de ejecución>` | `python -m src.worker` |
| `<PUERTO>` | `N/A` (si no usa puerto) |
| `<Dependencias de sistema>` | `redis, postgresql` |
| `<USUARIO>` | `mowgli` |
| `<RUTA_COMPLETA>` | `/home/mowgli/usipipo/usipipo-worker` |

---

## 📚 Referencias

- [systemd.exec(5) - Service configuration](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)
- [systemd.service(5) - Unit configuration](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
- [usipipo-backend/deploy/](../usipipo-backend/deploy/)
- [usipipo-telegram-bot/](../usipipo-telegram-bot/)
- [usipipo-landing/deploy/](../usipipo-landing/deploy/)

---

**Fecha de creación:** 2026-03-24  
**Autor:** uSipipo Team
