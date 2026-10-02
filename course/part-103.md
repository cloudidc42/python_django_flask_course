# Part 103: Security Advanced

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ OWASP Top 10 สำหรับ Python web applications
- ป้องกัน SQL Injection ด้วย parameterized queries และ ORM
- ป้องกัน XSS ด้วย template escaping และ CSP headers
- Implement CSRF protection
- ใช้ Rate limiting ด้วย slowapi สำหรับ FastAPI
- Implement OAuth2 / OIDC flow ด้วย authlib
- ใช้ secure library สำหรับ security headers
- เข้าใจ Secrets rotation patterns
- ใช้ bandit สำหรับ security scanning
- เข้าใจ Penetration testing concepts

---

## 1. OWASP Top 10 สำหรับ Python Web Apps

### OWASP Top 10 (2021) คือมาตรฐานความเสี่ยงที่พบบ่อยที่สุด

```python
# owasp_overview.py
# ภาพรวม OWASP Top 10 และการป้องกันสำหรับ Python

OWASP_TOP_10_2021 = {
    "A01": {
        "name": "Broken Access Control",
        "คำอธิบาย": "ผู้ใช้สามารถเข้าถึงข้อมูลหรือฟีเจอร์ที่ไม่มีสิทธิ์",
        "ตัวอย่าง": "เข้าถึง /admin โดยไม่ login, ดู order ของคนอื่น",
        "การป้องกัน": [
            "ตรวจสอบ authorization ทุก endpoint",
            "Deny by default - ปิดทุกอย่างก่อน แล้วเปิดเฉพาะที่อนุญาต",
            "Rate limit API calls",
            "Invalidate JWT tokens เมื่อ logout",
        ]
    },
    "A02": {
        "name": "Cryptographic Failures",
        "คำอธิบาย": "ข้อมูล sensitive ไม่ได้รับการเข้ารหัสที่เหมาะสม",
        "ตัวอย่าง": "เก็บ password เป็น plain text, ส่งข้อมูลบัตรเครดิตผ่าน HTTP",
        "การป้องกัน": [
            "ใช้ HTTPS ทุกที่",
            "Hash passwords ด้วย bcrypt/argon2",
            "เข้ารหัส data at rest",
            "ไม่ใช้ MD5/SHA1 สำหรับ security",
        ]
    },
    "A03": {
        "name": "Injection",
        "คำอธิบาย": "SQL, NoSQL, OS command injection",
        "ตัวอย่าง": "SELECT * FROM users WHERE id = '{user_input}'",
        "การป้องกัน": [
            "ใช้ parameterized queries",
            "ใช้ ORM",
            "Validate และ sanitize input",
            "ใช้ stored procedures",
        ]
    },
    "A04": {
        "name": "Insecure Design",
        "คำอธิบาย": "การออกแบบระบบที่ไม่คำนึงถึง security",
        "การป้องกัน": [
            "Threat modeling ตั้งแต่ต้น",
            "Secure design patterns",
            "Fail securely",
        ]
    },
    "A05": {
        "name": "Security Misconfiguration",
        "คำอธิบาย": "ตั้งค่า security ผิดหรือใช้ default settings",
        "ตัวอย่าง": "DEBUG=True ใน production, เปิด directory listing",
        "การป้องกัน": [
            "ปิด DEBUG mode ใน production",
            "ลบ default credentials",
            "ปิด unnecessary features",
        ]
    },
    "A06": {
        "name": "Vulnerable and Outdated Components",
        "คำอธิบาย": "ใช้ libraries ที่มีช่องโหว่",
        "การป้องกัน": [
            "อัพเดท dependencies สม่ำเสมอ",
            "ใช้ pip-audit หรือ safety check",
            "Monitor CVEs",
        ]
    },
    "A07": {
        "name": "Identification and Authentication Failures",
        "คำอธิบาย": "Authentication หรือ session management อ่อนแอ",
        "การป้องกัน": [
            "Multi-factor authentication",
            "ไม่ใช้ default passwords",
            "Secure session management",
            "Account lockout",
        ]
    },
    "A08": {
        "name": "Software and Data Integrity Failures",
        "คำอธิบาย": "ไม่ verify integrity ของ code หรือ data",
        "การป้องกัน": [
            "Verify package signatures",
            "ใช้ trusted sources",
            "CI/CD pipeline security",
        ]
    },
    "A09": {
        "name": "Security Logging and Monitoring Failures",
        "คำอธิบาย": "ไม่ log หรือ monitor security events",
        "การป้องกัน": [
            "Log authentication events",
            "Monitor failed logins",
            "Alert on anomalies",
        ]
    },
    "A10": {
        "name": "Server-Side Request Forgery (SSRF)",
        "คำอธิบาย": "Server ทำ request ไปยัง URL ที่ attacker กำหนด",
        "ตัวอย่าง": "URL parameter ที่ส่ง request ไปยัง internal services",
        "การป้องกัน": [
            "Validate URLs",
            "Whitelist allowed hosts",
            "ปิด access ไปยัง internal networks",
        ]
    }
}
```

---

## 2. SQL Injection Prevention

```python
# sql_injection_prevention.py
# การป้องกัน SQL Injection อย่างครอบคลุม

import sqlite3
import psycopg2
from sqlalchemy import create_engine, text, Column, Integer, String, select
from sqlalchemy.orm import Session, DeclarativeBase
from typing import Optional, List
import re

# ===== ❌ VULNERABLE CODE - อย่าใช้แบบนี้ =====
def get_user_VULNERABLE(conn, username: str):
    """ช่องโหว่ SQL Injection - อย่าทำแบบนี้!"""
    # attacker ส่ง: username = "admin' OR '1'='1"
    # query กลายเป็น: SELECT * FROM users WHERE username = 'admin' OR '1'='1'
    query = f"SELECT * FROM users WHERE username = '{username}'"
    cursor = conn.cursor()
    cursor.execute(query)  # DANGEROUS!
    return cursor.fetchone()


def search_products_VULNERABLE(conn, search_term: str):
    """อีกตัวอย่าง SQL Injection"""
    # attacker ส่ง: search_term = "'; DROP TABLE products; --"
    query = f"SELECT * FROM products WHERE name LIKE '%{search_term}%'"
    cursor = conn.cursor()
    cursor.execute(query)  # DANGEROUS!
    return cursor.fetchall()


# ===== ✅ SAFE CODE - ใช้ Parameterized Queries =====
def get_user_safe(conn, username: str):
    """ป้องกัน SQL Injection ด้วย parameterized query"""
    # ใช้ ? สำหรับ SQLite หรือ %s สำหรับ PostgreSQL
    query = "SELECT * FROM users WHERE username = ?"
    cursor = conn.cursor()
    cursor.execute(query, (username,))  # ส่ง parameter แยกจาก query
    return cursor.fetchone()


def search_products_safe(conn, search_term: str):
    """ค้นหาสินค้าอย่างปลอดภัย"""
    # % ใน LIKE ต้องอยู่ใน parameter ด้วย
    query = "SELECT * FROM products WHERE name LIKE ?"
    cursor = conn.cursor()
    cursor.execute(query, (f"%{search_term}%",))
    return cursor.fetchall()


def get_multiple_users_safe(conn, user_ids: List[int]):
    """ดึงหลาย users อย่างปลอดภัย"""
    if not user_ids:
        return []
    
    # ใช้ placeholders สำหรับ IN clause
    placeholders = ",".join("?" * len(user_ids))
    query = f"SELECT * FROM users WHERE id IN ({placeholders})"
    cursor = conn.cursor()
    cursor.execute(query, user_ids)
    return cursor.fetchall()


# ===== SQLAlchemy ORM - ปลอดภัยโดย default =====
class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    username = Column(String(100), nullable=False)
    email = Column(String(200), nullable=False)
    role = Column(String(50), default="user")


class ProductSearchService:
    """Service ที่ใช้ SQLAlchemy ORM อย่างปลอดภัย"""
    
    def __init__(self, session: Session):
        self.session = session
    
    def get_user_by_username(self, username: str) -> Optional[User]:
        """ดึง user ด้วย ORM (ปลอดภัยจาก SQL Injection)"""
        # ORM สร้าง parameterized query ให้อัตโนมัติ
        return self.session.query(User).filter(
            User.username == username
        ).first()
    
    def search_users(self, search_term: str, role: str = None) -> List[User]:
        """ค้นหา users ด้วย ORM"""
        query = self.session.query(User).filter(
            User.username.ilike(f"%{search_term}%")
        )
        
        if role:
            query = query.filter(User.role == role)
        
        return query.all()
    
    def get_users_by_ids(self, user_ids: List[int]) -> List[User]:
        """ดึง users หลายคนด้วย ORM"""
        return self.session.query(User).filter(
            User.id.in_(user_ids)
        ).all()
    
    def execute_raw_query_safely(self, username: str):
        """กรณีต้องใช้ raw SQL กับ SQLAlchemy"""
        # ใช้ text() กับ named parameters
        stmt = text("SELECT * FROM users WHERE username = :username")
        result = self.session.execute(stmt, {"username": username})
        return result.fetchone()


# ===== Input Validation Layer =====
class InputValidator:
    """Layer สำหรับ validate input ก่อนใช้งาน"""
    
    # Patterns ที่อนุญาต
    USERNAME_PATTERN = re.compile(r'^[a-zA-Z0-9_-]{3,50}$')
    EMAIL_PATTERN = re.compile(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
    
    @classmethod
    def validate_username(cls, username: str) -> str:
        """Validate และ sanitize username"""
        if not username or not isinstance(username, str):
            raise ValueError("Username ไม่ถูกต้อง")
        
        username = username.strip()
        
        if not cls.USERNAME_PATTERN.match(username):
            raise ValueError(
                "Username ต้องประกอบด้วย a-z, A-Z, 0-9, -, _ และมีความยาว 3-50 ตัวอักษร"
            )
        
        return username
    
    @classmethod
    def validate_email(cls, email: str) -> str:
        """Validate email"""
        if not email or not isinstance(email, str):
            raise ValueError("Email ไม่ถูกต้อง")
        
        email = email.strip().lower()
        
        if not cls.EMAIL_PATTERN.match(email):
            raise ValueError("รูปแบบ email ไม่ถูกต้อง")
        
        if len(email) > 200:
            raise ValueError("Email ยาวเกินไป")
        
        return email
    
    @classmethod
    def validate_integer(cls, value, min_val: int = None, max_val: int = None) -> int:
        """Validate integer value"""
        try:
            value = int(value)
        except (TypeError, ValueError):
            raise ValueError(f"ค่าต้องเป็นตัวเลข")
        
        if min_val is not None and value < min_val:
            raise ValueError(f"ค่าต้องไม่น้อยกว่า {min_val}")
        
        if max_val is not None and value > max_val:
            raise ValueError(f"ค่าต้องไม่มากกว่า {max_val}")
        
        return value
    
    @classmethod
    def sanitize_search_term(cls, term: str, max_length: int = 100) -> str:
        """Sanitize search term"""
        if not term:
            return ""
        
        # ลบ characters ที่อาจเป็น SQL keywords
        term = term.strip()[:max_length]
        
        # Escape % และ _ สำหรับ LIKE query
        term = term.replace('\\', '\\\\').replace('%', '\\%').replace('_', '\\_')
        
        return term


# ===== Django ORM Example =====
DJANGO_ORM_EXAMPLES = """
# ตัวอย่างที่ปลอดภัยใน Django

# ✅ ปลอดภัย - ใช้ ORM
from django.contrib.auth.models import User

# ค้นหาด้วย ORM
user = User.objects.get(username=username)
users = User.objects.filter(email__icontains=search_term)

# ✅ ปลอดภัย - raw query ด้วย parameters
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute(
        "SELECT id, username FROM auth_user WHERE username = %s",
        [username]
    )
    row = cursor.fetchone()

# ✅ ปลอดภัย - params ใน extra()
User.objects.extra(
    where=["username = %s"],
    params=[username]
)

# ❌ อันตราย - อย่าทำ
User.objects.raw(f"SELECT * FROM auth_user WHERE username = '{username}'")
User.objects.extra(where=[f"username = '{username}'"])
"""
```

---

## 3. XSS Prevention

```python
# xss_prevention.py
# การป้องกัน Cross-Site Scripting (XSS)

from fastapi import FastAPI, Request, Response
from fastapi.templating import Jinja2Templates
from fastapi.responses import HTMLResponse
import html
import bleach
from typing import Optional, List
import re

# ===== ❌ VULNERABLE - XSS =====
def render_user_content_VULNERABLE(user_input: str) -> str:
    """ช่องโหว่ XSS - อย่าทำแบบนี้!"""
    # attacker ส่ง: <script>document.cookie</script>
    return f"<div>ยินดีต้อนรับ: {user_input}</div>"  # DANGEROUS!


# ===== ✅ SAFE - HTML Escaping =====
def render_user_content_safe(user_input: str) -> str:
    """ป้องกัน XSS ด้วยการ escape HTML"""
    # html.escape() แปลง < > & " ' เป็น HTML entities
    safe_input = html.escape(user_input)
    return f"<div>ยินดีต้อนรับ: {safe_input}</div>"


# ===== Bleach Library - อนุญาต HTML บางส่วน =====
# กำหนด allowed tags และ attributes
ALLOWED_TAGS = [
    'p', 'br', 'strong', 'em', 'ul', 'ol', 'li',
    'h1', 'h2', 'h3', 'a', 'blockquote', 'code', 'pre'
]

ALLOWED_ATTRIBUTES = {
    'a': ['href', 'title'],
    '*': ['class']  # class attribute สำหรับทุก element
}

def sanitize_html_content(html_content: str) -> str:
    """
    Sanitize HTML content ที่ผู้ใช้ส่งมา
    อนุญาต HTML พื้นฐาน แต่ลบ script/iframe
    """
    # strip=True จะลบ tags ที่ไม่อยู่ใน list
    clean_html = bleach.clean(
        html_content,
        tags=ALLOWED_TAGS,
        attributes=ALLOWED_ATTRIBUTES,
        strip=True,
        strip_comments=True
    )
    return clean_html


def sanitize_url(url: str) -> str:
    """ป้องกัน javascript: URLs"""
    if not url:
        return "#"
    
    # ลบ whitespace และแปลงเป็น lowercase เพื่อ check
    url_lower = url.strip().lower()
    
    # Block dangerous protocols
    dangerous_protocols = ['javascript:', 'vbscript:', 'data:', 'file:']
    for protocol in dangerous_protocols:
        if url_lower.startswith(protocol):
            return "#"  # แทนที่ด้วย safe URL
    
    return url


# ===== Jinja2 Template Security =====
# Jinja2 escape HTML โดย default แต่ต้องระวัง |safe filter

JINJA2_SAFE_TEMPLATE = """
{# ✅ Safe: Jinja2 escape HTML โดย default #}
<p>ชื่อผู้ใช้: {{ username }}</p>

{# ❌ Dangerous: |safe ปิด auto-escaping #}
<p>{{ user_html_content | safe }}</p>  {# อย่าทำ! #}

{# ✅ Safe: sanitize ก่อนใช้ |safe #}
<p>{{ sanitized_content | safe }}</p>  {# OK ถ้า sanitized แล้ว #}

{# ✅ Safe: escape attribute values #}
<input value="{{ user_input }}" />  {# Jinja2 escape ให้อัตโนมัติ #}

{# ✅ Safe: escape ใน JavaScript context #}
<script>
    var username = {{ username | tojson }};  {# ใช้ tojson ใน JS context #}
</script>
"""


# ===== Content Security Policy (CSP) Headers =====
class CSPMiddleware:
    """Middleware สำหรับเพิ่ม Content Security Policy headers"""
    
    def __init__(
        self,
        app,
        environment: str = "production"
    ):
        self.app = app
        self.environment = environment
    
    def _get_csp_policy(self) -> str:
        """สร้าง CSP policy"""
        
        if self.environment == "development":
            # Development: relaxed policy สำหรับ hot reload
            return (
                "default-src 'self'; "
                "script-src 'self' 'unsafe-inline' 'unsafe-eval'; "
                "style-src 'self' 'unsafe-inline'; "
                "img-src 'self' data:;"
            )
        else:
            # Production: strict policy
            return (
                "default-src 'none'; "
                "script-src 'self'; "
                "style-src 'self' https://fonts.googleapis.com; "
                "font-src 'self' https://fonts.gstatic.com; "
                "img-src 'self' data: https:; "
                "connect-src 'self' https://api.myapp.com; "
                "frame-ancestors 'none'; "
                "base-uri 'self'; "
                "form-action 'self';"
            )
    
    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return
        
        async def send_with_headers(message):
            if message["type"] == "http.response.start":
                headers = list(message.get("headers", []))
                csp_policy = self._get_csp_policy()
                
                # เพิ่ม security headers
                security_headers = [
                    (b"content-security-policy", csp_policy.encode()),
                    (b"x-content-type-options", b"nosniff"),
                    (b"x-frame-options", b"DENY"),
                    (b"x-xss-protection", b"1; mode=block"),
                    (b"referrer-policy", b"strict-origin-when-cross-origin"),
                ]
                headers.extend(security_headers)
                message["headers"] = headers
            
            await send(message)
        
        await self.app(scope, receive, send_with_headers)


# ===== Output Encoding สำหรับ Context ต่างๆ =====
class OutputEncoder:
    """Encoder สำหรับ context ต่างๆ"""
    
    @staticmethod
    def html_encode(value: str) -> str:
        """HTML context encoding"""
        return html.escape(value, quote=True)
    
    @staticmethod
    def html_attr_encode(value: str) -> str:
        """HTML attribute encoding"""
        return html.escape(value, quote=True)
    
    @staticmethod
    def js_encode(value: str) -> str:
        """JavaScript string encoding"""
        import json
        # ใช้ json.dumps เพื่อ escape ให้ถูกต้อง
        return json.dumps(value)
    
    @staticmethod
    def url_encode(value: str) -> str:
        """URL encoding"""
        from urllib.parse import quote
        return quote(value, safe='')
    
    @staticmethod
    def css_encode(value: str) -> str:
        """CSS value encoding - อนุญาตเฉพาะ safe characters"""
        # อนุญาตเฉพาะ alphanumeric, spaces, และ CSS-safe chars
        return re.sub(r'[^a-zA-Z0-9\s\-_#.,;:%]', '', value)
```

---

## 4. CSRF Protection

```python
# csrf_protection.py
# CSRF Protection สำหรับ FastAPI

from fastapi import FastAPI, Request, HTTPException, Depends, Cookie
from fastapi.security import HTTPBearer
import secrets
import hashlib
import hmac
import time
from typing import Optional

# ===== CSRF Token Generation =====
SECRET_KEY = secrets.token_hex(32)  # ต้องเก็บเป็น secret และ random


def generate_csrf_token(session_id: str) -> str:
    """
    สร้าง CSRF token ที่ผูกกับ session
    
    ใช้ HMAC เพื่อ verify ว่า token ถูกสร้างโดย server ของเรา
    """
    timestamp = str(int(time.time()))
    # สร้าง message จาก session + timestamp
    message = f"{session_id}:{timestamp}"
    # HMAC ด้วย secret key
    mac = hmac.new(
        SECRET_KEY.encode(),
        message.encode(),
        hashlib.sha256
    ).hexdigest()
    
    # token = session_id:timestamp:hmac
    return f"{session_id}:{timestamp}:{mac}"


def validate_csrf_token(token: str, session_id: str, max_age: int = 3600) -> bool:
    """
    Validate CSRF token
    
    ตรวจสอบ:
    1. Format ถูกต้อง
    2. Session ID ตรงกัน
    3. HMAC ถูกต้อง (ไม่ถูก tamper)
    4. ไม่หมดอายุ
    """
    try:
        parts = token.split(":")
        if len(parts) != 3:
            return False
        
        token_session, timestamp, mac = parts
        
        # ตรวจสอบ session
        if token_session != session_id:
            return False
        
        # ตรวจสอบอายุ token
        token_age = int(time.time()) - int(timestamp)
        if token_age > max_age:
            return False
        
        # ตรวจสอบ HMAC
        message = f"{token_session}:{timestamp}"
        expected_mac = hmac.new(
            SECRET_KEY.encode(),
            message.encode(),
            hashlib.sha256
        ).hexdigest()
        
        # ใช้ hmac.compare_digest เพื่อป้องกัน timing attack
        return hmac.compare_digest(mac, expected_mac)
        
    except Exception:
        return False


# ===== FastAPI CSRF Dependency =====
class CSRFProtection:
    """Dependency สำหรับ CSRF protection ใน FastAPI"""
    
    def __init__(self, cookie_name: str = "csrf_token", header_name: str = "X-CSRF-Token"):
        self.cookie_name = cookie_name
        self.header_name = header_name
    
    async def __call__(
        self,
        request: Request,
        csrf_cookie: Optional[str] = Cookie(None, alias="csrf_token")
    ):
        # GET requests ไม่ต้องตรวจสอบ CSRF
        if request.method in ("GET", "HEAD", "OPTIONS"):
            return
        
        # ดึง CSRF token จาก header
        csrf_header = request.headers.get(self.header_name)
        
        if not csrf_header:
            raise HTTPException(
                status_code=403,
                detail="ไม่พบ CSRF token ใน request headers"
            )
        
        if not csrf_cookie:
            raise HTTPException(
                status_code=403,
                detail="ไม่พบ CSRF token ใน cookies"
            )
        
        # Double Submit Cookie pattern: header ต้องตรงกับ cookie
        if not hmac.compare_digest(csrf_header, csrf_cookie):
            raise HTTPException(
                status_code=403,
                detail="CSRF token ไม่ถูกต้อง"
            )


csrf_protect = CSRFProtection()

# ===== ตัวอย่างการใช้ CSRF ใน FastAPI =====
app = FastAPI()


@app.get("/api/csrf-token")
async def get_csrf_token(response: Response):
    """สร้างและส่ง CSRF token"""
    token = secrets.token_urlsafe(32)
    
    # ตั้ง CSRF token cookie
    response.set_cookie(
        key="csrf_token",
        value=token,
        httponly=False,  # JavaScript ต้องอ่านได้เพื่อส่งใน header
        secure=True,     # HTTPS เท่านั้น
        samesite="strict",
        max_age=3600
    )
    
    return {"csrf_token": token}


@app.post("/api/orders", dependencies=[Depends(csrf_protect)])
async def create_order_protected(order_data: dict):
    """Endpoint ที่ protected ด้วย CSRF"""
    return {"order_id": "ORD-001", "status": "created"}


# ===== Django CSRF =====
DJANGO_CSRF_EXAMPLES = """
# Django มี CSRF protection built-in

# settings.py
MIDDLEWARE = [
    ...
    'django.middleware.csrf.CsrfViewMiddleware',  # ต้องมี middleware นี้
    ...
]

# template
{% csrf_token %}  {# ใน HTML form ทุกอัน #}

# สำหรับ AJAX requests
const csrfToken = document.cookie
    .split(';')
    .find(c => c.trim().startsWith('csrftoken='))
    ?.split('=')[1];

fetch('/api/orders/', {
    method: 'POST',
    headers: {
        'X-CSRFToken': csrfToken,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(orderData)
});

# views.py - เพิ่ม CSRF exemption สำหรับ API ที่ใช้ JWT
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator

@method_decorator(csrf_exempt, name='dispatch')
class OrderAPIView(APIView):
    # ใช้ JWT authentication แทน session
    authentication_classes = [JWTAuthentication]
    ...
"""
```

---

## 5. Rate Limiting ด้วย slowapi

```python
# rate_limiting.py
# Rate Limiting สำหรับ FastAPI ด้วย slowapi

from fastapi import FastAPI, Request, HTTPException
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from slowapi.middleware import SlowAPIMiddleware
import redis
from typing import Optional
import hashlib

# ===== Basic Rate Limiting =====
# สร้าง Limiter ที่ใช้ IP address เป็น key
limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["1000/day", "100/hour"]  # global limits
)

app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
app.add_middleware(SlowAPIMiddleware)


# ===== ตัวอย่างการใช้งาน =====
@app.get("/api/public")
@limiter.limit("30/minute")  # 30 requests per minute per IP
async def public_endpoint(request: Request):
    """Public endpoint ที่มี rate limit"""
    return {"message": "สวัสดี"}


@app.post("/api/auth/login")
@limiter.limit("5/minute")  # จำกัด login attempts
async def login(request: Request, credentials: dict):
    """Login endpoint ที่มี strict rate limit"""
    return {"token": "jwt_token_here"}


@app.post("/api/auth/register")
@limiter.limit("3/hour")  # จำกัดการสมัครสมาชิก
async def register(request: Request, user_data: dict):
    """Registration endpoint"""
    return {"user_id": "USR-001"}


@app.post("/api/password/reset")
@limiter.limit("3/hour")  # จำกัด password reset
async def request_password_reset(request: Request, email_data: dict):
    """Password reset request"""
    return {"message": "ส่ง email reset แล้ว"}


# ===== Advanced Rate Limiting =====
class AdvancedRateLimiter:
    """Rate limiter แบบขั้นสูงด้วย Redis"""
    
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
    
    def _get_user_key(self, user_id: str, action: str) -> str:
        """สร้าง Redis key สำหรับ user + action"""
        return f"rate_limit:{action}:{user_id}"
    
    def _get_ip_key(self, ip: str, action: str) -> str:
        """สร้าง Redis key สำหรับ IP + action"""
        ip_hash = hashlib.md5(ip.encode()).hexdigest()
        return f"rate_limit:{action}:ip:{ip_hash}"
    
    def check_rate_limit(
        self,
        identifier: str,
        action: str,
        limit: int,
        window_seconds: int
    ) -> tuple[bool, int, int]:
        """
        ตรวจสอบ rate limit ด้วย sliding window counter
        
        Returns:
            (allowed, remaining, reset_after_seconds)
        """
        key = f"rate_limit:{action}:{identifier}"
        
        # ใช้ pipeline เพื่อ atomic operations
        pipe = self.redis.pipeline()
        pipe.incr(key)
        pipe.ttl(key)
        results = pipe.execute()
        
        current_count = results[0]
        ttl = results[1]
        
        # ตั้ง TTL ถ้าเป็น request แรก
        if ttl == -1:
            self.redis.expire(key, window_seconds)
            ttl = window_seconds
        
        remaining = max(0, limit - current_count)
        allowed = current_count <= limit
        
        return allowed, remaining, ttl
    
    def is_ip_blocked(self, ip: str) -> bool:
        """ตรวจสอบว่า IP ถูก block หรือไม่"""
        key = f"blocked_ip:{ip}"
        return self.redis.exists(key) > 0
    
    def block_ip(self, ip: str, duration_seconds: int = 3600):
        """Block IP address"""
        key = f"blocked_ip:{ip}"
        self.redis.setex(key, duration_seconds, "blocked")
    
    def track_failed_login(self, identifier: str) -> int:
        """ติดตาม failed login attempts"""
        key = f"failed_login:{identifier}"
        count = self.redis.incr(key)
        
        # TTL 15 นาทีสำหรับ failed attempts
        if count == 1:
            self.redis.expire(key, 900)
        
        # Block หลัง 10 failures
        if count >= 10:
            self.block_ip(identifier, 3600)
        
        return count
    
    def reset_failed_login(self, identifier: str):
        """Reset failed login count หลัง login สำเร็จ"""
        key = f"failed_login:{identifier}"
        self.redis.delete(key)


# ===== Rate Limit Dependency สำหรับ FastAPI =====
rate_limiter = AdvancedRateLimiter()


async def check_api_rate_limit(
    request: Request,
    limit: int = 100,
    window: int = 60,
    action: str = "api_call"
):
    """FastAPI Dependency สำหรับ rate limiting"""
    ip = request.client.host
    
    # ตรวจสอบว่า IP ถูก block หรือไม่
    if rate_limiter.is_ip_blocked(ip):
        raise HTTPException(
            status_code=429,
            detail="IP address ของคุณถูก block ชั่วคราว",
            headers={"Retry-After": "3600"}
        )
    
    # ตรวจสอบ rate limit
    allowed, remaining, reset_after = rate_limiter.check_rate_limit(
        identifier=ip,
        action=action,
        limit=limit,
        window_seconds=window
    )
    
    if not allowed:
        raise HTTPException(
            status_code=429,
            detail="คุณส่ง request มากเกินไป กรุณารอสักครู่",
            headers={
                "X-RateLimit-Limit": str(limit),
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(reset_after),
                "Retry-After": str(reset_after)
            }
        )
    
    # เพิ่ม rate limit headers ใน response
    request.state.rate_limit_remaining = remaining
    request.state.rate_limit_reset = reset_after
```

---

## 6. OAuth2 / OIDC ด้วย authlib

```python
# oauth2_oidc.py
# OAuth2 / OIDC implementation ด้วย authlib

from authlib.integrations.starlette_client import OAuth
from authlib.jose import jwt, JWTClaims
from authlib.jose.errors import JoseError
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.responses import RedirectResponse
from starlette.middleware.sessions import SessionMiddleware
import httpx
from typing import Optional
import time
import secrets

# ===== OAuth2 Configuration =====
oauth = OAuth()

# ตั้งค่า Google OAuth
oauth.register(
    name='google',
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_id='YOUR_GOOGLE_CLIENT_ID',
    client_secret='YOUR_GOOGLE_CLIENT_SECRET',
    client_kwargs={
        'scope': 'openid email profile',
        'code_challenge_method': 'S256',  # PKCE สำหรับ security
    }
)

# ตั้งค่า GitHub OAuth
oauth.register(
    name='github',
    client_id='YOUR_GITHUB_CLIENT_ID',
    client_secret='YOUR_GITHUB_CLIENT_SECRET',
    access_token_url='https://github.com/login/oauth/access_token',
    authorize_url='https://github.com/login/oauth/authorize',
    api_base_url='https://api.github.com/',
    client_kwargs={'scope': 'user:email'},
)

app = FastAPI()

# Session middleware สำหรับ OAuth state
app.add_middleware(
    SessionMiddleware,
    secret_key=secrets.token_hex(32),  # ต้องเป็น random secret
    https_only=True,  # production: True
    same_site="lax"
)


# ===== Google OAuth Endpoints =====
@app.get("/auth/google/login")
async def google_login(request: Request):
    """เริ่ม OAuth2 flow กับ Google"""
    redirect_uri = request.url_for("google_callback")
    
    # State สำหรับป้องกัน CSRF
    state = secrets.token_urlsafe(32)
    request.session['oauth_state'] = state
    
    return await oauth.google.authorize_redirect(
        request,
        redirect_uri,
        state=state
    )


@app.get("/auth/google/callback")
async def google_callback(request: Request):
    """รับ callback จาก Google"""
    
    # ตรวจสอบ state (CSRF protection)
    state = request.session.get('oauth_state')
    if not state or state != request.query_params.get('state'):
        raise HTTPException(status_code=400, detail="Invalid OAuth state")
    
    try:
        # แลก authorization code เอา access token
        token = await oauth.google.authorize_access_token(request)
        
        # ดึง user info จาก ID token
        user_info = token.get('userinfo')
        
        if not user_info:
            raise HTTPException(status_code=400, detail="ไม่ได้รับข้อมูล user")
        
        # บันทึก user หรืออัพเดทในระบบ
        user = await create_or_update_user(
            provider="google",
            provider_id=user_info['sub'],
            email=user_info['email'],
            name=user_info.get('name'),
            picture=user_info.get('picture')
        )
        
        # สร้าง session หรือ JWT
        session_token = create_session_token(user['id'])
        
        # Redirect ไปหน้า home
        response = RedirectResponse(url="/dashboard")
        response.set_cookie(
            key="session_token",
            value=session_token,
            httponly=True,
            secure=True,
            samesite="lax",
            max_age=86400  # 24 ชั่วโมง
        )
        return response
        
    except Exception as e:
        raise HTTPException(
            status_code=400,
            detail=f"OAuth callback error: {str(e)}"
        )


async def create_or_update_user(
    provider: str,
    provider_id: str,
    email: str,
    name: str = None,
    picture: str = None
) -> dict:
    """สร้างหรืออัพเดท user จาก OAuth provider"""
    # จำลองการบันทึกลง database
    return {
        "id": f"USR-{provider}_{provider_id[:8]}",
        "email": email,
        "name": name,
        "provider": provider
    }


def create_session_token(user_id: str) -> str:
    """สร้าง session token"""
    return secrets.token_urlsafe(32)


# ===== JWT Implementation =====
from authlib.jose import JsonWebToken, JWTClaims

JWT_SECRET = "your-super-secret-key"  # เก็บใน environment variable
ALGORITHM = "HS256"


class JWTManager:
    """Manager สำหรับ JWT tokens"""
    
    def __init__(self, secret: str, algorithm: str = "HS256"):
        self.secret = secret.encode()
        self.algorithm = algorithm
        self.jwt = JsonWebToken([algorithm])
    
    def create_access_token(
        self,
        subject: str,
        data: dict = None,
        expires_in: int = 3600  # 1 ชั่วโมง
    ) -> str:
        """สร้าง JWT access token"""
        now = int(time.time())
        
        payload = {
            "sub": subject,          # subject (user ID)
            "iat": now,              # issued at
            "exp": now + expires_in, # expiration
            "jti": secrets.token_hex(16),  # JWT ID (unique)
            **(data or {})
        }
        
        header = {"alg": self.algorithm}
        return self.jwt.encode(header, payload, self.secret).decode()
    
    def create_refresh_token(self, subject: str) -> str:
        """สร้าง refresh token"""
        return self.create_access_token(
            subject=subject,
            data={"type": "refresh"},
            expires_in=30 * 24 * 3600  # 30 วัน
        )
    
    def verify_token(self, token: str) -> dict:
        """Verify และ decode JWT token"""
        try:
            claims = self.jwt.decode(token, self.secret)
            claims.validate()  # ตรวจสอบ exp, iat, etc.
            return dict(claims)
            
        except JoseError as e:
            raise HTTPException(
                status_code=401,
                detail=f"Token ไม่ถูกต้อง: {str(e)}"
            )
    
    def refresh_access_token(self, refresh_token: str) -> str:
        """สร้าง access token ใหม่จาก refresh token"""
        claims = self.verify_token(refresh_token)
        
        if claims.get("type") != "refresh":
            raise HTTPException(
                status_code=401,
                detail="Token ประเภทผิด ต้องการ refresh token"
            )
        
        # สร้าง access token ใหม่
        return self.create_access_token(subject=claims["sub"])


jwt_manager = JWTManager(secret=JWT_SECRET)


# ===== FastAPI JWT Authentication =====
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")


async def get_current_user(token: str = Depends(oauth2_scheme)) -> dict:
    """Dependency สำหรับ get current user จาก JWT"""
    claims = jwt_manager.verify_token(token)
    user_id = claims.get("sub")
    
    if not user_id:
        raise HTTPException(
            status_code=401,
            detail="Token ไม่มีข้อมูล user"
        )
    
    # ดึงข้อมูล user จาก database
    user = await get_user_from_db(user_id)
    if not user:
        raise HTTPException(
            status_code=401,
            detail="ไม่พบ user"
        )
    
    return user


async def get_user_from_db(user_id: str) -> Optional[dict]:
    """ดึงข้อมูล user จาก database"""
    # จำลอง
    return {"id": user_id, "username": "testuser", "role": "user"}


@app.get("/api/profile")
async def get_profile(current_user: dict = Depends(get_current_user)):
    """Protected endpoint ที่ต้องการ JWT"""
    return {"user": current_user}
```

---

## 7. Secure Headers ด้วย secure library

```python
# secure_headers.py
# Security Headers สำหรับ Python web apps

from fastapi import FastAPI, Request, Response
from secure import Secure, SecureHeaders
import secure
from typing import Optional

# ===== secure library =====
# ติดตั้ง: pip install secure

# สร้าง Secure instance
secure_headers = Secure.with_default_headers()

# หรือ configure เอง
custom_secure = Secure(
    # Strict-Transport-Security
    hsts=secure.StrictTransportSecurity().max_age(63072000).include_subdomains().preload(),
    
    # X-Frame-Options
    xfo=secure.XFrameOptions().deny(),
    
    # X-Content-Type-Options
    xcto=secure.XContentTypeOptions(),
    
    # Content-Security-Policy
    csp=secure.ContentSecurityPolicy()
        .default_src("'none'")
        .script_src("'self'")
        .style_src("'self'", "https://fonts.googleapis.com")
        .font_src("'self'", "https://fonts.gstatic.com")
        .img_src("'self'", "data:", "https:")
        .connect_src("'self'")
        .frame_ancestors("'none'"),
    
    # Referrer-Policy
    referrer=secure.ReferrerPolicy().strict_origin_when_cross_origin(),
    
    # Permissions-Policy
    permissions=secure.PermissionsPolicy()
        .geolocation("'none'")
        .camera("'none'")
        .microphone("'none'")
        .payment("'self'"),
    
    # Cache-Control
    cache=secure.CacheControl().no_store().no_cache()
)

app = FastAPI()


@app.middleware("http")
async def add_security_headers(request: Request, call_next):
    """Middleware เพิ่ม security headers ทุก response"""
    response = await call_next(request)
    
    # เพิ่ม secure headers
    custom_secure.framework.fastapi(response)
    
    return response


# ===== Manual Security Headers =====
SECURITY_HEADERS = {
    # ป้องกัน clickjacking
    "X-Frame-Options": "DENY",
    
    # ป้องกัน MIME type sniffing
    "X-Content-Type-Options": "nosniff",
    
    # บังคับ HTTPS
    "Strict-Transport-Security": "max-age=63072000; includeSubDomains; preload",
    
    # ควบคุม referrer information
    "Referrer-Policy": "strict-origin-when-cross-origin",
    
    # ควบคุม browser features
    "Permissions-Policy": "camera=(), microphone=(), geolocation=()",
    
    # ป้องกัน XSS (legacy)
    "X-XSS-Protection": "1; mode=block",
    
    # ลบ server information
    "Server": "",  # ซ่อน server type
    
    # Content Security Policy
    "Content-Security-Policy": (
        "default-src 'none'; "
        "script-src 'self'; "
        "style-src 'self'; "
        "img-src 'self' data:; "
        "connect-src 'self'; "
        "frame-ancestors 'none'; "
        "base-uri 'self'; "
        "form-action 'self'"
    )
}


class SecurityHeadersMiddleware:
    """Middleware สำหรับเพิ่ม security headers"""
    
    def __init__(self, app, headers: dict = None):
        self.app = app
        self.headers = headers or SECURITY_HEADERS
    
    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return
        
        async def send_with_security_headers(message):
            if message["type"] == "http.response.start":
                headers = list(message.get("headers", []))
                
                for name, value in self.headers.items():
                    if value:  # ไม่เพิ่ม headers ที่ค่าว่าง
                        headers.append(
                            (name.lower().encode(), value.encode())
                        )
                
                message["headers"] = headers
            
            await send(message)
        
        await self.app(scope, receive, send_with_security_headers)
```

---

## 8. Secrets Rotation Patterns

```python
# secrets_rotation.py
# Pattern สำหรับการหมุนเวียน secrets อย่างปลอดภัย

import os
import json
import hashlib
import time
from typing import Optional, List, Dict
from datetime import datetime, timedelta
import boto3  # AWS Secrets Manager
from cryptography.fernet import Fernet

# ===== Environment Variables Pattern =====
class SecretManager:
    """Manager สำหรับจัดการ secrets"""
    
    def __init__(self):
        self._cache: Dict[str, dict] = {}
        self._cache_ttl = 300  # cache 5 นาที
    
    def get_secret(self, key: str, default: str = None) -> Optional[str]:
        """ดึง secret จาก environment variable"""
        value = os.environ.get(key, default)
        
        if value is None:
            raise ValueError(f"Secret '{key}' ไม่พบใน environment variables")
        
        return value
    
    def get_database_url(self) -> str:
        """ดึง database URL จาก environment"""
        return self.get_secret("DATABASE_URL")
    
    def get_jwt_secret(self) -> str:
        """ดึง JWT secret"""
        return self.get_secret("JWT_SECRET_KEY")
    
    def validate_secrets(self, required_keys: List[str]):
        """ตรวจสอบว่ามี required secrets ครบ"""
        missing = []
        for key in required_keys:
            if not os.environ.get(key):
                missing.append(key)
        
        if missing:
            raise EnvironmentError(
                f"Missing required secrets: {', '.join(missing)}"
            )


# ===== AWS Secrets Manager Pattern =====
class AWSSecretsManager:
    """ดึง secrets จาก AWS Secrets Manager"""
    
    def __init__(self, region_name: str = "ap-southeast-1"):
        self.client = boto3.client(
            "secretsmanager",
            region_name=region_name
        )
        self._cache: Dict[str, dict] = {}
        self._cache_ttl = 300  # 5 นาที
    
    def get_secret(self, secret_name: str, force_refresh: bool = False) -> dict:
        """ดึง secret จาก AWS Secrets Manager"""
        
        # ตรวจสอบ cache
        if not force_refresh and secret_name in self._cache:
            cached = self._cache[secret_name]
            if time.time() - cached['timestamp'] < self._cache_ttl:
                return cached['value']
        
        try:
            response = self.client.get_secret_value(SecretId=secret_name)
            
            if 'SecretString' in response:
                secret = json.loads(response['SecretString'])
            else:
                secret = {"binary": response['SecretBinary']}
            
            # บันทึก cache
            self._cache[secret_name] = {
                'value': secret,
                'timestamp': time.time()
            }
            
            return secret
            
        except Exception as e:
            raise RuntimeError(f"ไม่สามารถดึง secret '{secret_name}': {e}")
    
    def rotate_secret(self, secret_name: str, new_value: str):
        """หมุนเวียน secret"""
        self.client.put_secret_value(
            SecretId=secret_name,
            SecretString=new_value
        )
        # ลบ cache เพื่อดึงค่าใหม่
        self._cache.pop(secret_name, None)


# ===== Database Password Rotation =====
class DatabasePasswordRotator:
    """จัดการการหมุนเวียน database password"""
    
    def __init__(self, secrets_manager: AWSSecretsManager):
        self.secrets = secrets_manager
    
    def rotate_database_password(
        self,
        db_name: str,
        secret_name: str
    ):
        """
        Pattern การหมุนเวียน password แบบ dual-password
        
        ขั้นตอน:
        1. สร้าง password ใหม่
        2. เพิ่ม password ใหม่ใน database (เก็บทั้งเก่าและใหม่)
        3. อัพเดท secret ให้ใช้ password ใหม่
        4. ทดสอบ connection ด้วย password ใหม่
        5. ลบ password เก่าออก
        """
        
        # ดึง current credentials
        current_creds = self.secrets.get_secret(secret_name)
        old_password = current_creds.get('password')
        username = current_creds.get('username')
        
        # สร้าง password ใหม่
        new_password = self._generate_strong_password()
        
        try:
            # อัพเดท password ใน database
            self._add_new_password_to_db(username, new_password, db_name)
            
            # อัพเดท secret
            new_creds = {**current_creds, 'password': new_password}
            self.secrets.rotate_secret(
                secret_name,
                json.dumps(new_creds)
            )
            
            # ทดสอบ connection
            if not self._test_connection(new_creds, db_name):
                # Rollback ถ้าไม่ได้
                self.secrets.rotate_secret(
                    secret_name,
                    json.dumps(current_creds)
                )
                raise RuntimeError("Connection test ล้มเหลว - rollback แล้ว")
            
            # ลบ password เก่า
            self._remove_old_password_from_db(username, old_password, db_name)
            
        except Exception as e:
            raise RuntimeError(f"Rotation ล้มเหลว: {e}")
    
    def _generate_strong_password(self, length: int = 32) -> str:
        """สร้าง password ที่แข็งแกร่ง"""
        import secrets
        import string
        
        characters = string.ascii_letters + string.digits + "!@#$%^&*"
        return ''.join(secrets.choice(characters) for _ in range(length))
    
    def _add_new_password_to_db(self, username: str, password: str, db_name: str):
        """เพิ่ม password ใหม่ใน database"""
        # Implementation จริงจะ execute: ALTER USER username WITH PASSWORD 'new_password'
        pass
    
    def _test_connection(self, creds: dict, db_name: str) -> bool:
        """ทดสอบ database connection"""
        return True
    
    def _remove_old_password_from_db(self, username: str, old_password: str, db_name: str):
        """ลบ old password"""
        pass
```

---

## 9. Security Scanning ด้วย bandit

```bash
# bandit_usage.sh
# การใช้งาน bandit สำหรับ security scanning

# ติดตั้ง bandit
pip install bandit

# Scan ไฟล์เดียว
bandit app.py

# Scan ทั้ง project
bandit -r ./src/

# Scan และ output เป็น JSON
bandit -r ./src/ -f json -o bandit_report.json

# Scan เฉพาะ severity ระดับ HIGH และ MEDIUM
bandit -r ./src/ -l  # low
bandit -r ./src/ -ll # medium
bandit -r ./src/ -lll # high

# Exclude tests และ migration files
bandit -r ./src/ --exclude ./tests,./migrations

# ดู tests ทั้งหมดที่ bandit ทดสอบ
bandit -t B201  # ทดสอบ specific test ID
```

```python
# security_examples_for_bandit.py
# ตัวอย่าง code ที่ bandit จะตรวจพบ

import subprocess
import os
import hashlib
import random
import pickle
import yaml

# ===== B101: assert_used =====
# Bandit จะ flag: ไม่ใช้ assert ใน production code
def check_admin_FLAGGED(user):
    assert user.is_admin, "User is not admin"  # bandit flag: B101


# ===== B103: setting_noqa =====
# ===== B104: hardcoded_bind_all_interfaces =====
def start_server_FLAGGED():
    # bandit flag: bind to all interfaces
    import socket
    s = socket.socket()
    s.bind(('0.0.0.0', 8080))  # B104: ควรระบุ specific IP


# ===== B201: flask_debug_true =====
# ===== B105: hardcoded_password_string =====
password = "admin123"  # B105: hardcoded password


# ===== B108: probable_temporary_file =====
tmp_file = "/tmp/data.txt"  # B108: ใช้ tempfile module แทน


# ===== B301: pickle =====
def load_data_FLAGGED(data: bytes):
    return pickle.loads(data)  # B301: pickle ไม่ปลอดภัย


# ===== B303: use_of_md5 =====
def hash_password_FLAGGED(password: str) -> str:
    return hashlib.md5(password.encode()).hexdigest()  # B303: MD5 ไม่ปลอดภัย


# ===== B311: standard_pseudo_random_generators =====
def generate_token_FLAGGED() -> str:
    return str(random.randint(100000, 999999))  # B311: ใช้ secrets module แทน


# ===== B506: yaml_load =====
def load_config_FLAGGED(content: str) -> dict:
    return yaml.load(content)  # B506: ใช้ yaml.safe_load แทน


# ===== B602: subprocess_popen_with_shell_equals_true =====
def run_command_FLAGGED(user_input: str):
    subprocess.Popen(user_input, shell=True)  # B602: shell injection risk


# ===== ✅ Safe alternatives =====
def check_admin_safe(user):
    if not user.is_admin:
        raise PermissionError("User is not admin")


def hash_password_safe(password: str) -> str:
    # ใช้ bcrypt หรือ argon2
    import bcrypt
    salt = bcrypt.gensalt(rounds=12)
    return bcrypt.hashpw(password.encode(), salt).decode()


def generate_token_safe() -> str:
    import secrets
    return secrets.token_urlsafe(32)  # cryptographically secure


def load_config_safe(content: str) -> dict:
    return yaml.safe_load(content)  # safe_load ไม่ execute code


def run_command_safe(command: list):
    # ส่ง command เป็น list แทน string
    result = subprocess.run(
        command,  # ['ls', '-la']
        shell=False,  # ปิด shell
        capture_output=True,
        text=True,
        timeout=30
    )
    return result.stdout
```

```python
# bandit_config.py
# การตั้งค่า bandit ผ่าน configuration file

# .bandit หรือ pyproject.toml
BANDIT_CONFIG = """
[tool.bandit]
exclude_dirs = ["tests", "migrations", "venv"]
skips = ["B101", "B601"]  # skip specific tests
tests = ["B201", "B301", "B302"]  # run only specific tests

[tool.bandit.assert_used]
skips = ["*_test.py", "test_*.py"]  # skip asserts ใน test files
"""

# เพิ่ม bandit ใน CI/CD pipeline
CI_CONFIG = """
# .github/workflows/security.yml
name: Security Scan
on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install bandit safety
      
      - name: Run Bandit
        run: bandit -r src/ -f json -o bandit-report.json -ll
      
      - name: Check for known vulnerabilities
        run: safety check
      
      - name: Upload reports
        uses: actions/upload-artifact@v3
        with:
          name: security-reports
          path: bandit-report.json
"""
```

---

## 10. Penetration Testing Concepts

```python
# pentest_concepts.py
# แนวคิด Penetration Testing สำหรับ Python developers

"""
Penetration Testing คือการทดสอบระบบด้วยวิธีเดียวกับที่ attacker จะใช้
เพื่อค้นหาช่องโหว่ก่อนที่จะถูกโจมตีจริง

ขั้นตอนหลัก (Methodology):
1. Reconnaissance - เก็บข้อมูลเกี่ยวกับ target
2. Scanning - ค้นหา ports, services, vulnerabilities
3. Exploitation - ทดสอบการโจมตีจริง
4. Post-exploitation - ตรวจสอบผลกระทบ
5. Reporting - รายงานและแนะนำการแก้ไข
"""

# ===== Security Testing Tools =====
SECURITY_TESTING_TOOLS = {
    "Static Analysis (SAST)": {
        "bandit": "Python security linter",
        "semgrep": "Static analysis ที่กำหนด rules เองได้",
        "pylint": "Python linter ที่รวม security checks",
        "safety": "ตรวจสอบ dependencies ที่มีช่องโหว่",
        "pip-audit": "Audit Python packages"
    },
    "Dynamic Analysis (DAST)": {
        "OWASP ZAP": "Web app security scanner",
        "Burp Suite": "Web proxy และ security testing platform",
        "nikto": "Web server scanner",
        "nuclei": "Fast vulnerability scanner"
    },
    "API Testing": {
        "Postman": "API testing ด้วย security tests",
        "OWASP ZAP API Scan": "API security scanning",
        "pytest": "Custom security tests ด้วย Python"
    }
}

# ===== Security Test Suite ด้วย pytest =====
import pytest
import httpx

BASE_URL = "http://localhost:8080"


class SecurityTestSuite:
    """ชุดทดสอบ security สำหรับ API"""
    
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.client = httpx.Client()
    
    # ===== Authentication Tests =====
    def test_unauthenticated_access(self, protected_endpoints: list):
        """ทดสอบว่า protected endpoints ปฏิเสธ unauthenticated requests"""
        for endpoint in protected_endpoints:
            response = self.client.get(f"{self.base_url}{endpoint}")
            assert response.status_code in [401, 403], \
                f"Endpoint {endpoint} ควร return 401/403 แต่ได้ {response.status_code}"
    
    def test_sql_injection(self, login_endpoint: str):
        """ทดสอบ SQL injection ที่ login"""
        payloads = [
            "' OR '1'='1",
            "' OR 1=1 --",
            "'; DROP TABLE users; --",
            "admin'--",
            "1 UNION SELECT * FROM users",
        ]
        
        for payload in payloads:
            response = self.client.post(
                f"{self.base_url}{login_endpoint}",
                json={"username": payload, "password": "test"}
            )
            
            # ต้อง return 401 ไม่ใช่ 200
            assert response.status_code != 200, \
                f"SQL injection payload '{payload}' อาจสำเร็จ"
    
    def test_xss_prevention(self, endpoints_with_input: list):
        """ทดสอบ XSS prevention"""
        xss_payloads = [
            "<script>alert('xss')</script>",
            "<img src=x onerror=alert('xss')>",
            "javascript:alert('xss')",
            "'\"><script>alert('xss')</script>",
        ]
        
        for endpoint, field in endpoints_with_input:
            for payload in xss_payloads:
                response = self.client.post(
                    f"{self.base_url}{endpoint}",
                    json={field: payload}
                )
                
                # XSS payload ไม่ควรปรากฏ unescaped ใน response
                if response.status_code == 200:
                    assert "<script>" not in response.text.lower(), \
                        f"XSS payload ปรากฏใน response ที่ {endpoint}"
    
    def test_rate_limiting(self, endpoint: str, limit: int = 100):
        """ทดสอบ rate limiting"""
        exceeded = False
        
        for i in range(limit + 10):
            response = self.client.post(
                f"{self.base_url}{endpoint}",
                json={"username": "test", "password": "test"}
            )
            
            if response.status_code == 429:
                exceeded = True
                break
        
        assert exceeded, f"Rate limiting ไม่ทำงานที่ {endpoint}"
    
    def test_security_headers(self, endpoint: str = "/"):
        """ตรวจสอบ security headers"""
        response = self.client.get(f"{self.base_url}{endpoint}")
        headers = response.headers
        
        required_headers = {
            "x-content-type-options": "nosniff",
            "x-frame-options": "DENY",
            "strict-transport-security": None,  # ต้องมีแต่ค่าอะไรก็ได้
        }
        
        for header, expected_value in required_headers.items():
            assert header in headers, f"Missing security header: {header}"
            
            if expected_value:
                assert headers[header].lower() == expected_value.lower(), \
                    f"Header {header} ค่าไม่ถูกต้อง: {headers[header]}"
    
    def test_sensitive_data_exposure(self, endpoints: list):
        """ตรวจสอบว่าไม่มีข้อมูล sensitive ใน response"""
        sensitive_patterns = [
            "password",
            "secret",
            "api_key",
            "private_key",
            "token",
            "mysql://",
            "postgresql://",
        ]
        
        for endpoint in endpoints:
            response = self.client.get(f"{self.base_url}{endpoint}")
            response_text = response.text.lower()
            
            for pattern in sensitive_patterns:
                # ตรวจสอบว่าไม่มี sensitive data ใน error messages
                if response.status_code >= 400:
                    assert pattern not in response_text, \
                        f"Sensitive data '{pattern}' ปรากฏใน error response ที่ {endpoint}"
    
    def test_idor(self, resource_endpoint: str):
        """
        Test Insecure Direct Object Reference (IDOR)
        ผู้ใช้ไม่ควรเข้าถึงข้อมูลของผู้ใช้อื่น
        """
        # Login เป็น user A
        user_a_token = self._login_as_user("user_a@test.com", "password_a")
        
        # สร้าง resource ใน name user A
        resource_id = self._create_resource(user_a_token)
        
        # Login เป็น user B
        user_b_token = self._login_as_user("user_b@test.com", "password_b")
        
        # User B พยายามเข้าถึง resource ของ user A
        response = self.client.get(
            f"{self.base_url}{resource_endpoint}/{resource_id}",
            headers={"Authorization": f"Bearer {user_b_token}"}
        )
        
        # ต้อง return 403 Forbidden
        assert response.status_code == 403, \
            f"IDOR vulnerability: User B สามารถเข้าถึง resource ของ User A"
    
    def _login_as_user(self, email: str, password: str) -> str:
        """Helper: login และได้ token"""
        response = self.client.post(
            f"{self.base_url}/auth/login",
            json={"email": email, "password": password}
        )
        return response.json().get("access_token", "")
    
    def _create_resource(self, token: str) -> str:
        """Helper: สร้าง resource"""
        response = self.client.post(
            f"{self.base_url}/api/resources",
            headers={"Authorization": f"Bearer {token}"},
            json={"name": "Test Resource"}
        )
        return response.json().get("id", "")
```

---

## 11. Complete Security Implementation

```python
# complete_security.py
# ตัวอย่าง production-ready security setup

from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware
import secrets
from typing import Optional

# ===== สร้าง Secure FastAPI App =====
def create_secure_app(environment: str = "production") -> FastAPI:
    """สร้าง FastAPI app ที่มี security features ครบ"""
    
    app = FastAPI(
        title="Secure API",
        # ซ่อน API docs ใน production
        docs_url="/docs" if environment != "production" else None,
        redoc_url="/redoc" if environment != "production" else None,
        openapi_url="/openapi.json" if environment != "production" else None,
    )
    
    # ===== Middleware Stack =====
    
    # 1. Force HTTPS (production only)
    if environment == "production":
        app.add_middleware(HTTPSRedirectMiddleware)
    
    # 2. ป้องกัน Host header injection
    app.add_middleware(
        TrustedHostMiddleware,
        allowed_hosts=["myapp.com", "api.myapp.com", "localhost"]
    )
    
    # 3. Security headers
    @app.middleware("http")
    async def security_headers_middleware(request: Request, call_next):
        response = await call_next(request)
        
        # เพิ่ม security headers
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()"
        
        if environment == "production":
            response.headers["Strict-Transport-Security"] = \
                "max-age=63072000; includeSubDomains; preload"
        
        # ลบ headers ที่ expose ข้อมูล server
        response.headers.pop("server", None)
        response.headers.pop("x-powered-by", None)
        
        return response
    
    # 4. Request logging สำหรับ security monitoring
    @app.middleware("http")
    async def security_logging_middleware(request: Request, call_next):
        import structlog
        log = structlog.get_logger("security")
        
        # Log suspicious patterns
        user_agent = request.headers.get("user-agent", "")
        if any(scanner in user_agent.lower() for scanner in ["sqlmap", "nikto", "nessus"]):
            log.warning(
                "suspicious_scanner_detected",
                ip=request.client.host,
                user_agent=user_agent
            )
        
        response = await call_next(request)
        return response
    
    return app


# Security checklist
SECURITY_CHECKLIST = {
    "authentication": [
        "✅ ใช้ strong password hashing (bcrypt/argon2)",
        "✅ Multi-factor authentication",
        "✅ Account lockout หลัง failed attempts",
        "✅ Secure password reset flow",
        "✅ JWT token expiration",
        "✅ Refresh token rotation",
    ],
    "authorization": [
        "✅ Role-based access control (RBAC)",
        "✅ Resource ownership checks",
        "✅ Principle of least privilege",
        "✅ API key management",
    ],
    "input_validation": [
        "✅ Validate ทุก user input",
        "✅ Parameterized queries",
        "✅ HTML escaping",
        "✅ File upload validation",
        "✅ Request size limits",
    ],
    "output_security": [
        "✅ Security headers",
        "✅ Content Security Policy",
        "✅ HTTPS only",
        "✅ Sensitive data encryption",
    ],
    "infrastructure": [
        "✅ Secrets management (env vars / vault)",
        "✅ Regular dependency updates",
        "✅ Security scanning ใน CI/CD",
        "✅ Audit logging",
        "✅ Intrusion detection",
    ]
}
```

---

## 12. สรุป Part 103

✅ เข้าใจ OWASP Top 10 และการป้องกันแต่ละข้อสำหรับ Python apps  
✅ ป้องกัน SQL Injection ด้วย parameterized queries, ORM, และ input validation  
✅ ป้องกัน XSS ด้วย HTML escaping, bleach sanitization, และ CSP headers  
✅ Implement CSRF protection ด้วย HMAC tokens และ Double Submit Cookie pattern  
✅ ใช้ slowapi สำหรับ rate limiting ทั้ง basic และ advanced patterns  
✅ Implement OAuth2/OIDC flow ด้วย authlib สำหรับ Google และ GitHub  
✅ เพิ่ม security headers ด้วย secure library  
✅ จัดการ secrets ด้วย environment variables และ AWS Secrets Manager  
✅ ใช้ bandit สำหรับ security scanning ใน codebase  
✅ เข้าใจ penetration testing concepts และสร้าง security test suite  

---

## ➡️ ถัดไป: Part 104 - Machine Learning with Python

*Part 103/105 | Python Course - World-Class Level*
