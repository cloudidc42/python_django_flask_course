# Part 103: Security Advanced

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ OWASP Top 10 vulnerabilities สำหรับ Python
- ใช้ Secure Coding practices
- จัดการ Secrets Rotation
- Implement OAuth2/OIDC flows
- API Security best practices

---

## 1. OWASP Top 10 สำหรับ Python

### A01: Broken Access Control

```python
# security/access_control.py
"""
A01: Broken Access Control
ตัวอย่างช่องโหว่และวิธีป้องกัน
"""
from fastapi import FastAPI, HTTPException, Depends, Request
from sqlalchemy.orm import Session
from typing import Optional
import jwt
import os

app = FastAPI()

# ❌ ช่องโหว่: ไม่ตรวจสอบ ownership
@app.get("/orders/{order_id}")
async def get_order_insecure(order_id: int, db: Session = Depends()):
    """ทุกคนสามารถดู order ของคนอื่นได้!"""
    order = db.query(Order).filter(Order.id == order_id).first()
    return order  # ไม่ตรวจสอบว่าเป็น order ของ user นี้หรือไม่


# ✅ ป้องกัน: ตรวจสอบ ownership เสมอ
@app.get("/orders/{order_id}/secure")
async def get_order_secure(
    order_id: int,
    current_user: dict = Depends(get_current_user),
    db: Session = Depends()
):
    """ตรวจสอบว่า order เป็นของ user ที่ขอ"""
    order = db.query(Order).filter(Order.id == order_id).first()
    
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")
    
    # ตรวจสอบ ownership
    if order.user_id != current_user["user_id"] and not current_user.get("is_admin"):
        raise HTTPException(status_code=403, detail="Access denied")
    
    return order


# Role-Based Access Control (RBAC)
from enum import Enum
from typing import List
import functools


class Permission(Enum):
    READ_ORDERS = "read:orders"
    CREATE_ORDERS = "create:orders"
    DELETE_ORDERS = "delete:orders"
    MANAGE_USERS = "manage:users"
    VIEW_REPORTS = "view:reports"


ROLE_PERMISSIONS = {
    "customer": [Permission.READ_ORDERS, Permission.CREATE_ORDERS],
    "support": [Permission.READ_ORDERS, Permission.VIEW_REPORTS],
    "admin": list(Permission),  # All permissions
}


def require_permissions(*permissions: Permission):
    """Decorator สำหรับตรวจสอบ permissions"""
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            # ดึง current user จาก request
            request = None
            for arg in args:
                if isinstance(arg, Request):
                    request = arg
                    break
            
            if not request:
                raise HTTPException(status_code=500, detail="Request not found")
            
            # ดึง user และ role
            user = getattr(request.state, "user", None)
            if not user:
                raise HTTPException(status_code=401, detail="Not authenticated")
            
            user_role = user.get("role", "customer")
            user_permissions = ROLE_PERMISSIONS.get(user_role, [])
            
            # ตรวจสอบว่ามี permissions ครบ
            for permission in permissions:
                if permission not in user_permissions:
                    raise HTTPException(
                        status_code=403,
                        detail=f"Permission denied: {permission.value}"
                    )
            
            return await func(*args, **kwargs)
        return wrapper
    return decorator


@app.delete("/orders/{order_id}")
@require_permissions(Permission.DELETE_ORDERS)
async def delete_order(order_id: int, request: Request):
    """ลบ order (ต้อง permission DELETE_ORDERS)"""
    pass
```

### A02: Cryptographic Failures

```python
# security/cryptography.py
"""
A02: Cryptographic Failures
"""
import hashlib
import hmac
import secrets
import os
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.backends import default_backend
import base64


# ❌ ไม่ดี: ใช้ MD5 หรือ SHA1 สำหรับ passwords
def bad_hash_password(password: str) -> str:
    return hashlib.md5(password.encode()).hexdigest()


# ✅ ดี: ใช้ bcrypt หรือ argon2 สำหรับ passwords
from passlib.context import CryptContext

pwd_context = CryptContext(
    schemes=["argon2", "bcrypt"],
    deprecated="auto",
    argon2__rounds=4,
    argon2__memory_cost=65536,
    argon2__parallelism=2
)


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(password: str, hashed: str) -> bool:
    return pwd_context.verify(password, hashed)


# Symmetric Encryption ด้วย Fernet
class DataEncryptor:
    """Encrypt/decrypt sensitive data"""
    
    def __init__(self, key: bytes = None):
        if key:
            self.key = key
        else:
            # ดึง key จาก environment
            key_b64 = os.getenv("ENCRYPTION_KEY")
            if key_b64:
                self.key = base64.b64decode(key_b64)
            else:
                raise ValueError("ENCRYPTION_KEY not set")
        
        self.fernet = Fernet(self.key)
    
    @staticmethod
    def generate_key() -> str:
        """สร้าง encryption key ใหม่"""
        key = Fernet.generate_key()
        return base64.b64encode(key).decode()
    
    def encrypt(self, data: str) -> str:
        """Encrypt string data"""
        encrypted = self.fernet.encrypt(data.encode())
        return base64.b64encode(encrypted).decode()
    
    def decrypt(self, encrypted_data: str) -> str:
        """Decrypt data"""
        encrypted_bytes = base64.b64decode(encrypted_data)
        decrypted = self.fernet.decrypt(encrypted_bytes)
        return decrypted.decode()
    
    def encrypt_dict(self, data: dict) -> dict:
        """Encrypt sensitive fields ใน dict"""
        import json
        encrypted = {}
        for key, value in data.items():
            if isinstance(value, str):
                encrypted[key] = self.encrypt(value)
            else:
                encrypted[key] = value
        return encrypted


# Secure Token Generation
def generate_secure_token(length: int = 32) -> str:
    """สร้าง cryptographically secure token"""
    return secrets.token_urlsafe(length)


def generate_otp(length: int = 6) -> str:
    """สร้าง OTP ที่ปลอดภัย"""
    return "".join([str(secrets.randbelow(10)) for _ in range(length)])


# HMAC สำหรับ webhook signature verification
def generate_webhook_signature(payload: bytes, secret: str) -> str:
    """สร้าง HMAC signature สำหรับ webhook"""
    signature = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return f"sha256={signature}"


def verify_webhook_signature(payload: bytes, secret: str, signature: str) -> bool:
    """ตรวจสอบ webhook signature (constant-time comparison)"""
    expected = generate_webhook_signature(payload, secret)
    return hmac.compare_digest(signature, expected)
```

### A03: SQL Injection

```python
# security/sql_injection.py
"""
A03: SQL Injection Prevention
"""
from sqlalchemy import text
from sqlalchemy.orm import Session


# ❌ ช่องโหว่: SQL Injection
def get_user_by_name_vulnerable(name: str, db: Session) -> list:
    """ช่องโหว่: name = "'; DROP TABLE users; --" """
    query = f"SELECT * FROM users WHERE name = '{name}'"  # NEVER DO THIS!
    return db.execute(text(query)).fetchall()


# ✅ ป้องกัน: Parameterized queries
def get_user_by_name_safe(name: str, db: Session) -> list:
    """ปลอดภัย: ใช้ parameterized query"""
    query = text("SELECT * FROM users WHERE name = :name")
    return db.execute(query, {"name": name}).fetchall()


# ✅ ดีที่สุด: ใช้ SQLAlchemy ORM
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import select

Base = declarative_base()


class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)


def get_users_safe(name: str, db: Session) -> list:
    """ORM ป้องกัน SQL Injection โดยอัตโนมัติ"""
    return db.query(User).filter(User.name == name).all()


# Input Validation
from pydantic import BaseModel, validator, field_validator
import re


class UserCreateRequest(BaseModel):
    username: str
    email: str
    
    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        """ตรวจสอบ username"""
        if not re.match(r"^[a-zA-Z0-9_-]{3,50}$", v):
            raise ValueError(
                "Username must be 3-50 characters and contain only "
                "letters, numbers, underscores, or hyphens"
            )
        return v
    
    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        """ตรวจสอบ email format"""
        if not re.match(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$", v):
            raise ValueError("Invalid email format")
        return v.lower()


# XSS Prevention
import html
import bleach


def sanitize_html(content: str, allowed_tags: list = None) -> str:
    """ทำความสะอาด HTML input"""
    if allowed_tags is None:
        allowed_tags = ["b", "i", "em", "strong", "p", "br"]
    
    allowed_attrs = {
        "a": ["href", "title"],
        "img": ["src", "alt"],
    }
    
    # ใช้ bleach เพื่อ sanitize HTML
    cleaned = bleach.clean(
        content,
        tags=allowed_tags,
        attributes=allowed_attrs,
        strip=True
    )
    return cleaned


def escape_output(data: str) -> str:
    """Escape HTML characters สำหรับ output"""
    return html.escape(data, quote=True)
```

### A07: Identification and Authentication Failures

```python
# security/authentication.py
"""
A07: Authentication Best Practices
"""
import jwt
import secrets
import time
import hashlib
import os
from datetime import datetime, timedelta
from typing import Optional
from fastapi import HTTPException, Depends, Cookie, Header
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials


SECRET_KEY = os.getenv("JWT_SECRET_KEY", "change-this-in-production")
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 15    # Short-lived access token
REFRESH_TOKEN_EXPIRE_DAYS = 30


class AuthenticationService:
    """Authentication service"""
    
    def __init__(self, redis_client, db_session):
        self.redis = redis_client
        self.db = db_session
    
    def create_access_token(self, user_id: int, email: str, role: str) -> str:
        """สร้าง JWT access token"""
        payload = {
            "sub": str(user_id),  # subject
            "email": email,
            "role": role,
            "type": "access",
            "iat": int(time.time()),  # issued at
            "exp": int(time.time()) + (ACCESS_TOKEN_EXPIRE_MINUTES * 60),
            "jti": secrets.token_hex(16)  # JWT ID สำหรับ revocation
        }
        return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
    
    def create_refresh_token(self, user_id: int) -> str:
        """สร้าง refresh token"""
        token = secrets.token_urlsafe(64)
        
        # เก็บ refresh token ใน Redis
        key = f"refresh:{hashlib.sha256(token.encode()).hexdigest()}"
        self.redis.setex(
            key,
            REFRESH_TOKEN_EXPIRE_DAYS * 86400,
            str(user_id)
        )
        
        return token
    
    def verify_access_token(self, token: str) -> dict:
        """ตรวจสอบ JWT token"""
        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
            
            # ตรวจสอบว่า token ถูก revoke หรือไม่
            jti = payload.get("jti")
            if jti and self.redis.get(f"revoked:{jti}"):
                raise HTTPException(status_code=401, detail="Token has been revoked")
            
            return payload
        
        except jwt.ExpiredSignatureError:
            raise HTTPException(status_code=401, detail="Token expired")
        except jwt.InvalidTokenError as e:
            raise HTTPException(status_code=401, detail=f"Invalid token: {str(e)}")
    
    def refresh_tokens(self, refresh_token: str) -> dict:
        """Refresh access token ด้วย refresh token"""
        token_hash = hashlib.sha256(refresh_token.encode()).hexdigest()
        key = f"refresh:{token_hash}"
        
        user_id = self.redis.get(key)
        if not user_id:
            raise HTTPException(status_code=401, detail="Invalid refresh token")
        
        # Rotate refresh token (ป้องกัน token theft)
        self.redis.delete(key)
        
        # ดึง user data
        user = self.db.get_user(int(user_id))
        if not user or not user.is_active:
            raise HTTPException(status_code=401, detail="User not found or inactive")
        
        # สร้าง tokens ใหม่
        new_access_token = self.create_access_token(user.id, user.email, user.role)
        new_refresh_token = self.create_refresh_token(user.id)
        
        return {
            "access_token": new_access_token,
            "refresh_token": new_refresh_token,
            "token_type": "bearer"
        }
    
    def revoke_token(self, token: str):
        """Revoke access token (logout)"""
        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
            jti = payload.get("jti")
            exp = payload.get("exp", 0)
            
            if jti:
                # เก็บ JTI ใน blacklist จนกว่า token จะหมดอายุ
                ttl = max(0, exp - int(time.time()))
                if ttl > 0:
                    self.redis.setex(f"revoked:{jti}", ttl, "1")
        except jwt.InvalidTokenError:
            pass  # Token ไม่ valid อยู่แล้ว


# Multi-Factor Authentication (MFA)
class MFAService:
    """Two-Factor Authentication"""
    
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def generate_totp_secret(self) -> str:
        """สร้าง TOTP secret"""
        import pyotp
        return pyotp.random_base32()
    
    def get_totp_uri(self, secret: str, email: str, issuer: str = "MyApp") -> str:
        """ดึง TOTP URI สำหรับ QR code"""
        import pyotp
        totp = pyotp.TOTP(secret)
        return totp.provisioning_uri(email, issuer_name=issuer)
    
    def verify_totp(self, secret: str, code: str) -> bool:
        """ตรวจสอบ TOTP code"""
        import pyotp
        totp = pyotp.TOTP(secret)
        # valid_window=1 อนุญาต ±30 วินาที
        return totp.verify(code, valid_window=1)
    
    def send_sms_otp(self, phone: str) -> str:
        """ส่ง OTP ทาง SMS"""
        otp = generate_otp(6)
        
        # เก็บ OTP ใน Redis (หมดอายุใน 5 นาที)
        key = f"otp:sms:{phone}"
        self.redis.setex(key, 300, otp)
        
        # ส่ง SMS (ผ่าน Twilio, AWS SNS, etc.)
        # sms_client.send(phone, f"Your OTP is: {otp}")
        
        return "OTP sent"
    
    def verify_sms_otp(self, phone: str, code: str) -> bool:
        """ตรวจสอบ SMS OTP"""
        key = f"otp:sms:{phone}"
        stored_otp = self.redis.get(key)
        
        if not stored_otp:
            return False
        
        # ใช้ constant-time comparison
        is_valid = hmac.compare_digest(stored_otp, code)
        
        if is_valid:
            # ลบ OTP หลังใช้แล้ว (one-time use)
            self.redis.delete(key)
        
        return is_valid
```

---

## 2. OAuth2/OIDC Implementation

```python
# security/oauth2_oidc.py
"""
OAuth2 และ OpenID Connect Implementation
"""
import httpx
import jwt
import secrets
import hashlib
import base64
import os
from typing import Optional
from urllib.parse import urlencode, quote_plus


class OAuth2Client:
    """OAuth2 Authorization Code Flow with PKCE"""
    
    def __init__(
        self,
        client_id: str,
        client_secret: str,
        redirect_uri: str,
        authorization_endpoint: str,
        token_endpoint: str,
        userinfo_endpoint: str = None
    ):
        self.client_id = client_id
        self.client_secret = client_secret
        self.redirect_uri = redirect_uri
        self.authorization_endpoint = authorization_endpoint
        self.token_endpoint = token_endpoint
        self.userinfo_endpoint = userinfo_endpoint
    
    def _generate_pkce(self) -> tuple:
        """สร้าง PKCE code verifier และ challenge"""
        code_verifier = secrets.token_urlsafe(32)
        
        code_challenge = base64.urlsafe_b64encode(
            hashlib.sha256(code_verifier.encode()).digest()
        ).rstrip(b"=").decode()
        
        return code_verifier, code_challenge
    
    def get_authorization_url(
        self,
        state: str,
        scopes: list = None,
        use_pkce: bool = True
    ) -> tuple:
        """
        สร้าง Authorization URL
        Returns: (url, state, code_verifier)
        """
        params = {
            "response_type": "code",
            "client_id": self.client_id,
            "redirect_uri": self.redirect_uri,
            "state": state,
            "scope": " ".join(scopes or ["openid", "email", "profile"])
        }
        
        code_verifier = None
        if use_pkce:
            code_verifier, code_challenge = self._generate_pkce()
            params["code_challenge"] = code_challenge
            params["code_challenge_method"] = "S256"
        
        url = f"{self.authorization_endpoint}?{urlencode(params)}"
        return url, state, code_verifier
    
    async def exchange_code_for_tokens(
        self,
        code: str,
        state: str,
        expected_state: str,
        code_verifier: str = None
    ) -> dict:
        """แลก authorization code เป็น tokens"""
        
        # ตรวจสอบ state (ป้องกัน CSRF)
        if not hmac.compare_digest(state, expected_state):
            raise ValueError("State mismatch - possible CSRF attack")
        
        payload = {
            "grant_type": "authorization_code",
            "code": code,
            "redirect_uri": self.redirect_uri,
            "client_id": self.client_id,
            "client_secret": self.client_secret
        }
        
        if code_verifier:
            payload["code_verifier"] = code_verifier
        
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.token_endpoint,
                data=payload,
                headers={"Content-Type": "application/x-www-form-urlencoded"}
            )
            response.raise_for_status()
            tokens = response.json()
        
        # ตรวจสอบ ID Token ถ้ามี
        if "id_token" in tokens:
            tokens["user_info"] = self._decode_id_token(tokens["id_token"])
        
        return tokens
    
    def _decode_id_token(self, id_token: str) -> dict:
        """Decode และ verify ID Token"""
        # In production: ดึง public keys จาก JWKS endpoint และ verify signature
        payload = jwt.decode(
            id_token,
            options={"verify_signature": False}  # Simplified - ใน production ต้อง verify
        )
        return payload
    
    async def get_user_info(self, access_token: str) -> dict:
        """ดึง user info จาก userinfo endpoint"""
        if not self.userinfo_endpoint:
            return {}
        
        async with httpx.AsyncClient() as client:
            response = await client.get(
                self.userinfo_endpoint,
                headers={"Authorization": f"Bearer {access_token}"}
            )
            response.raise_for_status()
            return response.json()
    
    async def refresh_access_token(self, refresh_token: str) -> dict:
        """Refresh access token"""
        payload = {
            "grant_type": "refresh_token",
            "refresh_token": refresh_token,
            "client_id": self.client_id,
            "client_secret": self.client_secret
        }
        
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.token_endpoint,
                data=payload
            )
            response.raise_for_status()
            return response.json()


# Google OAuth2 Configuration
def create_google_oauth2_client() -> OAuth2Client:
    """สร้าง Google OAuth2 client"""
    return OAuth2Client(
        client_id=os.getenv("GOOGLE_CLIENT_ID"),
        client_secret=os.getenv("GOOGLE_CLIENT_SECRET"),
        redirect_uri=os.getenv("GOOGLE_REDIRECT_URI", "http://localhost:8000/auth/google/callback"),
        authorization_endpoint="https://accounts.google.com/o/oauth2/v2/auth",
        token_endpoint="https://oauth2.googleapis.com/token",
        userinfo_endpoint="https://www.googleapis.com/oauth2/v3/userinfo"
    )


# FastAPI OAuth2 endpoints
from fastapi import FastAPI, Request, Response
from fastapi.responses import RedirectResponse
import json

app = FastAPI()
google_client = create_google_oauth2_client()


@app.get("/auth/google/login")
async def google_login(request: Request, response: Response):
    """เริ่ม Google OAuth2 flow"""
    # สร้าง state token
    state = secrets.token_urlsafe(32)
    
    # สร้าง authorization URL
    url, state, code_verifier = google_client.get_authorization_url(
        state=state,
        scopes=["openid", "email", "profile"],
        use_pkce=True
    )
    
    # เก็บ state และ code_verifier ใน secure session/Redis
    # session["oauth_state"] = state
    # session["code_verifier"] = code_verifier
    
    return RedirectResponse(url=url)


@app.get("/auth/google/callback")
async def google_callback(
    request: Request,
    code: str,
    state: str
):
    """Handle Google OAuth2 callback"""
    # ดึง state และ code_verifier จาก session
    expected_state = "stored_state"  # จาก session
    code_verifier = "stored_verifier"  # จาก session
    
    # แลก code เป็น tokens
    tokens = await google_client.exchange_code_for_tokens(
        code=code,
        state=state,
        expected_state=expected_state,
        code_verifier=code_verifier
    )
    
    # ดึง user info
    user_info = tokens.get("user_info") or await google_client.get_user_info(
        tokens["access_token"]
    )
    
    # สร้างหรืออัพเดท user ใน database
    # user = await create_or_update_user(user_info)
    
    return {"user": user_info, "tokens": tokens}
```

---

## 3. API Security

```python
# security/api_security.py
"""
API Security Best Practices
"""
from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from starlette.middleware.sessions import SessionMiddleware
import time
import re


app = FastAPI()


# Security Headers Middleware
class SecurityHeadersMiddleware:
    """เพิ่ม security headers ทุก response"""
    
    async def __call__(self, request: Request, call_next):
        response = await call_next(request)
        
        # Prevent Clickjacking
        response.headers["X-Frame-Options"] = "DENY"
        
        # Prevent MIME sniffing
        response.headers["X-Content-Type-Options"] = "nosniff"
        
        # XSS Protection (Legacy browsers)
        response.headers["X-XSS-Protection"] = "1; mode=block"
        
        # HSTS (HTTPS only)
        response.headers["Strict-Transport-Security"] = (
            "max-age=31536000; includeSubDomains; preload"
        )
        
        # CSP - ควบคุม content ที่โหลดได้
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; "
            "script-src 'self' 'nonce-{nonce}'; "
            "style-src 'self' 'unsafe-inline'; "
            "img-src 'self' data: https:; "
            "connect-src 'self' https://api.example.com; "
            "frame-ancestors 'none'"
        )
        
        # Referrer Policy
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        
        # Permissions Policy
        response.headers["Permissions-Policy"] = (
            "geolocation=(), microphone=(), camera=()"
        )
        
        # Remove server information
        response.headers.pop("server", None)
        
        return response


# CORS Configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://app.example.com",
        "https://admin.example.com"
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    expose_headers=["X-Request-ID", "X-RateLimit-Remaining"],
    max_age=600
)

# Trusted Host
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["api.example.com", "*.example.com"]
)

# Session Middleware
app.add_middleware(
    SessionMiddleware,
    secret_key=os.getenv("SESSION_SECRET"),
    session_cookie="session",
    max_age=3600,
    same_site="lax",
    https_only=True  # Production เท่านั้น
)


# Input Sanitization Middleware
class InputSanitizationMiddleware:
    """Sanitize inputs ก่อน process"""
    
    async def __call__(self, request: Request, call_next):
        # ตรวจสอบ Content-Type
        content_type = request.headers.get("content-type", "")
        
        if "application/json" in content_type:
            try:
                body = await request.body()
                if len(body) > 10 * 1024 * 1024:  # 10MB limit
                    raise HTTPException(status_code=413, detail="Request too large")
                
                # ตรวจสอบ JSON ที่ malformed
                import json
                if body:
                    json.loads(body)
            except json.JSONDecodeError:
                raise HTTPException(status_code=400, detail="Invalid JSON")
        
        return await call_next(request)


# API Key Authentication
import hashlib
import hmac


class APIKeyAuth:
    """API Key based authentication"""
    
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def create_api_key(self, user_id: int, name: str, scopes: list) -> str:
        """สร้าง API key"""
        # สร้าง key: pk_live_xxxxx
        prefix = "pk_live"
        raw_key = secrets.token_urlsafe(32)
        api_key = f"{prefix}_{raw_key}"
        
        # Hash key ก่อนเก็บ (เหมือน password)
        key_hash = hashlib.sha256(api_key.encode()).hexdigest()
        
        # เก็บ metadata ใน Redis
        import json
        self.redis.hset(
            f"api_key:{key_hash}",
            mapping={
                "user_id": user_id,
                "name": name,
                "scopes": json.dumps(scopes),
                "created_at": time.time()
            }
        )
        
        # Return raw key (แสดงครั้งเดียว)
        return api_key
    
    def authenticate(self, api_key: str) -> dict:
        """ตรวจสอบ API key"""
        key_hash = hashlib.sha256(api_key.encode()).hexdigest()
        
        data = self.redis.hgetall(f"api_key:{key_hash}")
        if not data:
            raise HTTPException(status_code=401, detail="Invalid API key")
        
        # อัพเดท last used
        self.redis.hset(f"api_key:{key_hash}", "last_used", time.time())
        
        import json
        return {
            "user_id": int(data["user_id"]),
            "name": data["name"],
            "scopes": json.loads(data["scopes"])
        }


# Dependency สำหรับ API authentication
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

# JWT Bearer
jwt_bearer = HTTPBearer(auto_error=False)


async def get_current_user(
    api_key: Optional[str] = Depends(api_key_header),
    bearer: Optional[HTTPAuthorizationCredentials] = Depends(jwt_bearer)
) -> dict:
    """ดึง current user จาก API key หรือ JWT"""
    
    if api_key:
        # API Key authentication
        api_key_auth = APIKeyAuth(redis_client)
        return api_key_auth.authenticate(api_key)
    
    elif bearer:
        # JWT authentication
        auth_service = AuthenticationService(redis_client, db_session)
        payload = auth_service.verify_access_token(bearer.credentials)
        return {
            "user_id": int(payload["sub"]),
            "email": payload.get("email"),
            "role": payload.get("role", "customer")
        }
    
    raise HTTPException(
        status_code=401,
        detail="Authentication required",
        headers={"WWW-Authenticate": "Bearer"}
    )
```

---

## 4. Secrets Rotation

```python
# security/secrets_rotation.py
"""
Secrets Rotation - หลักการ Zero-Downtime Secret Rotation
"""
import boto3
import json
import time
from typing import List


class SecretsRotator:
    """จัดการการ rotate secrets"""
    
    def __init__(self, region: str = "ap-southeast-1"):
        self.sm = boto3.client("secretsmanager", region_name=region)
    
    def rotate_database_password(
        self,
        secret_id: str,
        db_host: str,
        db_name: str,
        db_username: str
    ):
        """Rotate database password แบบ zero-downtime"""
        
        # 1. ดึง current password
        current_secret = json.loads(
            self.sm.get_secret_value(SecretId=secret_id)["SecretString"]
        )
        
        # 2. สร้าง new password
        import secrets as sec
        new_password = sec.token_urlsafe(32)
        
        # 3. Create new password ใน database (ใช้ both passwords ชั่วคราว)
        import psycopg2
        
        try:
            conn = psycopg2.connect(
                host=db_host,
                database=db_name,
                user=db_username,
                password=current_secret["password"]
            )
            
            with conn.cursor() as cur:
                # สร้าง user ใหม่หรืออัพเดท password
                cur.execute(
                    f"ALTER USER {db_username} WITH PASSWORD %s",
                    (new_password,)
                )
                conn.commit()
        finally:
            conn.close()
        
        # 4. อัพเดท secret
        current_secret["password"] = new_password
        current_secret["rotated_at"] = time.time()
        
        self.sm.update_secret(
            SecretId=secret_id,
            SecretString=json.dumps(current_secret)
        )
        
        print(f"✅ Database password rotated for {secret_id}")
    
    def rotate_api_key(self, secret_id: str, api_service: str):
        """Rotate API key"""
        import secrets as sec
        
        new_key = sec.token_urlsafe(64)
        
        secret_data = {
            "api_key": new_key,
            "service": api_service,
            "rotated_at": time.time()
        }
        
        self.sm.update_secret(
            SecretId=secret_id,
            SecretString=json.dumps(secret_data)
        )
        
        print(f"✅ API key rotated for {api_service}")
    
    def setup_automatic_rotation(
        self,
        secret_id: str,
        rotation_lambda_arn: str,
        rotation_days: int = 30
    ):
        """ตั้งค่า automatic rotation"""
        self.sm.rotate_secret(
            SecretId=secret_id,
            RotationLambdaARN=rotation_lambda_arn,
            RotationRules={"AutomaticallyAfterDays": rotation_days}
        )
        print(f"✅ Automatic rotation set for {secret_id} every {rotation_days} days")
```

---

## 5. Security Testing

```python
# security/security_scanner.py
"""
Security Testing สำหรับ Python Applications
"""
import subprocess
import json
from pathlib import Path


class SecurityScanner:
    """เครื่องมือ scan security vulnerabilities"""
    
    def run_bandit(self, path: str = ".") -> dict:
        """
        Bandit: Static security analysis สำหรับ Python
        pip install bandit
        """
        result = subprocess.run(
            ["bandit", "-r", path, "-f", "json", "-ll"],
            capture_output=True,
            text=True
        )
        
        try:
            report = json.loads(result.stdout)
            issues = report.get("results", [])
            
            high_issues = [i for i in issues if i["issue_severity"] == "HIGH"]
            medium_issues = [i for i in issues if i["issue_severity"] == "MEDIUM"]
            
            print(f"\n=== Bandit Security Report ===")
            print(f"High severity issues: {len(high_issues)}")
            print(f"Medium severity issues: {len(medium_issues)}")
            
            for issue in high_issues[:5]:
                print(f"\n⚠️  HIGH: {issue['issue_text']}")
                print(f"   File: {issue['filename']}:{issue['line_number']}")
                print(f"   Code: {issue['code']}")
            
            return report
        except json.JSONDecodeError:
            print(f"Bandit output: {result.stdout}")
            return {}
    
    def run_safety(self) -> dict:
        """
        Safety: ตรวจสอบ vulnerabilities ใน dependencies
        pip install safety
        """
        result = subprocess.run(
            ["safety", "check", "--json"],
            capture_output=True,
            text=True
        )
        
        try:
            vulnerabilities = json.loads(result.stdout)
            
            print(f"\n=== Safety Dependency Check ===")
            if vulnerabilities:
                for vuln in vulnerabilities:
                    print(f"⚠️  {vuln[0]} {vuln[2]}: {vuln[3][:100]}")
            else:
                print("✅ No known vulnerabilities found!")
            
            return {"vulnerabilities": vulnerabilities}
        except json.JSONDecodeError:
            return {}
    
    def check_hardcoded_secrets(self, path: str = ".") -> list:
        """
        ตรวจสอบ hardcoded secrets ด้วย regex
        ควรใช้ detect-secrets หรือ truffleHog
        """
        import re
        import os
        
        patterns = {
            "AWS Access Key": r"AKIA[0-9A-Z]{16}",
            "AWS Secret Key": r"[0-9a-zA-Z/+]{40}",
            "Private Key": r"-----BEGIN (RSA |EC )?PRIVATE KEY-----",
            "Password in URL": r"://[^:]+:[^@]+@",
            "API Key Pattern": r"[aA][pP][iI]_?[kK][eE][yY]\s*=\s*['\"][^'\"]+['\"]",
        }
        
        findings = []
        
        for root, dirs, files in os.walk(path):
            # Skip certain directories
            dirs[:] = [d for d in dirs if d not in [".git", "node_modules", "__pycache__", ".venv"]]
            
            for filename in files:
                if filename.endswith((".py", ".env", ".yml", ".yaml", ".json")):
                    filepath = os.path.join(root, filename)
                    try:
                        with open(filepath, "r") as f:
                            content = f.read()
                        
                        for pattern_name, pattern in patterns.items():
                            matches = re.findall(pattern, content)
                            if matches:
                                findings.append({
                                    "file": filepath,
                                    "pattern": pattern_name,
                                    "count": len(matches)
                                })
                    except Exception:
                        pass
        
        if findings:
            print("\n⚠️  Potential hardcoded secrets found:")
            for finding in findings:
                print(f"  {finding['file']}: {finding['pattern']} ({finding['count']} matches)")
        else:
            print("✅ No obvious hardcoded secrets found")
        
        return findings
    
    def run_full_security_audit(self):
        """รัน full security audit"""
        print("🔒 Starting Security Audit...")
        
        print("\n1/3 Running Bandit static analysis...")
        bandit_results = self.run_bandit()
        
        print("\n2/3 Checking dependency vulnerabilities...")
        safety_results = self.run_safety()
        
        print("\n3/3 Checking for hardcoded secrets...")
        secret_findings = self.check_hardcoded_secrets()
        
        print("\n✅ Security audit completed!")
        return {
            "bandit": bandit_results,
            "safety": safety_results,
            "secrets": secret_findings
        }


# ตัวอย่างการใช้
if __name__ == "__main__":
    scanner = SecurityScanner()
    scanner.run_full_security_audit()
```

---

## 6. สรุป Part 103

✅ เข้าใจ OWASP Top 10 vulnerabilities และวิธีป้องกันใน Python
✅ ใช้ Role-Based Access Control (RBAC) สำหรับ authorization
✅ Secure password hashing ด้วย Argon2/bcrypt
✅ Symmetric Encryption ด้วย Fernet สำหรับ sensitive data
✅ ป้องกัน SQL Injection ด้วย parameterized queries และ ORM
✅ Implement OAuth2 Authorization Code Flow พร้อม PKCE
✅ API Security ด้วย security headers, CORS, rate limiting
✅ Secrets Rotation แบบ zero-downtime ด้วย AWS Secrets Manager
✅ Security testing ด้วย Bandit, Safety และ secret scanning

## ➡️ ถัดไป: Part 104 - Machine Learning with Python

*Part 103/105 | Python Course - World-Class Level*
