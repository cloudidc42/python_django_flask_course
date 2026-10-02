# Part 045: Security Basics
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- Hash passwords ด้วย bcrypt และ argon2
- สร้างและตรวจสอบ JWT tokens
- Validate input ป้องกัน injection
- ป้องกัน SQL Injection
- ป้องกัน XSS
- ตั้งค่า CORS อย่างถูกต้อง
- จัดการ HTTPS และ secrets

---

## 1. Password Hashing

```bash
pip install bcrypt argon2-cffi passlib
```

### 1.1 bcrypt

```python
import bcrypt

# === bcrypt ===
def hash_password_bcrypt(password: str) -> str:
    """Hash password ด้วย bcrypt"""
    # bcrypt ใส่ salt โดยอัตโนมัติ
    salt = bcrypt.gensalt(rounds=12)  # cost factor (10-12 ปกติ)
    hashed = bcrypt.hashpw(password.encode("utf-8"), salt)
    return hashed.decode("utf-8")


def verify_password_bcrypt(password: str, hashed: str) -> bool:
    """ตรวจสอบ password กับ hash"""
    return bcrypt.checkpw(
        password.encode("utf-8"), 
        hashed.encode("utf-8")
    )


# ทดสอบ
password = "MySecurePassword123!"
hashed = hash_password_bcrypt(password)
print(f"Original: {password}")
print(f"Hashed: {hashed}")
print(f"Verify correct: {verify_password_bcrypt(password, hashed)}")
print(f"Verify wrong: {verify_password_bcrypt('WrongPassword', hashed)}")
```

### 1.2 Argon2 (แนะนำ)

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError, VerificationError, InvalidHashError

# Argon2 เป็น winner ของ Password Hashing Competition 2015
# ปลอดภัยกว่า bcrypt สำหรับ modern hardware

ph = PasswordHasher(
    time_cost=3,       # จำนวน iterations
    memory_cost=65536, # 64MB RAM
    parallelism=1,     # threads
    hash_len=32,       # ความยาว hash
    salt_len=16        # ความยาว salt
)


def hash_password(password: str) -> str:
    """Hash password ด้วย Argon2id"""
    return ph.hash(password)


def verify_password(password: str, hashed: str) -> bool:
    """ตรวจสอบ password"""
    try:
        ph.verify(hashed, password)
        
        # Check ถ้าต้องการ rehash (เมื่อ parameters เปลี่ยน)
        if ph.check_needs_rehash(hashed):
            # Rehash ด้วย parameters ใหม่
            new_hash = hash_password(password)
            # อัปเดต hash ใน database
            print("Password needs rehash!")
        
        return True
    except VerifyMismatchError:
        return False  # Wrong password
    except (VerificationError, InvalidHashError):
        return False  # Invalid hash format


# ทดสอบ
password = "SecurePass@2024"
hashed = hash_password(password)
print(f"Argon2 hash: {hashed[:50]}...")
print(f"Verify: {verify_password(password, hashed)}")
print(f"Verify wrong: {verify_password('WrongPass', hashed)}")
```

### 1.3 ใช้ passlib (รองรับหลาย algorithms)

```python
from passlib.context import CryptContext

# สร้าง context ที่รองรับหลาย schemes
# deprecated schemes จะยังตรวจสอบได้แต่จะ rehash เมื่อ login
pwd_context = CryptContext(
    schemes=["argon2", "bcrypt"],
    deprecated="auto",
    # Argon2 parameters
    argon2__rounds=4,
    argon2__memory_cost=65536,
    # bcrypt parameters (legacy)
    bcrypt__rounds=12,
)


def hash_password_passlib(password: str) -> str:
    return pwd_context.hash(password)


def verify_password_passlib(password: str, hashed: str) -> tuple[bool, str | None]:
    """
    คืนค่า (valid, new_hash_if_needs_rehash)
    """
    try:
        is_valid, new_hash = pwd_context.verify_and_update(password, hashed)
        return is_valid, new_hash
    except Exception:
        return False, None


# ทดสอบ
p = "TestPassword!"
h = hash_password_passlib(p)
valid, new_hash = verify_password_passlib(p, h)
print(f"Valid: {valid}, Needs rehash: {new_hash is not None}")
```

---

## 2. JWT Tokens

```bash
pip install PyJWT
```

```python
import jwt
import secrets
from datetime import datetime, timedelta, timezone
from typing import Optional, Any
from dataclasses import dataclass

# === JWT Token Service ===
@dataclass
class TokenPayload:
    user_id: int
    email: str
    role: str = "user"


class JWTService:
    """Service สำหรับสร้างและตรวจสอบ JWT tokens"""
    
    def __init__(
        self, 
        secret_key: str,
        algorithm: str = "HS256",
        access_token_expire_minutes: int = 30,
        refresh_token_expire_days: int = 7
    ):
        self.secret_key = secret_key
        self.algorithm = algorithm
        self.access_expire = timedelta(minutes=access_token_expire_minutes)
        self.refresh_expire = timedelta(days=refresh_token_expire_days)
    
    def create_access_token(self, payload: TokenPayload) -> str:
        """สร้าง access token (short-lived)"""
        now = datetime.now(timezone.utc)
        
        data = {
            "sub": str(payload.user_id),  # subject
            "email": payload.email,
            "role": payload.role,
            "type": "access",
            "iat": now,                    # issued at
            "exp": now + self.access_expire,  # expiry
            "jti": secrets.token_hex(16),  # JWT ID (unique)
        }
        
        return jwt.encode(data, self.secret_key, algorithm=self.algorithm)
    
    def create_refresh_token(self, user_id: int) -> str:
        """สร้าง refresh token (long-lived)"""
        now = datetime.now(timezone.utc)
        
        data = {
            "sub": str(user_id),
            "type": "refresh",
            "iat": now,
            "exp": now + self.refresh_expire,
            "jti": secrets.token_hex(16),
        }
        
        return jwt.encode(data, self.secret_key, algorithm=self.algorithm)
    
    def decode_token(self, token: str) -> dict[str, Any]:
        """Decode และ verify token"""
        try:
            payload = jwt.decode(
                token,
                self.secret_key,
                algorithms=[self.algorithm],
                options={
                    "verify_exp": True,      # ตรวจสอบ expiry
                    "verify_iat": True,      # ตรวจสอบ issued at
                    "require": ["sub", "exp", "iat", "type"],
                }
            )
            return payload
            
        except jwt.ExpiredSignatureError:
            raise ValueError("Token has expired")
        except jwt.InvalidSignatureError:
            raise ValueError("Invalid token signature")
        except jwt.DecodeError:
            raise ValueError("Invalid token format")
        except jwt.InvalidTokenError as e:
            raise ValueError(f"Invalid token: {e}")
    
    def get_user_id_from_token(self, token: str) -> int:
        """ดึง user_id จาก token"""
        payload = self.decode_token(token)
        return int(payload["sub"])
    
    def is_access_token(self, token: str) -> bool:
        """ตรวจสอบว่าเป็น access token"""
        payload = self.decode_token(token)
        return payload.get("type") == "access"
    
    def refresh_access_token(
        self, 
        refresh_token: str, 
        user_payload: TokenPayload
    ) -> str:
        """สร้าง access token ใหม่จาก refresh token"""
        payload = self.decode_token(refresh_token)
        
        if payload.get("type") != "refresh":
            raise ValueError("Not a refresh token")
        
        if int(payload["sub"]) != user_payload.user_id:
            raise ValueError("Token user mismatch")
        
        return self.create_access_token(user_payload)


# ทดสอบ JWT
def demo_jwt():
    jwt_service = JWTService(
        secret_key=secrets.token_urlsafe(32),
        access_token_expire_minutes=30,
        refresh_token_expire_days=7
    )
    
    user = TokenPayload(user_id=1, email="alice@example.com", role="admin")
    
    # สร้าง tokens
    access_token = jwt_service.create_access_token(user)
    refresh_token = jwt_service.create_refresh_token(user.user_id)
    
    print(f"Access token: {access_token[:50]}...")
    print(f"Refresh token: {refresh_token[:50]}...")
    
    # Decode token
    payload = jwt_service.decode_token(access_token)
    print(f"\nDecoded payload:")
    for key, value in payload.items():
        print(f"  {key}: {value}")
    
    # ตรวจสอบ type
    print(f"\nIs access token: {jwt_service.is_access_token(access_token)}")
    print(f"User ID: {jwt_service.get_user_id_from_token(access_token)}")
    
    # Test expired token
    expired_service = JWTService(
        secret_key=secrets.token_urlsafe(32),
        access_token_expire_minutes=-1  # หมดอายุทันที
    )
    expired_token = expired_service.create_access_token(user)
    
    try:
        expired_service.decode_token(expired_token)
    except ValueError as e:
        print(f"\nExpected error: {e}")


demo_jwt()
```

---

## 3. SQL Injection Prevention

```python
import sqlite3
from typing import Any

# === ❌ SQL Injection Vulnerability ===
def unsafe_login(username: str, password: str):
    """อันตราย! ห้ามใช้"""
    conn = sqlite3.connect(":memory:")
    
    # ❌ String interpolation → SQL Injection
    query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
    
    # Attack: username = "' OR '1'='1' --"
    # Query จะเป็น: SELECT * FROM users WHERE username = '' OR '1'='1' --' AND password = ''
    # ผล: login ได้โดยไม่ต้องรู้ password!
    
    print(f"Unsafe query: {query}")


# === ✅ Parameterized Queries ===
def safe_login(username: str, password: str):
    """ปลอดภัย - ใช้ parameterized queries"""
    conn = sqlite3.connect(":memory:")
    conn.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY,
            username TEXT,
            password_hash TEXT
        )
    """)
    conn.execute(
        "INSERT INTO users VALUES (1, 'admin', 'hash123')"
    )
    
    # ✅ ใช้ ? placeholders - database จะ escape input อัตโนมัติ
    cursor = conn.execute(
        "SELECT * FROM users WHERE username = ? AND password_hash = ?",
        (username, password)  # Parameters แยกจาก query
    )
    
    user = cursor.fetchone()
    conn.close()
    return user


# ทดสอบ injection attempt
print("=== SQL Injection Test ===")
unsafe_login("admin' --", "anything")

# Safe version ป้องกันได้
result = safe_login("' OR '1'='1' --", "anything")
print(f"Safe login with injection attempt: {result}")  # None - ป้องกันได้!


# === SQLAlchemy (ORM) - ปลอดภัยกว่า Raw SQL ===
from sqlalchemy import create_engine, text
from sqlalchemy.orm import Session

def safe_with_sqlalchemy():
    engine = create_engine("sqlite:///:memory:")
    
    # ❌ อย่าทำแบบนี้ถ้าใช้ SQLAlchemy
    # session.execute(f"SELECT * FROM users WHERE name = '{name}'")
    
    # ✅ ใช้ ORM (ปลอดภัยที่สุด)
    # user = session.query(User).filter(User.name == name).first()
    
    # ✅ ถ้าต้องใช้ raw SQL ให้ใช้ text() กับ bindparams
    with Session(engine) as session:
        user_id = 1
        result = session.execute(
            text("SELECT * FROM users WHERE id = :user_id"),
            {"user_id": user_id}  # Named parameters
        )


# === Input Validation ===
import re
from pydantic import BaseModel, field_validator

class UserInput(BaseModel):
    username: str
    email: str
    age: int
    
    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        # อนุญาตเฉพาะ alphanumeric และ underscore
        if not re.match(r'^[a-zA-Z0-9_]{3,50}$', v):
            raise ValueError(
                "Username must be 3-50 chars, alphanumeric and underscore only"
            )
        return v
    
    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        if not re.match(pattern, v):
            raise ValueError("Invalid email format")
        return v.lower()
    
    @field_validator("age")
    @classmethod
    def validate_age(cls, v: int) -> int:
        if not 0 <= v <= 150:
            raise ValueError("Age must be between 0 and 150")
        return v


# ทดสอบ validation
try:
    user = UserInput(username="alice_123", email="alice@example.com", age=25)
    print(f"Valid user: {user}")
except Exception as e:
    print(f"Validation error: {e}")

try:
    user = UserInput(username="'; DROP TABLE users; --", email="x", age=999)
except Exception as e:
    print(f"SQL injection caught: {e}")
```

---

## 4. XSS Prevention

```python
import html
from markupsafe import Markup, escape  # pip install markupsafe

# === XSS Prevention ===

# ❌ XSS Vulnerability
def unsafe_template(user_input: str) -> str:
    """อันตราย - ไม่ escape HTML"""
    return f"<div>Hello, {user_input}!</div>"


# ✅ Safe HTML Escaping
def safe_template(user_input: str) -> str:
    """ปลอดภัย - escape HTML"""
    escaped = html.escape(user_input)
    return f"<div>Hello, {escaped}!</div>"


# ทดสอบ XSS
xss_payload = '<script>alert("XSS Attack!")</script>'
print("=== XSS Test ===")
print(f"Unsafe: {unsafe_template(xss_payload)}")
print(f"Safe: {safe_template(xss_payload)}")


# === Content Security Policy (CSP) Headers ===
"""
ตั้ง headers ใน FastAPI/Flask:
"""

# FastAPI
from fastapi import FastAPI, Request, Response
from fastapi.middleware.base import BaseHTTPMiddleware

app = FastAPI()

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Prevent XSS
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        
        # Content Security Policy
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; "
            "script-src 'self' 'nonce-{nonce}'; "
            "style-src 'self' https://fonts.googleapis.com; "
            "img-src 'self' data: https:; "
            "font-src 'self' https://fonts.gstatic.com; "
            "connect-src 'self' https://api.example.com; "
            "frame-ancestors 'none'"
        )
        
        # HSTS (ใช้ HTTPS เท่านั้น)
        response.headers["Strict-Transport-Security"] = (
            "max-age=31536000; includeSubDomains; preload"
        )
        
        # Referrer Policy
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        
        # Permissions Policy
        response.headers["Permissions-Policy"] = (
            "geolocation=(), microphone=(), camera=()"
        )
        
        return response

app.add_middleware(SecurityHeadersMiddleware)


# === HTML Sanitization ===
# pip install bleach
import bleach

def sanitize_html(content: str) -> str:
    """
    Sanitize HTML - อนุญาตเฉพาะ tags ที่ปลอดภัย
    ใช้สำหรับ rich text editors
    """
    allowed_tags = [
        "a", "abbr", "acronym", "b", "blockquote", "br",
        "code", "em", "i", "li", "ol", "p", "strong", "ul"
    ]
    allowed_attributes = {
        "a": ["href", "title", "rel"],
        "abbr": ["title"],
        "acronym": ["title"],
    }
    
    cleaned = bleach.clean(
        content,
        tags=allowed_tags,
        attributes=allowed_attributes,
        strip=True  # Remove disallowed tags instead of escaping
    )
    
    # Convert URLs to links
    cleaned = bleach.linkify(cleaned)
    
    return cleaned


xss_html = '<p>Hello <script>alert(1)</script> <b>World</b></p>'
print(f"\nSanitized HTML: {sanitize_html(xss_html)}")
```

---

## 5. CORS Configuration

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from typing import List

app = FastAPI()

# === CORS Settings ===
def get_cors_origins() -> List[str]:
    """ดึง allowed origins จาก config"""
    import os
    env = os.getenv("APP_ENV", "development")
    
    if env == "production":
        return [
            "https://myapp.com",
            "https://www.myapp.com",
            "https://admin.myapp.com",
        ]
    elif env == "staging":
        return [
            "https://staging.myapp.com",
            "https://staging-admin.myapp.com",
        ]
    else:
        # Development
        return [
            "http://localhost:3000",
            "http://localhost:8080",
            "http://127.0.0.1:3000",
        ]


# ตั้งค่า CORS Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=get_cors_origins(),    # Allowed origins
    allow_credentials=True,               # Allow cookies
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=[
        "Authorization",
        "Content-Type",
        "X-Request-ID",
    ],
    expose_headers=["X-Total-Count"],    # Headers ที่ client อ่านได้
    max_age=3600,                         # Preflight cache duration
)

# ❌ อย่าตั้ง allow_origins=["*"] ในระบบที่มี authentication
# เพราะ browser จะส่ง cookies ไปกับ request ได้ทุก origin


# === ตัวอย่าง Endpoints ===
@app.get("/api/data")
async def get_data():
    return {"data": "Protected data"}


@app.options("/api/data")
async def options_data():
    """Handle CORS preflight requests"""
    return {}
```

---

## 6. Rate Limiting

```python
from fastapi import FastAPI, Request, HTTPException
from collections import defaultdict
import time

app = FastAPI()

# === Simple In-Memory Rate Limiter ===
class RateLimiter:
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window = window_seconds
        self._requests: dict = defaultdict(list)
    
    def is_allowed(self, identifier: str) -> bool:
        now = time.time()
        window_start = now - self.window
        
        # ลบ requests เก่า
        self._requests[identifier] = [
            req_time for req_time in self._requests[identifier]
            if req_time > window_start
        ]
        
        # ตรวจสอบ limit
        if len(self._requests[identifier]) >= self.max_requests:
            return False
        
        self._requests[identifier].append(now)
        return True
    
    def get_reset_time(self, identifier: str) -> float:
        if not self._requests[identifier]:
            return 0
        oldest = min(self._requests[identifier])
        return oldest + self.window


# Rate limiters สำหรับ use cases ต่างๆ
login_limiter = RateLimiter(max_requests=5, window_seconds=300)  # 5 per 5 min
api_limiter = RateLimiter(max_requests=100, window_seconds=60)   # 100 per min


@app.post("/auth/login")
async def login(request: Request, username: str, password: str):
    ip = request.client.host
    
    if not login_limiter.is_allowed(ip):
        raise HTTPException(
            status_code=429,
            detail="Too many login attempts. Try again in 5 minutes.",
            headers={"Retry-After": str(int(login_limiter.get_reset_time(ip)))}
        )
    
    # Login logic...
    return {"message": "Login successful"}


# ใช้ slowapi สำหรับ production
# pip install slowapi
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/api/resource")
# @limiter.limit("100/minute")  # Uncomment เมื่อ setup slowapi
async def get_resource():
    return {"data": "resource"}
```

---

## 7. Secure File Upload

```python
from fastapi import FastAPI, UploadFile, HTTPException
from pathlib import Path
import magic  # pip install python-magic
import hashlib
import uuid

app = FastAPI()

ALLOWED_CONTENT_TYPES = {
    "image/jpeg", "image/png", "image/gif", "image/webp",
    "application/pdf",
}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB

UPLOAD_DIR = Path("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)


def validate_file(file: UploadFile) -> bool:
    """ตรวจสอบ file ก่อน upload"""
    
    # ตรวจสอบ extension
    allowed_extensions = {".jpg", ".jpeg", ".png", ".gif", ".webp", ".pdf"}
    ext = Path(file.filename).suffix.lower()
    if ext not in allowed_extensions:
        raise HTTPException(400, f"File type not allowed: {ext}")
    
    return True


async def save_file_securely(file: UploadFile) -> dict:
    """บันทึกไฟล์อย่างปลอดภัย"""
    
    validate_file(file)
    
    # อ่านเนื้อหาไฟล์
    content = await file.read()
    
    # ตรวจสอบขนาด
    if len(content) > MAX_FILE_SIZE:
        raise HTTPException(400, "File too large")
    
    # ตรวจสอบ content type จริง (ไม่ใช่แค่ extension)
    # magic ตรวจสอบ file signature (magic bytes)
    try:
        real_content_type = magic.from_buffer(content, mime=True)
        if real_content_type not in ALLOWED_CONTENT_TYPES:
            raise HTTPException(400, f"Invalid file content: {real_content_type}")
    except ImportError:
        pass  # Skip ถ้าไม่มี libmagic
    
    # สร้างชื่อไฟล์ใหม่ที่ปลอดภัย (ป้องกัน path traversal)
    ext = Path(file.filename).suffix.lower()
    safe_filename = f"{uuid.uuid4().hex}{ext}"
    
    # คำนวณ checksum
    file_hash = hashlib.sha256(content).hexdigest()
    
    # บันทึกไฟล์
    file_path = UPLOAD_DIR / safe_filename
    file_path.write_bytes(content)
    
    return {
        "original_name": file.filename,
        "saved_as": safe_filename,
        "size": len(content),
        "checksum": file_hash,
    }


@app.post("/upload")
async def upload_file(file: UploadFile):
    result = await save_file_securely(file)
    return result
```

---

## 8. Secrets Management Best Practices

```python
import secrets
import os
from pathlib import Path

# === สร้าง Secure Keys ===
def generate_keys():
    # Secret key สำหรับ JWT
    jwt_secret = secrets.token_urlsafe(32)
    print(f"JWT Secret: {jwt_secret}")
    
    # API Key
    api_key = secrets.token_hex(32)
    print(f"API Key: {api_key}")
    
    # One-time token (password reset, email verification)
    verification_token = secrets.token_urlsafe(24)
    print(f"Verification Token: {verification_token}")
    
    # Secure random number
    secure_int = secrets.randbelow(1000000)
    print(f"Secure OTP: {secure_int:06d}")


# === Environment Variable Validation ===
REQUIRED_SECRETS = [
    "SECRET_KEY",
    "DATABASE_URL",
]

def validate_environment():
    """ตรวจสอบ required secrets ตอน startup"""
    missing = []
    warnings = []
    
    for key in REQUIRED_SECRETS:
        if not os.environ.get(key):
            missing.append(key)
    
    # ตรวจสอบ weak secrets
    secret_key = os.environ.get("SECRET_KEY", "")
    if len(secret_key) < 32:
        warnings.append("SECRET_KEY is too short (minimum 32 characters)")
    
    if "changeme" in secret_key.lower() or "secret" == secret_key.lower():
        warnings.append("SECRET_KEY appears to be a default/weak value")
    
    if missing:
        raise EnvironmentError(
            f"Missing required environment variables: {', '.join(missing)}"
        )
    
    for warning in warnings:
        print(f"⚠️  Security Warning: {warning}")
    
    print("✅ Environment validation passed")


# ทดสอบ
os.environ["SECRET_KEY"] = secrets.token_urlsafe(32)
os.environ["DATABASE_URL"] = "sqlite:///./test.db"
validate_environment()
generate_keys()
```

---

## 9. Security Checklist

```python
"""
=== Security Checklist สำหรับ Python Web Apps ===

Authentication:
✅ ใช้ bcrypt/argon2 สำหรับ password hashing
✅ JWT tokens พร้อม expiry
✅ Refresh token rotation
✅ Rate limit login attempts
✅ Account lockout หลัง failed attempts
✅ Secure session cookies (HttpOnly, Secure, SameSite)

Authorization:
✅ ตรวจสอบ permissions ทุก endpoint
✅ Principle of Least Privilege
✅ ป้องกัน Insecure Direct Object Reference (IDOR)
✅ Row-level security สำหรับ multi-tenant apps

Input Validation:
✅ Validate ทุก user input (Pydantic)
✅ ใช้ parameterized queries
✅ Sanitize HTML content
✅ Validate file uploads (type, size, content)

Transport Security:
✅ HTTPS ทุก environments
✅ HSTS headers
✅ Redirect HTTP to HTTPS
✅ TLS 1.2+ เท่านั้น

Headers Security:
✅ Content-Security-Policy
✅ X-Content-Type-Options: nosniff
✅ X-Frame-Options: DENY
✅ X-XSS-Protection
✅ Referrer-Policy

Dependencies:
✅ อัปเดต dependencies สม่ำเสมอ
✅ ใช้ safety/pip-audit ตรวจสอบ vulnerabilities
✅ Pin dependency versions

Secrets:
✅ ไม่ hardcode secrets ใน code
✅ ใช้ environment variables
✅ ไม่ commit .env ลง git
✅ Rotate keys เป็นประจำ

Logging & Monitoring:
✅ Log authentication attempts
✅ Log authorization failures
✅ ไม่ log sensitive data (passwords, tokens)
✅ Monitor for suspicious activity
"""

print("Security checklist printed above")
```

---

## 10. สรุป Part 045

✅ **Password Hashing** - bcrypt (rounds=12), Argon2id (recommended)  
✅ **JWT Tokens** - access/refresh tokens, expiry, validation  
✅ **SQL Injection** - parameterized queries, ORM, pydantic validation  
✅ **XSS Prevention** - HTML escaping, bleach sanitization, CSP headers  
✅ **CORS** - proper origins, credentials, methods configuration  
✅ **Rate Limiting** - ป้องกัน brute force, abuse  
✅ **Secure File Upload** - type validation, path traversal prevention  
✅ **Secrets Management** - secure generation, environment validation  
✅ **Security Headers** - HSTS, CSP, X-Frame-Options  

**Golden Rules:**
- ไม่ไว้ใจ user input เลย
- Defense in depth - หลายชั้นป้องกัน
- Fail securely - error ต้องไม่เปิดเผยข้อมูล
- เข้าเรื่อง OWASP Top 10 ทุกปี

## ➡️ ถัดไป: Part 047 - CLI Applications
*Part 045/100+ | Python Course - Beginner to World-Class*
