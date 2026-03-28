# uSipipo Support Bot - Deployment Guide

**Date:** 2026-03-28  
**Version:** 1.0  

---

## 📋 Prerequisites

- Ubuntu 20.04+ or Debian 11+
- Docker 20.10+ (optional)
- Python 3.13
- Redis 6.0+
- systemd
- Git

---

## 🚀 Installation Options

### **Option 1: Native Installation (Recommended)**

#### **1. Install Dependencies**

```bash
# Install Python 3.13
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.13 python3.13-venv python3.13-dev

# Install uv package manager
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install Redis
sudo apt install redis-server
sudo systemctl enable redis
sudo systemctl start redis
```

#### **2. Clone Repository**

```bash
# Create directory
sudo mkdir -p /opt/usipipo-support-bot
sudo chown $USER:$USER /opt/usipipo-support-bot

# Clone repository
cd /opt/usipipo-support-bot
git clone https://github.com/uSipipo-Team/usipipo-support-bot.git .
```

#### **3. Configure Environment**

```bash
# Copy environment file
cp example.env .env

# Edit .env with your values
nano .env
```

**Environment Variables:**
```bash
# Bot Configuration
BOT_TOKEN=your_telegram_bot_token_support
BOT_USERNAME=uSipipoSupport_Bot

# Backend API
BACKEND_URL=https://api.usipipo.com
API_PREFIX=/api/v1

# Redis
REDIS_URL=redis://localhost:6379/1

# Logging
LOG_LEVEL=INFO
LOG_FILE=/var/log/usipipo/support-bot.log
```

#### **4. Install Dependencies**

```bash
# Sync dependencies
uv sync --frozen --no-dev

# Install pre-commit hooks (optional, for development)
uv run pre-commit install
```

#### **5. Create Log Directory**

```bash
sudo mkdir -p /var/log/usipipo
sudo chown $USER:$USER /var/log/usipipo
```

#### **6. Install systemd Service**

```bash
# Copy service file
sudo cp usipipo-support-bot.service /etc/systemd/system/

# Reload systemd
sudo systemctl daemon-reload

# Enable service (start on boot)
sudo systemctl enable usipipo-support-bot

# Start service
sudo systemctl start usipipo-support-bot

# Check status
sudo systemctl status usipipo-support-bot
```

#### **7. Verify Installation**

```bash
# Check logs
sudo journalctl -u usipipo-support-bot -f

# Test bot
# Send /start to @uSipipoSupport_Bot in Telegram
```

---

### **Option 2: Docker Installation**

#### **1. Install Docker**

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Install Docker Compose
sudo apt install docker-compose-plugin
```

#### **2. Build Image**

```bash
cd /opt/usipipo-support-bot
docker build -t usipipo-support-bot:latest .
```

#### **3. Run Container**

```bash
# Create .env file
cp example.env .env
nano .env

# Run container
docker run -d \
  --name usipipo-support-bot \
  --restart unless-stopped \
  --env-file .env \
  --network host \
  usipipo-support-bot:latest
```

#### **4. Verify**

```bash
# Check logs
docker logs -f usipipo-support-bot

# Check status
docker ps | grep usipipo-support-bot
```

---

### **Option 3: Docker Compose**

#### **1. Create docker-compose.yml**

```yaml
version: '3.8'

services:
  support-bot:
    build: .
    container_name: usipipo-support-bot
    restart: unless-stopped
    env_file: .env
    networks:
      - usipipo
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    container_name: usipipo-redis
    restart: unless-stopped
    networks:
      - usipipo
    volumes:
      - redis-data:/data

networks:
  usipipo:
    driver: bridge

volumes:
  redis-data:
```

#### **2. Start Services**

```bash
docker-compose up -d
```

#### **3. Verify**

```bash
docker-compose ps
docker-compose logs -f support-bot
```

---

## 🔧 Configuration

### **Environment Variables**

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `BOT_TOKEN` | ✅ | - | Telegram bot token |
| `BOT_USERNAME` | ✅ | - | Bot username (e.g., `uSipipoSupport_Bot`) |
| `BACKEND_URL` | ✅ | - | Backend API URL |
| `API_PREFIX` | ✅ | `/api/v1` | API path prefix |
| `REDIS_URL` | ✅ | `redis://localhost:6379/1` | Redis connection URL |
| `LOG_LEVEL` | ❌ | `INFO` | Logging level |
| `LOG_FILE` | ❌ | - | Log file path |

### **Telegram Bot Setup**

1. Open Telegram and search for [@BotFather](https://t.me/BotFather)
2. Send `/newbot` command
3. Follow instructions:
   - Choose bot name: `uSipipo Support`
   - Choose username: `uSipipoSupport_Bot`
4. Copy the bot token
5. Add token to `.env` file

### **Bot Commands Setup**

In BotFather, send `/setcommands` and configure:

```
start - Iniciar bot
help - Mostrar ayuda
tickets - Ver mis tickets
nuevoticket - Crear nuevo ticket
```

---

## 🔄 Update Procedure

### **Native Installation**

```bash
# Go to installation directory
cd /opt/usipipo-support-bot

# Pull latest changes
git pull origin main

# Update dependencies
uv sync --frozen --no-dev

# Restart service
sudo systemctl restart usipipo-support-bot

# Check status
sudo systemctl status usipipo-support-bot
```

### **Docker Installation**

```bash
# Pull latest changes
cd /opt/usipipo-support-bot
git pull origin main

# Rebuild image
docker build -t usipipo-support-bot:latest .

# Restart container
docker restart usipipo-support-bot

# Check logs
docker logs -f usipipo-support-bot
```

---

## 🐛 Troubleshooting

### **Bot Doesn't Respond**

```bash
# Check service status
sudo systemctl status usipipo-support-bot

# Check logs
sudo journalctl -u usipipo-support-bot -n 50

# Check Redis connection
redis-cli ping

# Check backend connectivity
curl https://api.usipipo.com/health
```

### **Authentication Errors**

```bash
# Check BOT_TOKEN in .env
cat .env | grep BOT_TOKEN

# Verify token with BotFather
# Send /mybots in Telegram → Select bot → Check token
```

### **Redis Connection Issues**

```bash
# Check Redis status
sudo systemctl status redis

# Test Redis connection
redis-cli ping

# Check Redis logs
sudo tail -f /var/log/redis/redis-server.log
```

### **Permission Issues**

```bash
# Fix log directory permissions
sudo chown -R $USER:$USER /var/log/usipipo

# Fix installation directory permissions
sudo chown -R $USER:$USER /opt/usipipo-support-bot
```

---

## 📊 Monitoring

### **Check Service Status**

```bash
# systemd status
sudo systemctl status usipipo-support-bot

# Real-time logs
sudo journalctl -u usipipo-support-bot -f

# Last 100 lines
sudo journalctl -u usipipo-support-bot -n 100
```

### **Resource Usage**

```bash
# Memory and CPU usage
systemd-cgtop

# Process status
ps aux | grep support-bot
```

### **Redis Monitoring**

```bash
# Redis info
redis-cli info

# Memory usage
redis-cli info memory

# Connected clients
redis-cli client list
```

---

## 🔒 Security Best Practices

1. **Protect .env file:**
   ```bash
   chmod 600 /opt/usipipo-support-bot/.env
   ```

2. **Use firewall:**
   ```bash
   sudo ufw enable
   sudo ufw allow out 443  # HTTPS outbound
   sudo ufw allow out 6379 # Redis (if remote)
   ```

3. **Enable automatic updates:**
   ```bash
   sudo apt install unattended-upgrades
   ```

4. **Monitor failed login attempts:**
   ```bash
   sudo grep "Failed" /var/log/auth.log
   ```

---

## 📝 Maintenance

### **Backup Configuration**

```bash
# Backup .env file
cp /opt/usipipo-support-bot/.env ~/backup-support-bot-env-$(date +%Y%m%d)

# Backup systemd service
sudo cp /etc/systemd/system/usipipo-support-bot.service ~/backup/
```

### **Log Rotation**

Create `/etc/logrotate.d/usipipo-support-bot`:

```
/var/log/usipipo/support-bot.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 $USER $USER
}
```

---

## 🆘 Support

For issues or questions:
- Check logs: `sudo journalctl -u usipipo-support-bot`
- Review documentation: `/home/mowgli/usipipo/usipipo-docs/support-bot/`
- Contact team: usipipo@gmail.com

---

**Last Updated:** 2026-03-28  
**Version:** 1.0
