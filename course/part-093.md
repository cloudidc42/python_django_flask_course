# Part 093: FastAPI Middleware and CORS
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- สร้าง custom Middleware
- ตั้งค่า CORS middleware
- สร้าง timing middleware
- สร้าง logging middleware
- ใช้ middleware สำหรับ authentication

---

## 1. Middleware คืออะไร?

Middleware คือโค้ดที่ทำงานระหว่าง request และ response ทุกครั้ง

```
Client → [Middleware 1] → [Middleware 2] → Route Handler → [Middleware 2] → [Middleware 1] → Client
```

ใช้สำหรับ:
- Logging
- Authentication
- Rate limiting
- CORS
- Compression
- Request timing

---

## 2. สร้าง Middleware

### วิธีที่ 1: @app.middleware("http")
```python
# middleware_basic.py

from fastapi import FastAPI, Request
from fastapi.responses import Response
import time

app = FastAPI()


@app.middleware("http")
async def timing_middleware(request: Request, call_next):
    """วัดเวลาในการประมวลผล request"""
    start_time = time.time()
    
    # ส่ง request ไปให้ handler ต่อไป
    response = await call_next(request)
    
    # คำนวณเวลา
    process_time = time.time() - start_time
    
    # เพิ่ม header ใน response
    response.headers["X-Process-Time"] = f"{process_time:.4f}s"
    
    return response


@app.middleware("http")
async def log_requests(request: Request, call_next):
    """Log ทุก request"""
    import logging
    logger = logging.getLogger(__name__)
    
    # Log request
    logger.info(f"→ {request.method} {request.url.path}")
    
    response = await call_next(request)
    
    # Log response
    logger.info(f"← {response.status_code} {request.url.path}")
    
    return response


@app.get("/")
def root():
    return {"message": "Hello!"}
```

### วิธีที่ 2: Starlette BaseHTTPMiddleware
```python
# starlette_middleware.py

from fastapi import FastAPI
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response
import time
import logging

logger = logging.getLogger(__name__)


class TimingMiddleware(BaseHTTPMiddleware):
    """Middleware วัดเวลา request"""
    
    async def dispatch(self, request: Request, call_next) -> Response:
        start_time = time.time()
        
        response = await call_next(request)
        
        process_time = time.time() - start_time
        response.headers["X-Process-Time"] = f"{process_time:.4f}s"
        
        return response


class RequestLoggingMiddleware(BaseHTTPMiddleware):
    """Middleware log requests"""
    
    def __init__(self, app, log_headers: bool = False):
        super().__init__(app)
        self.log_headers = log_headers
    
    async def dispatch(self, request: Request, call_next) -> Response:
        # Log request
        log_data = {
            "method": request.method,
            "url": str(request.url),
            "client": request.client.host if request.client else "unknown",
        }
        
        if self.log_headers:
            log_data["headers"] = dict(request.headers)
        
        logger.info(f"Request: {log_data}")
        
        response = await call_next(request)
        
        logger.info(f"Response: {response.status_code}")
        
        return response


class RateLimitMiddleware(BaseHTTPMiddleware):
    """Simple rate limiting middleware"""
    
    def __init__(self, app, max_requests: int = 100, window: int = 60):
        super().__init__(app)
        self.max_requests = max_requests
        self.window = window
        self.requests = {}  # ใน production ใช้ Redis
    
    async def dispatch(self, request: Request, call_next) -> Response:
        client_ip = request.client.host if request.client else "unknown"
        current_time = time.time()
        
        # ล้าง requests เก่า
        self.requests = {
            ip: times
            for ip, times in self.requests.items()
            if any(t > current_time - self.window for t in times)
        }
        
        # ตรวจสอบ rate limit
        if client_ip not in self.requests:
            self.requests[client_ip] = []
        
        # กรอง requests ใน window
        self.requests[client_ip] = [
            t for t in self.requests[client_ip]
            if t > current_time - self.window
        ]
        
        if len(self.requests[client_ip]) >= self.max_requests:
            from starlette.responses import JSONResponse
            return JSONResponse(
                {"error": "Rate limit exceeded", "retry_after": self.window},
                status_code=429,
                headers={"Retry-After": str(self.window)}
            )
        
        self.requests[client_ip].append(current_time)
        
        response = await call_next(request)
        
        # เพิ่ม rate limit headers
        remaining = self.max_requests - len(self.requests[client_ip])
        response.headers["X-RateLimit-Limit"] = str(self.max_requests)
        response.headers["X-RateLimit-Remaining"] = str(remaining)
        
        return response


# ใช้ middleware
app = FastAPI()

app.add_middleware(TimingMiddleware)
app.add_middleware(RequestLoggingMiddleware, log_headers=False)
app.add_middleware(RateLimitMiddleware, max_requests=100, window=60)
```

---

## 3. CORS Middleware

CORS (Cross-Origin Resource Sharing) อนุญาตให้ web browsers เข้าถึง API จาก origin อื่น

```python
# cors_setup.py

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# CORS Configuration
origins = [
    "http://localhost:3000",         # React development
    "http://localhost:8080",         # Vue development
    "https://myapp.com",             # Production frontend
    "https://www.myapp.com",
]

app.add_middleware(
    CORSMiddleware,
    
    # อนุญาต origins
    allow_origins=origins,
    # หรือ allow_origins=["*"] สำหรับ public API (ไม่แนะนำสำหรับ auth endpoints)
    
    # อนุญาต credentials (cookies, Authorization headers)
    allow_credentials=True,
    
    # อนุญาต HTTP methods
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    # หรือ allow_methods=["*"]
    
    # อนุญาต headers
    allow_headers=["*"],
    # หรือระบุเฉพาะ: allow_headers=["Content-Type", "Authorization"]
    
    # Expose headers ให้ browser อ่านได้
    expose_headers=["X-Request-ID", "X-Process-Time"],
    
    # Cache preflight request นานแค่ไหน (วินาที)
    max_age=600
)


@app.get("/api/data")
def get_data():
    return {"data": "accessible from other origins"}
```

### CORS สำหรับ Environment ต่างๆ
```python
# cors_by_environment.py

import os
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

env = os.environ.get("ENVIRONMENT", "development")

if env == "development":
    # Development: อนุญาตทุก origin
    origins = ["*"]
    allow_credentials = False  # ไม่ได้กับ wildcard
elif env == "staging":
    origins = [
        "https://staging.myapp.com",
        "http://localhost:3000"
    ]
    allow_credentials = True
else:  # production
    origins = [
        "https://myapp.com",
        "https://www.myapp.com",
        "https://app.myapp.com"
    ]
    allow_credentials = True

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=allow_credentials,
    allow_methods=["*"],
    allow_headers=["*"]
)
```

---

## 4. Security Middleware

```python
# security_middleware.py

from fastapi import FastAPI
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """เพิ่ม security headers"""
    
    async def dispatch(self, request: Request, call_next) -> Response:
        response = await call_next(request)
        
        # Security headers
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; "
            "script-src 'self' 'unsafe-inline'; "
            "style-src 'self' 'unsafe-inline'"
        )
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()"
        
        return response


class RequestIDMiddleware(BaseHTTPMiddleware):
    """เพิ่ม unique Request ID"""
    
    async def dispatch(self, request: Request, call_next) -> Response:
        import uuid
        
        # ดึงหรือสร้าง request ID
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        
        return response


app = FastAPI()
app.add_middleware(SecurityHeadersMiddleware)
app.add_middleware(RequestIDMiddleware)
```

---

## 5. Authentication Middleware

```python
# auth_middleware.py

from fastapi import FastAPI, Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse
import jwt

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"

# Paths ที่ไม่ต้องการ authentication
PUBLIC_PATHS = {
    "/",
    "/docs",
    "/redoc",
    "/openapi.json",
    "/auth/login",
    "/auth/register",
    "/health"
}


class JWTAuthMiddleware(BaseHTTPMiddleware):
    """Middleware ตรวจสอบ JWT token"""
    
    async def dispatch(self, request: Request, call_next) -> Response:
        # Skip authentication สำหรับ public paths
        if request.url.path in PUBLIC_PATHS:
            return await call_next(request)
        
        # Skip OPTIONS requests (CORS preflight)
        if request.method == "OPTIONS":
            return await call_next(request)
        
        # ตรวจสอบ Authorization header
        auth_header = request.headers.get("Authorization")
        
        if not auth_header or not auth_header.startswith("Bearer "):
            return JSONResponse(
                {"error": "Missing or invalid Authorization header"},
                status_code=401
            )
        
        token = auth_header.split(" ")[1]
        
        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
            # เก็บ user info ไว้ใน request state
            request.state.user = payload
        except jwt.ExpiredSignatureError:
            return JSONResponse({"error": "Token expired"}, status_code=401)
        except jwt.InvalidTokenError:
            return JSONResponse({"error": "Invalid token"}, status_code=401)
        
        return await call_next(request)


app = FastAPI()
app.add_middleware(JWTAuthMiddleware)
```

---

## 6. Complete Middleware Stack

```python
# complete_app.py

import os
import time
import uuid
import logging
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Complete Middleware Demo")

# 1. CORS (ต้องเป็นตัวแรกหรือตัวต้นๆ)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"]
)


class FullLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        request_id = str(uuid.uuid4())[:8]
        start = time.time()
        
        logger.info(f"[{request_id}] → {request.method} {request.url.path}")
        
        response = await call_next(request)
        
        duration = time.time() - start
        logger.info(f"[{request_id}] ← {response.status_code} ({duration:.3f}s)")
        
        response.headers["X-Request-ID"] = request_id
        response.headers["X-Response-Time"] = f"{duration:.3f}s"
        
        return response


# 2. Logging
app.add_middleware(FullLoggingMiddleware)


@app.get("/")
async def root():
    return {"message": "Middleware stack demo"}


@app.get("/health")
async def health():
    return {"status": "ok"}


if __name__ == "__main__":
    import uvicorn
    uvicorn.run("complete_app:app", host="0.0.0.0", port=8000, reload=True)
```

---

## 7. สรุป Part 093

✅ **Middleware** ทำงานระหว่าง request และ response ทุก request  
✅ **@app.middleware("http")** วิธีง่ายสุดในการสร้าง middleware  
✅ **BaseHTTPMiddleware** class-based approach ที่ reusable  
✅ **CORSMiddleware** จัดการ Cross-Origin Resource Sharing  
✅ **Security headers** เพิ่มความปลอดภัยให้ API  
✅ **Request ID** ใช้ track requests ใน logs  
✅ **Rate limiting** ป้องกัน abuse  

---

## ➡️ ถัดไป: Part 094 - FastAPI Testing

*Part 093/100+ | Python Course - Beginner to World-Class*
