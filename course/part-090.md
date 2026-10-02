# Part 090: FastAPI Advanced Features
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- สร้าง custom exception handlers
- เขียน middleware สำหรับ timing, logging, request ID
- ใช้ background tasks สำหรับงานที่ใช้เวลานาน
- จัดการ lifespan events (startup/shutdown)
- ใช้ custom response classes (HTML, File, Streaming)
- สร้าง Server-Sent Events (SSE)
- Integrate GraphQL ด้วย strawberry
- Customize OpenAPI documentation

---

## 1. Custom Exception Handlers

Exception handlers ช่วยให้จัดการ error อย่างสม่ำเสมอทั่วทั้งแอปพลิเคชัน

```python
# exception_handlers.py

from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import ValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException
from typing import Any
import logging
import traceback

# ตั้งค่า logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI()


# === Custom Exceptions ===

class AppException(Exception):
    """Base exception สำหรับแอปพลิเคชัน"""
    def __init__(self, message: str, code: str = "APP_ERROR", status_code: int = 400):
        self.message = message
        self.code = code
        self.status_code = status_code
        super().__init__(message)


class ResourceNotFoundException(AppException):
    """Exception สำหรับ resource ที่ไม่พบ"""
    def __init__(self, resource: str, resource_id: Any):
        super().__init__(
            message=f"ไม่พบ {resource} รหัส {resource_id}",
            code="RESOURCE_NOT_FOUND",
            status_code=404
        )
        self.resource = resource
        self.resource_id = resource_id


class InsufficientStockException(AppException):
    """Exception สำหรับสต็อกสินค้าไม่พอ"""
    def __init__(self, product_name: str, requested: int, available: int):
        super().__init__(
            message=f"สินค้า '{product_name}' มีสต็อกไม่พอ (ต้องการ {requested}, มี {available})",
            code="INSUFFICIENT_STOCK",
            status_code=409
        )
        self.product_name = product_name
        self.requested = requested
        self.available = available


class BusinessRuleException(AppException):
    """Exception สำหรับ business logic errors"""
    def __init__(self, message: str, rule: str):
        super().__init__(message=message, code="BUSINESS_RULE_VIOLATION", status_code=422)
        self.rule = rule


# === Exception Handlers ===

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    """จัดการ AppException และ subclasses ทั้งหมด"""
    logger.warning(f"App error on {request.url}: {exc.message}")

    return JSONResponse(
        status_code=exc.status_code,
        content={
            "success": False,
            "error": {
                "code": exc.code,
                "message": exc.message
            }
        }
    )


@app.exception_handler(ResourceNotFoundException)
async def resource_not_found_handler(request: Request, exc: ResourceNotFoundException):
    """จัดการ resource not found"""
    return JSONResponse(
        status_code=404,
        content={
            "success": False,
            "error": {
                "code": exc.code,
                "message": exc.message,
                "resource": exc.resource,
                "id": str(exc.resource_id)
            }
        }
    )


@app.exception_handler(InsufficientStockException)
async def insufficient_stock_handler(request: Request, exc: InsufficientStockException):
    """จัดการ insufficient stock"""
    return JSONResponse(
        status_code=409,
        content={
            "success": False,
            "error": {
                "code": exc.code,
                "message": exc.message,
                "product": exc.product_name,
                "requested": exc.requested,
                "available": exc.available
            }
        }
    )


@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    """จัดการ Pydantic validation errors ให้สวยงาม"""
    errors = []
    for error in exc.errors():
        field = " -> ".join(str(x) for x in error["loc"])
        errors.append({
            "field": field,
            "message": error["msg"],
            "type": error["type"]
        })

    return JSONResponse(
        status_code=422,
        content={
            "success": False,
            "error": {
                "code": "VALIDATION_ERROR",
                "message": "ข้อมูลที่ส่งมาไม่ถูกต้อง",
                "details": errors
            }
        }
    )


@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    """จัดการ HTTP exceptions"""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "success": False,
            "error": {
                "code": f"HTTP_{exc.status_code}",
                "message": exc.detail
            }
        },
        headers=exc.headers
    )


@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    """จัดการ exceptions ที่ไม่คาดคิด"""
    logger.error(
        f"Unexpected error on {request.url}: {exc}\n{traceback.format_exc()}"
    )

    return JSONResponse(
        status_code=500,
        content={
            "success": False,
            "error": {
                "code": "INTERNAL_SERVER_ERROR",
                "message": "เกิดข้อผิดพลาดภายในเซิร์ฟเวอร์"
                # ไม่บอกรายละเอียดให้ client ทราบ (security)
            }
        }
    )


# === Mock Data ===

products = {
    1: {"id": 1, "name": "มะม่วง", "price": 50, "stock": 10},
    2: {"id": 2, "name": "ทุเรียน", "price": 300, "stock": 5},
}


# === Endpoints ===

@app.get("/products/{product_id}")
def get_product(product_id: int):
    """ดึงสินค้า - raise custom exception ถ้าไม่พบ"""
    if product_id not in products:
        raise ResourceNotFoundException("สินค้า", product_id)
    return {"success": True, "data": products[product_id]}


@app.post("/orders/")
def create_order(product_id: int, quantity: int):
    """สร้าง order - raise custom exceptions"""
    # ตรวจสอบว่ามีสินค้า
    if product_id not in products:
        raise ResourceNotFoundException("สินค้า", product_id)

    product = products[product_id]

    # ตรวจสอบ business rules
    if quantity <= 0:
        raise BusinessRuleException(
            "จำนวนต้องมากกว่า 0",
            rule="POSITIVE_QUANTITY"
        )

    if quantity > 100:
        raise BusinessRuleException(
            "ไม่สามารถสั่งเกิน 100 ชิ้นต่อครั้ง",
            rule="MAX_ORDER_QUANTITY"
        )

    # ตรวจสอบสต็อก
    if quantity > product["stock"]:
        raise InsufficientStockException(
            product["name"],
            quantity,
            product["stock"]
        )

    # ลดสต็อก
    product["stock"] -= quantity

    return {
        "success": True,
        "data": {
            "product_name": product["name"],
            "quantity": quantity,
            "total": product["price"] * quantity
        }
    }
```

---

## 2. Middleware

Middleware ทำงานทุก request/response ก่อนถึง endpoint

```python
# middleware.py

from fastapi import FastAPI, Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.middleware.cors import CORSMiddleware
from typing import Callable
import time
import uuid
import logging
import json

app = FastAPI()

# ตั้งค่า logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


# === Timing Middleware ===

class TimingMiddleware(BaseHTTPMiddleware):
    """วัดเวลาในการประมวลผลแต่ละ request"""

    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        start_time = time.time()

        # ส่ง request ต่อไป
        response = await call_next(request)

        # คำนวณเวลา
        process_time = time.time() - start_time
        process_time_ms = round(process_time * 1000, 2)

        # เพิ่ม header บอกเวลา
        response.headers["X-Process-Time"] = f"{process_time_ms}ms"

        logger.info(f"{request.method} {request.url.path} - {process_time_ms}ms")

        return response


# === Request ID Middleware ===

class RequestIDMiddleware(BaseHTTPMiddleware):
    """เพิ่ม unique ID ให้แต่ละ request"""

    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        # ดึง request ID จาก header หรือสร้างใหม่
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))

        # เพิ่ม request_id ให้ request state (ให้ endpoint เข้าถึงได้)
        request.state.request_id = request_id

        response = await call_next(request)

        # เพิ่ม request ID กลับไปใน response header
        response.headers["X-Request-ID"] = request_id
        return response


# === Logging Middleware ===

class LoggingMiddleware(BaseHTTPMiddleware):
    """Log รายละเอียดของ request และ response"""

    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        # Log request info
        request_body = None
        if request.method in ["POST", "PUT", "PATCH"]:
            try:
                body = await request.body()
                if body:
                    request_body = body.decode("utf-8")[:500]  # จำกัด 500 ตัวอักษร
            except Exception:
                pass

        logger.info(
            f"REQUEST | {request.method} {request.url} | "
            f"Client: {request.client.host if request.client else 'unknown'} | "
            f"Body: {request_body[:100] if request_body else 'none'}"
        )

        # ประมวลผล request
        start_time = time.time()
        response = await call_next(request)
        duration = round((time.time() - start_time) * 1000, 2)

        # Log response info
        logger.info(
            f"RESPONSE | {request.method} {request.url.path} | "
            f"Status: {response.status_code} | "
            f"Duration: {duration}ms"
        )

        return response


# === Security Headers Middleware ===

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """เพิ่ม security headers ทุก response"""

    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        response = await call_next(request)

        # เพิ่ม security headers
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

        return response


# === Rate Limiting Middleware (Simple) ===

from collections import defaultdict
from datetime import datetime

class SimpleRateLimitMiddleware(BaseHTTPMiddleware):
    """Rate limiting แบบง่าย (ในระบบจริงใช้ Redis)"""

    def __init__(self, app, max_requests: int = 100, window_seconds: int = 60):
        super().__init__(app)
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.request_counts = defaultdict(list)  # {ip: [timestamps]}

    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        client_ip = request.client.host if request.client else "unknown"
        now = time.time()

        # ลบ requests ที่เก่าเกินไป
        self.request_counts[client_ip] = [
            t for t in self.request_counts[client_ip]
            if now - t < self.window_seconds
        ]

        # ตรวจสอบ rate limit
        if len(self.request_counts[client_ip]) >= self.max_requests:
            from fastapi.responses import JSONResponse
            return JSONResponse(
                status_code=429,
                content={"error": "Too Many Requests", "retry_after": self.window_seconds}
            )

        # บันทึก request
        self.request_counts[client_ip].append(now)

        return await call_next(request)


# === ลงทะเบียน Middleware (ลำดับสำคัญ - ทำงานย้อนกลับ) ===

app.add_middleware(TimingMiddleware)
app.add_middleware(RequestIDMiddleware)
app.add_middleware(LoggingMiddleware)
app.add_middleware(SecurityHeadersMiddleware)
app.add_middleware(SimpleRateLimitMiddleware, max_requests=100, window_seconds=60)

# CORS Middleware จาก Starlette
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "https://myapp.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
    allow_headers=["*"],
    expose_headers=["X-Request-ID", "X-Process-Time"]
)


# === ตัวอย่าง Endpoints ===

@app.get("/")
def root(request: Request):
    """ดู request ID จาก middleware"""
    return {
        "message": "Hello!",
        "request_id": request.state.request_id,
        "client_ip": request.client.host if request.client else "unknown"
    }


@app.get("/slow-endpoint/")
async def slow_endpoint():
    """Endpoint ที่ใช้เวลานาน - timing middleware จะแสดงเวลา"""
    import asyncio
    await asyncio.sleep(0.5)  # จำลองงานหนัก
    return {"message": "เสร็จแล้ว (ใช้เวลา ~500ms)"}
```

---

## 3. Background Tasks

สำหรับงานที่ใช้เวลานานและไม่ต้องรอผล

```python
# background_tasks.py

from fastapi import FastAPI, BackgroundTasks, Depends
from pydantic import BaseModel
from typing import Optional
import asyncio
import logging
import time
from datetime import datetime

app = FastAPI()
logger = logging.getLogger(__name__)


# === Functions สำหรับ Background Tasks ===

def send_email_notification(email: str, subject: str, body: str):
    """ส่งอีเมล (จำลอง - ใช้เวลานาน)"""
    logger.info(f"กำลังส่งอีเมลไปยัง {email}...")
    time.sleep(2)  # จำลองการส่งอีเมล
    logger.info(f"ส่งอีเมลไปยัง {email} สำเร็จ: {subject}")


async def send_push_notification(user_id: str, message: str):
    """ส่ง push notification (async)"""
    logger.info(f"กำลังส่ง push notification ไปยัง user {user_id}...")
    await asyncio.sleep(1)  # จำลอง async operation
    logger.info(f"ส่ง push notification ไปยัง user {user_id} สำเร็จ")


def write_audit_log(
    action: str,
    user_id: str,
    resource: str,
    details: dict
):
    """บันทึก audit log"""
    log_entry = {
        "timestamp": datetime.utcnow().isoformat(),
        "action": action,
        "user_id": user_id,
        "resource": resource,
        "details": details
    }
    logger.info(f"AUDIT LOG: {log_entry}")
    # ในระบบจริงเขียนลง database หรือ log file


def process_large_file(file_path: str, processing_type: str):
    """ประมวลผลไฟล์ขนาดใหญ่ (ใช้เวลานาน)"""
    logger.info(f"เริ่มประมวลผลไฟล์ {file_path}...")
    time.sleep(5)  # จำลองการประมวลผล
    logger.info(f"ประมวลผลไฟล์ {file_path} เสร็จสิ้น")


def generate_report(report_type: str, filters: dict):
    """สร้าง report (ใช้เวลานาน)"""
    logger.info(f"กำลังสร้าง {report_type} report...")
    time.sleep(3)
    logger.info(f"สร้าง {report_type} report เสร็จสิ้น")


# === Schemas ===

class OrderRequest(BaseModel):
    product_id: int
    quantity: int
    customer_email: str
    customer_name: str


class UserCreate(BaseModel):
    username: str
    email: str
    full_name: str


# === Endpoints ที่ใช้ Background Tasks ===

@app.post("/orders/", status_code=201)
def create_order(
    order: OrderRequest,
    background_tasks: BackgroundTasks
):
    """
    สร้าง order และส่ง email notification ใน background
    Client ได้รับ response ทันทีโดยไม่ต้องรอ email ส่ง
    """
    order_id = f"ORD-{int(time.time())}"

    # เพิ่ม background tasks (จะทำงานหลัง response ส่งออกไปแล้ว)
    background_tasks.add_task(
        send_email_notification,
        email=order.customer_email,
        subject=f"ยืนยัน Order #{order_id}",
        body=f"สวัสดี {order.customer_name}, ได้รับ order ของคุณแล้ว"
    )

    background_tasks.add_task(
        write_audit_log,
        action="CREATE_ORDER",
        user_id=order.customer_email,
        resource="order",
        details={"order_id": order_id, "product_id": order.product_id}
    )

    # ส่ง response ทันที (background tasks ยังไม่ได้ทำงาน)
    return {
        "order_id": order_id,
        "status": "confirmed",
        "message": "สร้าง order สำเร็จ กำลังส่งอีเมลยืนยัน..."
    }


@app.post("/users/", status_code=201)
async def create_user(
    user: UserCreate,
    background_tasks: BackgroundTasks
):
    """สร้าง user ใหม่พร้อม background tasks หลายอย่าง"""
    user_id = f"USR-{int(time.time())}"

    # ส่ง welcome email
    background_tasks.add_task(
        send_email_notification,
        email=user.email,
        subject="ยินดีต้อนรับสู่ระบบ!",
        body=f"สวัสดี {user.full_name}, ขอบคุณที่ลงทะเบียน"
    )

    # ส่ง push notification (async task ก็ได้)
    background_tasks.add_task(
        send_push_notification,
        user_id=user_id,
        message="ลงทะเบียนสำเร็จ! ยินดีต้อนรับ"
    )

    # บันทึก audit log
    background_tasks.add_task(
        write_audit_log,
        action="USER_REGISTERED",
        user_id=user_id,
        resource="user",
        details={"username": user.username, "email": user.email}
    )

    return {
        "user_id": user_id,
        "username": user.username,
        "message": "สร้าง user สำเร็จ"
    }


@app.post("/reports/generate/")
def generate_report_endpoint(
    report_type: str,
    background_tasks: BackgroundTasks,
    filters: dict = {}
):
    """สร้าง report ใน background"""
    report_id = f"RPT-{int(time.time())}"

    background_tasks.add_task(
        generate_report,
        report_type=report_type,
        filters=filters
    )

    return {
        "report_id": report_id,
        "status": "processing",
        "message": f"กำลังสร้าง {report_type} report, จะแจ้งเมื่อเสร็จ"
    }
```

---

## 4. Lifespan Events (Startup/Shutdown)

```python
# lifespan.py

from fastapi import FastAPI
from contextlib import asynccontextmanager
import asyncio
import logging
from typing import AsyncGenerator

logger = logging.getLogger(__name__)


# === Resources ที่ต้องจัดการ ===

class DatabasePool:
    """จำลอง database connection pool"""

    def __init__(self, url: str, max_connections: int = 10):
        self.url = url
        self.max_connections = max_connections
        self.connections = []
        self.is_connected = False

    async def connect(self):
        """สร้าง connection pool"""
        logger.info(f"กำลังเชื่อมต่อ database: {self.url}")
        await asyncio.sleep(0.5)  # จำลองการเชื่อมต่อ
        self.is_connected = True
        logger.info(f"เชื่อมต่อ database สำเร็จ (max: {self.max_connections} connections)")

    async def disconnect(self):
        """ปิด connection pool"""
        logger.info("กำลังปิด database connections...")
        await asyncio.sleep(0.1)
        self.is_connected = False
        logger.info("ปิด database connections สำเร็จ")


class CacheClient:
    """จำลอง Redis cache client"""

    def __init__(self, host: str = "localhost", port: int = 6379):
        self.host = host
        self.port = port
        self.is_connected = False
        self._cache = {}  # Mock cache

    async def connect(self):
        logger.info(f"กำลังเชื่อมต่อ Redis: {self.host}:{self.port}")
        await asyncio.sleep(0.2)
        self.is_connected = True
        logger.info("เชื่อมต่อ Redis สำเร็จ")

    async def disconnect(self):
        logger.info("กำลังปิด Redis connection...")
        self.is_connected = False
        logger.info("ปิด Redis connection สำเร็จ")

    async def get(self, key: str):
        return self._cache.get(key)

    async def set(self, key: str, value, ttl: int = 300):
        self._cache[key] = value


# === Global instances ===
db_pool = DatabasePool("postgresql://localhost/mydb")
cache = CacheClient()


# === Lifespan Context Manager ===

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator:
    """
    จัดการ startup และ shutdown ด้วย context manager
    
    โค้ดก่อน yield = startup
    โค้ดหลัง yield = shutdown
    """
    # ===== STARTUP =====
    logger.info("🚀 Starting application...")

    # เชื่อมต่อ database
    try:
        await db_pool.connect()
    except Exception as e:
        logger.error(f"ไม่สามารถเชื่อมต่อ database: {e}")
        raise

    # เชื่อมต่อ Redis
    try:
        await cache.connect()
    except Exception as e:
        logger.warning(f"ไม่สามารถเชื่อมต่อ Redis: {e} (จะทำงานโดยไม่มี cache)")

    # โหลด config หรือ data เริ่มต้น
    logger.info("โหลด configuration สำเร็จ")

    logger.info("✅ Application started successfully!")

    yield  # แอปทำงานที่นี่

    # ===== SHUTDOWN =====
    logger.info("🛑 Shutting down application...")

    # ปิด connections ตามลำดับ
    await cache.disconnect()
    await db_pool.disconnect()

    # รอ background tasks ที่ค้างอยู่
    await asyncio.sleep(0.1)

    logger.info("✅ Application shutdown complete")


# สร้าง app ด้วย lifespan
app = FastAPI(
    title="My App",
    lifespan=lifespan
)


# === Dependency สำหรับเข้าถึง resources ===

def get_db():
    """ดึง database pool"""
    if not db_pool.is_connected:
        from fastapi import HTTPException
        raise HTTPException(status_code=503, detail="Database ไม่พร้อมใช้งาน")
    return db_pool


def get_cache():
    """ดึง cache client"""
    return cache  # อาจจะ not connected แต่ไม่ error


# === Endpoints ===

@app.get("/health/")
async def health_check():
    """ตรวจสอบสถานะของระบบ"""
    return {
        "status": "healthy",
        "database": "connected" if db_pool.is_connected else "disconnected",
        "cache": "connected" if cache.is_connected else "disconnected"
    }


@app.get("/data/")
async def get_data():
    """ดึงข้อมูล (ใช้ database และ cache)"""
    # ลองดึงจาก cache ก่อน
    cached = await cache.get("data_key")
    if cached:
        return {"source": "cache", "data": cached}

    # ถ้าไม่มีใน cache ดึงจาก database
    if not db_pool.is_connected:
        from fastapi import HTTPException
        raise HTTPException(status_code=503, detail="ไม่สามารถเชื่อมต่อ database")

    data = {"items": ["a", "b", "c"], "from_db": True}

    # เก็บใน cache
    await cache.set("data_key", data, ttl=60)

    return {"source": "database", "data": data}
```

---

## 5. Custom Response Classes

```python
# custom_responses.py

from fastapi import FastAPI
from fastapi.responses import (
    HTMLResponse,
    FileResponse,
    StreamingResponse,
    JSONResponse,
    RedirectResponse,
    PlainTextResponse
)
import asyncio
from pathlib import Path
import io
import csv

app = FastAPI()


# === HTML Response ===

@app.get("/page/", response_class=HTMLResponse)
def get_html_page():
    """ส่ง HTML กลับไป"""
    html_content = """
    <!DOCTYPE html>
    <html lang="th">
    <head>
        <meta charset="UTF-8">
        <title>FastAPI HTML</title>
        <style>
            body { font-family: sans-serif; max-width: 800px; margin: 50px auto; }
            h1 { color: #009688; }
        </style>
    </head>
    <body>
        <h1>สวัสดีจาก FastAPI!</h1>
        <p>นี่คือ HTML response จาก FastAPI</p>
        <ul>
            <li>รองรับภาษาไทย</li>
            <li>ส่งเป็น HTML ได้โดยตรง</li>
        </ul>
    </body>
    </html>
    """
    return HTMLResponse(content=html_content, status_code=200)


# === File Response ===

@app.get("/download/sample-pdf/")
def download_pdf():
    """ดาวน์โหลดไฟล์ PDF"""
    file_path = Path("sample.pdf")

    # ตรวจสอบว่าไฟล์มีอยู่
    if not file_path.exists():
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail="ไม่พบไฟล์")

    return FileResponse(
        path=file_path,
        media_type="application/pdf",
        filename="รายงาน.pdf",  # ชื่อไฟล์ที่ user เห็น
        headers={"Content-Disposition": "attachment; filename=report.pdf"}
    )


@app.get("/download/image/{filename}")
def download_image(filename: str):
    """ดาวน์โหลดรูปภาพ"""
    file_path = Path(f"uploads/{filename}")

    if not file_path.exists():
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail="ไม่พบไฟล์")

    # กำหนด media type ตามนามสกุล
    suffix = file_path.suffix.lower()
    media_types = {
        ".jpg": "image/jpeg",
        ".jpeg": "image/jpeg",
        ".png": "image/png",
        ".gif": "image/gif",
        ".webp": "image/webp"
    }
    media_type = media_types.get(suffix, "application/octet-stream")

    return FileResponse(path=file_path, media_type=media_type)


# === Streaming Response ===

async def generate_large_data():
    """Generator สำหรับข้อมูลขนาดใหญ่"""
    for i in range(1000):
        yield f"row_{i}: ข้อมูลบรรทัดที่ {i}\n"
        await asyncio.sleep(0.001)  # ไม่ block event loop


@app.get("/stream/data/")
def stream_large_data():
    """Stream ข้อมูลขนาดใหญ่ทีละส่วน"""
    return StreamingResponse(
        generate_large_data(),
        media_type="text/plain; charset=utf-8",
        headers={"Content-Disposition": "attachment; filename=data.txt"}
    )


# === CSV Streaming ===

def generate_csv():
    """สร้าง CSV data แบบ streaming"""
    output = io.StringIO()
    writer = csv.writer(output)

    # Header
    writer.writerow(["ID", "ชื่อ", "ราคา", "สต็อก"])
    output.seek(0)
    yield output.read()
    output.seek(0)
    output.truncate()

    # Data rows
    products = [
        (1, "มะม่วง", 50, 100),
        (2, "ทุเรียน", 300, 50),
        (3, "มังคุด", 150, 75),
        (4, "ลำไย", 80, 200),
        (5, "เงาะ", 60, 150),
    ]

    for row in products:
        writer.writerow(row)
        output.seek(0)
        yield output.read()
        output.seek(0)
        output.truncate()


@app.get("/export/products.csv")
def export_csv():
    """Export สินค้าเป็น CSV"""
    return StreamingResponse(
        generate_csv(),
        media_type="text/csv; charset=utf-8-sig",  # utf-8-sig สำหรับ Excel
        headers={"Content-Disposition": "attachment; filename=products.csv"}
    )


# === Image Generation Streaming ===

def generate_image_bytes():
    """สร้าง image แบบ streaming (ตัวอย่างด้วย PNG bytes)"""
    try:
        from PIL import Image, ImageDraw, ImageFont
        import io

        # สร้างรูปภาพ
        img = Image.new('RGB', (400, 200), color=(0, 150, 136))
        draw = ImageDraw.Draw(img)
        draw.text((50, 80), "FastAPI", fill=(255, 255, 255))

        # แปลงเป็น bytes
        img_bytes = io.BytesIO()
        img.save(img_bytes, format='PNG')
        img_bytes.seek(0)
        yield img_bytes.read()

    except ImportError:
        # ถ้าไม่มี Pillow ส่ง placeholder
        yield b'\x89PNG\r\n'


@app.get("/generate/image/")
def generate_image():
    """สร้างและ stream รูปภาพ"""
    return StreamingResponse(
        generate_image_bytes(),
        media_type="image/png"
    )


# === Redirect Response ===

@app.get("/old-path/")
def old_endpoint():
    """Redirect ไปยัง URL ใหม่"""
    return RedirectResponse(url="/new-path/", status_code=301)  # Permanent redirect


@app.get("/new-path/")
def new_endpoint():
    return {"message": "นี่คือ URL ใหม่"}


# === Plain Text Response ===

@app.get("/robots.txt")
def robots_txt():
    """ส่ง robots.txt"""
    content = """User-agent: *
Disallow: /admin/
Disallow: /api/private/
Allow: /
"""
    return PlainTextResponse(content=content)
```

---

## 6. Server-Sent Events (SSE)

SSE ช่วยให้ server ส่งข้อมูลให้ client แบบ real-time (one-way)

```python
# sse.py

from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse, HTMLResponse
import asyncio
import json
import time
from datetime import datetime
from typing import AsyncGenerator

app = FastAPI()


# === SSE Helper ===

def format_sse(data: dict, event: str = None, id: str = None) -> str:
    """
    Format ข้อมูลเป็น SSE format
    
    SSE format:
    event: <event_name>  (optional)
    id: <event_id>       (optional)
    data: <json_data>
    
    (บรรทัดว่างเพื่อ terminate event)
    """
    lines = []
    if event:
        lines.append(f"event: {event}")
    if id:
        lines.append(f"id: {id}")
    lines.append(f"data: {json.dumps(data, ensure_ascii=False)}")
    lines.append("")  # บรรทัดว่าง = สิ้นสุด event
    return "\n".join(lines) + "\n"


# === SSE Generators ===

async def event_stream_counter() -> AsyncGenerator[str, None]:
    """ส่ง counter ทุกวินาที"""
    count = 0
    while True:
        count += 1
        data = {
            "count": count,
            "timestamp": datetime.utcnow().isoformat(),
            "message": f"นับ {count}"
        }
        yield format_sse(data, event="counter")

        if count >= 10:
            # ส่ง event สิ้นสุด
            yield format_sse({"message": "นับเสร็จแล้ว!"}, event="done")
            break

        await asyncio.sleep(1)


async def real_time_stock_prices() -> AsyncGenerator[str, None]:
    """จำลองราคาหุ้น real-time"""
    import random

    stocks = {"AAPL": 150.0, "GOOGL": 2800.0, "MSFT": 300.0}

    event_id = 0
    while True:
        event_id += 1

        # สุ่มการเปลี่ยนแปลงราคา
        for symbol in stocks:
            change = random.uniform(-5, 5)
            stocks[symbol] = max(1, stocks[symbol] + change)

        data = {
            "stocks": {
                symbol: {
                    "price": round(price, 2),
                    "currency": "USD"
                }
                for symbol, price in stocks.items()
            },
            "timestamp": datetime.utcnow().isoformat()
        }

        yield format_sse(data, event="stock_update", id=str(event_id))
        await asyncio.sleep(2)


async def system_metrics_stream() -> AsyncGenerator[str, None]:
    """ส่ง system metrics แบบ real-time"""
    import random

    for i in range(60):  # ส่ง 60 ครั้ง แล้วหยุด
        metrics = {
            "cpu_usage": random.uniform(10, 90),
            "memory_usage": random.uniform(30, 80),
            "requests_per_second": random.randint(10, 1000),
            "timestamp": datetime.utcnow().isoformat()
        }
        yield format_sse(metrics, event="metrics")
        await asyncio.sleep(1)

    # แจ้งว่าสิ้นสุด stream
    yield format_sse({"message": "Stream หยุดแล้ว"}, event="end")


async def notification_stream(user_id: str) -> AsyncGenerator[str, None]:
    """ส่ง notifications สำหรับ user เฉพาะ"""
    import random

    notifications = [
        {"type": "order", "message": "Order ของคุณถูกจัดส่งแล้ว"},
        {"type": "message", "message": "คุณมีข้อความใหม่"},
        {"type": "promotion", "message": "ลดราคา 50% วันนี้เท่านั้น!"},
        {"type": "system", "message": "ระบบจะ maintenance คืนนี้"},
    ]

    # ส่ง connected event
    yield format_sse(
        {"message": f"เชื่อมต่อสำเร็จ สำหรับ user {user_id}"},
        event="connected"
    )

    for notification in notifications:
        await asyncio.sleep(random.uniform(1, 3))  # สุ่มเวลา
        yield format_sse(notification, event="notification")

    yield format_sse({"message": "ไม่มีการแจ้งเตือนเพิ่มเติม"}, event="end")


# === SSE Endpoints ===

@app.get("/sse/counter/")
async def sse_counter(request: Request):
    """SSE endpoint สำหรับ counter"""
    async def stream():
        async for event in event_stream_counter():
            # ตรวจสอบว่า client ยังเชื่อมต่ออยู่
            if await request.is_disconnected():
                break
            yield event

    return StreamingResponse(
        stream(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no"  # สำหรับ Nginx
        }
    )


@app.get("/sse/stocks/")
async def sse_stocks(request: Request):
    """SSE endpoint สำหรับราคาหุ้น"""
    async def stream():
        async for event in real_time_stock_prices():
            if await request.is_disconnected():
                break
            yield event

    return StreamingResponse(
        stream(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache"}
    )


@app.get("/sse/notifications/{user_id}")
async def sse_notifications(user_id: str, request: Request):
    """SSE endpoint สำหรับ notifications ของ user"""
    async def stream():
        async for event in notification_stream(user_id):
            if await request.is_disconnected():
                break
            yield event

    return StreamingResponse(
        stream(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache"}
    )


# === Demo HTML Page ===

@app.get("/demo/sse/", response_class=HTMLResponse)
def sse_demo_page():
    """หน้าทดสอบ SSE"""
    return HTMLResponse("""
    <!DOCTYPE html>
    <html lang="th">
    <head>
        <meta charset="UTF-8">
        <title>SSE Demo</title>
        <style>
            body { font-family: sans-serif; padding: 20px; }
            #events { border: 1px solid #ccc; padding: 10px; height: 300px; overflow-y: auto; }
            .event { margin: 5px 0; padding: 5px; background: #f0f0f0; }
        </style>
    </head>
    <body>
        <h1>Server-Sent Events Demo</h1>
        <button onclick="startSSE()">เริ่ม Counter</button>
        <button onclick="stopSSE()">หยุด</button>
        <div id="events"></div>

        <script>
            let eventSource = null;

            function startSSE() {
                if (eventSource) eventSource.close();

                eventSource = new EventSource('/sse/counter/');

                eventSource.addEventListener('counter', (e) => {
                    const data = JSON.parse(e.data);
                    addEvent(`Counter: ${data.count} - ${data.message}`);
                });

                eventSource.addEventListener('done', (e) => {
                    const data = JSON.parse(e.data);
                    addEvent(`✅ ${data.message}`);
                    eventSource.close();
                });

                eventSource.onerror = () => {
                    addEvent('❌ การเชื่อมต่อขัดข้อง');
                    eventSource.close();
                };
            }

            function stopSSE() {
                if (eventSource) {
                    eventSource.close();
                    addEvent('⏹ หยุดการรับข้อมูล');
                }
            }

            function addEvent(msg) {
                const div = document.getElementById('events');
                const item = document.createElement('div');
                item.className = 'event';
                item.textContent = `[${new Date().toLocaleTimeString('th-TH')}] ${msg}`;
                div.appendChild(item);
                div.scrollTop = div.scrollHeight;
            }
        </script>
    </body>
    </html>
    """)
```

---

## 7. GraphQL ด้วย Strawberry

ติดตั้ง:
```bash
pip install strawberry-graphql[fastapi]
```

```python
# graphql_app.py

from fastapi import FastAPI
import strawberry
from strawberry.fastapi import GraphQLRouter
from typing import Optional, List
from datetime import datetime

app = FastAPI()


# === GraphQL Types ===

@strawberry.type
class Product:
    """GraphQL type สำหรับสินค้า"""
    id: int
    name: str
    price: float
    stock: int
    category: str
    created_at: datetime


@strawberry.type
class User:
    """GraphQL type สำหรับผู้ใช้"""
    id: int
    username: str
    email: str
    full_name: str


@strawberry.type
class Order:
    """GraphQL type สำหรับ order"""
    id: str
    user: User
    product: Product
    quantity: int
    total_price: float
    status: str


# === Mock Data ===

mock_products = [
    Product(id=1, name="มะม่วง", price=50.0, stock=100, category="ผลไม้", created_at=datetime.utcnow()),
    Product(id=2, name="ทุเรียน", price=300.0, stock=50, category="ผลไม้", created_at=datetime.utcnow()),
    Product(id=3, name="กล้วย", price=30.0, stock=200, category="ผลไม้", created_at=datetime.utcnow()),
]

mock_users = [
    User(id=1, username="alice", email="alice@example.com", full_name="Alice Smith"),
    User(id=2, username="bob", email="bob@example.com", full_name="Bob Jones"),
]


# === Input Types ===

@strawberry.input
class ProductInput:
    """Input type สำหรับสร้างสินค้า"""
    name: str
    price: float
    stock: int
    category: str


@strawberry.input
class OrderInput:
    """Input type สำหรับสร้าง order"""
    user_id: int
    product_id: int
    quantity: int


# === Queries ===

@strawberry.type
class Query:
    """GraphQL Queries"""

    @strawberry.field(description="ดึงสินค้าทั้งหมด")
    def products(
        self,
        category: Optional[str] = None,
        min_price: Optional[float] = None,
        max_price: Optional[float] = None
    ) -> List[Product]:
        """ดึงรายการสินค้าพร้อม filtering"""
        result = mock_products.copy()

        if category:
            result = [p for p in result if p.category == category]
        if min_price is not None:
            result = [p for p in result if p.price >= min_price]
        if max_price is not None:
            result = [p for p in result if p.price <= max_price]

        return result

    @strawberry.field(description="ดึงสินค้าตาม ID")
    def product(self, id: int) -> Optional[Product]:
        """ดึงสินค้าตาม ID"""
        for p in mock_products:
            if p.id == id:
                return p
        return None

    @strawberry.field(description="ดึง users ทั้งหมด")
    def users(self) -> List[User]:
        """ดึงรายการ users"""
        return mock_users

    @strawberry.field(description="ดึง user ตาม ID")
    def user(self, id: int) -> Optional[User]:
        """ดึง user ตาม ID"""
        for u in mock_users:
            if u.id == id:
                return u
        return None


# === Mutations ===

@strawberry.type
class Mutation:
    """GraphQL Mutations"""

    @strawberry.mutation(description="สร้างสินค้าใหม่")
    def create_product(self, input: ProductInput) -> Product:
        """สร้างสินค้าใหม่"""
        new_id = max(p.id for p in mock_products) + 1 if mock_products else 1
        new_product = Product(
            id=new_id,
            name=input.name,
            price=input.price,
            stock=input.stock,
            category=input.category,
            created_at=datetime.utcnow()
        )
        mock_products.append(new_product)
        return new_product

    @strawberry.mutation(description="อัปเดตราคาสินค้า")
    def update_product_price(self, product_id: int, new_price: float) -> Optional[Product]:
        """อัปเดตราคาสินค้า"""
        for product in mock_products:
            if product.id == product_id:
                # Strawberry types เป็น frozen dataclass ต้อง recreate
                idx = mock_products.index(product)
                updated = Product(
                    id=product.id,
                    name=product.name,
                    price=new_price,
                    stock=product.stock,
                    category=product.category,
                    created_at=product.created_at
                )
                mock_products[idx] = updated
                return updated
        return None


# === Schema และ Router ===

schema = strawberry.Schema(query=Query, mutation=Mutation)
graphql_app = GraphQLRouter(schema)

# เพิ่ม GraphQL route
app.include_router(graphql_app, prefix="/graphql")


# REST endpoints ยังคงทำงานได้ปกติ
@app.get("/")
def root():
    return {
        "message": "FastAPI + GraphQL",
        "graphql_url": "/graphql",
        "graphiql_url": "/graphql"  # GraphiQL UI
    }
```

ตัวอย่าง GraphQL queries:
```graphql
# ดึงสินค้าทั้งหมด
query {
  products {
    id
    name
    price
    stock
  }
}

# ดึงเฉพาะที่ต้องการ
query {
  products(category: "ผลไม้", minPrice: 50) {
    id
    name
    price
  }
}

# สร้างสินค้าใหม่
mutation {
  createProduct(input: {
    name: "ลำไย"
    price: 80.0
    stock: 150
    category: "ผลไม้"
  }) {
    id
    name
    price
  }
}
```

---

## 8. OpenAPI Customization

```python
# openapi_custom.py

from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi
from fastapi.openapi.docs import get_swagger_ui_html, get_redoc_html
from fastapi.responses import HTMLResponse, JSONResponse
from pydantic import BaseModel
from typing import Optional

# กำหนด metadata ให้ app
app = FastAPI(
    title="ร้านขายผลไม้ Thai Fruits API",
    description="""
## API สำหรับร้านขายผลไม้ไทย 🍎🥭

### Features
- ดูรายการสินค้า
- จัดการ orders
- ระบบสมาชิก

### Authentication
ใช้ JWT Bearer token สำหรับ endpoints ที่ต้องการ authentication
""",
    version="2.0.0",
    contact={
        "name": "Support Team",
        "email": "support@thaifruits.example.com",
        "url": "https://thaifruits.example.com"
    },
    license_info={
        "name": "MIT License",
        "url": "https://opensource.org/licenses/MIT"
    },
    terms_of_service="https://thaifruits.example.com/terms/",
    openapi_tags=[
        {
            "name": "products",
            "description": "จัดการข้อมูลสินค้า"
        },
        {
            "name": "orders",
            "description": "จัดการ orders"
        },
        {
            "name": "users",
            "description": "จัดการผู้ใช้"
        }
    ]
)


# === Custom OpenAPI Schema ===

def custom_openapi():
    """Custom OpenAPI schema"""
    if app.openapi_schema:
        return app.openapi_schema

    openapi_schema = get_openapi(
        title=app.title,
        version=app.version,
        description=app.description,
        routes=app.routes,
    )

    # เพิ่ม security scheme
    openapi_schema["components"]["securitySchemes"] = {
        "BearerAuth": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT",
            "description": "ใส่ JWT token ในรูปแบบ: Bearer <token>"
        },
        "ApiKeyHeader": {
            "type": "apiKey",
            "in": "header",
            "name": "X-API-Key",
            "description": "API Key สำหรับ service-to-service"
        }
    }

    # เพิ่ม custom info
    openapi_schema["info"]["x-logo"] = {
        "url": "https://example.com/logo.png",
        "altText": "Thai Fruits Logo"
    }

    app.openapi_schema = openapi_schema
    return app.openapi_schema


app.openapi = custom_openapi


# === Custom Docs Pages ===

@app.get("/docs", include_in_schema=False)
async def custom_swagger_ui():
    """Custom Swagger UI"""
    return get_swagger_ui_html(
        openapi_url="/openapi.json",
        title="Thai Fruits API - Swagger",
        swagger_js_url="https://cdn.jsdelivr.net/npm/swagger-ui-dist@5/swagger-ui-bundle.js",
        swagger_css_url="https://cdn.jsdelivr.net/npm/swagger-ui-dist@5/swagger-ui.css",
        swagger_favicon_url="https://fastapi.tiangolo.com/img/favicon.png",
        init_oauth={
            "usePkceWithAuthorizationCodeGrant": True
        }
    )


@app.get("/redoc", include_in_schema=False)
async def custom_redoc():
    """Custom ReDoc UI"""
    return get_redoc_html(
        openapi_url="/openapi.json",
        title="Thai Fruits API - Docs",
        redoc_js_url="https://cdn.jsdelivr.net/npm/redoc@latest/bundles/redoc.standalone.js"
    )


# === Endpoints พร้อม OpenAPI metadata ===

class Product(BaseModel):
    id: int
    name: str
    price: float


@app.get(
    "/products/",
    tags=["products"],
    summary="ดูรายการสินค้าทั้งหมด",
    description="ดึงรายการสินค้าทั้งหมดพร้อม pagination และ filtering",
    response_description="รายการสินค้า",
    responses={
        200: {"description": "สำเร็จ"},
        500: {"description": "Server error"}
    }
)
def list_products():
    """ดูรายการสินค้า"""
    return [
        {"id": 1, "name": "มะม่วง", "price": 50},
        {"id": 2, "name": "ทุเรียน", "price": 300}
    ]


@app.post(
    "/products/",
    tags=["products"],
    summary="สร้างสินค้าใหม่",
    status_code=201,
    deprecated=False
)
def create_product(product: Product):
    """สร้างสินค้าใหม่"""
    return product


@app.get(
    "/old-products/",
    tags=["products"],
    deprecated=True,  # บอก client ว่า endpoint นี้ deprecated
    summary="[Deprecated] ดูรายการสินค้า (ใช้ /products/ แทน)"
)
def list_products_old():
    """Endpoint เก่า - กรุณาใช้ /products/ แทน"""
    return {"message": "กรุณาใช้ /products/ แทน"}
```

---

## 9. สรุป Part 090

✅ **Custom Exception Handlers** - จัดการ errors อย่างสม่ำเสมอ, custom exception classes, global handler

✅ **Middleware** - TimingMiddleware, RequestIDMiddleware, LoggingMiddleware, SecurityHeaders, Rate Limiting, CORS

✅ **Background Tasks** - ทำงานหลัง response ส่งออก, สำหรับ email, notifications, audit logs

✅ **Lifespan Events** - startup/shutdown ด้วย `@asynccontextmanager`, จัดการ DB pool และ cache

✅ **HTMLResponse** - ส่ง HTML ตรงๆ จาก FastAPI

✅ **FileResponse** - ดาวน์โหลดไฟล์พร้อมกำหนด filename

✅ **StreamingResponse** - stream ข้อมูลขนาดใหญ่ทีละส่วน, CSV export

✅ **SSE** - Server-Sent Events สำหรับ real-time data แบบ one-way

✅ **GraphQL** - Strawberry integration, Query และ Mutation types

✅ **OpenAPI** - custom schema, security schemes, docs pages, endpoint metadata

---

## ➡️ ถัดไป: Part 091 - FastAPI Testing

ใน Part ถัดไปเราจะเรียนรู้:
- Unit testing ด้วย pytest
- TestClient สำหรับ integration tests
- Mocking dependencies
- Coverage reports

*Part 090/100+ | Python Course - Beginner to World-Class*
