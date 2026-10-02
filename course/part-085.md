# Part 085: Flask Deployment
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- Deploy Flask app ด้วย Gunicorn
- ตั้งค่า Nginx เป็น reverse proxy
- ตั้งค่า production settings
- ใช้ supervisor จัดการ process
- Deploy บน Ubuntu server

---

## 1. ทำไม Flask Development Server ไม่เหมาะ Production?

```
Flask Dev Server:
- Single-threaded — รับ request ทีละ 1
- ไม่มี auto-restart เมื่อ crash
- ไม่มี load balancing
- ไม่ปลอดภัยสำหรับ production
```

### WSGI Server คืออะไร?
WSGI (Web Server Gateway Interface) คือ standard interface ระหว่าง Python web application กับ web server

```
Client → Nginx → Gunicorn → Flask App
         (proxy)  (WSGI)    (Python)
```

---

## 2. Gunicorn

Gunicorn เป็น Python WSGI HTTP Server ที่นิยมใช้กับ Flask

### ติดตั้ง
```bash
pip install gunicorn
```

### รัน Gunicorn พื้นฐาน
```bash
# gunicorn <module>:<app_variable>
gunicorn app:app

# ระบุ host และ port
gunicorn --bind 0.0.0.0:8000 app:app

# ระบุ workers (แนะนำ: 2 * CPU cores + 1)
gunicorn --workers 4 --bind 0.0.0.0:8000 app:app

# ระบุ worker class (gevent สำหรับ async)
gunicorn --worker-class gevent --workers 4 --bind 0.0.0.0:8000 app:app
```

### gunicorn.conf.py
```python
# gunicorn.conf.py

import multiprocessing
import os

# จำนวน worker processes
# แนะนำ: (2 x $num_cores) + 1
workers = multiprocessing.cpu_count() * 2 + 1

# Worker class
# sync = default, blocking
# gevent = async, ต้องติดตั้ง gevent
# gthread = threaded
worker_class = 'sync'

# จำนวน threads ต่อ worker (ใช้กับ gthread)
threads = 2

# Timeout (วินาที)
timeout = 120

# Graceful timeout
graceful_timeout = 30

# ไฟล์ log
accesslog = '/var/log/myapp/gunicorn_access.log'
errorlog = '/var/log/myapp/gunicorn_error.log'
loglevel = 'warning'

# Bind address
bind = '0.0.0.0:8000'

# Daemonize (รันเป็น background process)
daemon = False  # ใช้ supervisor แทน

# Process naming
proc_name = 'myapp'

# Preload application (โหลด app ก่อนสร้าง workers)
preload_app = True

# Reload on code change (development เท่านั้น)
reload = False

# Max requests ก่อน restart worker (ป้องกัน memory leak)
max_requests = 1000
max_requests_jitter = 100  # random jitter

# เปิด keep-alive
keepalive = 2
```

### รัน Gunicorn ด้วย config file
```bash
gunicorn -c gunicorn.conf.py 'app:create_app()'
```

---

## 3. Nginx Configuration

Nginx ทำหน้าที่เป็น reverse proxy รับ traffic จาก internet และส่งต่อให้ Gunicorn

### ติดตั้ง Nginx
```bash
sudo apt update
sudo apt install nginx
```

### Nginx config สำหรับ Flask
```nginx
# /etc/nginx/sites-available/myapp

# Upstream group ของ Gunicorn workers
upstream flask_app {
    server 127.0.0.1:8000;
    # สำหรับ multiple instances:
    # server 127.0.0.1:8001;
    # server 127.0.0.1:8002;
}

# Redirect HTTP ไป HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    
    # ต้องมีไฟล์นี้สำหรับ Let's Encrypt
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    
    # Redirect ทุก request ไป HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS Server
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;
    
    # SSL Certificates (จาก Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    # SSL Settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Logging
    access_log /var/log/nginx/myapp_access.log;
    error_log /var/log/nginx/myapp_error.log;
    
    # Client body size (สำหรับ file upload)
    client_max_body_size 20M;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    
    # Static files ให้ Nginx serve โดยตรง (เร็วกว่า Python)
    location /static/ {
        alias /var/www/myapp/app/static/;
        expires 1y;  # Cache 1 ปี
        add_header Cache-Control "public, immutable";
    }
    
    # Uploads
    location /uploads/ {
        alias /var/www/myapp/uploads/;
        expires 30d;
    }
    
    # Favicon
    location /favicon.ico {
        alias /var/www/myapp/app/static/favicon.ico;
        expires 30d;
    }
    
    # Proxy ไปยัง Gunicorn
    location / {
        proxy_pass http://flask_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffer settings
        proxy_buffering on;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }
}
```

### Enable site
```bash
# สร้าง symlink เพื่อ enable
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

# ทดสอบ config
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

---

## 4. Systemd Service (แทน Supervisor)

```ini
# /etc/systemd/system/myapp.service

[Unit]
Description=Flask Application (MyApp)
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/myapp
EnvironmentFile=/var/www/myapp/.env

# Activate virtual environment และรัน Gunicorn
ExecStart=/var/www/myapp/venv/bin/gunicorn \
    --config gunicorn.conf.py \
    'app:create_app()'

ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

```bash
# Enable และ start service
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp

# ดู status
sudo systemctl status myapp

# ดู logs
sudo journalctl -u myapp -f
```

---

## 5. Deployment Script

```bash
#!/bin/bash
# deploy.sh — Script สำหรับ deploy

set -e  # หยุดถ้ามี error

APP_DIR="/var/www/myapp"
VENV_DIR="$APP_DIR/venv"
REPO_URL="https://github.com/username/myapp.git"
BRANCH="main"

echo "=== Starting deployment ==="

# 1. Pull latest code
cd $APP_DIR
git pull origin $BRANCH

# 2. Activate virtual environment
source $VENV_DIR/bin/activate

# 3. Install/update dependencies
pip install -r requirements.txt

# 4. Run database migrations
flask db upgrade

# 5. Collect static files (ถ้าต้องการ)
# flask assets build

# 6. Restart application
sudo systemctl restart myapp

# 7. Check status
if sudo systemctl is-active --quiet myapp; then
    echo "=== Deployment successful! ==="
else
    echo "=== ERROR: Application failed to start ==="
    sudo systemctl status myapp
    exit 1
fi

echo "=== Done ==="
```

---

## 6. Production Flask Settings

```python
# app/__init__.py — Production checks

import os
import logging
from logging.handlers import RotatingFileHandler


def create_app(config_name=None):
    app = Flask(__name__)
    
    # Load config
    config_name = config_name or os.environ.get('FLASK_ENV', 'production')
    app.config.from_object(config[config_name])
    
    # Production logging
    if not app.debug and not app.testing:
        # ตั้งค่า logging
        if not os.path.exists('logs'):
            os.mkdir('logs')
        
        file_handler = RotatingFileHandler(
            'logs/myapp.log',
            maxBytes=10240000,  # 10 MB
            backupCount=10
        )
        file_handler.setFormatter(logging.Formatter(
            '%(asctime)s %(levelname)s: %(message)s '
            '[in %(pathname)s:%(lineno)d]'
        ))
        file_handler.setLevel(logging.INFO)
        app.logger.addHandler(file_handler)
        app.logger.setLevel(logging.INFO)
        app.logger.info('MyApp startup')
    
    return app
```

### Production wsgi.py
```python
# wsgi.py — Entry point สำหรับ production

import os
from dotenv import load_dotenv

# โหลด environment variables
load_dotenv('/var/www/myapp/.env')

from app import create_app

# สร้าง app สำหรับ production
application = create_app('production')

# Gunicorn ใช้ตัวแปรชื่อ 'application'
# รัน: gunicorn wsgi:application
```

---

## 7. SSL Certificate ด้วย Let's Encrypt

```bash
# ติดตั้ง Certbot
sudo apt install certbot python3-certbot-nginx

# สร้าง SSL certificate
sudo certbot --nginx -d example.com -d www.example.com

# Auto-renewal (Certbot ตั้งให้อัตโนมัติ)
sudo certbot renew --dry-run  # ทดสอบ renewal

# ดู certificates
sudo certbot certificates
```

---

## 8. Checklist สำหรับ Production

```python
# checklist.py — ตรวจสอบก่อน deploy

import os
import sys


def check_production_readiness():
    """ตรวจสอบว่าพร้อมสำหรับ production"""
    
    checks = []
    
    # 1. SECRET_KEY ต้องไม่ใช่ default
    secret_key = os.environ.get('SECRET_KEY', '')
    if not secret_key or secret_key in ['dev-secret', 'change-this', '']:
        checks.append('❌ SECRET_KEY ไม่ปลอดภัย')
    else:
        checks.append('✅ SECRET_KEY ตั้งแล้ว')
    
    # 2. DEBUG ต้องปิด
    debug = os.environ.get('FLASK_DEBUG', 'false').lower()
    if debug == 'true':
        checks.append('❌ DEBUG=True (ต้องปิดใน production)')
    else:
        checks.append('✅ DEBUG ปิดแล้ว')
    
    # 3. DATABASE_URL ต้องตั้ง
    db_url = os.environ.get('DATABASE_URL', '')
    if not db_url:
        checks.append('❌ DATABASE_URL ไม่ได้ตั้ง')
    elif 'sqlite' in db_url:
        checks.append('⚠️  ใช้ SQLite (แนะนำ PostgreSQL สำหรับ production)')
    else:
        checks.append('✅ DATABASE_URL ตั้งแล้ว')
    
    # 4. HTTPS
    checks.append('ℹ️  ตรวจสอบว่าตั้ง HTTPS และ SSL certificate แล้ว')
    
    # แสดงผล
    print('=== Production Readiness Check ===')
    for check in checks:
        print(check)
    
    failed = [c for c in checks if c.startswith('❌')]
    if failed:
        print(f'\n{len(failed)} ปัญหาที่ต้องแก้ไขก่อน deploy!')
        return False
    
    print('\n✅ พร้อม deploy!')
    return True


if __name__ == '__main__':
    success = check_production_readiness()
    sys.exit(0 if success else 1)
```

---

## 9. สรุป Part 085

✅ **Gunicorn** เป็น WSGI server ที่เหมาะสำหรับ production  
✅ **Workers** ควรตั้งเป็น 2×CPU+1 เพื่อ performance ที่ดี  
✅ **Nginx** ทำหน้าที่ reverse proxy, serve static files, และ SSL termination  
✅ **Systemd** จัดการ process ให้ auto-start และ auto-restart  
✅ **Let's Encrypt** ให้ SSL certificate ฟรี  
✅ **Production settings**: DEBUG=False, HTTPS only, proper logging  

---

## ➡️ ถัดไป: Part 086 - FastAPI Basics

*Part 085/100+ | Python Course - Beginner to World-Class*
