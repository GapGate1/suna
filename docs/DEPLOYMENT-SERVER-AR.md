# دليل نشر سونا على السيرفر

هذا الدليل يشرح الطريقة الصحيحة لتشغيل منصة سونا (Suna) على سيرفر إنتاج (Production Server).

## جدول المحتويات

- [نظرة عامة](#نظرة-عامة)
- [المتطلبات](#المتطلبات)
- [إعداد البيئة](#إعداد-البيئة)
- [خيارات النشر](#خيارات-النشر)
  - [أ. النشر باستخدام Docker (موصى به)](#أ-النشر-باستخدام-docker-موصى-به)
  - [ب. النشر على الخدمات السحابية](#ب-النشر-على-الخدمات-السحابية)
- [إعداد Reverse Proxy](#إعداد-reverse-proxy)
- [إعداد SSL/HTTPS](#إعداد-sslhttps)
- [الأمان](#الأمان)
- [المراقبة والسجلات](#المراقبة-والسجلات)
- [النسخ الاحتياطي والصيانة](#النسخ-الاحتياطي-والصيانة)
- [حل المشاكل](#حل-المشاكل)

## نظرة عامة

منصة سونا تتكون من المكونات التالية:
1. **Backend API** (FastAPI) - واجهة برمجية للتعامل مع الطلبات
2. **Backend Worker** (Dramatiq) - معالجة المهام في الخلفية
3. **Frontend** (Next.js) - واجهة المستخدم
4. **Redis** - قاعدة بيانات للتخزين المؤقت والجلسات
5. **Supabase** - قاعدة البيانات والمصادقة
6. **Agent Sandbox** (Daytona) - بيئة تنفيذ معزولة للوكلاء

## المتطلبات

### متطلبات السيرفر

- **نظام التشغيل**: Ubuntu 20.04+ أو Debian 11+ (موصى به)
- **المعالج**: 2 CPU cores كحد أدنى (4+ موصى به)
- **الذاكرة**: 4GB RAM كحد أدنى (8GB+ موصى به)
- **التخزين**: 20GB مساحة فارغة كحد أدنى
- **الشبكة**: عنوان IP عام ونطاق (Domain)

### البرامج المطلوبة

```bash
# تحديث النظام
sudo apt update && sudo apt upgrade -y

# تثبيت Docker و Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# تثبيت Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# تثبيت Git
sudo apt install git -y

# إعادة تسجيل الدخول لتفعيل مجموعة Docker
newgrp docker
```

### الخدمات الخارجية المطلوبة

1. **Supabase Project** - إنشاء مشروع على https://supabase.com/
2. **مفاتيح API** (على الأقل واحد من مزودي LLM):
   - Anthropic API Key أو
   - OpenAI API Key أو
   - مزودين آخرين (Groq, OpenRouter, Gemini, إلخ)
3. **Tavily API Key** - للبحث على الويب
4. **Firecrawl API Key** - لاستخراج البيانات من المواقع
5. **Daytona API Key** - لبيئة تنفيذ الوكلاء
6. **RapidAPI Key** - لخدمات البيانات الإضافية

## إعداد البيئة

### 1. استنساخ المشروع

```bash
cd /opt
sudo git clone https://github.com/kortix-ai/suna.git
sudo chown -R $USER:$USER suna
cd suna
```

### 2. إعداد ملفات البيئة

#### Backend (.env)

```bash
cd /opt/suna/backend
cp .env.example .env
nano .env
```

إعدادات الإنتاج الموصى بها:

```env
# البيئة
ENV_MODE=production

# Supabase
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# Redis (استخدم redis للـ Docker)
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=your-strong-redis-password
REDIS_SSL=false

# مزود LLM (اختر واحد على الأقل)
ANTHROPIC_API_KEY=your-anthropic-key
# OPENAI_API_KEY=your-openai-key

# خدمات الويب
TAVILY_API_KEY=your-tavily-key
FIRECRAWL_API_KEY=your-firecrawl-key

# Daytona
DAYTONA_API_KEY=your-daytona-key
DAYTONA_SERVER_URL=https://app.daytona.io/api
DAYTONA_TARGET=us

# خدمات البيانات
RAPID_API_KEY=your-rapidapi-key

# التشفير والأمان
MCP_CREDENTIAL_ENCRYPTION_KEY=your-generated-fernet-key
TRIGGER_WEBHOOK_SECRET=your-random-secret-string

# عنوان الويب هوك (استبدل بنطاقك)
WEBHOOK_BASE_URL=https://your-domain.com

# اختياري: المراقبة
# LANGFUSE_PUBLIC_KEY=your-langfuse-public-key
# LANGFUSE_SECRET_KEY=your-langfuse-secret-key
```

لتوليد مفتاح التشفير:

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

## خيارات النشر

### أ. النشر باستخدام Docker (موصى به)

#### 1. إنشاء ملف docker-compose للإنتاج

قم بإنشاء `docker-compose.prod.yaml`:

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

#### 2. إنشاء ملف بيئة للـ Docker Compose

```bash
nano .env
```

```env
REDIS_PASSWORD=your-strong-redis-password
```

#### 3. تشغيل الخدمات

```bash
# سحب آخر الصور
docker compose -f docker-compose.prod.yaml pull

# تشغيل الخدمات
docker compose -f docker-compose.prod.yaml up -d

# التحقق من حالة الخدمات
docker compose -f docker-compose.prod.yaml ps

# عرض السجلات
docker compose -f docker-compose.prod.yaml logs -f
```

### ب. النشر على الخدمات السحابية

#### AWS EC2

```bash
# 1. إنشاء EC2 Instance (Ubuntu 22.04)
# اختر نوع Instance: t3.medium أو أكبر
# افتح المنافذ: 80, 443, 22

# 2. الاتصال بالسيرفر
ssh -i your-key.pem ubuntu@your-server-ip

# 3. اتبع خطوات التثبيت أعلاه
```

#### DigitalOcean Droplet

```bash
# 1. إنشاء Droplet (Ubuntu 22.04)
# اختر الحجم: Basic - 4GB RAM / 2 vCPUs

# 2. الاتصال بالسيرفر
ssh root@your-droplet-ip

# 3. اتبع خطوات التثبيت أعلاه
```

#### Google Cloud Platform

```bash
# 1. إنشاء Compute Engine Instance
# النوع: e2-standard-2

# 2. الاتصال بالسيرفر
gcloud compute ssh your-instance-name

# 3. اتبع خطوات التثبيت أعلاه
```

## إعداد Reverse Proxy

### استخدام Nginx

#### 1. تثبيت Nginx

```bash
sudo apt install nginx -y
```

#### 2. إنشاء ملف الإعداد

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

#### 3. تفعيل الإعداد

```bash
sudo ln -s /etc/nginx/sites-available/suna /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### استخدام Caddy (بديل أسهل مع SSL تلقائي)

```bash
# تثبيت Caddy
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy

# إنشاء Caddyfile
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
# إعادة تشغيل Caddy
sudo systemctl restart caddy
```

## إعداد SSL/HTTPS

### استخدام Certbot (مع Nginx)

```bash
# تثبيت Certbot
sudo apt install certbot python3-certbot-nginx -y

# الحصول على شهادة SSL
sudo certbot --nginx -d your-domain.com -d api.your-domain.com

# التجديد التلقائي
sudo certbot renew --dry-run
```

### استخدام Caddy
Caddy يوفر SSL تلقائياً عبر Let's Encrypt - لا حاجة لإعداد إضافي!

## الأمان

### 1. جدار الحماية (Firewall)

```bash
# تفعيل UFW
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable

# التحقق من الحالة
sudo ufw status
```

### 2. تحديثات الأمان التلقائية

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 3. حماية SSH

```bash
# تعطيل تسجيل الدخول بكلمة المرور
sudo nano /etc/ssh/sshd_config
# غيّر: PasswordAuthentication no

sudo systemctl restart sshd
```

### 4. المتغيرات البيئية الحساسة

- **لا تقم أبداً** بتضمين ملفات `.env` في Git
- استخدم كلمات مرور قوية لـ Redis
- احفظ نسخة احتياطية آمنة من مفاتيح API

## المراقبة والسجلات

### عرض سجلات Docker

```bash
# جميع الخدمات
docker compose -f docker-compose.prod.yaml logs -f

# خدمة محددة
docker compose -f docker-compose.prod.yaml logs -f backend
docker compose -f docker-compose.prod.yaml logs -f worker
docker compose -f docker-compose.prod.yaml logs -f frontend
```

### إعداد مراقبة Uptime

يُنصح باستخدام خدمات مثل:
- **UptimeRobot** - https://uptimerobot.com
- **Pingdom** - https://www.pingdom.com
- **StatusCake** - https://www.statuscake.com

### مراقبة الموارد

```bash
# استخدام htop
sudo apt install htop -y
htop

# مراقبة Docker
docker stats

# مساحة القرص
df -h
```

## النسخ الاحتياطي والصيانة

### 1. نسخ احتياطي لقاعدة البيانات

Supabase يوفر نسخ احتياطي تلقائي. للنسخ اليدوي:

```bash
# النسخ الاحتياطي من Supabase Dashboard
# Project Settings → Database → Backups
```

### 2. نسخ احتياطي لبيانات Redis

```bash
# إنشاء نسخة احتياطية
docker compose -f docker-compose.prod.yaml exec redis redis-cli SAVE

# نسخ ملف البيانات
sudo cp /var/lib/docker/volumes/suna_redis_data/_data/dump.rdb ~/backups/
```

### 3. نسخ احتياطي لملفات الإعداد

```bash
# إنشاء نسخة احتياطية
tar -czf suna-config-backup-$(date +%Y%m%d).tar.gz \
  /opt/suna/backend/.env \
  /opt/suna/frontend/.env.local \
  /opt/suna/docker-compose.prod.yaml
```

### 4. جدولة النسخ الاحتياطي

```bash
# إضافة مهمة cron
crontab -e

# نسخ احتياطي يومي عند الساعة 2 صباحاً
0 2 * * * /opt/suna/scripts/backup.sh
```

### 5. التحديثات

```bash
# سحب آخر التحديثات
cd /opt/suna
git pull

# تحديث صور Docker
docker compose -f docker-compose.prod.yaml pull

# إعادة تشغيل الخدمات
docker compose -f docker-compose.prod.yaml up -d
```

## حل المشاكل

### الخدمات لا تعمل

```bash
# التحقق من حالة الخدمات
docker compose -f docker-compose.prod.yaml ps

# إعادة تشغيل خدمة محددة
docker compose -f docker-compose.prod.yaml restart backend

# إعادة بناء الخدمات
docker compose -f docker-compose.prod.yaml up -d --build
```

### مشاكل الذاكرة

```bash
# التحقق من استخدام الذاكرة
free -h

# تنظيف صور Docker غير المستخدمة
docker system prune -a
```

### مشاكل SSL

```bash
# تجديد شهادات SSL
sudo certbot renew --force-renewal
sudo systemctl restart nginx
```

### سجلات الأخطاء

```bash
# سجلات Nginx
sudo tail -f /var/log/nginx/error.log

# سجلات خدمة محددة
docker compose -f docker-compose.prod.yaml logs --tail=100 backend
```

## الدعم

للحصول على المساعدة:
- انضم إلى Discord: https://discord.gg/Py6pCBUUPw
- افتح Issue على GitHub: https://github.com/kortix-ai/suna/issues
- راجع دليل Self-Hosting: [SELF-HOSTING.md](./SELF-HOSTING.md)

---

**تم إنشاء هذا الدليل لمساعدتك في نشر سونا على سيرفر الإنتاج بشكل آمن وفعال.**
