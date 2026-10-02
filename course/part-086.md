# Part 086: FastAPI Basics
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- ติดตั้งและ setup FastAPI
- สร้าง path operations (endpoints)
- กำหนด request และ response models
- ดู automatic documentation (Swagger/ReDoc)
- เข้าใจ Type Hints ใน FastAPI

---

## 1. FastAPI คืออะไร?

FastAPI เป็น modern Python web framework สำหรับสร้าง API ที่:
- **เร็วมาก**: เทียบเท่า Node.js และ Go
- **Type-safe**: ใช้ Python type hints
- **Auto docs**: สร้าง Swagger UI อัตโนมัติ
- **Async support**: รองรับ async/await
- **Validation**: ตรวจสอบข้อมูลด้วย Pydantic

### เปรียบเทียบกับ Flask

| Feature | Flask | FastAPI |
|---------|-------|---------|
| Type hints | ไม่บังคับ | บังคับ (เป็น core feature) |
| Auto docs | ต้องติดตั้ง extension | มีในตัว |
| Async | ต้องใช้ extension | native support |
| Speed | ปานกลาง | เร็วมาก |
| Learning curve | ง่าย | ง่าย-ปานกลาง |

---

## 2. ติดตั้ง FastAPI

```bash
# ติดตั้ง FastAPI และ Uvicorn (ASGI server)
pip install fastapi uvicorn[standard]

# สำหรับ development เพิ่ม:
pip install httpx  # สำหรับ testing
```

---

## 3. Hello World

```python
# main.py

from fastapi import FastAPI

# สร้าง FastAPI application
app = FastAPI(
    title="My First FastAPI",
    description="ตัวอย่าง FastAPI application",
    version="1.0.0"
)


# Path operation (endpoint)
# @app.<http_method>("<path>")
@app.get("/")
def read_root():
    """หน้าแรก — ส่งคืน JSON อัตโนมัติ"""
    return {"message": "สวัสดี FastAPI!"}


@app.get("/hello/{name}")
def say_hello(name: str):
    """ทักทายด้วยชื่อ — ใช้ path parameter"""
    return {"message": f"สวัสดี {name}!"}
```

### รัน Application
```bash
# รัน development server (auto-reload)
uvicorn main:app --reload

# ระบุ host และ port
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### ดู Documentation
```
http://localhost:8000/docs       → Swagger UI (interactive)
http://localhost:8000/redoc      → ReDoc (readable)
http://localhost:8000/openapi.json → OpenAPI schema
```

---

## 4. Path Operations

```python
# path_operations.py

from fastapi import FastAPI

app = FastAPI()

# GET — ดึงข้อมูล
@app.get("/items")
def get_items():
    return [{"id": 1, "name": "Item 1"}, {"id": 2, "name": "Item 2"}]


# POST — สร้างข้อมูลใหม่
@app.post("/items")
def create_item():
    return {"message": "สร้าง item แล้ว"}


# PUT — แก้ไขข้อมูลทั้งหมด
@app.put("/items/{item_id}")
def update_item(item_id: int):
    return {"message": f"แก้ไข item {item_id} แล้ว"}


# PATCH — แก้ไขบางส่วน
@app.patch("/items/{item_id}")
def partial_update_item(item_id: int):
    return {"message": f"แก้ไขบางส่วนของ item {item_id} แล้ว"}


# DELETE — ลบข้อมูล
@app.delete("/items/{item_id}")
def delete_item(item_id: int):
    return {"message": f"ลบ item {item_id} แล้ว"}


# HEAD — เหมือน GET แต่ไม่ส่ง body
@app.head("/items")
def head_items():
    return None


# OPTIONS — ดู HTTP methods ที่รองรับ
@app.options("/items")
def options_items():
    return None
```

---

## 5. Pydantic Models (Request/Response Models)

```python
# models.py

from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List
from datetime import datetime


class UserBase(BaseModel):
    """Base model ที่ใช้ร่วมกัน"""
    username: str = Field(..., min_length=3, max_length=80, description="ชื่อผู้ใช้")
    email: EmailStr = Field(..., description="อีเมล")


class UserCreate(UserBase):
    """สำหรับสร้าง user (มี password)"""
    password: str = Field(..., min_length=8, description="รหัสผ่าน")


class UserUpdate(BaseModel):
    """สำหรับแก้ไข user (ทุก field optional)"""
    username: Optional[str] = Field(None, min_length=3, max_length=80)
    email: Optional[EmailStr] = None
    bio: Optional[str] = None


class UserResponse(UserBase):
    """สำหรับ response (ไม่มี password)"""
    id: int
    is_active: bool
    created_at: datetime
    
    class Config:
        # อนุญาตให้สร้าง model จาก ORM objects
        from_attributes = True  # Pydantic v2 (เดิม: orm_mode = True)


class ItemBase(BaseModel):
    name: str = Field(..., min_length=1, max_length=200)
    description: Optional[str] = None
    price: float = Field(..., gt=0, description="ราคา (ต้องมากกว่า 0)")
    tax: float = Field(default=0.0, ge=0)
    tags: List[str] = []


class ItemCreate(ItemBase):
    pass  # เหมือน ItemBase


class ItemResponse(ItemBase):
    id: int
    
    class Config:
        from_attributes = True
```

### ใช้ Models ใน Endpoints
```python
# main.py

from fastapi import FastAPI, HTTPException, status
from typing import List
from models import UserCreate, UserResponse, ItemCreate, ItemResponse

app = FastAPI()

# ข้อมูลตัวอย่าง
fake_users_db = {}
next_user_id = 1


@app.post(
    "/users",
    response_model=UserResponse,    # กำหนด response schema
    status_code=status.HTTP_201_CREATED,  # HTTP status code
    summary="สร้าง user ใหม่",
    description="สร้าง user ใหม่พร้อม username, email, และ password",
    tags=["Users"]                  # จัด group ใน Swagger
)
def create_user(user: UserCreate):
    """
    สร้าง user ใหม่:
    - **username**: ต้องมี 3-80 ตัวอักษร
    - **email**: ต้องเป็นรูปแบบ email ที่ถูกต้อง
    - **password**: ต้องมีอย่างน้อย 8 ตัวอักษร
    """
    global next_user_id
    
    # ตรวจสอบซ้ำ
    for u in fake_users_db.values():
        if u['username'] == user.username:
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="username นี้มีอยู่แล้ว"
            )
    
    # สร้าง user
    from datetime import datetime, timezone
    new_user = {
        'id': next_user_id,
        'username': user.username,
        'email': user.email,
        'password_hash': f'hashed_{user.password}',  # จริงๆ ควร hash
        'is_active': True,
        'created_at': datetime.now(timezone.utc)
    }
    fake_users_db[next_user_id] = new_user
    next_user_id += 1
    
    return new_user


@app.get(
    "/users",
    response_model=List[UserResponse],
    tags=["Users"]
)
def get_users():
    """ดึงรายการ users ทั้งหมด"""
    return list(fake_users_db.values())


@app.get(
    "/users/{user_id}",
    response_model=UserResponse,
    tags=["Users"]
)
def get_user(user_id: int):
    """ดึงข้อมูล user ตาม ID"""
    user = fake_users_db.get(user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"ไม่พบ user ID {user_id}"
        )
    return user
```

---

## 6. Async Endpoints

```python
# async_endpoints.py

from fastapi import FastAPI
import asyncio
import httpx

app = FastAPI()


# Async endpoint — ใช้ async def แทน def
@app.get("/async-hello")
async def async_hello():
    """Async endpoint"""
    await asyncio.sleep(0.1)  # simulate async operation
    return {"message": "Hello from async!"}


@app.get("/fetch-data")
async def fetch_external_data():
    """ดึงข้อมูลจาก external API แบบ async"""
    async with httpx.AsyncClient() as client:
        response = await client.get("https://jsonplaceholder.typicode.com/todos/1")
        data = response.json()
    return data


# เมื่อไหรใช้ async def vs def?
# - async def: เมื่อมี I/O operations (database, HTTP, file)
# - def: เมื่อเป็น CPU-bound operations (คำนวณ)
# - FastAPI รองรับทั้งสองแบบ

@app.get("/sync-endpoint")
def sync_endpoint():
    """Sync endpoint — ก็ใช้งานได้ปกติ"""
    # FastAPI จะรัน sync function ใน thread pool อัตโนมัติ
    return {"message": "Sync is fine too!"}
```

---

## 7. HTTP Exceptions

```python
# http_exceptions.py

from fastapi import FastAPI, HTTPException, status

app = FastAPI()


@app.get("/items/{item_id}")
def get_item(item_id: int):
    """ตัวอย่างการใช้ HTTP exceptions"""
    
    items = {1: "Apple", 2: "Banana", 3: "Cherry"}
    
    if item_id not in items:
        # สร้าง HTTP exception
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"ไม่พบ item ID {item_id}"
        )
    
    return {"id": item_id, "name": items[item_id]}


# Custom Exception Handler
from fastapi import Request
from fastapi.responses import JSONResponse


class CustomError(Exception):
    def __init__(self, message: str, code: int = 400):
        self.message = message
        self.code = code


@app.exception_handler(CustomError)
async def custom_error_handler(request: Request, exc: CustomError):
    return JSONResponse(
        status_code=exc.code,
        content={"error": exc.message, "status": exc.code}
    )


@app.get("/trigger-error")
def trigger_error():
    raise CustomError("นี่คือ custom error", code=422)


# HTTP Status Codes ที่ใช้บ่อย
STATUS_CODES = {
    200: "OK — สำเร็จ",
    201: "Created — สร้างแล้ว",
    204: "No Content — สำเร็จ ไม่มี body",
    400: "Bad Request — ข้อมูลไม่ถูกต้อง",
    401: "Unauthorized — ต้อง authenticate",
    403: "Forbidden — ไม่มีสิทธิ์",
    404: "Not Found — ไม่พบ",
    409: "Conflict — ข้อมูลซ้ำ",
    422: "Unprocessable Entity — validation error",
    500: "Internal Server Error — server error"
}
```

---

## 8. Response Models

```python
# response_models.py

from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional, Any


app = FastAPI()


class SuccessResponse(BaseModel):
    """Standard success response"""
    success: bool = True
    message: str
    data: Optional[Any] = None


class ErrorResponse(BaseModel):
    """Standard error response"""
    success: bool = False
    error: str
    detail: Optional[str] = None


class PaginatedResponse(BaseModel):
    """Paginated response"""
    items: list
    total: int
    page: int
    per_page: int
    pages: int


@app.get(
    "/products",
    response_model=PaginatedResponse,
    responses={
        200: {"description": "รายการสินค้า"},
        400: {"model": ErrorResponse, "description": "ข้อมูลไม่ถูกต้อง"}
    }
)
def get_products(page: int = 1, per_page: int = 10):
    """ดึงรายการสินค้าพร้อม pagination"""
    all_products = [
        {"id": i, "name": f"Product {i}"}
        for i in range(1, 51)  # 50 products
    ]
    
    # Paginate
    start = (page - 1) * per_page
    end = start + per_page
    items = all_products[start:end]
    total = len(all_products)
    
    return {
        "items": items,
        "total": total,
        "page": page,
        "per_page": per_page,
        "pages": (total + per_page - 1) // per_page
    }
```

---

## 9. Application Metadata และ Tags

```python
# app_metadata.py

from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

# กำหนด tags metadata
tags_metadata = [
    {
        "name": "Users",
        "description": "จัดการผู้ใช้",
    },
    {
        "name": "Posts",
        "description": "จัดการบทความ",
    },
    {
        "name": "Auth",
        "description": "Authentication และ Authorization",
    },
]

app = FastAPI(
    title="Blog API",
    description="""
# Blog API 📝

API สำหรับระบบ Blog พร้อม authentication

## Features
* **Users** — จัดการผู้ใช้
* **Posts** — จัดการบทความ
* **Authentication** — JWT-based auth
    """,
    version="1.0.0",
    terms_of_service="http://example.com/terms/",
    contact={
        "name": "API Support",
        "url": "http://example.com/support",
        "email": "support@example.com"
    },
    license_info={
        "name": "MIT",
        "url": "https://opensource.org/licenses/MIT"
    },
    openapi_tags=tags_metadata
)


@app.get("/users", tags=["Users"])
def get_users():
    return []


@app.get("/posts", tags=["Posts"])
def get_posts():
    return []
```

---

## 10. ตัวอย่าง Complete App

```python
# complete_app.py

from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List
from datetime import datetime, timezone

app = FastAPI(
    title="My Blog API",
    description="FastAPI Blog Application",
    version="1.0.0"
)


# Pydantic Models
class PostBase(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    content: str = Field(..., min_length=1)
    published: bool = False


class PostCreate(PostBase):
    pass


class PostResponse(PostBase):
    id: int
    created_at: datetime
    
    class Config:
        from_attributes = True


# ข้อมูลในหน่วยความจำ
posts_db: dict = {}
next_id = 1


@app.get("/", tags=["Root"])
def root():
    return {"message": "ยินดีต้อนรับสู่ Blog API", "version": "1.0.0"}


@app.get("/posts", response_model=List[PostResponse], tags=["Posts"])
def list_posts(published_only: bool = False):
    """ดึงรายการบทความ"""
    posts = list(posts_db.values())
    if published_only:
        posts = [p for p in posts if p['published']]
    return posts


@app.post("/posts", response_model=PostResponse,
          status_code=status.HTTP_201_CREATED, tags=["Posts"])
def create_post(post: PostCreate):
    """สร้างบทความใหม่"""
    global next_id
    new_post = {
        "id": next_id,
        "title": post.title,
        "content": post.content,
        "published": post.published,
        "created_at": datetime.now(timezone.utc)
    }
    posts_db[next_id] = new_post
    next_id += 1
    return new_post


@app.get("/posts/{post_id}", response_model=PostResponse, tags=["Posts"])
def get_post(post_id: int):
    """ดึงบทความตาม ID"""
    post = posts_db.get(post_id)
    if not post:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"ไม่พบบทความ ID {post_id}"
        )
    return post


@app.put("/posts/{post_id}", response_model=PostResponse, tags=["Posts"])
def update_post(post_id: int, post: PostCreate):
    """แก้ไขบทความ"""
    existing = posts_db.get(post_id)
    if not existing:
        raise HTTPException(status_code=404, detail="ไม่พบบทความ")
    
    existing.update(post.model_dump())
    posts_db[post_id] = existing
    return existing


@app.delete("/posts/{post_id}", status_code=status.HTTP_204_NO_CONTENT, tags=["Posts"])
def delete_post(post_id: int):
    """ลบบทความ"""
    if post_id not in posts_db:
        raise HTTPException(status_code=404, detail="ไม่พบบทความ")
    del posts_db[post_id]
    return None


if __name__ == "__main__":
    import uvicorn
    uvicorn.run("complete_app:app", host="0.0.0.0", port=8000, reload=True)
```

---

## 11. สรุป Part 086

✅ **FastAPI** เป็น modern web framework ที่ใช้ type hints  
✅ **uvicorn** เป็น ASGI server สำหรับรัน FastAPI  
✅ **Path operations** สร้างด้วย decorator `@app.get()`, `@app.post()` etc.  
✅ **Pydantic models** กำหนด structure ของ request/response data  
✅ **Automatic docs** ดูได้ที่ `/docs` (Swagger) และ `/redoc`  
✅ **HTTPException** สำหรับส่ง HTTP errors  
✅ **async def** รองรับ async operations  

---

## ➡️ ถัดไป: Part 087 - FastAPI Path Parameters and Query

*Part 086/100+ | Python Course - Beginner to World-Class*
