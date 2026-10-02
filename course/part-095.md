# Part 095 - FastAPI Authentication

## เป้าหมายการเรียนรู้

- ใช้ OAuth2 with Password flow
- สร้างและ validate JWT tokens
- ใช้ Dependencies (Depends) สำหรับ auth
- Security utilities
- Middleware
- Rate limiting

---

## 1. Authentication แบบต่างๆ ใน FastAPI

```
1. OAuth2 Password Flow - username/password -> JWT token
2. OAuth2 Bearer Token - API key หรือ token ใน header
3. HTTP Basic Auth - username:password encoded ใน header
4. API Key - ใน header, query, cookie
```

---

## 2. OAuth2 + JWT Setup

```bash
pip install python-jose[cryptography] passlib[bcrypt]
```

```python
# app/auth/security.py - JWT utilities

from datetime import datetime, timedelta, timezone
from typing import Optional, Any
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import HTTPException, status
from app.config import get_settings

settings = get_settings()

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    """Hash password ด้วย bcrypt"""
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    """ตรวจสอบ password"""
    return pwd_context.verify(plain_password, hashed_password)


def create_access_token(
    subject: Any,
    expires_delta: Optional[timedelta] = None,
    additional_claims: dict = None
) -> str:
    """สร้าง JWT access token"""
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(
            minutes=settings.access_token_expire_minutes
        )
    
    payload = {
        "sub": str(subject),    # subject (user ID)
        "exp": expire,          # expiration
        "iat": datetime.now(timezone.utc),  # issued at
        "type": "access"
    }
    
    if additional_claims:
        payload.update(additional_claims)
    
    return jwt.encode(payload, settings.secret_key, algorithm=settings.algorithm)


def create_refresh_token(subject: Any) -> str:
    """สร้าง JWT refresh token"""
    expire = datetime.now(timezone.utc) + timedelta(
        days=settings.refresh_token_expire_days
    )
    
    payload = {
        "sub": str(subject),
        "exp": expire,
        "iat": datetime.now(timezone.utc),
        "type": "refresh"
    }
    
    return jwt.encode(payload, settings.secret_key, algorithm=settings.algorithm)


def decode_token(token: str) -> dict:
    """Decode และ validate JWT token"""
    try:
        payload = jwt.decode(
            token,
            settings.secret_key,
            algorithms=[settings.algorithm]
        )
        return payload
    except JWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token ไม่ถูกต้องหรือหมดอายุ",
            headers={"WWW-Authenticate": "Bearer"},
        )
```

---

## 3. Auth Schemas

```python
# app/schemas/auth.py

from pydantic import BaseModel
from typing import Optional


class LoginRequest(BaseModel):
    username: str
    password: str


class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int  # วินาที


class RefreshRequest(BaseModel):
    refresh_token: str


class TokenData(BaseModel):
    """ข้อมูลที่ decode จาก token"""
    user_id: Optional[int] = None
    username: Optional[str] = None
    role: Optional[str] = None
    scopes: list[str] = []
```

---

## 4. Auth Router

```python
# app/routers/auth.py

from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.ext.asyncio import AsyncSession
from datetime import timedelta
from app.database import get_db
from app.repositories.user import UserRepository
from app.auth.security import (
    create_access_token, create_refresh_token,
    decode_token, verify_password
)
from app.schemas.auth import TokenResponse, RefreshRequest
from app.config import get_settings

router = APIRouter(prefix="/auth", tags=["authentication"])
settings = get_settings()


@router.post("/login", response_model=TokenResponse)
async def login(
    # OAuth2PasswordRequestForm รับ username + password จาก form data
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db)
):
    """Login ด้วย username/password"""
    repo = UserRepository(db)
    
    # ค้นหา user (username หรือ email)
    user = await repo.get_by_username(form_data.username)
    if not user:
        user = await repo.get_by_email(form_data.username)
    
    # ตรวจสอบ credentials
    if not user or not verify_password(form_data.password, user.password_hash):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Username หรือ Password ไม่ถูกต้อง",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="บัญชีถูกระงับ"
        )
    
    # สร้าง tokens
    access_token = create_access_token(
        subject=user.id,
        additional_claims={
            "username": user.username,
            "role": "admin" if user.is_admin else "user"
        }
    )
    refresh_token = create_refresh_token(subject=user.id)
    
    # อัปเดต last_login
    await repo.update_last_login(user.id)
    
    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token,
        expires_in=settings.access_token_expire_minutes * 60
    )


@router.post("/refresh", response_model=TokenResponse)
async def refresh_token(
    request: RefreshRequest,
    db: AsyncSession = Depends(get_db)
):
    """สร้าง access token ใหม่ด้วย refresh token"""
    try:
        payload = decode_token(request.refresh_token)
        
        # ตรวจสอบว่าเป็น refresh token
        if payload.get("type") != "refresh":
            raise HTTPException(status_code=401, detail="ต้องการ refresh token")
        
        user_id = int(payload.get("sub"))
        
    except Exception:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Refresh token ไม่ถูกต้อง"
        )
    
    # ตรวจสอบว่า user ยังอยู่
    repo = UserRepository(db)
    user = await repo.get(user_id)
    
    if not user or not user.is_active:
        raise HTTPException(status_code=401, detail="ผู้ใช้ไม่พบหรือถูกระงับ")
    
    # สร้าง tokens ใหม่
    new_access_token = create_access_token(
        subject=user.id,
        additional_claims={"username": user.username, "role": "admin" if user.is_admin else "user"}
    )
    new_refresh_token = create_refresh_token(subject=user.id)
    
    return TokenResponse(
        access_token=new_access_token,
        refresh_token=new_refresh_token,
        expires_in=settings.access_token_expire_minutes * 60
    )
```

---

## 5. Dependencies

```python
# app/dependencies/auth.py - Auth dependencies

from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.repositories.user import UserRepository
from app.auth.security import decode_token
from app.models.user import User

# OAuth2 scheme - ดึง token จาก "Authorization: Bearer <token>"
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")


async def get_token_payload(token: str = Depends(oauth2_scheme)) -> dict:
    """Decode token และส่งคืน payload"""
    return decode_token(token)


async def get_current_user(
    payload: dict = Depends(get_token_payload),
    db: AsyncSession = Depends(get_db)
) -> User:
    """ดึง user ปัจจุบันจาก token"""
    user_id = payload.get("sub")
    
    if not user_id:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token ไม่ถูกต้อง"
        )
    
    repo = UserRepository(db)
    user = await repo.get(int(user_id))
    
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="ไม่พบผู้ใช้"
        )
    
    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="บัญชีถูกระงับ"
        )
    
    return user


async def get_current_active_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """ตรวจสอบว่า user active"""
    # (already checked in get_current_user)
    return current_user


async def get_admin_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """ต้องเป็น admin user"""
    if not current_user.is_admin:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="ต้องการสิทธิ์ admin"
        )
    return current_user


class RequireScopes:
    """Dependency ที่ตรวจสอบ scopes"""
    
    def __init__(self, required_scopes: list[str]):
        self.required_scopes = required_scopes
    
    async def __call__(
        self,
        payload: dict = Depends(get_token_payload),
        db: AsyncSession = Depends(get_db)
    ) -> User:
        token_scopes = payload.get("scopes", [])
        
        for scope in self.required_scopes:
            if scope not in token_scopes:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail=f"ต้องการ scope: {scope}"
                )
        
        user_id = int(payload["sub"])
        repo = UserRepository(db)
        return await repo.get_or_404(user_id)


# ─────────────────────────────────────────
# ใช้งาน dependencies ใน routes
# ─────────────────────────────────────────

from fastapi import APIRouter

router = APIRouter()


@router.get("/me")
async def get_me(current_user: User = Depends(get_current_user)):
    """ดูข้อมูลตัวเอง - ต้อง login"""
    return {
        "id": current_user.id,
        "username": current_user.username,
        "email": current_user.email,
        "is_admin": current_user.is_admin
    }


@router.get("/admin/dashboard")
async def admin_dashboard(admin: User = Depends(get_admin_user)):
    """Admin only endpoint"""
    return {"message": f"Welcome admin {admin.username}"}


@router.get("/posts/create")
async def create_post_page(
    user: User = Depends(RequireScopes(["posts:write"]))
):
    """ต้องมี scope posts:write"""
    return {"message": "Create post form"}
```

---

## 6. Optional Authentication

```python
# app/dependencies/optional_auth.py

from fastapi import Depends
from fastapi.security import OAuth2PasswordBearer
from typing import Optional
from app.models.user import User

# auto_error=False ทำให้ไม่ raise error ถ้าไม่มี token
oauth2_scheme_optional = OAuth2PasswordBearer(
    tokenUrl="/api/v1/auth/login",
    auto_error=False
)


async def get_optional_user(
    token: Optional[str] = Depends(oauth2_scheme_optional),
    db: AsyncSession = Depends(get_db)
) -> Optional[User]:
    """ดึง current user ถ้ามี token, หรือ None ถ้าไม่มี"""
    if not token:
        return None
    
    try:
        payload = decode_token(token)
        user_id = int(payload.get("sub", 0))
        repo = UserRepository(db)
        user = await repo.get(user_id)
        return user if user and user.is_active else None
    except Exception:
        return None


# ใช้งาน
@router.get("/posts")
async def list_posts(
    current_user: Optional[User] = Depends(get_optional_user)
):
    """แสดงโพสต์ - login จะเห็นโพสต์ส่วนตัวด้วย"""
    if current_user:
        # แสดงทั้ง public และ private posts
        return {"message": f"Welcome {current_user.username}!", "show_private": True}
    else:
        # แสดงแค่ public posts
        return {"message": "Guest user", "show_private": False}
```

---

## 7. Middleware

```python
# app/middleware/auth.py - Auth middleware

from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.types import ASGIApp
import time
import logging

logger = logging.getLogger(__name__)


class RequestLoggingMiddleware(BaseHTTPMiddleware):
    """Middleware สำหรับ log ทุก request"""
    
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()
        
        # Log request
        logger.info(f"→ {request.method} {request.url.path}")
        
        # Process request
        response = await call_next(request)
        
        # Log response
        duration = time.time() - start_time
        logger.info(
            f"← {request.method} {request.url.path} "
            f"[{response.status_code}] {duration:.3f}s"
        )
        
        # เพิ่ม response time header
        response.headers["X-Response-Time"] = f"{duration:.3f}s"
        
        return response


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """เพิ่ม security headers"""
    
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Security headers
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        
        return response


# ลงทะเบียน middleware ใน main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(RequestLoggingMiddleware)
app.add_middleware(SecurityHeadersMiddleware)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## 8. Rate Limiting

```bash
pip install slowapi
```

```python
# app/middleware/rate_limit.py - Rate limiting

from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import FastAPI, Request, Depends
from app.dependencies.auth import get_current_user

# Limiter instance
limiter = Limiter(key_func=get_remote_address)


def setup_rate_limiting(app: FastAPI):
    """ตั้งค่า rate limiting สำหรับ app"""
    app.state.limiter = limiter
    app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)


# ใช้งาน decorator บน routes
from fastapi import APIRouter

router = APIRouter()


@router.post("/auth/login")
@limiter.limit("5/minute")  # สูงสุด 5 ครั้งต่อนาที
async def login(request: Request):
    pass


@router.post("/auth/register")
@limiter.limit("3/hour")  # สูงสุด 3 ครั้งต่อชั่วโมง
async def register(request: Request):
    pass


@router.get("/search")
@limiter.limit("30/minute")
async def search(request: Request):
    pass


# Custom rate limit key (ใช้ user ID แทน IP)
def get_user_id_or_ip(request: Request) -> str:
    """ใช้ user ID เป็น rate limit key ถ้า login แล้ว"""
    user = getattr(request.state, "user", None)
    if user:
        return f"user:{user.id}"
    return get_remote_address(request)


user_limiter = Limiter(key_func=get_user_id_or_ip)
```

---

## Exercises

### Exercise 1: Token Blacklisting
สร้าง token blacklist ด้วย Redis:
- เมื่อ logout ให้ add token ใน blacklist
- ตรวจสอบ blacklist ก่อน process request
- TTL ตาม token expiry

### Exercise 2: Role-Based Access Control
สร้าง RBAC system:
- Roles: admin, editor, viewer
- Permissions: create, read, update, delete
- Middleware ตรวจสอบ role

### Exercise 3: API Key Authentication
สร้าง API key authentication:
- Generate API keys
- Validate ใน header: `X-API-Key`
- Rate limit ตาม API key

---

## สรุป

สิ่งที่เรียนรู้:
- **JWT tokens** - สร้างและ validate
- **OAuth2** with Password flow
- **Dependencies** - get_current_user, get_admin_user
- **Optional auth** สำหรับ public/private endpoints
- **Middleware** logging, security headers
- **Rate limiting** ป้องกัน abuse

---

## ลิงก์ Part ถัดไป

➡️ [Part 096 - FastAPI Advanced](./part-096.md)
