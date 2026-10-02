# Part 084: Flask Configuration
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- จัดการ configuration ด้วย Config classes
- แยก config ตาม environment (dev/prod/test)
- ใช้ environment variables สำหรับ secrets
- ใช้ python-dotenv กับ Flask
- Best practices สำหรับ configuration

---

## 1. วิธีตั้งค่า Flask Configuration

### วิธีที่ 1: ตั้งตรงๆ (ไม่แนะนำสำหรับ production)
```python
app = Flask(__name__)
app.config['SECRET_KEY'] = 'my-secret-key'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///myapp.db'
app.config['DEBUG'] = True
```

### วิธีที่ 2: จาก object/class (แนะนำ)
```python
class Config:
    SECRET_KEY = 'my-secret'
    DEBUG = False

app.config.from_object(Config)
```

### วิธีที่ 3: จากไฟล์ .py
```python
app.config.from_pyfile('config.py')
```

### วิธีที่ 4: จาก environment variables
```python
app.config.from_envvar('MYAPP_SETTINGS')  # ไฟล์ที่กำหนดใน env var
```

---

## 2. Config Classes Pattern

```python
# config.py

import os
from datetime import timedelta


class Config:
    """Base configuration — ใช้ร่วมกันทุก environment"""
    
    # Application
    APP_NAME = 'MyApp'
    APP_VERSION = '1.0.0'
    
    # Security
    SECRET_KEY = os.environ.get('SECRET_KEY', 'fallback-secret-key')
    WTF_CSRF_ENABLED = True
    
    # Database
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SQLALCHEMY_ECHO = False
    
    # Session
    PERMANENT_SESSION_LIFETIME = timedelta(hours=24)
    SESSION_COOKIE_SECURE = False
    SESSION_COOKIE_HTTPONLY = True
    SESSION_COOKIE_SAMESITE = 'Lax'
    
    # JWT
    JWT_SECRET_KEY = os.environ.get('JWT_SECRET_KEY', SECRET_KEY)
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(hours=1)
    JWT_REFRESH_TOKEN_EXPIRES = timedelta(days=30)
    
    # Email
    MAIL_SERVER = os.environ.get('MAIL_SERVER', 'localhost')
    MAIL_PORT = int(os.environ.get('MAIL_PORT', 587))
    MAIL_USE_TLS = True
    MAIL_USERNAME = os.environ.get('MAIL_USERNAME')
    MAIL_PASSWORD = os.environ.get('MAIL_PASSWORD')
    MAIL_DEFAULT_SENDER = os.environ.get('MAIL_DEFAULT_SENDER', 'noreply@myapp.com')
    
    # File Upload
    MAX_CONTENT_LENGTH = 16 * 1024 * 1024  # 16 MB
    UPLOAD_FOLDER = os.path.join(os.path.dirname(__file__), 'uploads')
    ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif', 'pdf'}
    
    # Pagination
    POSTS_PER_PAGE = 10
    USERS_PER_PAGE = 20
    
    # Cache
    CACHE_TYPE = 'SimpleCache'
    CACHE_DEFAULT_TIMEOUT = 300  # 5 นาที
    
    @staticmethod
    def init_app(app):
        """เรียกเมื่อสร้าง app (override ใน subclass ได้)"""
        pass


class DevelopmentConfig(Config):
    """Development environment"""
    
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = os.environ.get(
        'DEV_DATABASE_URL',
        'sqlite:///dev.db'
    )
    SQLALCHEMY_ECHO = True  # แสดง SQL queries ใน console
    
    # ปิด CSRF ใน dev (อาจเปิดก็ได้)
    WTF_CSRF_ENABLED = True
    
    # Email: ใช้ mailtrap หรือ console backend
    MAIL_SERVER = 'smtp.mailtrap.io'
    MAIL_PORT = 2525
    MAIL_USE_TLS = True
    MAIL_USERNAME = os.environ.get('MAILTRAP_USERNAME')
    MAIL_PASSWORD = os.environ.get('MAILTRAP_PASSWORD')
    
    # Logging
    LOG_LEVEL = 'DEBUG'
    
    @staticmethod
    def init_app(app):
        print('Running in Development mode')


class TestingConfig(Config):
    """Testing environment"""
    
    TESTING = True
    DEBUG = True
    
    # ใช้ in-memory database
    SQLALCHEMY_DATABASE_URI = 'sqlite:///:memory:'
    
    # ปิด CSRF ใน tests
    WTF_CSRF_ENABLED = False
    
    # ปิด rate limiting ใน tests
    RATELIMIT_ENABLED = False
    
    # ปิด email จริงใน tests
    MAIL_SUPPRESS_SEND = True
    
    # ใช้ simple cache ไม่มี expiry ใน tests
    CACHE_TYPE = 'SimpleCache'
    CACHE_DEFAULT_TIMEOUT = 0
    
    LOG_LEVEL = 'ERROR'


class ProductionConfig(Config):
    """Production environment"""
    
    DEBUG = False
    TESTING = False
    
    # Database จาก environment variable เท่านั้น
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL')
    
    # Security เข้มงวดขึ้น
    SESSION_COOKIE_SECURE = True   # HTTPS only
    REMEMBER_COOKIE_SECURE = True
    WTF_CSRF_SSL_STRICT = True
    
    # Connection pooling สำหรับ production
    SQLALCHEMY_ENGINE_OPTIONS = {
        'pool_size': 10,
        'pool_recycle': 3600,
        'pool_pre_ping': True,
        'max_overflow': 20
    }
    
    # Logging
    LOG_LEVEL = 'WARNING'
    
    @classmethod
    def init_app(cls, app):
        Config.init_app(app)
        
        # ตรวจสอบ required env vars ตอน startup
        required_vars = ['SECRET_KEY', 'DATABASE_URL']
        missing = [var for var in required_vars if not os.environ.get(var)]
        if missing:
            raise ValueError(f'ต้องตั้ง environment variables: {", ".join(missing)}')
        
        # Log ไปไฟล์ใน production
        import logging
        from logging.handlers import RotatingFileHandler
        
        if not app.debug:
            handler = RotatingFileHandler(
                'app.log',
                maxBytes=10 * 1024 * 1024,  # 10 MB
                backupCount=5
            )
            handler.setLevel(logging.WARNING)
            app.logger.addHandler(handler)


class StagingConfig(ProductionConfig):
    """Staging environment (คล้าย production แต่ไม่ใช่ production จริง)"""
    
    DEBUG = False
    SQLALCHEMY_DATABASE_URI = os.environ.get('STAGING_DATABASE_URL')
    LOG_LEVEL = 'INFO'


# Map ชื่อ config
config = {
    'development': DevelopmentConfig,
    'testing': TestingConfig,
    'production': ProductionConfig,
    'staging': StagingConfig,
    'default': DevelopmentConfig
}
```

---

## 3. App Factory ใช้ Config

```python
# app/__init__.py

import os
from flask import Flask
from .config import config


def create_app(config_name=None):
    """App Factory"""
    if config_name is None:
        # อ่านจาก environment variable
        config_name = os.environ.get('FLASK_ENV', 'default')
    
    app = Flask(__name__)
    
    # โหลด config
    app.config.from_object(config[config_name])
    config[config_name].init_app(app)
    
    # สร้างโฟลเดอร์ที่จำเป็น
    _ensure_dirs(app)
    
    # Initialize extensions
    _init_extensions(app)
    
    # Register blueprints
    _register_blueprints(app)
    
    return app


def _ensure_dirs(app):
    """สร้างโฟลเดอร์ที่จำเป็นถ้ายังไม่มี"""
    dirs = [
        app.config.get('UPLOAD_FOLDER', 'uploads'),
        'logs'
    ]
    for d in dirs:
        os.makedirs(d, exist_ok=True)
```

---

## 4. Environment Variables ด้วย python-dotenv

### ติดตั้ง
```bash
pip install python-dotenv
```

### ไฟล์ .env
```bash
# .env — ไม่ commit ไป git!

# Application
FLASK_ENV=development
SECRET_KEY=dev-secret-key-abc123

# Database
DATABASE_URL=postgresql://user:password@localhost/myapp

# JWT
JWT_SECRET_KEY=jwt-secret-key-xyz789

# Email
MAIL_SERVER=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=myapp@gmail.com
MAIL_PASSWORD=app-specific-password

# Third-party APIs
STRIPE_API_KEY=sk_test_xxxxx
SENDGRID_API_KEY=SG.xxxxx
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=xxxxx
AWS_S3_BUCKET=my-app-bucket
```

### .env.example (commit ไปได้)
```bash
# .env.example — template สำหรับ .env

# Application
FLASK_ENV=development
SECRET_KEY=change-this-to-random-string

# Database
DATABASE_URL=sqlite:///myapp.db

# JWT
JWT_SECRET_KEY=change-this-jwt-secret

# Email
MAIL_SERVER=smtp.example.com
MAIL_PORT=587
MAIL_USERNAME=
MAIL_PASSWORD=

# Third-party APIs
STRIPE_API_KEY=
SENDGRID_API_KEY=
```

### โหลด .env ใน Flask
```python
# run.py

from dotenv import load_dotenv
import os

# โหลด .env ก่อน import Flask
load_dotenv()

from app import create_app

app = create_app(os.environ.get('FLASK_ENV', 'development'))

if __name__ == '__main__':
    app.run()
```

### ใน config.py
```python
# config.py

import os

# python-dotenv โหลดแล้วตั้งแต่ run.py
# ดึงค่าจาก environment variables
class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY')
    DATABASE_URL = os.environ.get('DATABASE_URL')
    
    # Fallback values สำหรับ development
    if not SECRET_KEY:
        SECRET_KEY = 'fallback-dev-only-secret'
        print('WARNING: Using default SECRET_KEY. Set SECRET_KEY env var in production!')
```

---

## 5. Configuration Validation

```python
# config.py

import os
from typing import Optional


class ConfigValidator:
    """ตรวจสอบ configuration"""
    
    REQUIRED_IN_PRODUCTION = [
        'SECRET_KEY',
        'DATABASE_URL',
        'JWT_SECRET_KEY',
    ]
    
    RECOMMENDED = [
        'MAIL_USERNAME',
        'MAIL_PASSWORD',
    ]
    
    @classmethod
    def validate(cls, config_obj, env='development'):
        """ตรวจสอบ config และแสดง warnings"""
        errors = []
        warnings = []
        
        if env == 'production':
            for key in cls.REQUIRED_IN_PRODUCTION:
                value = os.environ.get(key) or getattr(config_obj, key, None)
                if not value:
                    errors.append(f'Missing required: {key}')
        
        for key in cls.RECOMMENDED:
            value = os.environ.get(key) or getattr(config_obj, key, None)
            if not value:
                warnings.append(f'Recommended but missing: {key}')
        
        # ตรวจสอบ SECRET_KEY ไม่ใช่ default
        secret_key = getattr(config_obj, 'SECRET_KEY', '')
        if secret_key in ['change-this', 'secret', 'dev-secret-key', '']:
            if env == 'production':
                errors.append('SECRET_KEY ต้องเปลี่ยนจาก default!')
            else:
                warnings.append('SECRET_KEY ใช้ค่า default (OK สำหรับ dev)')
        
        if errors:
            raise ValueError('Configuration errors:\n' + '\n'.join(errors))
        
        for w in warnings:
            print(f'Config Warning: {w}')
```

---

## 6. Dynamic Configuration

```python
# config_manager.py — อัปเดต config ขณะ runtime

from flask import current_app


class FeatureFlags:
    """จัดการ feature flags"""
    
    _flags = {}
    
    @classmethod
    def enable(cls, feature):
        cls._flags[feature] = True
    
    @classmethod
    def disable(cls, feature):
        cls._flags[feature] = False
    
    @classmethod
    def is_enabled(cls, feature, default=False):
        return cls._flags.get(feature, default)


# ใช้ใน route
@app.route('/checkout')
def checkout():
    if FeatureFlags.is_enabled('new_checkout'):
        return render_template('checkout_v2.html')
    return render_template('checkout.html')
```

---

## 7. ตัวอย่าง Complete Configuration

```python
# config.py — Production ready

import os
import logging
from datetime import timedelta
from dotenv import load_dotenv

load_dotenv()


def get_env(key, default=None, required=False):
    """Helper สำหรับดึง env var"""
    value = os.environ.get(key, default)
    if required and not value:
        raise EnvironmentError(f'Environment variable {key} is required')
    return value


class BaseConfig:
    # App
    SECRET_KEY = get_env('SECRET_KEY', required=True) if os.environ.get('FLASK_ENV') == 'production' else get_env('SECRET_KEY', 'dev-only-secret')
    DEBUG = False
    TESTING = False
    
    # Database
    SQLALCHEMY_DATABASE_URI = get_env('DATABASE_URL', 'sqlite:///app.db')
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SQLALCHEMY_ENGINE_OPTIONS = {'pool_pre_ping': True}
    
    # Security
    WTF_CSRF_ENABLED = True
    WTF_CSRF_TIME_LIMIT = 3600
    SESSION_COOKIE_HTTPONLY = True
    SESSION_COOKIE_SAMESITE = 'Lax'
    
    # JWT
    JWT_SECRET_KEY = get_env('JWT_SECRET_KEY', SECRET_KEY)
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(hours=1)
    JWT_REFRESH_TOKEN_EXPIRES = timedelta(days=30)
    
    # Pagination
    DEFAULT_PAGE_SIZE = 20
    MAX_PAGE_SIZE = 100
    
    # File Upload
    MAX_CONTENT_LENGTH = 16 * 1024 * 1024
    UPLOAD_FOLDER = get_env('UPLOAD_FOLDER', 'uploads')
    
    @classmethod
    def from_env(cls):
        """Factory method: เลือก config ตาม FLASK_ENV"""
        env = os.environ.get('FLASK_ENV', 'development')
        configs = {
            'development': DevelopmentConfig,
            'testing': TestingConfig,
            'production': ProductionConfig,
        }
        return configs.get(env, DevelopmentConfig)


class DevelopmentConfig(BaseConfig):
    DEBUG = True
    SQLALCHEMY_ECHO = True
    LOG_LEVEL = logging.DEBUG


class TestingConfig(BaseConfig):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = 'sqlite:///:memory:'
    WTF_CSRF_ENABLED = False
    LOG_LEVEL = logging.ERROR


class ProductionConfig(BaseConfig):
    SESSION_COOKIE_SECURE = True
    REMEMBER_COOKIE_SECURE = True
    LOG_LEVEL = logging.WARNING
    
    SQLALCHEMY_ENGINE_OPTIONS = {
        'pool_size': 10,
        'pool_recycle': 3600,
        'pool_pre_ping': True,
        'max_overflow': 20,
    }
```

---

## 8. .gitignore สำหรับ Configuration

```
# .gitignore

# Environment variables (สำคัญมาก!)
.env
.env.local
.env.production
*.env

# Database files
*.db
*.sqlite3

# Upload files
uploads/
media/

# Logs
logs/
*.log

# Python
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
env/
venv/
.venv/
pip-log.txt

# Flask
instance/
.webassets-cache

# Testing
.pytest_cache/
.coverage
htmlcov/
```

---

## 9. สรุป Part 084

✅ **Config classes** แยก configuration ตาม environment อย่างชัดเจน  
✅ **Environment variables** เก็บ secrets ไม่ให้อยู่ใน code  
✅ **python-dotenv** โหลด .env file ได้ง่าย  
✅ **Validation** ตรวจสอบ config ตอน startup ป้องกัน deploy ที่มีปัญหา  
✅ **Never commit .env** ใช้ .env.example แทน  
✅ **App Factory** รับ config_name เพื่อสร้าง app ด้วย config ที่ต้องการ  

---

## ➡️ ถัดไป: Part 085 - Flask Deployment

*Part 084/100+ | Python Course - Beginner to World-Class*
