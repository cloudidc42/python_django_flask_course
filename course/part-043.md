# Part 043: Docker สำหรับ Python
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Docker concepts
- เขียน Dockerfile สำหรับ Python apps
- Docker Compose สำหรับหลาย services
- Best practices สำหรับ production
- Docker กับ Django, Flask, FastAPI

---

## 1. Docker Concepts

```
Container = สภาพแวดล้อมที่แยกออกมา (ไม่ขึ้นกับ host OS)
Image = template สำหรับสร้าง container
Dockerfile = คำสั่งสร้าง image
Docker Hub = registry เก็บ images
Docker Compose = จัดการหลาย containers
```

---

## 2. Dockerfile สำหรับ FastAPI

```dockerfile
# Dockerfile
FROM python:3.11-slim

# สร้าง non-root user (security best practice)
RUN addgroup --system app && adduser --system --group app

# ตั้ง working directory
WORKDIR /app

# Copy requirements ก่อน (Docker layer caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy โค้ด
COPY . .

# เปลี่ยนเป็น non-root user
USER app

# Expose port
EXPOSE 8000

# รัน app
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 3. Multi-stage Build (Production)

```dockerfile
# Dockerfile.prod
# Stage 1: Builder
FROM python:3.11 AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Final image (เล็กกว่า)
FROM python:3.11-slim

# Copy packages จาก builder
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

WORKDIR /app
COPY . .

RUN addgroup --system app && adduser --system --group app
USER app

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

---

## 4. .dockerignore

```
# .dockerignore
.git/
.venv/
venv/
__pycache__/
*.pyc
*.pyo
*.egg-info/
.env
.env.local
*.log
.pytest_cache/
htmlcov/
dist/
build/
node_modules/
README.md
docs/
tests/
```

---

## 5. Docker Compose สำหรับ Full Stack App

```yaml
# docker-compose.yml
version: '3.9'

services:
  # FastAPI/Django App
  web:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/myapp
      - REDIS_URL=redis://redis:6379
      - SECRET_KEY=your-secret-key
      - DEBUG=False
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - ./:/app  # development only
    restart: unless-stopped

  # PostgreSQL
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis (สำหรับ cache/celery)
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  # Celery Worker
  celery:
    build: .
    command: celery -A tasks worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  # Nginx (reverse proxy)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./static:/static
      - ./media:/media
    depends_on:
      - web

volumes:
  postgres_data:
  redis_data:
```

---

## 6. nginx.conf

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream app {
        server web:8000;
    }

    server {
        listen 80;
        server_name example.com;

        # Static files
        location /static/ {
            alias /static/;
        }

        location /media/ {
            alias /media/;
        }

        # API proxy
        location / {
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

---

## 7. Docker Compose Commands

```bash
# Build images
docker-compose build

# รัน services
docker-compose up -d          # background
docker-compose up             # foreground (เห็น logs)

# หยุด
docker-compose down
docker-compose down -v        # ลบ volumes ด้วย

# ดู logs
docker-compose logs -f web    # follow logs ของ web
docker-compose logs -f        # ทุก services

# รัน command ใน container
docker-compose exec web bash
docker-compose exec web python manage.py migrate

# Build + Run
docker-compose up --build

# Scale
docker-compose up --scale web=3

# ดู running services
docker-compose ps
```

---

## 8. Environment Variables

```bash
# .env (ไม่ commit)
DATABASE_URL=postgresql://postgres:password@db:5432/myapp
SECRET_KEY=super-secret-key-here
DEBUG=False
REDIS_URL=redis://redis:6379
ALLOWED_HOSTS=example.com,www.example.com

# .env.example (commit ได้)
DATABASE_URL=postgresql://user:password@localhost/dbname
SECRET_KEY=change-this-in-production
DEBUG=True
REDIS_URL=redis://localhost:6379
ALLOWED_HOSTS=localhost,127.0.0.1
```

```python
# settings.py / config.py
import os
from pathlib import Path
from dotenv import load_dotenv  # pip install python-dotenv

load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./dev.db")
SECRET_KEY = os.getenv("SECRET_KEY", "dev-secret-key")
DEBUG = os.getenv("DEBUG", "True").lower() == "true"
```

---

## 9. Django กับ Docker

```dockerfile
# Dockerfile.django
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN python manage.py collectstatic --noinput

EXPOSE 8000
CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

```yaml
# docker-compose.django.yml
services:
  web:
    build:
      dockerfile: Dockerfile.django
    command: >
      sh -c "python manage.py migrate &&
             python manage.py collectstatic --noinput &&
             gunicorn myproject.wsgi:application --bind 0.0.0.0:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:pass@db:5432/myapp
    depends_on:
      - db
```

---

## 10. สรุป Part 043

✅ **Dockerfile** - สร้าง image สำหรับ Python app  
✅ **Multi-stage build** - image เล็กสำหรับ production  
✅ **.dockerignore** - ไม่ copy ไฟล์ที่ไม่จำเป็น  
✅ **Docker Compose** - จัดการหลาย services  
✅ **Environment variables** - config management  
✅ **Nginx** - reverse proxy  
✅ **Best practices** - non-root user, health checks  

---

## ➡️ ถัดไป: Part 044 - CI/CD Pipeline

*Part 043/100+ | Python Course - Beginner to World-Class*
