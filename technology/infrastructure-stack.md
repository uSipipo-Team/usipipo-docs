# 🏢 Infrastructure Stack - Infraestructura y Deployment

> Documento detallado de la infraestructura, deployment y operaciones del ecosistema uSipipo

---

## 📊 Visión General

La infraestructura de uSipipo está diseñada para ser **escalable, segura y de alta disponibilidad**, utilizando una combinación de servicios en la nube y servidores dedicados.

---

## 🏗️ Arquitectura de Infraestructura

### **Diagrama de Producción**

```
                                    Internet
                                        │
                                        ▼
                        ┌───────────────────────────────┐
                        │   Cloudflare / DNS            │
                        │   (DNS + DDoS Protection)     │
                        └───────────────┬───────────────┘
                                        │
                                        ▼
                        ┌───────────────────────────────┐
                        │   Caddy Reverse Proxy         │
                        │   usipipo.duckdns.org         │
                        │   TLS Automático              │
                        └───────────────┬───────────────┘
                                        │
                ┌───────────────────────┼───────────────────────┐
                │                       │                       │
                ▼                       ▼                       ▼
    ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
    │   Landing Page    │   │   Backend API     │   │   MiniApp Web     │
    │   Flask :5000     │   │   FastAPI :8000   │   │   Flask :5001     │
    │   systemd         │   │   systemd         │   │   systemd         │
    └───────────────────┘   └───────────────────┘   └───────────────────┘
                │                       │
                │                       │
                ▼                       ▼
    ┌───────────────────┐   ┌───────────────────────────────────────────┐
    │   Telegram Bot    │   │           Data Layer                      │
    │   PTB Polling     │   │  ┌─────────────┐  ┌─────────────┐        │
    │   systemd         │   │  │ PostgreSQL  │  │    Redis    │        │
    └───────────────────┘   │  │   :5432     │  │   :6379     │        │
                            │  │  systemd    │  │  systemd    │        │
                            │  └─────────────┘  └─────────────┘        │
                            └───────────────────────────────────────────┘
                                        │
                                        ▼
                            ┌───────────────────────────────┐
                            │       VPN Servers             │
                            │  ┌──────────┐ ┌──────────┐   │
                            │  │WireGuard │ │  Outline │   │
                            │  │  :51820  │ │  :8080   │   │
                            │  └──────────┘ └──────────┘   │
                            └───────────────────────────────┘
```

---

## 🖥️ Servidores

### **Especificaciones de Producción**

| Servidor | CPU | RAM | Disco | SO | Propósito |
|----------|-----|-----|-------|----|-----------|
| **App Server** | 4 vCPU | 8 GB | 80 GB SSD | Ubuntu 22.04 | Backend, Landing, Bot |
| **DB Server** | 4 vCPU | 8 GB | 100 GB SSD | Ubuntu 22.04 | PostgreSQL |
| **Cache Server** | 2 vCPU | 4 GB | - | Ubuntu 22.04 | Redis |
| **VPN Server 1** | 2 vCPU | 4 GB | 40 GB SSD | Ubuntu 22.04 | WireGuard |
| **VPN Server 2** | 2 vCPU | 4 GB | 40 GB SSD | Ubuntu 22.04 | Outline |

---

### **Configuración de Red**

```
Red Principal: 10.0.0.0/24

┌─────────────────────────────────────────────────────────┐
│                    Red Privada                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  10.0.0.1  - VPN Gateway (WireGuard)                   │
│  10.0.0.2  - App Server                                │
│  10.0.0.3  - DB Server (PostgreSQL)                    │
│  10.0.0.4  - Cache Server (Redis)                      │
│  10.0.0.5  - VPN Server 2 (Outline)                    │
│                                                         │
│  10.0.0.100-200 - VPN Clients (dinámico)               │
│                                                         │
└─────────────────────────────────────────────────────────┘

Puertos expuestos:
- 443/tcp  - HTTPS (Caddy)
- 51820/udp - WireGuard VPN
- 8080/tcp - Outline API (solo interno)
```

---

## 🔧 Configuración de Servicios

### **PostgreSQL**

```ini
# /etc/postgresql/15/main/postgresql.conf

# Conexiones
listen_addresses = '*'
port = 5432
max_connections = 200

# Memoria
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 64MB
maintenance_work_mem = 512MB

# WAL
wal_level = replica
max_wal_senders = 3
wal_keep_size = 64MB

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on

# Autovacuum
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 1min
autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 50
```

---

### **Redis**

```ini
# /etc/redis/redis.conf

bind 127.0.0.1
port 6379
timeout 0
tcp-keepalive 300
daemonize yes
supervised systemd
pidfile /var/run/redis/redis-server.pid
loglevel notice
logfile /var/log/redis/redis-server.log
databases 16

# Snapshotting
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis

# Memoria
maxmemory 2gb
maxmemory-policy allkeys-lru

# Append only mode
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Seguridad
requirepass <strong_password>
protected-mode yes

# Lazy freeing
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes
```

---

### **Caddy (Reverse Proxy)**

```
# /etc/caddy/Caddyfile

usipipo.duckdns.org {
    # Logging
    log {
        output file /var/log/caddy/access.log
        format json
    }
    
    # TLS
    tls dev@usipipo.com
    
    # Security headers
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        X-XSS-Protection "1; mode=block"
        Referrer-Policy "strict-origin-when-cross-origin"
        Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline' fonts.googleapis.com; font-src 'self' fonts.gstatic.com"
    }
    
    # Rate limiting
    @ratelimit {
        not path /api/v1/webhooks/*
    }
    respond @ratelimit {
        # Custom rate limit handling
    }
    
    # Landing Page
    @landing {
        not path /miniapp/*
        not path /api/*
    }
    reverse_proxy @landing localhost:5000 {
        health_uri /health
        health_interval 30s
        health_timeout 5s
    }
    
    # MiniApp Web
    @miniapp path /miniapp/*
    reverse_proxy @miniapp localhost:5001 {
        health_uri /health
        health_interval 30s
        health_timeout 5s
    }
    
    # Backend API
    @api path /api/*
    reverse_proxy @api localhost:8000 {
        health_uri /health
        health_interval 10s
        health_timeout 5s
        flush_interval -1
    }
}

# HTTP redirect
http://usipipo.duckdns.org {
    redir https://usipipo.duckdns.org{uri} permanent
}
```

---

## 📦 systemd Services

### **Backend API**

```ini
# /etc/systemd/system/usipipo-backend.service

[Unit]
Description=uSipipo Backend API
Documentation=https://github.com/uSipipo-Team/usipipo-docs
After=network.target postgresql.service redis.service
Wants=postgresql.service redis.service

[Service]
Type=notify
User=usipipo
Group=usipipo
WorkingDirectory=/opt/usipipo/usipipo-backend
EnvironmentFile=/opt/usipipo/.env
ExecStart=/opt/usipipo/.venv/bin/python -m src
ExecReload=/bin/kill -s HUP $MAINPID

# Restart
Restart=always
RestartSec=10
SuccessExitStatus=143
TimeoutStopSec=30

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/usipipo/usipipo-backend/logs
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
RestrictNamespaces=true
RestrictRealtime=true
RestrictSUIDSGID=true
MemoryDenyWriteExecute=true
LockPersonality=true

# Capabilities
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=

# Resources
LimitNOFILE=65536
LimitNPROC=4096
MemoryMax=2G
CPUQuota=200%

[Install]
WantedBy=multi-user.target
```

---

### **Telegram Bot**

```ini
# /etc/systemd/system/usipipo-bot.service

[Unit]
Description=uSipipo Telegram Bot
Documentation=https://github.com/uSipipo-Team/usipipo-docs
After=network.target redis.service
Wants=redis.service

[Service]
Type=simple
User=usipipo
Group=usipipo
WorkingDirectory=/opt/usipipo/usipipo-telegram-bot
EnvironmentFile=/opt/usipipo/.env
ExecStart=/opt/usipipo/.venv/bin/python -m src

# Restart
Restart=always
RestartSec=10
TimeoutStopSec=30

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/usipipo/usipipo-telegram-bot/logs

# Resources
LimitNOFILE=65536
LimitNPROC=4096
MemoryMax=512M

[Install]
WantedBy=multi-user.target
```

---

### **Landing Page**

```ini
# /etc/systemd/system/usipipo-landing.service

[Unit]
Description=uSipipo Landing Page
Documentation=https://github.com/uSipipo-Team/usipipo-docs
After=network.target

[Service]
Type=notify
User=usipipo
Group=usipipo
WorkingDirectory=/opt/usipipo/usipipo-landing
EnvironmentFile=/opt/usipipo/.env
ExecStart=/opt/usipipo/.venv/bin/python -m src

# Restart
Restart=always
RestartSec=10
TimeoutStopSec=30

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true

# Resources
LimitNOFILE=65536
MemoryMax=512M

[Install]
WantedBy=multi-user.target
```

---

### **WireGuard VPN**

```ini
# /etc/systemd/system/wg-quick@wg0.service
# (Provisto por paquete wireguard-tools)

[Unit]
Description=WireGuard via wg-quick(8) for wg0
After=network-online.target nss-lookup.target
Wants=network-online.target nss-lookup.target
Documentation=man:wg-quick(8)
Documentation=man:wg(8)

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/usr/bin/ip link add wg0 type wireguard
ExecStartPre=/usr/bin/wg setconf wg0 /etc/wireguard/wg0.conf
ExecStart=/usr/bin/ip link set dev wg0 up
ExecStop=/usr/bin/ip link del dev wg0

[Install]
WantedBy=multi-user.target
```

---

## 🔐 Seguridad

### **Firewall (UFW)**

```bash
# /etc/ufw/rules

# Default policies
DEFAULT_INPUT_POLICY="DROP"
DEFAULT_OUTPUT_POLICY="ACCEPT"
DEFAULT_FORWARD_POLICY="DROP"

# Allow rules
-A ufw-before-input -p tcp --dport 22 -j ACCEPT        # SSH (limitar por IP)
-A ufw-before-input -p tcp --dport 443 -j ACCEPT        # HTTPS
-A ufw-before-input -p udp --dport 51820 -j ACCEPT      # WireGuard
-A ufw-before-input -p tcp --dport 5432 -j ACCEPT       # PostgreSQL (solo internal)
-A ufw-before-input -p tcp --dport 6379 -j ACCEPT       # Redis (solo internal)

# Rate limiting
-A ufw-before-input -p tcp --dport 22 -m state --state NEW -m recent --set
-A ufw-before-input -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
```

**Comandos:**
```bash
# Habilitar firewall
ufw enable

# Ver estado
ufw status verbose

# Allow SSH (solo desde IPs específicas)
ufw allow from 192.168.1.0/24 to any port 22

# Allow HTTPS
ufw allow 443/tcp

# Allow WireGuard
ufw allow 51820/udp
```

---

### **SSH Hardening**

```ini
# /etc/ssh/sshd_config

# Autenticación
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes

# Root login
PermitRootLogin prohibit-password
AllowUsers usipipo

# Puertos
Port 22
# O cambiar a puerto no estándar
# Port 2222

# Rate limiting
MaxAuthTries 3
MaxStartups 10:30:60
LoginGraceTime 60

# Logging
SyslogFacility AUTH
LogLevel VERBOSE

# Forwarding
AllowAgentForwarding no
AllowTcpForwarding no
X11Forwarding no

# Misc
Protocol 2
ClientAliveInterval 300
ClientAliveCountMax 2
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server

# Keys
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
```

---

### **Fail2Ban**

```ini
# /etc/fail2ban/jail.local

[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5
backend = auto
usedns = warn
logencoding = auto
enabled = false
mode = normal

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400

[nginx-limit-req]
enabled = true
port = http,https
filter = nginx-limit-req
logpath = /var/log/nginx/error.log
maxretry = 10
bantime = 3600

[ufw]
enabled = true
filter = ufw
logpath = /var/log/ufw.log
maxretry = 3
bantime = 86400
```

---

## 📊 Monitoring

### **Prometheus Configuration**

```yaml
# /etc/prometheus/prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: []

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'backend'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics'

  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['localhost:9121']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```

---

### **Grafana Dashboards**

**Dashboards principales:**

1. **Backend Overview**
   - Request rate
   - Latency (p50, p95, p99)
   - Error rate
   - Active connections

2. **Database Metrics**
   - Connections
   - Query rate
   - Cache hit ratio
   - Replication lag

3. **VPN Metrics**
   - Active connections
   - Data transfer
   - Server health
   - Key usage

4. **Business Metrics**
   - New users
   - Payments
   - Active subscriptions
   - Revenue

---

## 🔄 Backup y Recovery

### **PostgreSQL Backup**

```bash
#!/bin/bash
# /opt/usipipo/scripts/backup-postgres.sh

set -e

BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
DB_NAME="usipipo_prod"
DB_USER="usipipo"

# Create backup
pg_dump -U $DB_USER -h localhost -F c -b -v $DB_NAME > $BACKUP_DIR/$DB_NAME-$DATE.dump

# Compress
gzip $BACKUP_DIR/$DB_NAME-$DATE.dump

# Delete old backups (keep 7 days)
find $BACKUP_DIR -name "*.dump.gz" -mtime +7 -delete

# Upload to S3 (optional)
# aws s3 cp $BACKUP_DIR/$DB_NAME-$DATE.dump.gz s3://usipipo-backups/postgresql/
```

**Cron:**
```bash
# /etc/cron.d/postgresql-backup
0 2 * * * usipipo /opt/usipipo/scripts/backup-postgres.sh >> /var/log/postgresql-backup.log 2>&1
```

---

### **Redis Backup**

```bash
#!/bin/bash
# /opt/usipipo/scripts/backup-redis.sh

set -e

BACKUP_DIR="/var/backups/redis"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
REDIS_HOST="localhost"
REDIS_PORT="6379"

# Trigger BGSAVE
redis-cli -h $REDIS_HOST -p $REDIS_PORT BGSAVE

# Wait for save
sleep 5

# Copy dump file
cp /var/lib/redis/dump.rdb $BACKUP_DIR/dump-$DATE.rdb

# Compress
gzip $BACKUP_DIR/dump-$DATE.rdb

# Delete old backups (keep 3 days)
find $BACKUP_DIR -name "*.rdb.gz" -mtime +3 -delete
```

---

### **Recovery Procedures**

**PostgreSQL Restore:**
```bash
# Stop application
systemctl stop usipipo-backend

# Restore
gunzip /var/backups/postgresql/usipipo_prod-2026-03-27_02-00-00.dump.gz
pg_restore -U usipipo -h localhost -d usipipo_prod /var/backups/postgresql/usipipo_prod-2026-03-27_02-00-00.dump

# Start application
systemctl start usipipo-backend
```

**Redis Restore:**
```bash
# Stop Redis
systemctl stop redis

# Restore dump
gunzip /var/backups/redis/dump-2026-03-27_02-00-00.rdb.gz
cp /var/backups/redis/dump-2026-03-27_02-00-00.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb

# Start Redis
systemctl start redis
```

---

## 🚀 CI/CD

### **GitHub Actions Workflow**

```yaml
# .github/workflows/deploy.yml

name: Deploy to Production

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/usipipo/usipipo-backend
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}
            docker-compose down
            docker-compose up -d
            docker system prune -f
```

---

## 📈 Escalamiento

### **Horizontal Scaling**

```
                    Load Balancer (HAProxy)
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Backend  │    │ Backend  │    │ Backend  │
    │   :8000  │    │   :8000  │    │   :8000  │
    └────┬─────┘    └────┬─────┘    └────┬─────┘
         │               │               │
         └───────────────┼───────────────┘
                         │
                         ▼
                  ┌─────────────┐
                  │  PostgreSQL │
                  │  (Primary)  │
                  └──────┬──────┘
                         │
                         ▼
                  ┌─────────────┐
                  │  PostgreSQL │
                  │  (Replica)  │
                  └─────────────┘
```

---

### **Auto-scaling Rules**

| Métrica | Threshold | Acción |
|---------|-----------|--------|
| CPU > 80% | 5 min | Agregar instancia |
| CPU < 30% | 10 min | Remover instancia |
| Memoria > 85% | 5 min | Agregar instancia |
| Request rate > 1000/s | 2 min | Agregar instancia |

---

## 📚 Recursos Relacionados

- [Stack Overview](stack-overview.md)
- [Backend Stack](backend-stack.md)
- [Security Requirements](../apis/webhooks-integration.md)

---

**Última actualización:** 2026-03-27
