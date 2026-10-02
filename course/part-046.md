# Part 046: API Design Principles
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ REST API design principles
- URL naming conventions
- HTTP methods และ status codes
- Pagination, Filtering, Sorting
- Versioning
- Error responses
- API Documentation

---

## 1. REST API Principles

```
REST = Representational State Transfer
6 Constraints:
1. Client-Server
2. Stateless
3. Cacheable
4. Uniform Interface
5. Layered System
6. Code on Demand (optional)
```

---

## 2. URL Design

```
Resource-based URLs (ไม่ใช่ action-based):

✅ ดี:
GET    /users              - ดูรายชื่อ users
POST   /users              - สร้าง user ใหม่
GET    /users/123          - ดู user id=123
PUT    /users/123          - แก้ไข user ทั้งหมด
PATCH  /users/123          - แก้ไข user บางส่วน
DELETE /users/123          - ลบ user

GET    /users/123/posts    - posts ของ user 123
GET    /posts/456/comments - comments ของ post 456

❌ ไม่ดี:
GET  /getUsers
POST /createUser
GET  /deleteUser?id=123
POST /users/updateUser/123
```

---

## 3. HTTP Status Codes

```python
# 2xx Success
200 OK                  - GET, PUT, PATCH สำเร็จ
201 Created             - POST สร้างสำเร็จ (ส่ง Location header)
204 No Content          - DELETE สำเร็จ ไม่มี body

# 3xx Redirect
301 Moved Permanently   - redirect ถาวร
302 Found               - redirect ชั่วคราว

# 4xx Client Error
400 Bad Request         - request ผิดรูปแบบ/ข้อมูลไม่ถูกต้อง
401 Unauthorized        - ยังไม่ได้ authentication
403 Forbidden           - authenticated แต่ไม่มีสิทธิ์
404 Not Found           - ไม่พบ resource
409 Conflict            - conflict เช่น email ซ้ำ
422 Unprocessable Entity - validation error (FastAPI ใช้นี้)
429 Too Many Requests   - rate limit exceeded

# 5xx Server Error
500 Internal Server Error - server crash
502 Bad Gateway         - upstream error
503 Service Unavailable - server overload/maintenance
```

---

## 4. Response Format ที่ดี

```python
# Standard error response
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Any, Optional

class APIResponse(BaseModel):
    success: bool
    message: str
    data: Optional[Any] = None

class ErrorResponse(BaseModel):
    success: bool = False
    error: str
    details: Optional[Any] = None
    code: Optional[str] = None  # error code สำหรับ client

# FastAPI example
app = FastAPI()

@app.get("/users/{user_id}", response_model=APIResponse)
async def get_user(user_id: int):
    user = await fetch_user(user_id)
    if not user:
        raise HTTPException(
            status_code=404,
            detail={
                "success": False,
                "error": "User not found",
                "code": "USER_NOT_FOUND"
            }
        )
    return {
        "success": True,
        "message": "User retrieved successfully",
        "data": user
    }

# Error handler
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc):
    return JSONResponse(
        status_code=422,
        content={
            "success": False,
            "error": "Validation error",
            "details": exc.errors(),
            "code": "VALIDATION_ERROR"
        }
    )
```

---

## 5. Pagination

```python
from fastapi import FastAPI, Query
from pydantic import BaseModel
from typing import Generic, TypeVar, List

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: List[T]
    total: int
    page: int
    per_page: int
    pages: int
    has_next: bool
    has_prev: bool

def paginate(query, page: int, per_page: int):
    total = query.count()
    items = query.offset((page-1)*per_page).limit(per_page).all()
    pages = (total + per_page - 1) // per_page
    
    return {
        "items": items,
        "total": total,
        "page": page,
        "per_page": per_page,
        "pages": pages,
        "has_next": page < pages,
        "has_prev": page > 1,
    }

@app.get("/posts/")
async def list_posts(
    page: int = Query(default=1, ge=1),
    per_page: int = Query(default=10, ge=1, le=100),
    sort_by: str = Query(default="created_at"),
    order: str = Query(default="desc", regex="^(asc|desc)$"),
    search: str = Query(default=None),
):
    query = db.query(Post)
    
    if search:
        query = query.filter(Post.title.ilike(f"%{search}%"))
    
    if order == "desc":
        query = query.order_by(desc(getattr(Post, sort_by, Post.created_at)))
    else:
        query = query.order_by(asc(getattr(Post, sort_by, Post.created_at)))
    
    return paginate(query, page, per_page)
```

---

## 6. API Versioning

```python
# URL versioning (ชัดเจนที่สุด)
# /api/v1/users
# /api/v2/users

from fastapi import FastAPI
from fastapi import APIRouter

app = FastAPI()

# v1 router
v1_router = APIRouter(prefix="/api/v1")

@v1_router.get("/users")
async def get_users_v1():
    return {"version": "v1", "users": []}

# v2 router
v2_router = APIRouter(prefix="/api/v2")

@v2_router.get("/users")
async def get_users_v2():
    return {"version": "v2", "users": [], "total": 0, "page": 1}

app.include_router(v1_router)
app.include_router(v2_router)
```

---

## 7. Filtering

```python
@app.get("/products/")
async def list_products(
    category: Optional[str] = None,
    min_price: Optional[float] = None,
    max_price: Optional[float] = None,
    in_stock: Optional[bool] = None,
    brand: Optional[List[str]] = Query(default=None),  # ?brand=Nike&brand=Adidas
):
    filters = {}
    if category:
        filters["category"] = category
    if in_stock is not None:
        filters["in_stock"] = in_stock
    
    # Build dynamic query
    query = db.query(Product)
    
    if category:
        query = query.filter(Product.category == category)
    if min_price:
        query = query.filter(Product.price >= min_price)
    if max_price:
        query = query.filter(Product.price <= max_price)
    if in_stock is not None:
        query = query.filter(Product.in_stock == in_stock)
    if brand:
        query = query.filter(Product.brand.in_(brand))
    
    return query.all()
```

---

## 8. API Documentation (OpenAPI/Swagger)

```python
from fastapi import FastAPI

app = FastAPI(
    title="My API",
    description="""
## My Awesome API

This API provides access to:
* **Users** - manage user accounts
* **Posts** - blog posts
* **Comments** - post comments

### Authentication
Use Bearer token in Authorization header:
```
Authorization: Bearer <your-token>
```
    """,
    version="1.0.0",
    terms_of_service="https://example.com/terms",
    contact={
        "name": "API Support",
        "url": "https://example.com/support",
        "email": "api@example.com",
    },
    license_info={
        "name": "MIT",
        "url": "https://opensource.org/licenses/MIT",
    },
    docs_url="/docs",       # Swagger UI
    redoc_url="/redoc",     # ReDoc
    openapi_url="/openapi.json",
)

# OpenAPI URL
# http://localhost:8000/docs      - Swagger UI
# http://localhost:8000/redoc     - ReDoc
# http://localhost:8000/openapi.json - OpenAPI spec
```

---

## 9. สรุป Part 046

✅ **RESTful URLs** - resource-based, ไม่ใช่ action-based  
✅ **HTTP Methods** - GET/POST/PUT/PATCH/DELETE ที่ถูกต้อง  
✅ **Status Codes** - 2xx/4xx/5xx ที่ถูกต้อง  
✅ **Response Format** - standard success/error format  
✅ **Pagination** - page-based, cursor-based  
✅ **Filtering/Sorting** - query parameters  
✅ **Versioning** - /api/v1, /api/v2  
✅ **Documentation** - OpenAPI/Swagger  

---

## ➡️ ถัดไป: Part 047 - HTTP และ Requests Library

*Part 046/100+ | Python Course - Beginner to World-Class*
