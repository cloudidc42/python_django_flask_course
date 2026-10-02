# Part 071: Django Middleware

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจแนวคิด Middleware ใน Django
- รู้จัก built-in middlewares
- สร้าง Custom Middleware
- เข้าใจ request/response processing pipeline
- ตัวอย่าง authentication middleware

---

## 1. Middleware คืออะไร?

Middleware เป็น layer ที่อยู่ระหว่าง request กับ view (และระหว่าง view กับ response)

```
Request → Middleware 1 → Middleware 2 → Middleware 3 → View
                                                         |
Response ← Middleware 1 ← Middleware 2 ← Middleware 3 ←

Onion Model:
┌─────────────────────────────────────────┐
│  SecurityMiddleware                     │
│  ┌───────────────────────────────────┐  │
│  │  SessionMiddleware                │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │  AuthenticationMiddleware   │  │  │
│  │  │  ┌───────────────────────┐  │  │  │
│  │  │  │       View            │  │  │  │
│  │  │  └───────────────────────┘  │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

---

## 2. Built-in Middlewares

```python
# settings.py
MIDDLEWARE = [
    # ความปลอดภัย: HTTPS, HSTS, X-Frame-Options
    'django.middleware.security.SecurityMiddleware',
    
    # จัดการ session
    'django.contrib.sessions.middleware.SessionMiddleware',
    
    # Content-Type negotiation, URL normalization
    'django.middleware.common.CommonMiddleware',
    
    # CSRF protection
    'django.middleware.csrf.CsrfViewMiddleware',
    
    # กำหนด request.user จาก session
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    
    # Messages framework
    'django.contrib.messages.middleware.MessageMiddleware',
    
    # Clickjacking protection (X-Frame-Options header)
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

### SecurityMiddleware

```python
# settings.py
# SecurityMiddleware settings
SECURE_HSTS_SECONDS = 31536000          # 1 ปี
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_SSL_REDIRECT = True              # redirect HTTP → HTTPS
SECURE_CONTENT_TYPE_NOSNIFF = True      # X-Content-Type-Options: nosniff
X_FRAME_OPTIONS = 'DENY'               # ป้องกัน Clickjacking
SECURE_BROWSER_XSS_FILTER = True        # X-XSS-Protection header
```

---

## 3. Custom Middleware

### Function-based Middleware (แนะนำ - Django 1.10+)

```python
# middleware.py

def simple_middleware(get_response):
    """
    Function-based middleware รูปแบบ decorator
    
    get_response: callable ที่เรียก middleware ถัดไปหรือ view
    """
    
    # One-time setup code (เรียกครั้งเดียวตอน startup)
    print('Middleware initialized')
    
    def middleware(request):
        # ===== ก่อน view =====
        # แก้ไข request ได้ที่นี่
        
        # เรียก view หรือ middleware ถัดไป
        response = get_response(request)
        
        # ===== หลัง view =====
        # แก้ไข response ได้ที่นี่
        
        return response
    
    return middleware
```

### Class-based Middleware

```python
class LoggingMiddleware:
    """Middleware บันทึก request/response log"""
    
    def __init__(self, get_response):
        self.get_response = get_response
        # Setup ครั้งเดียว
        import logging
        self.logger = logging.getLogger('django.request')
    
    def __call__(self, request):
        """เรียกทุก request"""
        import time
        
        # ===== ก่อน view =====
        start_time = time.time()
        
        # Log request
        self.logger.info(
            f'Request: {request.method} {request.path} '
            f'from {self.get_client_ip(request)}'
        )
        
        # เรียก view
        response = self.get_response(request)
        
        # ===== หลัง view =====
        duration = time.time() - start_time
        
        # Log response
        self.logger.info(
            f'Response: {response.status_code} '
            f'for {request.path} '
            f'in {duration:.3f}s'
        )
        
        # เพิ่ม header ลง response
        response['X-Response-Time'] = f'{duration:.3f}s'
        
        return response
    
    def get_client_ip(self, request):
        """ดึง IP จาก request"""
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            return x_forwarded_for.split(',')[0].strip()
        return request.META.get('REMOTE_ADDR', '')
    
    def process_exception(self, request, exception):
        """เรียกเมื่อ view raise exception"""
        self.logger.error(
            f'Exception in {request.path}: {exception}',
            exc_info=True
        )
        # return None เพื่อให้ exception propagate ต่อ
        # return response เพื่อ handle เอง
        return None
    
    def process_template_response(self, request, response):
        """เรียกเมื่อ view return TemplateResponse (lazy rendering)"""
        # แก้ไข template context ได้ที่นี่
        return response
```

---

## 4. ตัวอย่าง Custom Middlewares

### Rate Limiting Middleware

```python
# middleware.py
from django.core.cache import cache
from django.http import JsonResponse
import time

class RateLimitMiddleware:
    """
    จำกัดจำนวน requests ต่อ IP
    Default: 100 requests per minute
    """
    
    def __init__(self, get_response):
        self.get_response = get_response
        self.rate_limit = getattr(settings, 'RATE_LIMIT_REQUESTS', 100)
        self.rate_limit_window = getattr(settings, 'RATE_LIMIT_WINDOW', 60)  # seconds
    
    def __call__(self, request):
        # เฉพาะ API endpoints
        if not request.path.startswith('/api/'):
            return self.get_response(request)
        
        ip = self.get_client_ip(request)
        cache_key = f'rate_limit_{ip}'
        
        # ดึงจำนวน requests ปัจจุบัน
        requests_count = cache.get(cache_key, 0)
        
        if requests_count >= self.rate_limit:
            return JsonResponse(
                {
                    'error': 'Too Many Requests',
                    'message': f'เกินขีดจำกัด {self.rate_limit} requests ต่อนาที',
                    'retry_after': cache.ttl(cache_key)
                },
                status=429
            )
        
        # เพิ่มจำนวน requests
        if requests_count == 0:
            cache.set(cache_key, 1, self.rate_limit_window)
        else:
            cache.incr(cache_key)
        
        response = self.get_response(request)
        
        # เพิ่ม headers
        response['X-RateLimit-Limit'] = str(self.rate_limit)
        response['X-RateLimit-Remaining'] = str(
            max(0, self.rate_limit - requests_count - 1)
        )
        
        return response
    
    def get_client_ip(self, request):
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            return x_forwarded_for.split(',')[0].strip()
        return request.META.get('REMOTE_ADDR', '')
```

### Maintenance Mode Middleware

```python
from django.conf import settings
from django.http import HttpResponse
from django.template.loader import render_to_string

class MaintenanceModeMiddleware:
    """แสดงหน้า Maintenance เมื่อเปิด maintenance mode"""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # ตรวจสอบ maintenance mode
        if getattr(settings, 'MAINTENANCE_MODE', False):
            # อนุญาต admin และ IPs ที่กำหนด
            allowed_ips = getattr(settings, 'MAINTENANCE_ALLOWED_IPS', [])
            client_ip = self.get_client_ip(request)
            
            if (request.path.startswith('/admin/') or 
                client_ip in allowed_ips):
                return self.get_response(request)
            
            # แสดงหน้า maintenance
            html = render_to_string('maintenance.html')
            return HttpResponse(html, status=503)
        
        return self.get_response(request)
    
    def get_client_ip(self, request):
        return request.META.get('REMOTE_ADDR', '')
```

### API Version Middleware

```python
class APIVersionMiddleware:
    """จัดการ API versioning จาก URL หรือ header"""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # ดึง version จาก URL (/api/v1/, /api/v2/)
        import re
        match = re.match(r'^/api/v(\d+)/', request.path)
        
        if match:
            request.api_version = int(match.group(1))
        else:
            # ดึงจาก Accept header
            accept = request.META.get('HTTP_ACCEPT', '')
            version_match = re.search(r'version=(\d+)', accept)
            request.api_version = int(version_match.group(1)) if version_match else 1
        
        return self.get_response(request)
```

### Request Context Middleware

```python
import threading

_thread_local = threading.local()

class RequestContextMiddleware:
    """เก็บ request ใน thread local สำหรับ access จากทุกที่"""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        _thread_local.request = request
        
        response = self.get_response(request)
        
        # Clean up
        if hasattr(_thread_local, 'request'):
            del _thread_local.request
        
        return response


def get_current_request():
    """ดึง current request จากทุกที่"""
    return getattr(_thread_local, 'request', None)


def get_current_user():
    """ดึง current user จากทุกที่"""
    request = get_current_request()
    if request and hasattr(request, 'user'):
        return request.user
    return None
```

---

## 5. Middleware สำหรับ Authentication

```python
class JWTAuthMiddleware:
    """Middleware ตรวจสอบ JWT token"""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # เฉพาะ API endpoints
        if request.path.startswith('/api/'):
            self.authenticate_jwt(request)
        
        return self.get_response(request)
    
    def authenticate_jwt(self, request):
        """ตรวจสอบ JWT token ใน Authorization header"""
        auth_header = request.META.get('HTTP_AUTHORIZATION', '')
        
        if not auth_header.startswith('Bearer '):
            return
        
        token = auth_header[7:]  # ลบ "Bearer " ออก
        
        try:
            from rest_framework_simplejwt.tokens import AccessToken
            from django.contrib.auth import get_user_model
            
            User = get_user_model()
            
            # Decode token
            validated_token = AccessToken(token)
            user_id = validated_token['user_id']
            
            # ดึง user
            user = User.objects.get(id=user_id, is_active=True)
            request.user = user
            request._jwt_token = validated_token
            
        except Exception:
            pass  # token ไม่ valid - ปล่อยให้ DRF จัดการ
```

---

## 6. การลงทะเบียน Middleware

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    
    # Custom middlewares
    'myapp.middleware.LoggingMiddleware',
    'myapp.middleware.RateLimitMiddleware',
    'myapp.middleware.MaintenanceModeMiddleware',
]
```

**ลำดับสำคัญ**: Middleware ทำงานจากบนลงล่าง (request) และล่างขึ้นบน (response)

---

## 7. Middleware สำหรับ CORS

```python
# middleware.py (หรือใช้ django-cors-headers)
class CORSMiddleware:
    """Custom CORS Middleware"""
    
    def __init__(self, get_response):
        self.get_response = get_response
        from django.conf import settings
        self.allowed_origins = getattr(settings, 'CORS_ALLOWED_ORIGINS', [])
        self.allow_all = getattr(settings, 'CORS_ALLOW_ALL_ORIGINS', False)
    
    def __call__(self, request):
        response = self.get_response(request)
        
        origin = request.META.get('HTTP_ORIGIN', '')
        
        if self.allow_all or origin in self.allowed_origins:
            response['Access-Control-Allow-Origin'] = origin or '*'
            response['Access-Control-Allow-Methods'] = 'GET, POST, PUT, PATCH, DELETE, OPTIONS'
            response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization, X-CSRFToken'
            response['Access-Control-Allow-Credentials'] = 'true'
            response['Access-Control-Max-Age'] = '86400'
        
        return response
    
    def process_view(self, request, view_func, view_args, view_kwargs):
        """จัดการ OPTIONS (preflight) request"""
        if request.method == 'OPTIONS':
            from django.http import HttpResponse
            response = HttpResponse()
            origin = request.META.get('HTTP_ORIGIN', '')
            if self.allow_all or origin in self.allowed_origins:
                response['Access-Control-Allow-Origin'] = origin or '*'
                response['Access-Control-Allow-Methods'] = 'GET, POST, PUT, PATCH, DELETE, OPTIONS'
                response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
                response.status_code = 200
            return response
        return None
```

---

## 8. Middleware Testing

```python
# tests/test_middleware.py
from django.test import TestCase, RequestFactory
from django.contrib.auth import get_user_model
from unittest.mock import patch, MagicMock
import time

User = get_user_model()

class LoggingMiddlewareTest(TestCase):
    
    def setUp(self):
        self.factory = RequestFactory()
    
    def test_response_time_header(self):
        """ทดสอบว่า middleware เพิ่ม X-Response-Time header"""
        response = self.client.get('/')
        self.assertIn('X-Response-Time', response)
    
    def test_logging_called(self):
        """ทดสอบว่า logging ถูกเรียก"""
        with patch('myapp.middleware.logging') as mock_logger:
            response = self.client.get('/')
            # ตรวจสอบว่า logger.info ถูกเรียก
            # mock_logger.getLogger.return_value.info.assert_called()


class RateLimitMiddlewareTest(TestCase):
    
    def test_normal_request(self):
        """Request ปกติต้องผ่าน"""
        response = self.client.get('/api/articles/')
        self.assertNotEqual(response.status_code, 429)
    
    def test_rate_limit_exceeded(self):
        """เกิน rate limit ต้อง 429"""
        from django.core.cache import cache
        from django.conf import settings
        
        # จำลองว่าเกิน limit
        ip = '127.0.0.1'
        cache.set(f'rate_limit_{ip}', 101)  # เกิน 100
        
        response = self.client.get('/api/articles/')
        self.assertEqual(response.status_code, 429)
        
        # cleanup
        cache.delete(f'rate_limit_{ip}')
```

---

## 9. สรุป Part 071

✅ **Middleware** เป็น layer ที่ประมวลผล request/response ก่อนถึง view
✅ **Built-in middlewares** เช่น SecurityMiddleware, SessionMiddleware, AuthenticationMiddleware
✅ **Function-based middleware** ใช้ closure pattern สะดวก
✅ **Class-based middleware** มี `__init__`, `__call__`, `process_exception` methods
✅ **ลำดับ** ใน MIDDLEWARE list สำคัญมาก - request จากบนลงล่าง, response จากล่างขึ้นบน
✅ Custom middleware เหมาะสำหรับ: logging, rate limiting, authentication, maintenance mode
✅ **process_view** เรียกก่อน view แต่หลัง URL routing
✅ **process_exception** เรียกเมื่อ view raise exception

## ➡️ ถัดไป: Part 072 - Django Celery Integration

*Part 071/100+ | Python Course - Beginner to World-Class*
