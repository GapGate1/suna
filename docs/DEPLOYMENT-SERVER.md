# Suna Server Deployment Guide

This guide explains the correct way to deploy and run the Suna platform on a production server.

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Environment Setup](#environment-setup)
- [Deployment Options](#deployment-options)
  - [A. Docker Deployment (Recommended)](#a-docker-deployment-recommended)
  - [B. Cloud Provider Deployment](#b-cloud-provider-deployment)
- [Reverse Proxy Setup](#reverse-proxy-setup)
- [SSL/HTTPS Configuration](#sslhttps-configuration)
- [Security](#security)
- [Monitoring and Logging](#monitoring-and-logging)
- [Backup and Maintenance](#backup-and-maintenance)
- [Troubleshooting](#troubleshooting)

## Overview

The Suna platform consists of the following components:
1. **Backend API** (FastAPI) - REST endpoints and request handling
2. **Backend Worker** (Dramatiq) - Background task processing
3. **Frontend** (Next.js) - User interface
4. **Redis** - Caching and session management
5. **Supabase** - Database and authentication
6. **Agent Sandbox** (Daytona) - Isolated execution environment for agents

## Requirements

### Server Requirements

- **Operating System**: Ubuntu 20.04+ or Debian 11+ (recommended)
- **CPU**: 2 cores minimum (4+ recommended)
- **RAM**: 4GB minimum (8GB+ recommended)
- **Storage**: 20GB free space minimum
- **Network**: Public IP address and domain name

### Required Software

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker & Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Install Git
sudo apt install git -y

# Re-login to activate Docker group
newgrp docker
```

### External Services Required

1. **Supabase Project** - Create at https://supabase.com/
2. **API Keys** (at least one LLM provider):
   - Anthropic API Key or
   - OpenAI API Key or
   - Other providers (Groq, OpenRouter, Gemini, etc.)
3. **Tavily API Key** - For web search
4. **Firecrawl API Key** - For web scraping
5. **Daytona API Key** - For agent execution environment
6. **RapidAPI Key** - For additional data services

## Environment Setup

### 1. Clone the Repository

```bash
cd /opt
sudo git clone https://github.com/kortix-ai/suna.git
sudo chown -R $USER:$USER suna
cd suna
```

### 2. Configure Environment Files

#### Backend (.env)

```bash
cd /opt/suna/backend
cp .env.example .env
nano .env
```

Recommended production settings:

```env
# Environment
ENV_MODE=production

# Supabase
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# Redis (use redis for Docker)
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=your-strong-redis-password
REDIS_SSL=false

# LLM provider (choose at least one)
ANTHROPIC_API_KEY=your-anthropic-key
# OPENAI_API_KEY=your-openai-key

# Web services
TAVILY_API_KEY=your-tavily-key
FIRECRAWL_API_KEY=your-firecrawl-key

# Daytona
DAYTONA_API_KEY=your-daytona-key
DAYTONA_SERVER_URL=https://app.daytona.io/api
DAYTONA_TARGET=us

# Data services
RAPID_API_KEY=your-rapidapi-key

# Encryption and security
MCP_CREDENTIAL_ENCRYPTION_KEY=your-generated-fernet-key
TRIGGER_WEBHOOK_SECRET=your-random-secret-string

# Webhook URL (replace with your domain)
WEBHOOK_BASE_URL=https://your-domain.com

# Optional: Monitoring
# LANGFUSE_PUBLIC_KEY=your-langfuse-public-key
# LANGFUSE_SECRET_KEY=your-langfuse-secret-key
```

To generate an encryption key:

```bash
python3 << 'PY'
from cryptography.fernet import Fernet
print(Fernet.generate_key().decode())
PY
```

#### Frontend (.env.local)

```bash
cd /opt/suna/frontend
cp .env.example .env.local
nano .env.local
```

```env
NEXT_PUBLIC_ENV_MODE=production
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
NEXT_PUBLIC_BACKEND_URL=https://api.your-domain.com
NEXT_PUBLIC_URL=https://your-domain.com
```

## Deployment Options

### A. Docker Deployment (Recommended)

#### 1. Create Production Docker Compose File

Create `docker-compose.prod.yaml`:

```bash
cd /opt/suna
nano docker-compose.prod.yaml
```

```yaml
services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    ports:
      - "127.0.0.1:6379:6379"
    volumes:
      - redis_data:/data
      - ./backend/services/docker/redis.conf:/usr/local/etc/redis/redis.conf:ro
    command: redis-server /usr/local/etc/redis/redis.conf --requirepass ${REDIS_PASSWORD} --save 60 1 --loglevel warning
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  backend:
    image: ghcr.io/suna-ai/suna-backend:latest
    platform: linux/amd64
    restart: unless-stopped
    ports:
      - "127.0.0.1:8000:8000"
    volumes:
      - ./backend/.env:/app/.env:ro
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - REDIS_SSL=False
    depends_on:
      redis:
        condition: service_healthy
      worker:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  worker:
    image: ghcr.io/suna-ai/suna-backend:latest
    platform: linux/amd64
    restart: unless-stopped
    command: uv run dramatiq --skip-logging --processes 4 --threads 4 run_agent_background
    volumes:
      - ./backend/.env:/app/.env:ro
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - REDIS_SSL=False
    depends_on:
      redis:
        condition: service_healthy

  frontend:
    image: ghcr.io/suna-ai/suna-frontend:latest
    restart: unless-stopped
    init: true
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      - NODE_ENV=production
    depends_on:
      - backend

volumes:
  redis_data:
```

**Note on Docker Image Versions:**
This configuration uses `latest` tags to stay consistent with the project's default setup. For production environments, consider pinning to specific version tags (e.g., `v1.2.3`) when they become available to prevent unexpected updates. You can check available versions at:
- Backend: https://github.com/kortix-ai/suna/pkgs/container/suna-backend
- Frontend: https://github.com/kortix-ai/suna/pkgs/container/suna-frontend

#### 2. Create Environment File for Docker Compose

```bash
nano .env
```

```env
REDIS_PASSWORD=your-strong-redis-password
```

#### 3. Start Services

```bash
# Pull latest images
docker compose -f docker-compose.prod.yaml pull

# Start services
docker compose -f docker-compose.prod.yaml up -d

# Check service status
docker compose -f docker-compose.prod.yaml ps

# View logs
docker compose -f docker-compose.prod.yaml logs -f
```

### B. Cloud Provider Deployment

#### AWS EC2

```bash
# 1. Create EC2 Instance (Ubuntu 22.04)
# Choose Instance type: t3.medium or larger
# Open ports: 80, 443, 22

# 2. Connect to server
ssh -i your-key.pem ubuntu@your-server-ip

# 3. Follow installation steps above
```

#### DigitalOcean Droplet

```bash
# 1. Create Droplet (Ubuntu 22.04)
# Choose size: Basic - 4GB RAM / 2 vCPUs

# 2. Connect to server
ssh root@your-droplet-ip

# 3. Follow installation steps above
```

#### Google Cloud Platform

```bash
# 1. Create Compute Engine Instance
# Type: e2-standard-2

# 2. Connect to server
gcloud compute ssh your-instance-name

# 3. Follow installation steps above
```

## Reverse Proxy Setup

### Using Nginx

#### 1. Install Nginx

```bash
sudo apt install nginx -y
```

#### 2. Create Configuration File

```bash
sudo nano /etc/nginx/sites-available/suna
```

```nginx
# Frontend
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Backend API
server {
    listen 80;
    server_name api.your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts for long-running requests
        proxy_read_timeout 300s;
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
    }
}
```

#### 3. Enable Configuration

```bash
sudo ln -s /etc/nginx/sites-available/suna /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Using Caddy (Easier Alternative with Automatic SSL)

```bash
# Install Caddy
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy

# Create Caddyfile
sudo nano /etc/caddy/Caddyfile
```

```caddyfile
# Frontend
your-domain.com {
    reverse_proxy localhost:3000
}

# Backend API
api.your-domain.com {
    reverse_proxy localhost:8000 {
        transport http {
            read_timeout 300s
        }
    }
}
```

```bash
# Restart Caddy
sudo systemctl restart caddy
```

## SSL/HTTPS Configuration

### Using Certbot (with Nginx)

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Obtain SSL certificate
sudo certbot --nginx -d your-domain.com -d api.your-domain.com

# Test auto-renewal
sudo certbot renew --dry-run
```

### Using Caddy
Caddy provides automatic SSL via Let's Encrypt - no additional setup needed!

## Security

### 1. Firewall Setup

```bash
# Enable UFW
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable

# Check status
sudo ufw status
```

### 2. Automatic Security Updates

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 3. SSH Hardening

```bash
# Disable password authentication
sudo nano /etc/ssh/sshd_config
# Change: PasswordAuthentication no

sudo systemctl restart sshd
```

### 4. Sensitive Environment Variables

- **Never** commit `.env` files to Git
- Use strong passwords for Redis
- Keep secure backups of API keys

## Monitoring and Logging

### View Docker Logs

```bash
# All services
docker compose -f docker-compose.prod.yaml logs -f

# Specific service
docker compose -f docker-compose.prod.yaml logs -f backend
docker compose -f docker-compose.prod.yaml logs -f worker
docker compose -f docker-compose.prod.yaml logs -f frontend
```

### Setup Uptime Monitoring

Recommended services:
- **UptimeRobot** - https://uptimerobot.com
- **Pingdom** - https://www.pingdom.com
- **StatusCake** - https://www.statuscake.com

### Resource Monitoring

```bash
# Using htop
sudo apt install htop -y
htop

# Docker monitoring
docker stats

# Disk space
df -h
```

## Backup and Maintenance

### 1. Database Backup

Supabase provides automatic backups. For manual backups:

```bash
# Backup from Supabase Dashboard
# Project Settings → Database → Backups
```

### 2. Redis Data Backup

```bash
# Create backup
docker compose -f docker-compose.prod.yaml exec redis redis-cli SAVE

# Copy data file
sudo cp /var/lib/docker/volumes/suna_redis_data/_data/dump.rdb ~/backups/
```

### 3. Configuration Backup

```bash
# Create backup
tar -czf suna-config-backup-$(date +%Y%m%d).tar.gz \
  /opt/suna/backend/.env \
  /opt/suna/frontend/.env.local \
  /opt/suna/docker-compose.prod.yaml
```

### 4. Schedule Backups

```bash
# Add cron job
crontab -e

# Daily backup at 2 AM
0 2 * * * /opt/suna/scripts/backup.sh
```

### 5. Updates

```bash
# Pull latest updates
cd /opt/suna
git pull

# Update Docker images
docker compose -f docker-compose.prod.yaml pull

# Restart services
docker compose -f docker-compose.prod.yaml up -d
```

## Troubleshooting

### Services Not Running

```bash
# Check service status
docker compose -f docker-compose.prod.yaml ps

# Restart specific service
docker compose -f docker-compose.prod.yaml restart backend

# Rebuild services
docker compose -f docker-compose.prod.yaml up -d --build
```

### Memory Issues

```bash
# Check memory usage
free -h

# Clean unused Docker images
docker system prune -a
```

### SSL Issues

```bash
# Renew SSL certificates
sudo certbot renew --force-renewal
sudo systemctl restart nginx
```

### Error Logs

```bash
# Nginx logs
sudo tail -f /var/log/nginx/error.log

# Specific service logs
docker compose -f docker-compose.prod.yaml logs --tail=100 backend
```

## Support

For assistance:
- Join Discord: https://discord.gg/Py6pCBUUPw
- Open GitHub Issue: https://github.com/kortix-ai/suna/issues
- Review Self-Hosting Guide: [SELF-HOSTING.md](./SELF-HOSTING.md)

---

**This guide was created to help you deploy Suna on a production server securely and efficiently.**
