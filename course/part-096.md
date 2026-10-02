# Part 096 - FastAPI Advanced

## เป้าหมายการเรียนรู้

- ใช้ Background Tasks สำหรับงานที่ไม่ต้องรอผล
- สร้าง WebSocket server สำหรับ real-time communication
- เขียน Custom Middleware
- เขียน Tests ด้วย TestClient และ pytest-asyncio
- Deploy ด้วย Docker และ uvicorn
- ตัวอย่าง production-ready API

---

## 1. Background Tasks

```python
# app/routers/email.py - Background Tasks

from fastapi import APIRouter, BackgroundTasks, Depends
from pydantic import BaseModel, EmailStr
import asyncio
import logging

logger = logging.getLogger(__name__)
router = APIRouter()


class EmailRequest(BaseModel):
    to: EmailStr
    subject: str
    body: str


# ─────────────────────────────────────────
# Sync background task
# ─────────────────────────────────────────

def send_email_sync(to: str, subject: str, body: str):
    """ส่ง email แบบ sync (รันใน thread pool)"""
    # จำลองการส่ง email
    import time
    time.sleep(2)  # จำลองว่าใช้เวลา
    logger.info(f"Email ส่งถึง {to}: {subject}")


# ─────────────────────────────────────────
# Async background task
# ─────────────────────────────────────────

async def send_email_async(to: str, subject: str, body: str):
    """ส่ง email แบบ async"""
    await asyncio.sleep(2)  # จำลองการส่ง email
    logger.info(f"Email ส่งถึง {to}: {subject}")


async def process_image(image_id: int, filters: list[str]):
    """ประมวลผลรูปภาพใน background"""
    await asyncio.sleep(5)  # จำลองการประมวลผล
    logger.info(f"Image {image_id} processed with filters: {filters}")


async def cleanup_temp_files(directory: str):
    """ลบไฟล์ temp ใน background"""
    import os
    import glob
    for f in glob.glob(f"{directory}/*.tmp"):
        os.remove(f)
    logger.info(f"Cleaned up temp files in {directory}")


# ─────────────────────────────────────────
# Routes
# ─────────────────────────────────────────

@router.post("/send-email")
async def send_email_endpoint(
    request: EmailRequest,
    background_tasks: BackgroundTasks
):
    """ส่ง email ใน background - ตอบกลับทันที"""
    # เพิ่ม task ให้รันใน background
    background_tasks.add_task(
        send_email_async,
        to=request.to,
        subject=request.subject,
        body=request.body
    )
    
    return {
        "message": "รับคำขอแล้ว กำลังส่ง email",
        "status": "queued"
    }


@router.post("/upload-and-process")
async def upload_image(
    image_id: int,
    background_tasks: BackgroundTasks
):
    """อัปโหลดรูปและประมวลผลใน background"""
    # สามารถเพิ่ม task หลายอันได้
    background_tasks.add_task(process_image, image_id, ["resize", "compress"])
    background_tasks.add_task(send_email_async, "admin@example.com", "New Image", f"Image {image_id} uploaded")
    background_tasks.add_task(cleanup_temp_files, "/tmp/uploads")
    
    return {"image_id": image_id, "status": "processing"}
```

---

## 2. WebSocket

```python
# app/routers/chat.py - WebSocket Chat

from fastapi import APIRouter, WebSocket, WebSocketDisconnect, Depends
from typing import Optional
import json
import asyncio

router = APIRouter()


# ─────────────────────────────────────────
# Connection Manager - จัดการ WebSocket connections
# ─────────────────────────────────────────

class ConnectionManager:
    def __init__(self):
        # {room_id: {client_id: WebSocket}}
        self.rooms: dict[str, dict[str, WebSocket]] = {}
        # {client_id: username}
        self.usernames: dict[str, str] = {}
    
    async def connect(self, websocket: WebSocket, room_id: str, client_id: str, username: str):
        """เชื่อมต่อ client เข้า room"""
        await websocket.accept()
        
        if room_id not in self.rooms:
            self.rooms[room_id] = {}
        
        self.rooms[room_id][client_id] = websocket
        self.usernames[client_id] = username
        
        # แจ้งคนอื่นว่ามีคนเข้ามา
        await self.broadcast_to_room(
            room_id,
            {
                "type": "system",
                "message": f"{username} เข้าร่วม chat",
                "room_id": room_id,
                "online_count": len(self.rooms[room_id])
            },
            exclude_client=None  # แจ้งทุกคนรวมถึงตัวเอง
        )
    
    def disconnect(self, room_id: str, client_id: str):
        """ตัดการเชื่อมต่อ client"""
        if room_id in self.rooms:
            self.rooms[room_id].pop(client_id, None)
            if not self.rooms[room_id]:
                del self.rooms[room_id]
        self.usernames.pop(client_id, None)
    
    async def send_personal(self, message: dict, client_id: str, room_id: str):
        """ส่งข้อความถึง client คนเดียว"""
        if room_id in self.rooms and client_id in self.rooms[room_id]:
            ws = self.rooms[room_id][client_id]
            await ws.send_json(message)
    
    async def broadcast_to_room(
        self,
        room_id: str,
        message: dict,
        exclude_client: Optional[str] = None
    ):
        """ส่งข้อความหา client ทุกคนใน room"""
        if room_id not in self.rooms:
            return
        
        disconnected = []
        for client_id, websocket in self.rooms[room_id].items():
            if client_id == exclude_client:
                continue
            try:
                await websocket.send_json(message)
            except Exception:
                disconnected.append(client_id)
        
        # ลบ connections ที่ตัดแล้ว
        for client_id in disconnected:
            self.disconnect(room_id, client_id)
    
    def get_room_users(self, room_id: str) -> list[str]:
        """ดูรายชื่อ users ใน room"""
        if room_id not in self.rooms:
            return []
        return [self.usernames.get(cid, cid) for cid in self.rooms[room_id]]


# สร้าง singleton
manager = ConnectionManager()


# ─────────────────────────────────────────
# WebSocket endpoint
# ─────────────────────────────────────────

@router.websocket("/ws/chat/{room_id}/{client_id}")
async def websocket_chat(
    websocket: WebSocket,
    room_id: str,
    client_id: str,
    username: str = "Anonymous"
):
    """WebSocket endpoint สำหรับ chat"""
    await manager.connect(websocket, room_id, client_id, username)
    
    try:
        while True:
            # รอรับข้อความ
            data = await websocket.receive_json()
            
            message_type = data.get("type", "message")
            
            if message_type == "message":
                # ส่งข้อความไปทุกคนใน room
                await manager.broadcast_to_room(
                    room_id,
                    {
                        "type": "message",
                        "from": username,
                        "client_id": client_id,
                        "text": data.get("text", ""),
                        "timestamp": data.get("timestamp")
                    }
                )
            
            elif message_type == "ping":
                # ตอบ pong กลับ
                await manager.send_personal(
                    {"type": "pong"},
                    client_id,
                    room_id
                )
            
            elif message_type == "users":
                # ส่งรายชื่อ users ใน room
                await manager.send_personal(
                    {
                        "type": "users",
                        "users": manager.get_room_users(room_id)
                    },
                    client_id,
                    room_id
                )
    
    except WebSocketDisconnect:
        manager.disconnect(room_id, client_id)
        await manager.broadcast_to_room(
            room_id,
            {
                "type": "system",
                "message": f"{username} ออกจาก chat",
                "online_count": len(manager.rooms.get(room_id, {}))
            }
        )
```

---

## 3. Custom Middleware

```python
# app/middleware/custom.py

from fastapi import FastAPI, Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
import time
import uuid
import json
import logging

logger = logging.getLogger(__name__)


class RequestIDMiddleware(BaseHTTPMiddleware):
    """เพิ่ม Request ID ให้แต่ละ request"""
    
    async def dispatch(self, request: Request, call_next):
        # สร้าง request ID
        request_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
        
        # เก็บไว้ใน request state
        request.state.request_id = request_id
        
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        
        return response


class TimingMiddleware(BaseHTTPMiddleware):
    """วัดเวลา request"""
    
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        duration_ms = (time.perf_counter() - start) * 1000
        
        response.headers["X-Process-Time"] = f"{duration_ms:.2f}ms"
        
        # Log ถ้าช้าเกิน 1 วินาที
        if duration_ms > 1000:
            logger.warning(
                f"Slow request: {request.method} {request.url.path} "
                f"took {duration_ms:.0f}ms"
            )
        
        return response


class MaintenanceModeMiddleware(BaseHTTPMiddleware):
    """Maintenance mode middleware"""
    
    def __init__(self, app, maintenance_mode: bool = False):
        super().__init__(app)
        self.maintenance_mode = maintenance_mode
    
    async def dispatch(self, request: Request, call_next):
        if self.maintenance_mode:
            # อนุญาต health check
            if request.url.path == "/health":
                return await call_next(request)
            
            return Response(
                content=json.dumps({
                    "error": "Service Unavailable",
                    "message": "ระบบกำลังปรับปรุง กรุณากลับมาใหม่ภายหลัง"
                }),
                status_code=503,
                media_type="application/json",
                headers={"Retry-After": "3600"}
            )
        
        return await call_next(request)


# ลงทะเบียน middleware
def setup_middleware(app: FastAPI):
    app.add_middleware(RequestIDMiddleware)
    app.add_middleware(TimingMiddleware)
    app.add_middleware(
        MaintenanceModeMiddleware,
        maintenance_mode=False
    )
```

---

## 4. Testing

```python
# tests/conftest.py

import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

from app.main import app
from app.database import Base, get_db


# ─────────────────────────────────────────
# Test database
# ─────────────────────────────────────────

TEST_DATABASE_URL = "sqlite+aiosqlite:///./test.db"

test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
TestSessionLocal = async_sessionmaker(test_engine, expire_on_commit=False)


async def override_get_db():
    """Override database dependency สำหรับ test"""
    async with TestSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()


@pytest_asyncio.fixture(scope="session")
async def setup_database():
    """สร้าง test database"""
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await test_engine.dispose()


@pytest_asyncio.fixture
async def db_session(setup_database):
    """Database session สำหรับแต่ละ test"""
    async with TestSessionLocal() as session:
        yield session
        await session.rollback()


@pytest_asyncio.fixture
async def client(setup_database):
    """HTTP client สำหรับ test"""
    app.dependency_overrides[get_db] = override_get_db
    
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as ac:
        yield ac
    
    app.dependency_overrides.clear()


@pytest_asyncio.fixture
async def auth_client(client, db_session):
    """HTTP client ที่ login แล้ว"""
    # สร้าง test user
    response = await client.post("/api/v1/auth/register", json={
        "username": "testuser",
        "email": "test@example.com",
        "password": "TestPass123!"
    })
    
    # Login
    response = await client.post("/api/v1/auth/login", data={
        "username": "testuser",
        "password": "TestPass123!"
    })
    
    token = response.json()["access_token"]
    client.headers["Authorization"] = f"Bearer {token}"
    
    yield client
```

```python
# tests/test_auth.py

import pytest
import pytest_asyncio
from httpx import AsyncClient


class TestRegister:
    """ทดสอบ Register"""
    
    @pytest.mark.asyncio
    async def test_register_success(self, client: AsyncClient):
        response = await client.post("/api/v1/auth/register", json={
            "username": "newuser",
            "email": "new@example.com",
            "password": "Password123!"
        })
        
        assert response.status_code == 201
        data = response.json()
        assert data["username"] == "newuser"
        assert "password" not in data
    
    @pytest.mark.asyncio
    async def test_register_duplicate_email(self, client: AsyncClient):
        payload = {
            "username": "user1",
            "email": "same@example.com",
            "password": "Password123!"
        }
        await client.post("/api/v1/auth/register", json=payload)
        
        # ลงทะเบียนซ้ำ
        payload["username"] = "user2"
        response = await client.post("/api/v1/auth/register", json=payload)
        
        assert response.status_code == 409
    
    @pytest.mark.asyncio
    async def test_register_weak_password(self, client: AsyncClient):
        response = await client.post("/api/v1/auth/register", json={
            "username": "user3",
            "email": "user3@example.com",
            "password": "weak"
        })
        
        assert response.status_code == 422


class TestLogin:
    """ทดสอบ Login"""
    
    @pytest.mark.asyncio
    async def test_login_success(self, client: AsyncClient):
        # สร้าง user ก่อน
        await client.post("/api/v1/auth/register", json={
            "username": "loginuser",
            "email": "login@example.com",
            "password": "Password123!"
        })
        
        response = await client.post("/api/v1/auth/login", data={
            "username": "loginuser",
            "password": "Password123!"
        })
        
        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert "refresh_token" in data
        assert data["token_type"] == "bearer"
    
    @pytest.mark.asyncio
    async def test_login_wrong_password(self, client: AsyncClient):
        response = await client.post("/api/v1/auth/login", data={
            "username": "loginuser",
            "password": "WrongPass!"
        })
        
        assert response.status_code == 401
    
    @pytest.mark.asyncio
    async def test_protected_without_token(self, client: AsyncClient):
        response = await client.get("/api/v1/users/me")
        assert response.status_code == 401
    
    @pytest.mark.asyncio
    async def test_protected_with_token(self, auth_client: AsyncClient):
        response = await auth_client.get("/api/v1/users/me")
        assert response.status_code == 200
```

```python
# tests/test_posts.py

import pytest
from httpx import AsyncClient


class TestPosts:
    """ทดสอบ Post CRUD"""
    
    @pytest.mark.asyncio
    async def test_create_post(self, auth_client: AsyncClient):
        response = await auth_client.post("/api/v1/posts", json={
            "title": "Test Post",
            "content": "Test content",
            "published": True
        })
        
        assert response.status_code == 201
        data = response.json()
        assert data["title"] == "Test Post"
        assert "id" in data
    
    @pytest.mark.asyncio
    async def test_get_posts(self, auth_client: AsyncClient):
        response = await auth_client.get("/api/v1/posts")
        
        assert response.status_code == 200
        data = response.json()
        assert isinstance(data, list)
    
    @pytest.mark.asyncio
    async def test_get_post_not_found(self, auth_client: AsyncClient):
        response = await auth_client.get("/api/v1/posts/99999")
        assert response.status_code == 404
    
    @pytest.mark.asyncio
    async def test_update_post(self, auth_client: AsyncClient):
        # สร้าง post
        create_response = await auth_client.post("/api/v1/posts", json={
            "title": "Original Title",
            "content": "Original content"
        })
        post_id = create_response.json()["id"]
        
        # อัปเดต
        response = await auth_client.put(f"/api/v1/posts/{post_id}", json={
            "title": "Updated Title"
        })
        
        assert response.status_code == 200
        assert response.json()["title"] == "Updated Title"
    
    @pytest.mark.asyncio
    async def test_delete_post(self, auth_client: AsyncClient):
        # สร้าง post
        create_response = await auth_client.post("/api/v1/posts", json={
            "title": "To Delete",
            "content": "Will be deleted"
        })
        post_id = create_response.json()["id"]
        
        # ลบ
        response = await auth_client.delete(f"/api/v1/posts/{post_id}")
        assert response.status_code == 204
        
        # ตรวจสอบว่าลบแล้ว
        get_response = await auth_client.get(f"/api/v1/posts/{post_id}")
        assert get_response.status_code == 404
```

---

## 5. Production-Ready API

```python
# app/main.py - Production app

from contextlib import asynccontextmanager
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import logging

from app.config import get_settings
from app.database import init_db
from app.routers import auth, users, posts
from app.middleware.custom import setup_middleware

settings = get_settings()
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup / Shutdown events"""
    logger.info("Starting up...")
    await init_db()
    yield
    logger.info("Shutting down...")


app = FastAPI(
    title=settings.app_name,
    version="1.0.0",
    docs_url="/docs" if settings.debug else None,
    redoc_url="/redoc" if settings.debug else None,
    lifespan=lifespan
)

# Middleware
setup_middleware(app)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Routers
app.include_router(auth.router, prefix="/api/v1")
app.include_router(users.router, prefix="/api/v1")
app.include_router(posts.router, prefix="/api/v1")


@app.get("/health")
async def health_check():
    return {"status": "ok", "version": "1.0.0"}


# Global exception handler
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unhandled error: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={"detail": "Internal server error"}
    )
```

---

## 6. Dockerfile

```dockerfile
# Dockerfile

FROM python:3.12-slim AS base
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# Dependencies
FROM base AS deps
COPY requirements.txt .
RUN pip install --upgrade pip && pip install -r requirements.txt

# Final image
FROM base AS final
COPY --from=deps /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=deps /usr/local/bin /usr/local/bin
COPY . .

RUN adduser --disabled-password --gecos "" appuser && chown -R appuser /app
USER appuser

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s CMD python -c "import httpx; httpx.get('http://localhost:8000/health')"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

```yaml
# docker-compose.yml

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://user:pass@db:5432/mydb
      - SECRET_KEY=${SECRET_KEY}
      - DEBUG=false
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

---

## Exercises

### Exercise 1: Chat with Authentication
เพิ่ม JWT auth ให้ WebSocket:
- ส่ง token ใน query param `?token=...`
- Validate token ก่อน accept connection
- แสดงชื่อจริงจาก token

### Exercise 2: Task Queue
ใช้ Redis + Background Tasks:
- สร้าง task queue ด้วย Redis
- Track สถานะ task (pending/running/done/failed)
- Endpoint ดูสถานะ task ด้วย task_id

### Exercise 3: Full Test Suite
เขียน tests ให้ครบ:
- Auth tests (register/login/refresh)
- CRUD tests
- WebSocket tests ด้วย `pytest-asyncio` + `websockets`
- Coverage > 80%

---

## สรุป

สิ่งที่เรียนรู้:
- **Background Tasks** - งานที่ไม่ต้องรอผล เช่น ส่ง email
- **WebSocket** - real-time communication
- **Middleware** - Request ID, timing, maintenance mode
- **Testing** - TestClient, pytest-asyncio, fixtures
- **Deployment** - Docker multi-stage, docker-compose
- **Production** - lifespan, CORS, exception handler, health check

---

จบหลักสูตร FastAPI แล้ว! ขอให้โชคดีในการพัฒนา! 🎉
