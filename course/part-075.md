# Part 075: Django Deployment and Production
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- จัดการ Settings ด้วย django-environ หรือ python-decouple ได้
- ตั้งค่า Production Settings ที่ปลอดภัย (DEBUG=False, ALLOWED_HOSTS, PostgreSQL)
- ตั้งค่า Gunicorn สำหรับ Production Server
- ใช้ WhiteNoise สำหรับ Static Files
- ตั้งค่า Nginx เป็น Reverse Proxy
- เชื่อมต่อ PostgreSQL ด้วย psycopg2
- จัดการ Environment Variables อย่างปลอดภัย
- ตั้งค่า HTTPS/SSL ด้วย Let's Encrypt
- ผ่าน Deployment Checklist ของ Django
- Deploy ไปยัง Heroku, Railway, หรือ Render
- สร้าง Complete Production Docker Compose

---

## 1. ทำไมต้องแยก Settings สำหรับ Production?

```
Development Settings:
✅ DEBUG = True (แสดง Error details)
✅ SQLite (ง่าย ไม่ต้องตั้งค่า)
✅ Secret Key ง่ายๆ (ไม่สำคัญ)
✅ Email ส่งไปยัง Console
✅ Static files serve โดย Django
✅ CORS เปิดกว้าง

Production Settings:
❌ DEBUG = False (ซ่อน Error details จาก Users)
✅ PostgreSQL (เสถียร, Scale ได้)
✅ Secret Key ยาว Random (สำคัญมาก!)
✅ Email จริงด้วย SMTP
✅ Static files serve โดย Nginx/CDN
✅ CORS ตั้งค่าเฉพาะ domains ที่อนุญาต
```

---

## 2. django-environ สำหรับจัดการ Settings

```bash
# ติดตั้ง
pip install django-environ
```

```python
# ไฟล์ .env (อยู่ใน Root directory เดียวกับ manage.py)
# ห้าม Commit ไฟล์นี้ไปยัง Git!
DEBUG=True
SECRET_KEY=your-very-secret-key-here-change-in-production
DATABASE_URL=postgres://user:password@localhost:5432/mydb
ALLOWED_HOSTS=localhost,127.0.0.1
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_HOST_USER=myapp@gmail.com
EMAIL_HOST_PASSWORD=my-app-password
EMAIL_USE_TLS=True
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_STORAGE_BUCKET_NAME=my-app-bucket
REDIS_URL=redis://localhost:6379/0
SENTRY_DSN=https://xxx@sentry.io/123
```

```python
# myproject/settings/base.py
import environ
import os
from pathlib import Path

# สร้าง environ instance
env = environ.Env(
    # กำหนด Default values และ Type Casting
    DEBUG=(bool, False),
    ALLOWED_HOSTS=(list, []),
    EMAIL_PORT=(int, 587),
    EMAIL_USE_TLS=(bool, True),
)

# อ่านไฟล์ .env
BASE_DIR = Path(__file__).resolve().parent.parent
environ.Env.read_env(BASE_DIR / '.env')

# ============================================================
# Core Settings
# ============================================================

# ดึงค่าจาก Environment Variable
SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')
ALLOWED_HOSTS = env('ALLOWED_HOSTS')

# Application definition
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # Third-party
    'rest_framework',
    'corsheaders',
    'whitenoise.runserver_nostatic',  # สำหรับ Development
    # Local apps
    'myapp',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # ต้องอยู่หลัง SecurityMiddleware
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'myproject.urls'

# ============================================================
# Database
# ============================================================

# django-environ อ่าน DATABASE_URL อัตโนมัติ
DATABASES = {
    'default': env.db()
    # ตัวอย่าง DATABASE_URL:
    # sqlite:///db.sqlite3
    # postgres://user:pass@localhost:5432/dbname
    # mysql://user:pass@localhost:3306/dbname
}

# ============================================================
# Email
# ============================================================

EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = env('EMAIL_HOST', default='smtp.gmail.com')
EMAIL_PORT = env('EMAIL_PORT', default=587)
EMAIL_HOST_USER = env('EMAIL_HOST_USER', default='')
EMAIL_HOST_PASSWORD = env('EMAIL_HOST_PASSWORD', default='')
EMAIL_USE_TLS = env('EMAIL_USE_TLS', default=True)
DEFAULT_FROM_EMAIL = env('DEFAULT_FROM_EMAIL', default='noreply@example.com')

# ============================================================
# Static & Media Files
# ============================================================

STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'  # สำหรับ collectstatic
STATICFILES_DIRS = [BASE_DIR / 'static']

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

# WhiteNoise Static Files Compression
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

# ============================================================
# Redis Cache
# ============================================================

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': env('REDIS_URL', default='redis://localhost:6379/0'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}

# ============================================================
# Celery
# ============================================================

CELERY_BROKER_URL = env('REDIS_URL', default='redis://localhost:6379/0')
CELERY_RESULT_BACKEND = env('REDIS_URL', default='redis://localhost:6379/0')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
```

---

## 3. python-decouple (ทางเลือก)

```bash
pip install python-decouple
```

```python
# myproject/settings/base.py (ใช้ python-decouple)
from decouple import config, Csv

# ดึงค่าจาก .env หรือ Environment Variables
SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='localhost', cast=Csv())

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME', default='mydb'),
        'USER': config('DB_USER', default='postgres'),
        'PASSWORD': config('DB_PASSWORD', default=''),
        'HOST': config('DB_HOST', default='localhost'),
        'PORT': config('DB_PORT', default='5432'),
    }
}

# Email
EMAIL_HOST = config('EMAIL_HOST', default='smtp.gmail.com')
EMAIL_PORT = config('EMAIL_PORT', default=587, cast=int)
EMAIL_HOST_USER = config('EMAIL_HOST_USER', default='')
EMAIL_HOST_PASSWORD = config('EMAIL_HOST_PASSWORD', default='')
EMAIL_USE_TLS = config('EMAIL_USE_TLS', default=True, cast=bool)

# AWS S3
AWS_ACCESS_KEY_ID = config('AWS_ACCESS_KEY_ID', default='')
AWS_SECRET_ACCESS_KEY = config('AWS_SECRET_ACCESS_KEY', default='')
AWS_STORAGE_BUCKET_NAME = config('AWS_STORAGE_BUCKET_NAME', default='')
```

---

## 4. Production Settings.py

```python
# myproject/settings/production.py
"""
Production Settings - ใช้เฉพาะใน Production Environment
"""
from .base import *
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

# ============================================================
# Security Settings
# ============================================================

DEBUG = False

# Hosts ที่อนุญาตให้เข้าถึง (ต้องตั้งค่า)
ALLOWED_HOSTS = env('ALLOWED_HOSTS')
# ตัวอย่าง: ALLOWED_HOSTS=myapp.com,www.myapp.com

# HTTPS Settings
SECURE_SSL_REDIRECT = True           # Redirect HTTP → HTTPS
SECURE_HSTS_SECONDS = 31536000       # HTTP Strict Transport Security 1 ปี
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True

# Cookie Security
SESSION_COOKIE_SECURE = True         # ส่ง Cookie เฉพาะผ่าน HTTPS
CSRF_COOKIE_SECURE = True            # ส่ง CSRF Cookie เฉพาะผ่าน HTTPS
SESSION_COOKIE_HTTPONLY = True       # JavaScript ไม่สามารถเข้าถึง Cookie ได้
SESSION_COOKIE_AGE = 86400           # Session หมดอายุใน 1 วัน

# X-Frame-Options
X_FRAME_OPTIONS = 'DENY'

# ============================================================
# Database - PostgreSQL
# ============================================================

DATABASES = {
    'default': env.db()
    # DATABASE_URL=postgres://user:password@db-host:5432/dbname
}

# Database Connection Pooling
DATABASES['default']['CONN_MAX_AGE'] = 600      # Connection Pool Timeout 10 นาที
DATABASES['default']['OPTIONS'] = {
    'connect_timeout': 10,
    'sslmode': 'require',                         # บังคับ SSL สำหรับ Database
}

# ============================================================
# Static Files
# ============================================================

STATIC_ROOT = BASE_DIR / 'staticfiles'
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

# ============================================================
# Media Files - AWS S3 (Optional)
# ============================================================

USE_S3 = env('USE_S3', default=False, cast=bool)

if USE_S3:
    AWS_ACCESS_KEY_ID = env('AWS_ACCESS_KEY_ID')
    AWS_SECRET_ACCESS_KEY = env('AWS_SECRET_ACCESS_KEY')
    AWS_STORAGE_BUCKET_NAME = env('AWS_STORAGE_BUCKET_NAME')
    AWS_S3_REGION_NAME = env('AWS_S3_REGION_NAME', default='ap-southeast-1')
    AWS_S3_CUSTOM_DOMAIN = f'{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com'
    AWS_S3_OBJECT_PARAMETERS = {'CacheControl': 'max-age=86400'}
    AWS_DEFAULT_ACL = 'public-read'

    # Media Files
    DEFAULT_FILE_STORAGE = 'myproject.storage.MediaStorage'
    MEDIA_URL = f'https://{AWS_S3_CUSTOM_DOMAIN}/media/'

# ============================================================
# Logging
# ============================================================

LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
        'simple': {
            'format': '{levelname} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': BASE_DIR / 'logs' / 'django.log',
            'maxBytes': 1024 * 1024 * 5,   # 5 MB
            'backupCount': 5,
            'formatter': 'verbose',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'WARNING',
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'file'],
            'level': env('DJANGO_LOG_LEVEL', default='ERROR'),
            'propagate': False,
        },
        'myapp': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
            'propagate': False,
        },
    },
}

# ============================================================
# Sentry Error Tracking
# ============================================================

SENTRY_DSN = env('SENTRY_DSN', default='')

if SENTRY_DSN:
    sentry_sdk.init(
        dsn=SENTRY_DSN,
        integrations=[DjangoIntegration()],
        traces_sample_rate=0.1,         # Sample 10% ของ Transactions
        send_default_pii=False,
        environment='production',
    )

# ============================================================
# Email Production
# ============================================================

EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = env('EMAIL_HOST')
EMAIL_PORT = env('EMAIL_PORT', default=587)
EMAIL_HOST_USER = env('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = env('EMAIL_HOST_PASSWORD')
EMAIL_USE_TLS = True
DEFAULT_FROM_EMAIL = env('DEFAULT_FROM_EMAIL')
SERVER_EMAIL = env('SERVER_EMAIL', default=DEFAULT_FROM_EMAIL)
ADMINS = [('Admin', env('ADMIN_EMAIL', default=''))]
```

---

## 5. Gunicorn Setup

```bash
# ติดตั้ง Gunicorn
pip install gunicorn

# รัน Gunicorn พื้นฐาน
gunicorn myproject.wsgi:application

# รัน Gunicorn พร้อม Options
gunicorn myproject.wsgi:application \
    --workers 4 \
    --threads 2 \
    --bind 0.0.0.0:8000 \
    --timeout 120 \
    --log-level info
```

```ini
# gunicorn.conf.py - ไฟล์ Configuration
import multiprocessing

# Worker จำนวน = 2 * CPU cores + 1
workers = multiprocessing.cpu_count() * 2 + 1
# ตัวอย่าง: 2 cores = 5 workers

# Worker Class
worker_class = 'sync'           # sync (default), gevent, uvicorn.workers.UvicornWorker

# Threads ต่อ Worker (สำหรับ sync worker)
threads = 2

# Binding
bind = '0.0.0.0:8000'          # ฟังทุก Interfaces
# หรือ Unix Socket (เร็วกว่าสำหรับ Nginx)
# bind = 'unix:/run/gunicorn.sock'

# Timeout
timeout = 120                   # Worker timeout 120 วินาที
graceful_timeout = 30           # Graceful shutdown timeout
keepalive = 5                   # HTTP Keep-alive

# Logging
accesslog = '/var/log/gunicorn/access.log'
errorlog = '/var/log/gunicorn/error.log'
loglevel = 'info'
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s"'

# Process Management
pidfile = '/run/gunicorn.pid'
user = 'www-data'
group = 'www-data'

# Max Requests (ป้องกัน Memory Leak)
max_requests = 1000
max_requests_jitter = 100

# Preload App (โหลด Django ก่อน Fork Workers - ประหยัด Memory)
preload_app = True

# Worker Connections (สำหรับ Async Workers)
worker_connections = 1000

# Django Settings
raw_env = [
    'DJANGO_SETTINGS_MODULE=myproject.settings.production',
]
```

```bash
# รัน Gunicorn ด้วย Config file
gunicorn -c gunicorn.conf.py myproject.wsgi:application

# รัน ด้วย Async Worker (ต้องติดตั้ง uvicorn)
gunicorn myproject.asgi:application \
    --worker-class uvicorn.workers.UvicornWorker \
    --workers 4 \
    --bind 0.0.0.0:8000
```

```ini
# /etc/systemd/system/gunicorn.service
# Systemd Service สำหรับ Gunicorn

[Unit]
Description=Gunicorn Django Application
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/myapp
ExecStart=/var/www/myapp/venv/bin/gunicorn \
    -c /var/www/myapp/gunicorn.conf.py \
    myproject.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true
Restart=always
RestartSec=10

# Environment Variables
EnvironmentFile=/var/www/myapp/.env

[Install]
WantedBy=multi-user.target
```

```bash
# เปิดใช้งาน Gunicorn Service
sudo systemctl daemon-reload
sudo systemctl enable gunicorn
sudo systemctl start gunicorn
sudo systemctl status gunicorn

# Restart เมื่อ Deploy โค้ดใหม่
sudo systemctl reload gunicorn
```

---

## 6. WhiteNoise สำหรับ Static Files

```bash
# ติดตั้ง WhiteNoise
pip install whitenoise
```

```python
# myproject/settings/base.py

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    # WhiteNoise ต้องอยู่ลำดับที่ 2 (หลัง SecurityMiddleware)
    'whitenoise.middleware.WhiteNoiseMiddleware',
    # ... Middleware อื่นๆ
]

# Static Files Settings
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

# WhiteNoise - Compress และ Cache Static Files
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
# CompressedManifestStaticFilesStorage:
# - Compress ไฟล์ (gzip, brotli)
# - เพิ่ม Hash ใน filename (cache busting)
# - ตัวอย่าง: main.css → main.abc123.css
```

```bash
# collectstatic - รวม Static files ทั้งหมดไปยัง STATIC_ROOT
python manage.py collectstatic --noinput

# ตรวจสอบ Static files
python manage.py collectstatic --dry-run
```

---

## 7. Nginx Configuration

```nginx
# /etc/nginx/sites-available/myapp.conf
# Nginx Reverse Proxy สำหรับ Django

# Upstream (Gunicorn)
upstream django_app {
    server unix:/run/gunicorn.sock;
    # หรือ
    # server 127.0.0.1:8000;
}

# Redirect HTTP → HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name myapp.com www.myapp.com;

    # Let's Encrypt Verification
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Redirect ทั้งหมดไป HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS Server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name myapp.com www.myapp.com;

    # SSL Certificates (จาก Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

    # SSL Settings (Modern Configuration)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_stapling on;
    ssl_stapling_verify on;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:;" always;

    # Logs
    access_log /var/log/nginx/myapp_access.log;
    error_log /var/log/nginx/myapp_error.log;

    # Max Upload Size
    client_max_body_size 10M;

    # Static Files (serve โดย Nginx โดยตรง - เร็วกว่า Django มาก)
    location /static/ {
        alias /var/www/myapp/staticfiles/;
        expires 1y;                         # Cache 1 ปี
        add_header Cache-Control "public, immutable";
        gzip_static on;                     # Serve .gz files ถ้ามี
        access_log off;                     # ไม่ Log Static files
    }

    # Media Files
    location /media/ {
        alias /var/www/myapp/media/;
        expires 1M;                         # Cache 1 เดือน
        add_header Cache-Control "public";
        access_log off;
    }

    # Favicon
    location = /favicon.ico {
        alias /var/www/myapp/staticfiles/favicon.ico;
        access_log off;
        log_not_found off;
    }

    # robots.txt
    location = /robots.txt {
        alias /var/www/myapp/staticfiles/robots.txt;
        access_log off;
        log_not_found off;
    }

    # Proxy ไปยัง Gunicorn
    location / {
        proxy_pass http://django_app;

        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 30s;
        proxy_send_timeout 120s;
        proxy_read_timeout 120s;

        # Buffer Settings
        proxy_buffering on;
        proxy_buffer_size 8k;
        proxy_buffers 8 8k;
        proxy_busy_buffers_size 16k;

        # WebSocket Support (ถ้าใช้ Django Channels)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

```bash
# เปิดใช้งาน Nginx Config
sudo ln -s /etc/nginx/sites-available/myapp.conf /etc/nginx/sites-enabled/
sudo nginx -t          # ทดสอบ Config
sudo systemctl reload nginx
```

---

## 8. PostgreSQL with psycopg2

```bash
# ติดตั้ง psycopg2
pip install psycopg2-binary  # สำหรับ Development
pip install psycopg2          # สำหรับ Production (ต้องมี libpq-dev)

# Ubuntu: ติดตั้ง Dependencies
sudo apt-get install -y libpq-dev python3-dev

# สร้าง PostgreSQL Database
sudo -u postgres psql

# ใน PostgreSQL shell:
CREATE DATABASE myapp_db;
CREATE USER myapp_user WITH PASSWORD 'secure_password_here';
ALTER ROLE myapp_user SET client_encoding TO 'utf8';
ALTER ROLE myapp_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE myapp_user SET timezone TO 'Asia/Bangkok';
GRANT ALL PRIVILEGES ON DATABASE myapp_db TO myapp_user;
\q
```

```python
# myproject/settings/production.py

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST', default='localhost'),
        'PORT': env('DB_PORT', default='5432'),
        'OPTIONS': {
            'connect_timeout': 10,
            'sslmode': env('DB_SSLMODE', default='prefer'),
            # Pool settings
            'pool_pre_ping': True,
        },
        'CONN_MAX_AGE': 600,   # Connection Pool Timeout 10 นาที
    }
}
```

```python
# ใช้ DATABASE_URL (ง่ายกว่า)
# .env:
# DATABASE_URL=postgresql://myapp_user:secure_password@localhost:5432/myapp_db

# settings.py:
import dj_database_url
DATABASES = {
    'default': dj_database_url.config(
        default=env('DATABASE_URL'),
        conn_max_age=600
    )
}
```

```bash
# Migrations
python manage.py migrate

# สร้าง Superuser
python manage.py createsuperuser

# Backup PostgreSQL
pg_dump -U myapp_user myapp_db > backup.sql

# Restore PostgreSQL
psql -U myapp_user myapp_db < backup.sql
```

---

## 9. Environment Variables Management

```bash
# .gitignore - อย่า Commit .env ไปยัง Git!
.env
.env.local
.env.production
*.env
```

```bash
# .env.example - เก็บใน Git เป็น Template (ไม่มีค่าจริง)
DEBUG=False
SECRET_KEY=generate-a-strong-secret-key-here
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
ALLOWED_HOSTS=yourdomain.com,www.yourdomain.com
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_HOST_USER=your-email@gmail.com
EMAIL_HOST_PASSWORD=your-app-password
EMAIL_USE_TLS=True
DEFAULT_FROM_EMAIL=noreply@yourdomain.com
REDIS_URL=redis://localhost:6379/0
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_STORAGE_BUCKET_NAME=
SENTRY_DSN=
```

```python
# Generate Strong Secret Key
import secrets
import string

def generate_secret_key(length=50):
    """สร้าง Django SECRET_KEY ที่แข็งแกร่ง"""
    chars = string.ascii_letters + string.digits + '!@#$%^&*(-_=+)'
    return ''.join(secrets.choice(chars) for _ in range(length))

print(generate_secret_key())

# หรือใช้ Django
# python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

```bash
# ใช้ direnv สำหรับ Local Development
# ติดตั้ง direnv
curl -sfL https://direnv.net/install.sh | bash

# สร้าง .envrc
echo "dotenv" > .envrc
direnv allow .

# หรือใช้ python-dotenv
pip install python-dotenv
```

```python
# ใน manage.py หรือ wsgi.py
import os
from dotenv import load_dotenv

# โหลด .env file
load_dotenv()

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings.production')
```

---

## 10. HTTPS/SSL ด้วย Let's Encrypt

```bash
# ติดตั้ง Certbot บน Ubuntu
sudo apt update
sudo apt install certbot python3-certbot-nginx

# ขอ SSL Certificate
sudo certbot --nginx -d myapp.com -d www.myapp.com

# ทดสอบ Renewal
sudo certbot renew --dry-run

# Certificate จะ Renew อัตโนมัติผ่าน Systemd Timer
sudo systemctl status certbot.timer
```

```python
# myproject/settings/production.py
# หลังจากได้ SSL Certificate แล้ว

# Force HTTPS
SECURE_SSL_REDIRECT = True

# HSTS Headers
SECURE_HSTS_SECONDS = 31536000        # 1 ปี
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Secure Cookies
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

# Trust Nginx Headers (สำหรับ Reverse Proxy)
USE_X_FORWARDED_HOST = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

---

## 11. Deployment Checklist

```bash
# รัน Django Deployment Checklist
python manage.py check --deploy

# ผลลัพธ์ที่ควรผ่านทั้งหมด:
# System check identified no issues (0 silenced).
```

```python
# Checklist ที่สำคัญ:

# ✅ 1. DEBUG = False
DEBUG = False

# ✅ 2. SECRET_KEY ยาวและ Random
SECRET_KEY = env('SECRET_KEY')  # ไม่ Hardcode!

# ✅ 3. ALLOWED_HOSTS ตั้งค่าถูกต้อง
ALLOWED_HOSTS = ['myapp.com', 'www.myapp.com']

# ✅ 4. Database ใช้ PostgreSQL
# ✅ 5. Email ตั้งค่าจริง
# ✅ 6. Static Files ตั้งค่า STATIC_ROOT

# ✅ 7. Security Middleware
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    # ...
]

# ✅ 8. Password Validators
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {'min_length': 8},
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

# ✅ 9. HTTPS Settings
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

# ✅ 10. X-Content-Type-Options
SECURE_CONTENT_TYPE_NOSNIFF = True

# ✅ 11. X-XSS-Protection
SECURE_BROWSER_XSS_FILTER = True

# ✅ 12. Clickjacking Protection
X_FRAME_OPTIONS = 'DENY'
```

---

## 12. Heroku Deployment

```bash
# ติดตั้ง Heroku CLI
# https://devcenter.heroku.com/articles/heroku-cli

# Login
heroku login

# สร้าง App
heroku create myapp-name

# เพิ่ม PostgreSQL
heroku addons:create heroku-postgresql:mini

# ตั้งค่า Environment Variables
heroku config:set DEBUG=False
heroku config:set SECRET_KEY="$(python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())")"
heroku config:set DJANGO_SETTINGS_MODULE=myproject.settings.production
heroku config:set ALLOWED_HOSTS=myapp-name.herokuapp.com

# Deploy
git push heroku main

# รัน Migrations
heroku run python manage.py migrate

# Collect Static
heroku run python manage.py collectstatic --noinput

# สร้าง Superuser
heroku run python manage.py createsuperuser
```

```
# Procfile - บอก Heroku วิธีรัน App
web: gunicorn myproject.wsgi:application --config gunicorn.conf.py
release: python manage.py migrate
worker: celery -A myproject worker --loglevel=info
```

```python
# requirements.txt สำหรับ Heroku
Django==4.2.7
gunicorn==21.2.0
psycopg2-binary==2.9.9
whitenoise==6.6.0
django-environ==0.11.2
dj-database-url==2.1.0
```

---

## 13. Railway Deployment

```bash
# ติดตั้ง Railway CLI
npm install -g @railway/cli

# Login
railway login

# สร้าง Project
railway new

# เพิ่ม PostgreSQL
railway add postgresql

# Deploy
railway up

# ดู Logs
railway logs

# รัน Commands
railway run python manage.py migrate
railway run python manage.py createsuperuser
```

```yaml
# railway.toml
[build]
builder = "NIXPACKS"

[deploy]
startCommand = "gunicorn myproject.wsgi:application"
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 10

[deploy.healthcheckPath]
healthcheckPath = "/health/"
healthcheckTimeout = 100
```

---

## 14. Render Deployment

```yaml
# render.yaml
services:
  - type: web
    name: myapp
    env: python
    buildCommand: |
      pip install -r requirements.txt
      python manage.py collectstatic --noinput
    startCommand: gunicorn myproject.wsgi:application
    envVars:
      - key: DEBUG
        value: false
      - key: SECRET_KEY
        generateValue: true
      - key: DATABASE_URL
        fromDatabase:
          name: myapp-db
          property: connectionString
      - key: DJANGO_SETTINGS_MODULE
        value: myproject.settings.production

databases:
  - name: myapp-db
    databaseName: myapp
    user: myapp
    plan: free
```

---

## 15. Complete Production docker-compose.yml

```dockerfile
# Dockerfile
FROM python:3.11-slim

# ตั้งค่า Environment
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV DJANGO_SETTINGS_MODULE=myproject.settings.production

# ติดตั้ง System Dependencies
RUN apt-get update && apt-get install -y \
    libpq-dev \
    gcc \
    curl \
    && rm -rf /var/lib/apt/lists/*

# สร้าง Directory
WORKDIR /app

# ติดตั้ง Python Dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy Source Code
COPY . .

# Collect Static Files
RUN python manage.py collectstatic --noinput

# สร้าง Non-root User
RUN useradd --create-home appuser && chown -R appuser:appuser /app
USER appuser

# Expose Port
EXPOSE 8000

# Health Check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1

# รัน Gunicorn
CMD ["gunicorn", "myproject.wsgi:application", "--config", "gunicorn.conf.py"]
```

```yaml
# docker-compose.yml (Production)
version: '3.9'

# Shared Environment Variables
x-django-env: &django-env
  DJANGO_SETTINGS_MODULE: myproject.settings.production
  DEBUG: "False"
  SECRET_KEY: ${SECRET_KEY}
  DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
  REDIS_URL: redis://redis:6379/0
  ALLOWED_HOSTS: ${ALLOWED_HOSTS}
  EMAIL_HOST: ${EMAIL_HOST}
  EMAIL_HOST_USER: ${EMAIL_HOST_USER}
  EMAIL_HOST_PASSWORD: ${EMAIL_HOST_PASSWORD}
  SENTRY_DSN: ${SENTRY_DSN}

services:
  # ============================================================
  # Database - PostgreSQL
  # ============================================================
  db:
    image: postgres:15-alpine
    restart: unless-stopped
    volumes:
      - postgres_data:/var/lib/postgresql/data
      # Init Scripts
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-myapp_db}
      POSTGRES_USER: ${POSTGRES_USER:-myapp_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-myapp_user}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # ============================================================
  # Cache - Redis
  # ============================================================
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # ============================================================
  # Django Application
  # ============================================================
  web:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    volumes:
      - media_files:/app/media
      - static_files:/app/staticfiles
    environment:
      <<: *django-env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    expose:
      - "8000"
    networks:
      - backend
      - frontend
    # รัน Migrations และ Start Server
    command: >
      sh -c "
        python manage.py migrate --noinput &&
        python manage.py collectstatic --noinput &&
        gunicorn myproject.wsgi:application --config gunicorn.conf.py
      "

  # ============================================================
  # Celery Worker
  # ============================================================
  celery_worker:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    command: celery -A myproject worker --loglevel=info --concurrency=4
    volumes:
      - media_files:/app/media
    environment:
      <<: *django-env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend

  # ============================================================
  # Celery Beat (Scheduler)
  # ============================================================
  celery_beat:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    command: celery -A myproject beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    volumes:
      - media_files:/app/media
    environment:
      <<: *django-env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend

  # ============================================================
  # Nginx Reverse Proxy
  # ============================================================
  nginx:
    image: nginx:1.25-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./docker/nginx/conf.d:/etc/nginx/conf.d:ro
      - static_files:/var/www/static:ro
      - media_files:/var/www/media:ro
      - certbot_conf:/etc/letsencrypt:ro
      - certbot_www:/var/www/certbot:ro
      - nginx_logs:/var/log/nginx
    depends_on:
      - web
    networks:
      - frontend
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ============================================================
  # Certbot (SSL Certificate)
  # ============================================================
  certbot:
    image: certbot/certbot
    volumes:
      - certbot_conf:/etc/letsencrypt
      - certbot_www:/var/www/certbot
    entrypoint: /bin/sh -c "trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;"

# ============================================================
# Volumes
# ============================================================
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  media_files:
    driver: local
  static_files:
    driver: local
  certbot_conf:
    driver: local
  certbot_www:
    driver: local
  nginx_logs:
    driver: local

# ============================================================
# Networks
# ============================================================
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # Backend network ไม่ expose ออกสู่ภายนอก
```

```bash
# รัน Production Docker Compose
docker-compose up -d

# ดู Logs
docker-compose logs -f web

# รัน Django Commands
docker-compose exec web python manage.py migrate
docker-compose exec web python manage.py createsuperuser

# Restart Service
docker-compose restart web

# Update และ Deploy ใหม่
git pull
docker-compose build web
docker-compose up -d --no-deps web

# Backup Database
docker-compose exec db pg_dump -U myapp_user myapp_db > backup.sql

# Restore Database
docker-compose exec -T db psql -U myapp_user myapp_db < backup.sql
```

---

## 16. CI/CD Pipeline ด้วย GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  DJANGO_SETTINGS_MODULE: myproject.settings.test

jobs:
  # ============================================================
  # Test
  # ============================================================
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install Dependencies
        run: pip install -r requirements.txt

      - name: Run Tests
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/test_db
          SECRET_KEY: test-secret-key-for-ci
          DEBUG: "False"
        run: |
          python manage.py migrate
          pytest --cov=myapp --cov-report=xml -v

      - name: Upload Coverage
        uses: codecov/codecov-action@v3

  # ============================================================
  # Deploy
  # ============================================================
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Server
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/myapp
            git pull origin main
            source venv/bin/activate
            pip install -r requirements.txt
            python manage.py migrate --noinput
            python manage.py collectstatic --noinput
            sudo systemctl reload gunicorn
            echo "Deploy completed!"
```

---

## 17. Health Check Endpoint

```python
# myapp/views.py
from django.http import JsonResponse
from django.db import connection
from django.core.cache import cache

def health_check(request):
    """
    Health Check Endpoint
    ใช้สำหรับ Load Balancer และ Container Orchestration
    """
    health = {
        'status': 'healthy',
        'checks': {}
    }

    # ตรวจสอบ Database
    try:
        with connection.cursor() as cursor:
            cursor.execute('SELECT 1')
        health['checks']['database'] = 'ok'
    except Exception as e:
        health['checks']['database'] = f'error: {str(e)}'
        health['status'] = 'unhealthy'

    # ตรวจสอบ Cache (Redis)
    try:
        cache.set('health_check', 'ok', 30)
        val = cache.get('health_check')
        if val == 'ok':
            health['checks']['cache'] = 'ok'
        else:
            health['checks']['cache'] = 'error: cache read failed'
            health['status'] = 'unhealthy'
    except Exception as e:
        health['checks']['cache'] = f'error: {str(e)}'
        health['status'] = 'unhealthy'

    status_code = 200 if health['status'] == 'healthy' else 503
    return JsonResponse(health, status=status_code)
```

```python
# myproject/urls.py
from django.urls import path
from myapp.views import health_check

urlpatterns = [
    path('health/', health_check, name='health-check'),
    # ...
]
```

---

## 18. Deploy Script อัตโนมัติ

```bash
#!/bin/bash
# deploy.sh - Script สำหรับ Deploy แบบ Manual

set -e  # หยุดถ้า Error

echo "🚀 เริ่ม Deploy..."

# ไปยัง Project Directory
cd /var/www/myapp

# Pull Code ใหม่
echo "📥 Pull latest code..."
git pull origin main

# Activate Virtual Environment
source venv/bin/activate

# ติดตั้ง Dependencies ใหม่ (ถ้ามี)
echo "📦 Install dependencies..."
pip install -r requirements.txt --quiet

# รัน Migrations
echo "🗄️ Run migrations..."
python manage.py migrate --noinput

# Collect Static Files
echo "📁 Collect static files..."
python manage.py collectstatic --noinput

# Restart Gunicorn
echo "🔄 Restart Gunicorn..."
sudo systemctl reload gunicorn

# Restart Celery Workers (ถ้ามี)
echo "⚙️ Restart Celery..."
sudo systemctl restart celery celery-beat

echo "✅ Deploy สำเร็จ!"

# ส่ง Notification (Optional)
# curl -X POST https://hooks.slack.com/... -d '{"text": "Deploy completed!"}'
```

```bash
chmod +x deploy.sh
./deploy.sh
```

---

## 19. Performance Optimization

```python
# myproject/settings/production.py

# ============================================================
# Database Connection Pooling ด้วย pgBouncer (แนะนำ)
# หรือใช้ CONN_MAX_AGE
# ============================================================

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST'),
        'PORT': env('DB_PORT', default='5432'),
        # Connection Pool: ไม่ปิด Connection หลัง Request
        'CONN_MAX_AGE': 600,  # 10 นาที
        # ใช้กับ pgBouncer: ต้องปิด CONN_MAX_AGE
        # 'CONN_MAX_AGE': 0,
    }
}

# ============================================================
# Cache Configuration
# ============================================================

CACHE_MIDDLEWARE_ALIAS = 'default'
CACHE_MIDDLEWARE_SECONDS = 300    # Cache 5 นาที
CACHE_MIDDLEWARE_KEY_PREFIX = 'myapp'

# ============================================================
# Session ใช้ Cache แทน Database
# ============================================================

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'

# ============================================================
# Template Caching
# ============================================================

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'OPTIONS': {
            'context_processors': [
                # ...
            ],
            # Enable Template Caching (ใน Production)
            'loaders': [
                ('django.template.loaders.cached.Loader', [
                    'django.template.loaders.filesystem.Loader',
                    'django.template.loaders.app_directories.Loader',
                ]),
            ],
        },
    },
]
```

---

## 20. สรุป Part 075

✅ **django-environ / python-decouple** - จัดการ Environment Variables อย่างปลอดภัย  
✅ **Production Settings** - DEBUG=False, ALLOWED_HOSTS, Security Headers, HTTPS  
✅ **PostgreSQL** - ใช้ psycopg2, Connection Pooling, Secure SSL  
✅ **Gunicorn** - Workers, Threads, Timeout, Systemd Service  
✅ **WhiteNoise** - Serve Static Files พร้อม Compression และ Cache  
✅ **Nginx** - Reverse Proxy, SSL Termination, Static Files, Security Headers  
✅ **HTTPS/SSL** - Let's Encrypt ด้วย Certbot, Auto-renewal  
✅ **Deployment Checklist** - `python manage.py check --deploy`  
✅ **Heroku/Railway/Render** - Cloud Platform Deployment Steps  
✅ **Docker Compose** - Complete Production Stack พร้อม Celery, Redis, Nginx  
✅ **CI/CD** - GitHub Actions สำหรับ Auto-test และ Deploy  
✅ **Health Check** - Endpoint สำหรับ Load Balancer  
✅ **Performance** - Connection Pooling, Cache, Template Caching  

### Quick Reference: Production Checklist

```bash
# 1. ตั้งค่า Environment Variables
cp .env.example .env
nano .env  # แก้ไขค่าจริง

# 2. ติดตั้ง Dependencies
pip install -r requirements.txt

# 3. รัน Migrations
python manage.py migrate

# 4. Collect Static Files
python manage.py collectstatic --noinput

# 5. ตรวจสอบ Deployment Checklist
python manage.py check --deploy

# 6. รัน Gunicorn
gunicorn myproject.wsgi:application -c gunicorn.conf.py

# 7. หรือรัน Docker Compose
docker-compose up -d

# 8. ขอ SSL Certificate
sudo certbot --nginx -d mydomain.com

# 9. ทดสอบ Application
curl https://mydomain.com/health/
```

---

## ➡️ ถัดไป: Part 076 - Django Advanced Topics

*Part 075/100+ | Python Course - Beginner to World-Class*
