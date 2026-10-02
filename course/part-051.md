# Part 51: Django Introduction and Setup

## เป้าหมายของบทเรียน

- เข้าใจว่า Django คืออะไรและทำงานอย่างไร
- เข้าใจ MVT (Model-View-Template) Pattern
- ติดตั้ง Django และสร้าง project แรก
- เข้าใจโครงสร้างไฟล์ Django project ทุกไฟล์
- กำหนดค่า settings.py
- รัน development server และทดสอบใน browser
- สร้าง Hello World ด้วย Django

---

## 1. Django คืออะไร?

Django เป็น web framework สำหรับภาษา Python ที่พัฒนาโดย Adrian Holovaty และ Simon Willison ในปี 2003 และเปิดให้ใช้งานเป็น open source ในปี 2005

**จุดเด่นของ Django:**
- "Batteries included" - มีทุกอย่างในตัว
- DRY (Don't Repeat Yourself) principle
- Security ระดับสูง (ป้องกัน XSS, CSRF, SQL Injection)
- Scalable สูง (ใช้งานโดย Instagram, Pinterest, Disqus)
- Admin interface ที่สร้างอัตโนมัติ
- ORM (Object-Relational Mapping) อันทรงพลัง
- รองรับ PostgreSQL, MySQL, SQLite, Oracle

**เว็บไซต์ที่ใช้ Django:**
- Instagram
- Pinterest  
- Mozilla
- National Geographic
- Disqus
- Bitbucket

---

## 2. MVT Pattern

Django ใช้ pattern ที่เรียกว่า **MVT (Model-View-Template)** ซึ่งคล้ายกับ MVC แต่มีความต่างเล็กน้อย:

```
Browser Request
      |
      v
   URLs.py  ------>  View (views.py)
                          |
                    Template (.html)
                          |
                       Model
                          |
                       Database
```

### M - Model (โมเดล)
- เป็นตัวแทนของข้อมูลในฐานข้อมูล
- กำหนดโครงสร้างตาราง (schema)
- มี ORM ให้ query ข้อมูลได้โดยไม่ต้องเขียน SQL
- อยู่ใน `models.py`

### V - View (วิว)
- ทำหน้าที่เป็น "Controller" ใน MVC
- รับ request, ประมวลผล, และส่ง response กลับ
- อยู่ใน `views.py`

### T - Template (เทมเพลต)
- ทำหน้าที่แสดงผล HTML
- มี template language พิเศษของ Django
- อยู่ในโฟลเดอร์ `templates/`

**เปรียบเทียบ MVT vs MVC:**

| MVT (Django) | MVC (ทั่วไป) | หน้าที่ |
|---|---|---|
| Model | Model | จัดการข้อมูล |
| View | Controller | ประมวลผล logic |
| Template | View | แสดงผล |

---

## 3. ติดตั้ง Django

### สร้าง Virtual Environment

```bash
# สร้างโฟลเดอร์โปรเจค
mkdir myblog
cd myblog

# สร้าง virtual environment
python -m venv venv

# เปิดใช้งาน virtual environment
# บน Linux/Mac:
source venv/bin/activate

# บน Windows:
venv\Scripts\activate

# ตรวจสอบว่า venv เปิดอยู่ (จะเห็น (venv) นำหน้า prompt)
```

### ติดตั้ง Django

```bash
# ติดตั้ง Django เวอร์ชันล่าสุด
pip install django

# หรือระบุเวอร์ชัน
pip install django==4.2

# ตรวจสอบเวอร์ชัน
python -m django --version
# ควรได้ผลลัพธ์: 4.2.x หรือใหม่กว่า

# บันทึก dependencies
pip freeze > requirements.txt
```

### สร้าง Django Project

```bash
# สร้าง project ชื่อ mysite
django-admin startproject mysite

# โครงสร้างที่ได้:
# mysite/
# ├── manage.py
# └── mysite/
#     ├── __init__.py
#     ├── settings.py
#     ├── urls.py
#     ├── asgi.py
#     └── wsgi.py

# เข้าไปใน project
cd mysite
```

### รัน Development Server

```bash
# รัน server
python manage.py runserver

# จะเห็น output:
# Watching for file changes with StatReloader
# Performing system checks...
# System check identified no issues (0 silenced).
# October 02, 2026 - 10:00:00
# Django version 4.2, using settings 'mysite.settings'
# Starting development server at http://127.0.0.1:8000/
# Quit the server with CONTROL-C.

# เปิด browser ไปที่ http://127.0.0.1:8000/
# จะเห็นหน้า welcome ของ Django
```

---

## 4. Django Project Structure

```
mysite/                    # root directory
├── manage.py             # command-line utility
└── mysite/               # project package
    ├── __init__.py       # Python package marker
    ├── settings.py       # project settings
    ├── urls.py           # URL declarations (main)
    ├── asgi.py           # ASGI entry point
    └── wsgi.py           # WSGI entry point
```

### manage.py

```python
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys

def main():
    """Run administrative tasks."""
    # กำหนด settings module
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'mysite.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)

if __name__ == '__main__':
    main()
```

คำสั่งที่ใช้กับ manage.py:
```bash
python manage.py runserver          # รัน development server
python manage.py startapp myapp     # สร้าง app ใหม่
python manage.py makemigrations     # สร้างไฟล์ migration
python manage.py migrate            # apply migrations
python manage.py createsuperuser    # สร้าง admin user
python manage.py shell              # เปิด Python shell
python manage.py collectstatic      # รวม static files
python manage.py test               # รัน tests
python manage.py dbshell            # เปิด database shell
```

### __init__.py

ไฟล์ว่างที่บอกว่า directory นี้เป็น Python package

### wsgi.py

```python
"""
WSGI config for mysite project.
ใช้สำหรับ deploy ใน production environment ที่รองรับ WSGI
"""
import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'mysite.settings')

application = get_wsgi_application()
```

### asgi.py

```python
"""
ASGI config for mysite project.
ใช้สำหรับ deploy ที่รองรับ async (ASGI) เช่น Channels
"""
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'mysite.settings')

application = get_asgi_application()
```

---

## 5. settings.py Configuration

```python
"""
Django settings for mysite project.
"""

from pathlib import Path

# สร้าง path ไปยัง base directory ของ project
BASE_DIR = Path(__file__).resolve().parent.parent

# ==============================
# ความปลอดภัย
# ==============================

# Secret key สำหรับ cryptographic signing
# ห้ามเปิดเผยใน production!
SECRET_KEY = 'django-insecure-your-secret-key-here'

# เปิด debug mode ระหว่าง development
# ต้องปิดใน production!
DEBUG = True

# hosts ที่อนุญาตให้เข้าถึง
ALLOWED_HOSTS = []  # ใน production ใส่ domain จริง เช่น ['example.com', 'www.example.com']

# ==============================
# Application Definition
# ==============================

INSTALLED_APPS = [
    # Django built-in apps
    'django.contrib.admin',       # Admin interface
    'django.contrib.auth',        # Authentication system
    'django.contrib.contenttypes',# Content type framework
    'django.contrib.sessions',    # Session framework
    'django.contrib.messages',    # Messaging framework
    'django.contrib.staticfiles', # Static file management
    
    # เพิ่ม apps ของเราที่นี่
    # 'blog',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',    # Security headers
    'django.contrib.sessions.middleware.SessionMiddleware',  # Session handling
    'django.middleware.common.CommonMiddleware',        # Common HTTP operations
    'django.middleware.csrf.CsrfViewMiddleware',        # CSRF protection
    'django.contrib.auth.middleware.AuthenticationMiddleware',  # Authentication
    'django.contrib.messages.middleware.MessageMiddleware',     # Messages
    'django.middleware.clickjacking.XFrameOptionsMiddleware',   # Clickjacking protection
]

# กำหนดไฟล์ URLs หลักของ project
ROOT_URLCONF = 'mysite.urls'

# ==============================
# Templates Configuration
# ==============================

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],  # โฟลเดอร์ templates หลัก
        'APP_DIRS': True,  # ให้ค้นหา templates ใน app แต่ละตัวด้วย
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

WSGI_APPLICATION = 'mysite.wsgi.application'

# ==============================
# Database
# ==============================

# Default: SQLite (เหมาะสำหรับ development)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# PostgreSQL (สำหรับ production)
# DATABASES = {
#     'default': {
#         'ENGINE': 'django.db.backends.postgresql',
#         'NAME': 'mydb',
#         'USER': 'myuser',
#         'PASSWORD': 'mypassword',
#         'HOST': 'localhost',
#         'PORT': '5432',
#     }
# }

# ==============================
# Password Validation
# ==============================

AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

# ==============================
# Internationalization
# ==============================

LANGUAGE_CODE = 'th'        # ภาษาไทย
TIME_ZONE = 'Asia/Bangkok'  # timezone ไทย
USE_I18N = True             # เปิด internationalization
USE_TZ = True               # เปิด timezone-aware datetimes

# ==============================
# Static Files
# ==============================

STATIC_URL = 'static/'

# โฟลเดอร์ static files ที่ใช้ใน development
STATICFILES_DIRS = [
    BASE_DIR / 'static',
]

# โฟลเดอร์ที่ collectstatic จะรวมไฟล์ (ใช้ใน production)
# STATIC_ROOT = BASE_DIR / 'staticfiles'

# ==============================
# Media Files (ไฟล์ที่ผู้ใช้ upload)
# ==============================

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

# ==============================
# Default Primary Key
# ==============================

DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
```

---

## 6. สร้าง Django App

ใน Django project หนึ่งประกอบด้วย "apps" หลายตัว แต่ละ app ทำหน้าที่เฉพาะอย่าง

```bash
# สร้าง app ชื่อ blog
python manage.py startapp blog

# โครงสร้าง app:
# blog/
# ├── migrations/        # database migrations
# │   └── __init__.py
# ├── __init__.py        # Python package marker
# ├── admin.py           # Admin configuration
# ├── apps.py            # App configuration
# ├── models.py          # Data models
# ├── tests.py           # Unit tests
# └── views.py           # View functions
```

### ลงทะเบียน App ใน settings.py

```python
# mysite/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # เพิ่ม app ของเรา
    'blog',  # หรือใช้ 'blog.apps.BlogConfig' (แนะนำ)
]
```

### apps.py

```python
# blog/apps.py
from django.apps import AppConfig

class BlogConfig(AppConfig):
    # ชื่อ app
    name = 'blog'
    # ชื่อที่แสดงใน admin (ภาษาไทยได้)
    verbose_name = 'บล็อก'
    
    def ready(self):
        """
        ทำงานเมื่อ app พร้อมใช้งาน
        ใช้สำหรับ import signals เป็นต้น
        """
        pass  # import blog.signals
```

---

## 7. Hello World ด้วย Django

### Step 1: สร้าง View

```python
# blog/views.py
from django.http import HttpResponse
from django.shortcuts import render

# View แบบ Function-Based (FBV) อย่างง่าย
def hello_world(request):
    """
    View สำหรับแสดงข้อความ Hello World
    รับ request object และส่งกลับ HttpResponse
    """
    return HttpResponse('<h1>สวัสดีชาวโลก! Hello from Django!</h1>')


# View ที่ใช้ template
def index(request):
    """
    View หน้าแรกของ blog
    ส่ง context ไปยัง template
    """
    # ข้อมูลที่จะส่งไปยัง template
    context = {
        'title': 'ยินดีต้อนรับสู่บล็อกของเรา',
        'message': 'Django Web Framework',
        'version': '4.2',
    }
    # render() รับ request, template path, และ context
    return render(request, 'blog/index.html', context)
```

### Step 2: สร้าง URL Patterns ของ App

```python
# blog/urls.py (สร้างไฟล์ใหม่)
from django.urls import path
from . import views  # import views จาก app เดียวกัน

# namespace สำหรับ reverse URL
app_name = 'blog'

urlpatterns = [
    # path(route, view, name)
    path('', views.index, name='index'),           # /blog/
    path('hello/', views.hello_world, name='hello'),  # /blog/hello/
]
```

### Step 3: Include URLs ใน main urls.py

```python
# mysite/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    # include URLs จาก blog app
    path('blog/', include('blog.urls')),
    # หรือ include ที่ root URL
    # path('', include('blog.urls')),
]
```

### Step 4: สร้าง Template

```bash
# สร้างโฟลเดอร์ templates
mkdir -p blog/templates/blog
```

```html
<!-- blog/templates/blog/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <!-- แสดงค่า title จาก context -->
    <title>{{ title }}</title>
</head>
<body>
    <h1>{{ title }}</h1>
    <p>ยินดีต้อนรับสู่ {{ message }} version {{ version }}</p>
    
    <!-- Django template tag สำหรับ URL -->
    <a href="{% url 'blog:hello' %}">ทดสอบ Hello World</a>
</body>
</html>
```

### Step 5: ทดสอบ

```bash
# รัน server
python manage.py runserver

# เปิด browser ไปที่:
# http://127.0.0.1:8000/blog/        -> หน้า index
# http://127.0.0.1:8000/blog/hello/  -> Hello World
```

---

## 8. ตัวอย่าง Hello World แบบสมบูรณ์

สร้าง project ใหม่แบบเต็มรูปแบบ:

```bash
# สร้าง project
django-admin startproject hellodemo
cd hellodemo

# สร้าง app
python manage.py startapp pages

# สร้างโครงสร้าง templates
mkdir -p pages/templates/pages
mkdir -p static/css
```

### pages/views.py

```python
# pages/views.py
from django.shortcuts import render
from django.http import HttpResponse
import datetime

def home(request):
    """
    หน้าแรกของเว็บไซต์
    แสดงข้อมูล welcome และเวลาปัจจุบัน
    """
    # เวลาปัจจุบัน
    now = datetime.datetime.now()
    
    context = {
        'page_title': 'หน้าแรก - Hello Django',
        'heading': 'ยินดีต้อนรับสู่ Django!',
        'current_time': now.strftime('%d/%m/%Y %H:%M:%S'),
        'features': [
            'MVT Pattern ที่ทรงพลัง',
            'ORM สำหรับจัดการฐานข้อมูล',
            'Admin Interface อัตโนมัติ',
            'Security ระดับสูง',
            'Community ขนาดใหญ่',
        ],
    }
    return render(request, 'pages/home.html', context)


def about(request):
    """
    หน้า About
    """
    context = {
        'page_title': 'เกี่ยวกับเรา',
        'team_members': [
            {'name': 'สมชาย ใจดี', 'role': 'Backend Developer'},
            {'name': 'สมหญิง รักงาน', 'role': 'Frontend Developer'},
            {'name': 'มานะ พัฒนา', 'role': 'Full Stack Developer'},
        ]
    }
    return render(request, 'pages/about.html', context)


def api_info(request):
    """
    ตัวอย่าง JSON response
    """
    import json
    data = {
        'status': 'success',
        'message': 'Django API is working!',
        'version': '1.0',
        'timestamp': str(datetime.datetime.now()),
    }
    return HttpResponse(
        json.dumps(data, ensure_ascii=False),
        content_type='application/json'
    )
```

### pages/urls.py

```python
# pages/urls.py
from django.urls import path
from . import views

app_name = 'pages'

urlpatterns = [
    path('', views.home, name='home'),
    path('about/', views.about, name='about'),
    path('api/info/', views.api_info, name='api_info'),
]
```

### hellodemo/urls.py

```python
# hellodemo/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('pages.urls')),  # include ที่ root
]
```

### hellodemo/settings.py (เพิ่ม app)

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # app ของเรา
    'pages',
]
```

### pages/templates/pages/home.html

```html
<!-- pages/templates/pages/home.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ page_title }}</title>
    {% load static %}
    <style>
        /* CSS inline สำหรับตัวอย่าง */
        body {
            font-family: 'Sarabun', Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            background-color: #f5f5f5;
        }
        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 40px;
            border-radius: 10px;
            text-align: center;
            margin-bottom: 30px;
        }
        .features {
            background: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        .features li {
            padding: 8px 0;
            border-bottom: 1px solid #eee;
        }
        .nav {
            margin: 20px 0;
        }
        .nav a {
            margin-right: 15px;
            color: #667eea;
            text-decoration: none;
            font-weight: bold;
        }
        .time-badge {
            background: rgba(255,255,255,0.2);
            padding: 5px 15px;
            border-radius: 20px;
            font-size: 0.9em;
            margin-top: 10px;
            display: inline-block;
        }
    </style>
</head>
<body>
    <!-- Navigation -->
    <nav class="nav">
        <a href="{% url 'pages:home' %}">หน้าแรก</a>
        <a href="{% url 'pages:about' %}">เกี่ยวกับเรา</a>
        <a href="{% url 'pages:api_info' %}">API Info</a>
    </nav>
    
    <!-- Header -->
    <div class="header">
        <h1>{{ heading }}</h1>
        <p>เว็บแอปพลิเคชันแรกของเราด้วย Django</p>
        <!-- แสดงเวลาจาก context -->
        <div class="time-badge">⏰ {{ current_time }}</div>
    </div>
    
    <!-- Features List -->
    <div class="features">
        <h2>✨ คุณสมบัติของ Django</h2>
        <!-- วนลูปแสดง features จาก context -->
        <ul>
            {% for feature in features %}
            <li>✅ {{ feature }}</li>
            {% endfor %}
        </ul>
    </div>
    
    <!-- Admin link -->
    <p style="margin-top: 20px; text-align: center;">
        <a href="/admin/" style="color: #764ba2;">เข้าสู่ Admin Interface →</a>
    </p>
</body>
</html>
```

### pages/templates/pages/about.html

```html
<!-- pages/templates/pages/about.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{{ page_title }}</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
        .team-card {
            background: white;
            border: 1px solid #ddd;
            border-radius: 8px;
            padding: 20px;
            margin: 10px 0;
        }
        .role { color: #666; font-style: italic; }
    </style>
</head>
<body>
    <h1>{{ page_title }}</h1>
    
    <h2>ทีมงานของเรา</h2>
    
    <!-- วนลูปแสดงสมาชิกทีม -->
    {% for member in team_members %}
    <div class="team-card">
        <h3>{{ member.name }}</h3>
        <p class="role">{{ member.role }}</p>
    </div>
    {% empty %}
    <!-- แสดงเมื่อ list ว่างเปล่า -->
    <p>ยังไม่มีสมาชิกในทีม</p>
    {% endfor %}
    
    <a href="{% url 'pages:home' %}">← กลับหน้าแรก</a>
</body>
</html>
```

### รัน Migrations และ Superuser

```bash
# apply migrations เริ่มต้น
python manage.py migrate

# สร้าง admin superuser
python manage.py createsuperuser
# Username: admin
# Email: admin@example.com
# Password: xxxxxxxx

# รัน server
python manage.py runserver
```

---

## 9. Django Admin Setup

```bash
# เปิด browser ไปที่ http://127.0.0.1:8000/admin/
# ใส่ username และ password ที่สร้างไว้
```

---

## 10. URL Patterns เพิ่มเติม

```python
# hellodemo/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('pages.urls')),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
# เพิ่ม static() ใน development เพื่อ serve media files
```

---

## 11. การใช้ Django Shell

```bash
# เปิด Django interactive shell
python manage.py shell

# ใน shell:
>>> import django
>>> django.__version__
'4.2.x'

>>> from django.conf import settings
>>> settings.DEBUG
True

>>> from django.test.utils import setup_test_environment
>>> setup_test_environment()

# ออกจาก shell
>>> exit()
```

---

## 12. Best Practices สำหรับ Django Project

### โครงสร้าง Project แบบ Production

```
myproject/
├── config/              # แยก config ออกมา
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py      # base settings
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
├── apps/                # แยก apps ไว้ในโฟลเดอร์เดียวกัน
│   ├── blog/
│   ├── users/
│   └── shop/
├── static/              # static files
├── media/               # media files (uploads)
├── templates/           # global templates
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   └── production.txt
├── .env                 # environment variables (ห้าม commit!)
├── .gitignore
└── manage.py
```

### .gitignore สำหรับ Django

```
# Python
__pycache__/
*.py[cod]
*.pyo
*.pyd
.Python
*.egg
*.egg-info/
dist/
build/

# Virtual Environment
venv/
env/
.venv/

# Django
*.log
local_settings.py
db.sqlite3
db.sqlite3-journal
media/

# Static files
staticfiles/
static_collected/

# Environment variables
.env
.env.local

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db
```

### ใช้ Environment Variables

```bash
# ติดตั้ง python-decouple
pip install python-decouple
```

```python
# mysite/settings.py
from decouple import config, Csv

# ดึงค่าจาก .env file หรือ environment variable
SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='localhost', cast=Csv())

DATABASE_URL = config('DATABASE_URL', default='sqlite:///db.sqlite3')
```

```bash
# .env file
SECRET_KEY=your-super-secret-key-here
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=sqlite:///db.sqlite3
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Personal Portfolio
สร้าง Django project สำหรับ personal portfolio website ที่มี:
- หน้า Home แสดงชื่อและคำแนะนำตัว
- หน้า Projects แสดงรายการ projects (เป็น list ใน context)
- หน้า Contact แสดงข้อมูลติดต่อ
- Navigation ที่ใช้ `{% url %}` tag

### แบบฝึกหัดที่ 2: ศึกษา Django Settings
ลองเปลี่ยนค่าใน settings.py และสังเกตผลลัพธ์:
1. เปลี่ยน `LANGUAGE_CODE` เป็น `'en-us'` และ `'th'` - สังเกตความต่างใน admin
2. เปลี่ยน `TIME_ZONE` เป็น `'UTC'` และ `'Asia/Bangkok'`
3. เพิ่ม custom middleware และดูว่ามันทำงานอย่างไร

### แบบฝึกหัดที่ 3: สร้าง Multiple Apps
สร้าง Django project ที่มี 2 apps:
- `blog` app: หน้าแสดงบทความ (รายการ + รายละเอียด)
- `contact` app: หน้าติดต่อเรา

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- Django คืออะไรและทำไมต้องใช้
- MVT Pattern และความแตกต่างจาก MVC
- การติดตั้ง Django และสร้าง project
- โครงสร้างไฟล์ทุกไฟล์ใน Django project
- การกำหนดค่า settings.py
- การสร้าง app และลงทะเบียน
- การสร้าง View, URL, และ Template เบื้องต้น
- Best practices สำหรับ Django project

---

## บทถัดไป

➡️ **[Part 52: Django Models](part-052.md)** - เรียนรู้การสร้าง Model, Field types, ORM queries, และ QuerySet API
