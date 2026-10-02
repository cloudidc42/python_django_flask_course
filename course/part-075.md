# Part 075: Django Deployment

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ตั้งค่า settings สำหรับ production
- ใช้ gunicorn เป็น WSGI server
- จัดการ static files ด้วย whitenoise
- ตั้งค่า PostgreSQL
- จัดการ environment variables
- รัน collectstatic
- Production deployment checklist

---

## 1. Settings สำหรับ Production

### แยก Settings

```
myproject/
├── settings/
│   ├── __init__.py
│   ├── base.py          # settings พื้นฐาน
│   ├── development.py   # สำหรับ dev
│   └── production.py    # สำหรับ production
```

```python
# settings/base.py - Settings พื้นฐาน
import os
from pathlib import Path
from decouple import config, Csv  # pip install python-decouple

BASE_DIR = Path(__file__).resolve().parent.parent.parent

SECRET_KEY = config('SECRET_KEY')  # อ่านจาก env

DEBUG = config('DEBUG', default=False, cast=bool)

ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='', cast=Csv())

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
    
    # Local apps
    'accounts',
    'articles',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # สำหรับ static files
    'corsheaders.middleware.CorsMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

# Database - อ่านจาก DATABASE_URL
import dj_database_url  # pip install dj-database-url

DATABASES = {
    'default': dj_database_url.config(
        default=config('DATABASE_URL', default='sqlite:///db.sqlite3'),
        conn_max_age=600,
        conn_health_checks=True,
    )
}

# Password validation
AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator', 'OPTIONS': {'min_length': 8}},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]

# Internationalization
LANGUAGE_CODE = 'th'
TIME_ZONE = 'Asia/Bangkok'
USE_I18N = True
USE_TZ = True

# Static files
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]

# Whitenoise - serve static files ใน production
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

# Media files
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')

# Auth model
AUTH_USER_MODEL = 'accounts.User'

# Email
EMAIL_BACKEND = config(
    'EMAIL_BACKEND',
    default='django.core.mail.backends.console.EmailBackend'
)
EMAIL_HOST = config('EMAIL_HOST', default='smtp.gmail.com')
EMAIL_PORT = config('EMAIL_PORT', default=587, cast=int)
EMAIL_HOST_USER = config('EMAIL_HOST_USER', default='')
EMAIL_HOST_PASSWORD = config('EMAIL_HOST_PASSWORD', default='')
EMAIL_USE_TLS = config('EMAIL_USE_TLS', default=True, cast=bool)
DEFAULT_FROM_EMAIL = config('DEFAULT_FROM_EMAIL', default='noreply@example.com')

# Celery
CELERY_BROKER_URL = config('CELERY_BROKER_URL', default='redis://localhost:6379/0')
CELERY_RESULT_BACKEND = config('CELERY_RESULT_BACKEND', default='django-db')

# Logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
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
            'filename': os.path.join(BASE_DIR, 'logs', 'django.log'),
            'maxBytes': 1024 * 1024 * 5,  # 5 MB
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
            'level': config('LOG_LEVEL', default='INFO'),
            'propagate': False,
        },
        'myapp': {
            'handlers': ['console', 'file'],
            'level': 'DEBUG',
            'propagate': False,
        },
    },
}
```

```python
# settings/production.py - Production-specific settings
from .base import *

# Security settings
DEBUG = False

# HTTPS
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
X_FRAME_OPTIONS = 'DENY'

# PostgreSQL
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST', default='localhost'),
        'PORT': config('DB_PORT', default='5432'),
        'CONN_MAX_AGE': 600,
        'OPTIONS': {
            'sslmode': 'require',  # ใช้ SSL กับ PostgreSQL
        },
    }
}

# Cache - Redis
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': config('REDIS_URL', default='redis://localhost:6379/1'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}

# Email - SendGrid หรือ AWS SES
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'

# CORS
CORS_ALLOWED_ORIGINS = config('CORS_ALLOWED_ORIGINS', cast=Csv(), default='')
CORS_ALLOW_CREDENTIALS = True

# AWS S3 สำหรับ media files (optional)
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
AWS_ACCESS_KEY_ID = config('AWS_ACCESS_KEY_ID', default='')
AWS_SECRET_ACCESS_KEY = config('AWS_SECRET_ACCESS_KEY', default='')
AWS_STORAGE_BUCKET_NAME = config('AWS_STORAGE_BUCKET_NAME', default='')
AWS_S3_REGION_NAME = config('AWS_S3_REGION_NAME', default='ap-southeast-1')
```

```python
# settings/development.py - Development settings
from .base import *

DEBUG = True

ALLOWED_HOSTS = ['*']

# SQLite สำหรับ development
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# Debug toolbar
INSTALLED_APPS += ['debug_toolbar']
MIDDLEWARE.insert(0, 'debug_toolbar.middleware.DebugToolbarMiddleware')
INTERNAL_IPS = ['127.0.0.1']

# แสดงอีเมลใน console
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

---

## 2. Environment Variables

```bash
# .env - ไม่เพิ่มใน git!
SECRET_KEY=your-super-secret-key-here-change-this
DEBUG=False
ALLOWED_HOSTS=example.com,www.example.com
DATABASE_URL=postgresql://user:password@localhost:5432/mydb

# หรือแยก DB settings
DB_NAME=mydb
DB_USER=myuser
DB_PASSWORD=mypassword
DB_HOST=localhost
DB_PORT=5432

# Redis
REDIS_URL=redis://localhost:6379/1
CELERY_BROKER_URL=redis://localhost:6379/0

# Email
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_HOST_USER=your@email.com
EMAIL_HOST_PASSWORD=your-app-password
EMAIL_USE_TLS=True
DEFAULT_FROM_EMAIL=noreply@example.com

# AWS
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_STORAGE_BUCKET_NAME=my-bucket

# CORS
CORS_ALLOWED_ORIGINS=https://example.com,https://app.example.com
```

```python
# .gitignore
.env
.env.*
*.pyc
__pycache__/
db.sqlite3
media/
staticfiles/
logs/
*.log
```

---

## 3. Gunicorn

```bash
pip install gunicorn
```

```bash
# รัน gunicorn
gunicorn myproject.wsgi:application

# พร้อม options
gunicorn myproject.wsgi:application \
    --bind 0.0.0.0:8000 \
    --workers 4 \
    --worker-class gthread \
    --threads 2 \
    --timeout 120 \
    --max-requests 1000 \
    --max-requests-jitter 50 \
    --log-file /var/log/gunicorn/gunicorn.log \
    --access-logfile /var/log/gunicorn/access.log \
    --error-logfile /var/log/gunicorn/error.log \
    --log-level info
```

```ini
# gunicorn.conf.py
bind = "0.0.0.0:8000"
workers = 4                    # 2-4 x CPU cores
worker_class = "gthread"       # threaded workers
threads = 2                    # threads per worker
worker_connections = 1000
timeout = 120
keepalive = 5
max_requests = 1000
max_requests_jitter = 50
reload = False                 # True ใน development

# Logging
accesslog = "/var/log/gunicorn/access.log"
errorlog = "/var/log/gunicorn/error.log"
loglevel = "info"

# Security
limit_request_line = 4096
limit_request_fields = 100
```

```bash
# รัน ด้วย config file
gunicorn -c gunicorn.conf.py myproject.wsgi:application
```

---

## 4. Nginx Configuration

```nginx
# /etc/nginx/sites-available/myproject
server {
    listen 80;
    server_name example.com www.example.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;
    
    # SSL
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # Static files
    location /static/ {
        alias /home/ubuntu/myproject/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
    }
    
    # Media files
    location /media/ {
        alias /home/ubuntu/myproject/media/;
    }
    
    # Proxy to Gunicorn
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

---

## 5. Static Files และ collectstatic

```python
# settings/base.py
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]

# Whitenoise
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

```bash
# รัน collectstatic ก่อน deploy
python manage.py collectstatic --no-input

# collectstatic จะ:
# 1. รวม static files จากทุก apps
# 2. เก็บไว้ใน STATIC_ROOT (staticfiles/)
# 3. Whitenoise จะ compress และเพิ่ม hash ให้ filename
```

---

## 6. PostgreSQL

```bash
# ติดตั้ง PostgreSQL
sudo apt install postgresql postgresql-contrib

# สร้าง database และ user
sudo -u postgres psql
CREATE DATABASE mydb;
CREATE USER myuser WITH PASSWORD 'mypassword';
ALTER ROLE myuser SET client_encoding TO 'utf8';
ALTER ROLE myuser SET default_transaction_isolation TO 'read committed';
ALTER ROLE myuser SET timezone TO 'Asia/Bangkok';
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
\q

# ติดตั้ง psycopg2
pip install psycopg2-binary

# รัน migrations
python manage.py migrate
```

---

## 7. Systemd Service

```ini
# /etc/systemd/system/gunicorn.service
[Unit]
Description=Gunicorn daemon for Django
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/myproject
ExecStart=/home/ubuntu/myproject/venv/bin/gunicorn \
    -c gunicorn.conf.py \
    myproject.wsgi:application
Restart=on-failure
RestartSec=5s

# Environment
Environment="DJANGO_SETTINGS_MODULE=myproject.settings.production"
EnvironmentFile=/home/ubuntu/myproject/.env

[Install]
WantedBy=multi-user.target
```

```bash
# เปิดใช้ service
sudo systemctl enable gunicorn
sudo systemctl start gunicorn
sudo systemctl status gunicorn

# reload เมื่อแก้ไข
sudo systemctl reload gunicorn
```

---

## 8. Deployment Checklist

```python
# ตรวจสอบด้วย Django check
python manage.py check --deploy
```

### Checklist ก่อน Deploy

```
SECURITY:
□ SECRET_KEY ต้องเป็น random string ยาวอย่างน้อย 50 ตัวอักษร
□ DEBUG = False
□ ALLOWED_HOSTS กำหนดเฉพาะ domain ที่ใช้
□ SECURE_SSL_REDIRECT = True (ถ้าใช้ HTTPS)
□ SESSION_COOKIE_SECURE = True
□ CSRF_COOKIE_SECURE = True
□ SECURE_HSTS_SECONDS กำหนดแล้ว
□ X_FRAME_OPTIONS = 'DENY'
□ ไม่มี sensitive data ใน code (passwords, API keys)

DATABASE:
□ ใช้ PostgreSQL (ไม่ใช้ SQLite)
□ Database backup ตั้งค่าแล้ว
□ Database connection pool ตั้งค่าแล้ว

STATIC FILES:
□ รัน collectstatic แล้ว
□ Whitenoise หรือ CDN ตั้งค่าแล้ว

PERFORMANCE:
□ Cache ตั้งค่าแล้ว (Redis/Memcached)
□ Database query optimization (select_related, prefetch_related)
□ Database indexes ตั้งค่าแล้ว

MONITORING:
□ Error tracking (Sentry) ตั้งค่าแล้ว
□ Log ตั้งค่าแล้ว
□ Backup strategy วางแผนแล้ว

EMAIL:
□ Email backend ตั้งค่าแล้ว
□ DEFAULT_FROM_EMAIL กำหนดแล้ว

DEPENDENCIES:
□ requirements.txt อัปเดตแล้ว
□ pip install -r requirements.txt รันแล้ว
□ python manage.py migrate รันแล้ว
□ python manage.py collectstatic รันแล้ว
```

---

## 9. Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy project
COPY . .

# Collect static files
RUN python manage.py collectstatic --no-input

# Run gunicorn
CMD ["gunicorn", "-c", "gunicorn.conf.py", "myproject.wsgi:application"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    command: gunicorn -c gunicorn.conf.py myproject.wsgi:application
    volumes:
      - .:/app
      - static_volume:/app/staticfiles
      - media_volume:/app/media
    env_file:
      - .env
    depends_on:
      - db
      - redis
    ports:
      - "8000:8000"
  
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
  
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
  
  celery:
    build: .
    command: celery -A myproject worker --loglevel=info
    env_file:
      - .env
    depends_on:
      - db
      - redis
  
  celery-beat:
    build: .
    command: celery -A myproject beat --loglevel=info
    env_file:
      - .env
    depends_on:
      - db
      - redis
  
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - static_volume:/static
      - media_volume:/media
    depends_on:
      - web

volumes:
  postgres_data:
  redis_data:
  static_volume:
  media_volume:
```

---

## 10. สรุป Part 075

✅ **แยก settings** base/development/production ลดความซ้ำ
✅ **Environment variables** ใช้ python-decouple หรือ os.environ เก็บ secrets
✅ **Gunicorn** เป็น WSGI server production-ready
✅ **Whitenoise** serve static files โดยไม่ต้องพึ่ง nginx
✅ **collectstatic** รวม static files ก่อน deploy เสมอ
✅ **PostgreSQL** แทน SQLite ใน production
✅ **Nginx** เป็น reverse proxy รับ request ก่อนส่งให้ Gunicorn
✅ **Docker** ทำให้ deploy consistent ทุก environment
✅ รัน `python manage.py check --deploy` ตรวจสอบความปลอดภัย

## ➡️ ถัดไป: Part 076 - FastAPI Introduction

*Part 075/100+ | Python Course - Beginner to World-Class*
