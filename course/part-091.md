# Part 091 - FastAPI Introduction

## เป้าหมายการเรียนรู้

- เข้าใจว่า FastAPI คืออะไรและทำไมถึงเร็ว
- ติดตั้งและสร้าง Hello World
- ทำงานกับ Path Operations (GET/POST/PUT/PATCH/DELETE)
- ใช้ Interactive Docs (Swagger UI, ReDoc)
- เปรียบเทียบกับ Django และ Flask

---

## 1. FastAPI คืออะไร?

FastAPI เป็น modern, high-performance web framework สำหรับสร้าง APIs ด้วย Python พัฒนาโดย Sebastián Ramírez (tiangolo) และ release ในปี 2018

### 1.1 ทำไม FastAPI ถึงเร็ว?

```
FastAPI สร้างบน:
1. Starlette - ASGI framework (Asynchronous)
2. Pydantic v2 - Data validation ที่เขียนด้วย Rust
3. Uvicorn/Hypercorn - ASGI server

WSGI (Flask, Django) vs ASGI (FastAPI):
- WSGI: Synchronous - รอ I/O ทีละ request
- ASGI: Asynchronous - จัดการ I/O พร้อมกันได้หลาย requests

ผลลัพธ์: FastAPI เร็วพอๆ กับ NodeJS และ Go!
(จากการทดสอบ TechEmpower Framework Benchmarks)
```

### 1.2 คุณสมบัติหลัก

```
✅ Fast - เร็วมากเทียบชั้น NodeJS
✅ Fast to code - ลด bugs ~40%, ใช้เวลาพัฒนาน้อยลง ~50%
✅ Fewer bugs - Validation อัตโนมัติ ลด human errors
✅ Intuitive - Editor completion ดีเยี่ยม
✅ Easy - เรียนรู้ง่าย
✅ Short - Minimize code duplication
✅ Robust - Production-ready
✅ Standards-based - OpenAPI, JSON Schema
```

---

## 2. เปรียบเทียบ FastAPI vs Flask vs Django

| หัวข้อ | FastAPI | Flask | Django |
|--------|---------|-------|--------|
| ประเภท | API Framework | Micro Framework | Full-stack |
| Async support | Native | ต้องใช้ extension | Limited |
| Auto docs | ✅ Built-in | ❌ | ❌ |
| Type hints | ✅ Required | Optional | Optional |
| ORM | ไม่มีในตัว | ไม่มีในตัว | มี |
| Validation | Pydantic (auto) | Manual/WTForms | Forms/DRF |
| Performance | สูงมาก | สูง | ปานกลาง |
| Learning curve | ปานกลาง | ต่ำ | สูง |
| เหมาะกับ | APIs, Microservices | APIs, Small apps | Full-stack apps |

---

## 3. ติดตั้ง FastAPI

```bash
# สร้าง virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate   # Windows

# ติดตั้ง FastAPI และ Uvicorn (ASGI server)
pip install fastapi uvicorn[standard]

# ติดตั้ง packages เพิ่มเติมที่ใช้บ่อย
pip install fastapi uvicorn[standard] \
    sqlalchemy aiosqlite \
    pydantic pydantic-settings \
    python-jose[cryptography] \
    passlib[bcrypt] \
    python-multipart \
    python-dotenv

# ตรวจสอบ version
python -c "import fastapi; print(fastapi.__version__)"
```

---

## 4. Hello World

```python
# main.py - FastAPI Hello World

from fastapi import FastAPI

# สร้าง FastAPI instance
app = FastAPI(
    title="My First FastAPI",
    description="API สำหรับเรียน FastAPI",
    version="1.0.0"
)

# Path Operation (GET /)
@app.get("/")
def root():
    """หน้าแรก"""
    return {"message": "Hello, FastAPI!"}


# Path Operation พร้อม type hints
@app.get("/hello/{name}")
def hello(name: str):
    """ทักทายด้วยชื่อ"""
    return {"message": f"สวัสดี, {name}!"}
```

```bash
# รัน development server
uvicorn main:app --reload

# รันบน port อื่น
uvicorn main:app --reload --port 8001

# Output:
# INFO:     Uvicorn running on http://127.0.0.1:8000
# INFO:     Started reloader process
```

เปิด browser:
- `http://127.0.0.1:8000` - Hello World
- `http://127.0.0.1:8000/docs` - **Swagger UI (Interactive Docs)**
- `http://127.0.0.1:8000/redoc` - **ReDoc**

---

## 5. Interactive Documentation

FastAPI สร้าง docs อัตโนมัติจาก code โดยไม่ต้องเขียน documentation เพิ่ม!

### 5.1 Swagger UI (/docs)

```python
# main.py - ปรับแต่ง docs

from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

app = FastAPI(
    title="Product API",
    description="""
    ## Product Management API
    
    API สำหรับจัดการสินค้าในร้านค้า
    
    ### Features
    - จัดการสินค้า (CRUD)
    - ค้นหาและกรองสินค้า
    - จัดการหมวดหมู่
    """,
    version="2.0.0",
    contact={
        "name": "Dev Team",
        "email": "dev@example.com",
    },
    license_info={
        "name": "MIT",
    },
    # เปิด/ปิด docs endpoints
    docs_url="/docs",      # Swagger UI
    redoc_url="/redoc",    # ReDoc
    openapi_url="/openapi.json",  # OpenAPI schema
)
```

### 5.2 ปิด Docs ใน Production

```python
# production: ปิด docs เพื่อความปลอดภัย
app = FastAPI(
    docs_url=None,     # ปิด Swagger
    redoc_url=None,    # ปิด ReDoc
)
```

---

## 6. Path Operations

```python
# path_operations.py - HTTP Methods ทั้งหมด

from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

# Mock database
items_db = {
    1: {"id": 1, "name": "สมุดโน้ต", "price": 59.0},
    2: {"id": 2, "name": "ปากกา", "price": 25.0},
}
next_id = 3


# Pydantic model สำหรับ request/response
class ItemCreate(BaseModel):
    name: str
    price: float
    description: Optional[str] = None


class ItemUpdate(BaseModel):
    name: Optional[str] = None
    price: Optional[float] = None
    description: Optional[str] = None


class ItemResponse(BaseModel):
    id: int
    name: str
    price: float
    description: Optional[str] = None


# ─────────────────────────────────────────
# GET - ดูข้อมูล
# ─────────────────────────────────────────

@app.get("/items", response_model=list[ItemResponse])
def list_items():
    """ดูสินค้าทั้งหมด"""
    return list(items_db.values())


@app.get("/items/{item_id}", response_model=ItemResponse)
def get_item(item_id: int):
    """ดูสินค้าเดียว"""
    if item_id not in items_db:
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail="ไม่พบสินค้า")
    return items_db[item_id]


# ─────────────────────────────────────────
# POST - สร้างใหม่
# ─────────────────────────────────────────

@app.post("/items", response_model=ItemResponse, status_code=201)
def create_item(item: ItemCreate):
    """สร้างสินค้าใหม่"""
    global next_id
    new_item = {"id": next_id, **item.dict()}
    items_db[next_id] = new_item
    next_id += 1
    return new_item


# ─────────────────────────────────────────
# PUT - แทนที่ทั้งหมด
# ─────────────────────────────────────────

@app.put("/items/{item_id}", response_model=ItemResponse)
def replace_item(item_id: int, item: ItemCreate):
    """แทนที่สินค้า (ต้องส่งทุก fields)"""
    if item_id not in items_db:
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail="ไม่พบสินค้า")
    
    updated = {"id": item_id, **item.dict()}
    items_db[item_id] = updated
    return updated


# ─────────────────────────────────────────
# PATCH - อัปเดตบางส่วน
# ─────────────────────────────────────────

@app.patch("/items/{item_id}", response_model=ItemResponse)
def update_item(item_id: int, item: ItemUpdate):
    """อัปเดตสินค้าบางส่วน"""
    if item_id not in items_db:
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail="ไม่พบสินค้า")
    
    # อัปเดตเฉพาะ fields ที่ส่งมา
    existing = items_db[item_id].copy()
    update_data = item.dict(exclude_unset=True)  # เอาเฉพาะ fields ที่ตั้งค่า
    existing.update(update_data)
    items_db[item_id] = existing
    return existing


# ─────────────────────────────────────────
# DELETE - ลบ
# ─────────────────────────────────────────

@app.delete("/items/{item_id}", status_code=204)
def delete_item(item_id: int):
    """ลบสินค้า"""
    if item_id not in items_db:
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail="ไม่พบสินค้า")
    
    del items_db[item_id]
    # return None (204 No Content)
```

---

## 7. Type Hints และ Automatic Validation

FastAPI ใช้ Python type hints สำหรับ:
1. Validation อัตโนมัติ
2. Editor autocompletion
3. Documentation อัตโนมัติ

```python
# type_hints_demo.py

from fastapi import FastAPI, Path, Query
from pydantic import BaseModel
from typing import Optional, List
from datetime import date, datetime
from enum import Enum

app = FastAPI()


class Status(str, Enum):
    """Enum สำหรับ status"""
    active = "active"
    inactive = "inactive"
    pending = "pending"


# ─────────────────────────────────────────
# Path Parameters พร้อม validation
# ─────────────────────────────────────────

@app.get("/users/{user_id}")
def get_user(
    user_id: int = Path(
        title="User ID",
        description="ID ของ user",
        gt=0,          # ต้องมากกว่า 0
        le=999999      # ต้องน้อยกว่าหรือเท่ากับ 999999
    )
):
    return {"user_id": user_id}


# ─────────────────────────────────────────
# Query Parameters
# ─────────────────────────────────────────

@app.get("/items")
def search_items(
    q: Optional[str] = Query(None, min_length=2, max_length=50),
    page: int = Query(1, ge=1),
    per_page: int = Query(10, ge=1, le=100),
    status: Optional[Status] = None,
    min_price: Optional[float] = Query(None, ge=0),
    max_price: Optional[float] = Query(None, ge=0),
    tags: Optional[List[str]] = Query(None),  # ?tags=a&tags=b
):
    """ค้นหาสินค้า
    
    - **q**: คำค้นหา
    - **page**: หน้าที่
    - **per_page**: จำนวนต่อหน้า
    - **status**: สถานะสินค้า
    """
    return {
        "query": q,
        "page": page,
        "per_page": per_page,
        "status": status,
        "price_range": [min_price, max_price],
        "tags": tags
    }


# ─────────────────────────────────────────
# Request Body
# ─────────────────────────────────────────

class UserCreate(BaseModel):
    username: str
    email: str
    age: Optional[int] = None
    birth_date: Optional[date] = None
    tags: List[str] = []


@app.post("/users")
def create_user(user: UserCreate):
    """
    FastAPI แปลงและ validate ข้อมูลอัตโนมัติ:
    - age ต้องเป็น integer
    - birth_date ต้องเป็น "YYYY-MM-DD"
    - tags ต้องเป็น array of strings
    """
    return {
        "user": user.dict(),
        "received_at": datetime.now()
    }
```

---

## 8. Async/Await

```python
# async_demo.py - Async operations

import asyncio
from fastapi import FastAPI
import httpx  # Async HTTP client

app = FastAPI()


# Sync function - ยังใช้ได้ใน FastAPI
@app.get("/sync")
def sync_endpoint():
    """Synchronous - FastAPI รันใน thread pool"""
    return {"type": "sync"}


# Async function - ใช้ประโยชน์จาก ASGI
@app.get("/async")
async def async_endpoint():
    """Asynchronous - รันใน event loop"""
    await asyncio.sleep(0.1)  # simulate async I/O
    return {"type": "async"}


# Async database query
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """ดึงข้อมูล user แบบ async"""
    # async database call
    user = await get_user_from_db(user_id)  # async function
    return user


async def get_user_from_db(user_id: int):
    """Mock async database call"""
    await asyncio.sleep(0.01)  # simulate DB latency
    return {"id": user_id, "name": "Alice"}


# Async external API call
@app.get("/github/{username}")
async def get_github_user(username: str):
    """ดึงข้อมูล GitHub user (async HTTP)"""
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://api.github.com/users/{username}"
        )
        if response.status_code == 404:
            from fastapi import HTTPException
            raise HTTPException(404, "ไม่พบ GitHub user")
        return response.json()


# ทำหลาย async operations พร้อมกัน
@app.get("/dashboard")
async def get_dashboard():
    """ดึงข้อมูลหลายส่วนพร้อมกัน"""
    # รันพร้อมกัน (parallel) - เร็วกว่ารันทีละอัน
    users_task = get_total_users()
    posts_task = get_total_posts()
    orders_task = get_recent_orders()
    
    total_users, total_posts, recent_orders = await asyncio.gather(
        users_task,
        posts_task,
        orders_task
    )
    
    return {
        "total_users": total_users,
        "total_posts": total_posts,
        "recent_orders": recent_orders
    }


async def get_total_users():
    await asyncio.sleep(0.05)
    return 1500

async def get_total_posts():
    await asyncio.sleep(0.03)
    return 8900

async def get_recent_orders():
    await asyncio.sleep(0.04)
    return [{"id": 1}, {"id": 2}]
```

---

## 9. Application Structure

```python
# main.py - Complete FastAPI application

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from contextlib import asynccontextmanager
import logging

# Setup logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Lifecycle events (startup/shutdown)"""
    # Startup
    logger.info("Starting up...")
    # await init_db()  # async startup tasks
    
    yield  # รัน application
    
    # Shutdown
    logger.info("Shutting down...")
    # await close_db()  # cleanup


# สร้าง FastAPI application
app = FastAPI(
    title="My API",
    description="Complete FastAPI Example",
    version="1.0.0",
    lifespan=lifespan,
)

# CORS Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "https://example.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


# Include routers (Blueprints equivalent)
from app.routers import users, posts, auth
app.include_router(auth.router, prefix="/api/v1/auth", tags=["auth"])
app.include_router(users.router, prefix="/api/v1/users", tags=["users"])
app.include_router(posts.router, prefix="/api/v1/posts", tags=["posts"])


# Health check
@app.get("/health", tags=["system"])
async def health_check():
    return {"status": "healthy", "version": "1.0.0"}


# Root
@app.get("/", tags=["system"])
async def root():
    return {
        "message": "Welcome to My API",
        "docs": "/docs",
        "redoc": "/redoc"
    }
```

```
# โครงสร้างโปรเจกต์ FastAPI
my_fastapi_app/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI instance
│   ├── config.py            # Settings (Pydantic Settings)
│   ├── database.py          # Database setup
│   ├── models/              # SQLAlchemy models
│   │   ├── user.py
│   │   └── post.py
│   ├── schemas/             # Pydantic schemas
│   │   ├── user.py
│   │   └── post.py
│   ├── routers/             # Route handlers
│   │   ├── auth.py
│   │   ├── users.py
│   │   └── posts.py
│   ├── services/            # Business logic
│   │   └── auth_service.py
│   ├── dependencies/        # Depends() functions
│   │   └── auth.py
│   └── middleware/          # Custom middleware
├── tests/
├── alembic/                 # Database migrations
├── requirements.txt
└── .env
```

---

## 10. ตัวอย่าง Complete Simple API

```python
# complete_example.py - FastAPI app สมบูรณ์

from fastapi import FastAPI, HTTPException, Query, Path, status
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime

app = FastAPI(
    title="Book Store API",
    description="API สำหรับจัดการร้านหนังสือ",
    version="1.0.0"
)

# ─────────────────────────────────────────
# Schemas
# ─────────────────────────────────────────

class BookBase(BaseModel):
    title: str = Field(..., min_length=1, max_length=200, description="ชื่อหนังสือ")
    author: str = Field(..., min_length=1, max_length=100)
    price: float = Field(..., gt=0, description="ราคาต้องมากกว่า 0")
    isbn: Optional[str] = Field(None, pattern=r'^\d{10}(\d{3})?$')
    genre: Optional[str] = None
    published_year: Optional[int] = Field(None, ge=1000, le=2100)


class BookCreate(BookBase):
    pass


class BookUpdate(BaseModel):
    title: Optional[str] = Field(None, min_length=1, max_length=200)
    author: Optional[str] = Field(None, min_length=1, max_length=100)
    price: Optional[float] = Field(None, gt=0)
    genre: Optional[str] = None


class BookResponse(BookBase):
    id: int
    created_at: datetime
    
    class Config:
        from_attributes = True


# ─────────────────────────────────────────
# Database (in-memory)
# ─────────────────────────────────────────

books_db: dict[int, dict] = {
    1: {
        "id": 1, "title": "Clean Code", "author": "Robert C. Martin",
        "price": 650.0, "isbn": "0132350882", "genre": "Programming",
        "published_year": 2008, "created_at": datetime.now()
    },
    2: {
        "id": 2, "title": "Python Crash Course", "author": "Eric Matthes",
        "price": 480.0, "isbn": "1593276036", "genre": "Programming",
        "published_year": 2019, "created_at": datetime.now()
    },
}
next_id = 3


# ─────────────────────────────────────────
# Endpoints
# ─────────────────────────────────────────

@app.get(
    "/books",
    response_model=dict,
    summary="ดูรายการหนังสือทั้งหมด",
    tags=["books"]
)
async def list_books(
    search: Optional[str] = Query(None, description="ค้นหาชื่อหนังสือหรือผู้แต่ง"),
    genre: Optional[str] = Query(None, description="กรองตาม genre"),
    min_price: Optional[float] = Query(None, ge=0),
    max_price: Optional[float] = Query(None, ge=0),
    page: int = Query(1, ge=1),
    per_page: int = Query(10, ge=1, le=50)
):
    """
    ดูรายการหนังสือทั้งหมด พร้อม filtering และ pagination
    
    - **search**: ค้นหาในชื่อและผู้แต่ง
    - **genre**: กรองตาม genre
    - **min_price**, **max_price**: ช่วงราคา
    """
    books = list(books_db.values())
    
    if search:
        books = [b for b in books if 
                 search.lower() in b['title'].lower() or 
                 search.lower() in b['author'].lower()]
    
    if genre:
        books = [b for b in books if b.get('genre', '').lower() == genre.lower()]
    
    if min_price is not None:
        books = [b for b in books if b['price'] >= min_price]
    
    if max_price is not None:
        books = [b for b in books if b['price'] <= max_price]
    
    total = len(books)
    start = (page - 1) * per_page
    paginated = books[start:start + per_page]
    
    return {
        "data": paginated,
        "pagination": {
            "page": page,
            "per_page": per_page,
            "total": total,
            "pages": (total + per_page - 1) // per_page
        }
    }


@app.get("/books/{book_id}", response_model=BookResponse, tags=["books"])
async def get_book(
    book_id: int = Path(..., title="Book ID", ge=1)
):
    """ดูหนังสือเดียว"""
    if book_id not in books_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"ไม่พบหนังสือ ID {book_id}"
        )
    return books_db[book_id]


@app.post(
    "/books",
    response_model=BookResponse,
    status_code=status.HTTP_201_CREATED,
    tags=["books"]
)
async def create_book(book: BookCreate):
    """สร้างหนังสือใหม่"""
    global next_id
    new_book = {
        "id": next_id,
        **book.dict(),
        "created_at": datetime.now()
    }
    books_db[next_id] = new_book
    next_id += 1
    return new_book


@app.patch("/books/{book_id}", response_model=BookResponse, tags=["books"])
async def update_book(
    book_id: int = Path(..., ge=1),
    book_data: BookUpdate = None
):
    """อัปเดตหนังสือบางส่วน"""
    if book_id not in books_db:
        raise HTTPException(status_code=404, detail="ไม่พบหนังสือ")
    
    book = books_db[book_id].copy()
    update_data = book_data.dict(exclude_unset=True)
    book.update(update_data)
    books_db[book_id] = book
    return book


@app.delete(
    "/books/{book_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    tags=["books"]
)
async def delete_book(book_id: int = Path(..., ge=1)):
    """ลบหนังสือ"""
    if book_id not in books_db:
        raise HTTPException(status_code=404, detail="ไม่พบหนังสือ")
    del books_db[book_id]


if __name__ == "__main__":
    import uvicorn
    uvicorn.run("complete_example:app", host="0.0.0.0", port=8000, reload=True)
```

---

## Exercises

### Exercise 1: Hello FastAPI
สร้าง FastAPI app ที่มี:
- `GET /` - Welcome message
- `GET /about` - ข้อมูล API
- `GET /items/{item_id}` - ดู item พร้อม path validation
- `GET /search?q=&page=&per_page=` - ค้นหาพร้อม query params

### Exercise 2: TODO API
สร้าง TODO API ด้วย FastAPI:
- CRUD สำหรับ todos
- Filter ตาม status (done/pending)
- Priority levels (low/medium/high)
- Swagger UI ที่ document ดี

### Exercise 3: Async API
สร้าง async API ที่:
- ดึงข้อมูลจาก external API (httpx)
- ทำ parallel requests ด้วย asyncio.gather
- แสดงความแตกต่างระหว่าง sync/async ในด้านประสิทธิภาพ

---

## สรุป

สิ่งที่เรียนรู้ใน Part นี้:
- **FastAPI** เป็น modern high-performance framework
- **ASGI** ทำให้ async/await ทำงานได้ดี
- **Pydantic** จัดการ validation อัตโนมัติ
- **Interactive Docs** สร้างจาก type hints โดยอัตโนมัติ
- **Path Operations** ทุก HTTP methods
- **Type hints** ช่วย validation และ documentation

---

## ลิงก์ Part ถัดไป

➡️ [Part 092 - FastAPI Request and Response](./part-092.md)
