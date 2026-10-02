# Part 092 - FastAPI Request and Response

## เป้าหมายการเรียนรู้

- จัดการ Path parameters และ Query parameters
- ใช้ Request Body ด้วย Pydantic models
- กำหนด Response models และ status codes
- อัปโหลดไฟล์
- รับ Form data
- จัดการ Headers และ Cookies

---

## 1. Path Parameters

```python
from fastapi import FastAPI, Path, HTTPException
from typing import Optional

app = FastAPI()

# ─────────────────────────────────────────
# Basic Path Parameter
# ─────────────────────────────────────────

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    """item_id แปลงเป็น int อัตโนมัติ"""
    return {"item_id": item_id}


# หลาย path parameters
@app.get("/users/{user_id}/posts/{post_id}")
async def get_user_post(user_id: int, post_id: int):
    return {"user_id": user_id, "post_id": post_id}


# ─────────────────────────────────────────
# Path Parameter พร้อม Validation
# ─────────────────────────────────────────

@app.get("/products/{product_id}")
async def get_product(
    product_id: int = Path(
        title="Product ID",
        description="ID ของสินค้า (ต้องมากกว่า 0)",
        gt=0,         # greater than
        le=9999999,   # less than or equal
        example=1
    )
):
    return {"product_id": product_id}


# String path parameter
@app.get("/users/{username}")
async def get_user_by_username(
    username: str = Path(
        min_length=3,
        max_length=50,
        pattern=r'^[a-zA-Z0-9_]+$',  # alphanumeric + underscore
        example="alice"
    )
):
    return {"username": username}


# ─────────────────────────────────────────
# Enum path parameter
# ─────────────────────────────────────────

from enum import Enum

class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"


@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    """Path parameter ที่เป็น Enum"""
    if model_name == ModelName.alexnet:
        return {"model": model_name, "message": "Deep Learning FTW!"}
    if model_name.value == "lenet":
        return {"model": model_name, "message": "LeCNN all the images"}
    return {"model": model_name, "message": "Have some residuals"}


# ─────────────────────────────────────────
# File path parameter
# ─────────────────────────────────────────

@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    """:path รับ / ในชื่อไฟล์ได้"""
    # /files/uploads/2024/image.png -> file_path = "uploads/2024/image.png"
    return {"file_path": file_path}
```

---

## 2. Query Parameters

```python
from fastapi import FastAPI, Query
from typing import Optional, List, Annotated

app = FastAPI()


# ─────────────────────────────────────────
# Basic Query Parameters
# ─────────────────────────────────────────

@app.get("/items")
async def list_items(
    skip: int = 0,           # default 0
    limit: int = 10,         # default 10
    active: bool = True,     # default True, รับ "true", "1", "yes"
    name: Optional[str] = None  # optional
):
    """
    GET /items?skip=0&limit=10&active=true&name=test
    """
    return {
        "skip": skip,
        "limit": limit,
        "active": active,
        "name": name
    }


# ─────────────────────────────────────────
# Query Parameter พร้อม Validation
# ─────────────────────────────────────────

@app.get("/search")
async def search(
    q: Annotated[
        Optional[str],
        Query(
            title="Query string",
            description="คำค้นหา",
            min_length=2,
            max_length=100,
            example="python"
        )
    ] = None,
    page: Annotated[int, Query(ge=1, le=1000, example=1)] = 1,
    per_page: Annotated[int, Query(ge=1, le=100, example=20)] = 20,
    sort: Annotated[
        Optional[str],
        Query(pattern=r'^(asc|desc)$')
    ] = "desc",
):
    return {
        "query": q,
        "page": page,
        "per_page": per_page,
        "sort": sort
    }


# ─────────────────────────────────────────
# Multiple values สำหรับ key เดียวกัน
# ─────────────────────────────────────────

@app.get("/filter")
async def filter_items(
    # GET /filter?tags=python&tags=flask&tags=web
    tags: Optional[List[str]] = Query(None, description="หลาย tags"),
    
    # GET /filter?ids=1&ids=2&ids=3
    ids: Optional[List[int]] = Query(None),
):
    return {"tags": tags, "ids": ids}


# ─────────────────────────────────────────
# Required vs Optional
# ─────────────────────────────────────────

@app.get("/products")
async def get_products(
    category: str,              # Required (ไม่มี default)
    brand: Optional[str] = None,  # Optional
    min_price: float = 0.0,    # Optional with default
):
    return {
        "category": category,
        "brand": brand,
        "min_price": min_price
    }
```

---

## 3. Request Body

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List
from datetime import date

app = FastAPI()


# ─────────────────────────────────────────
# Basic Request Body
# ─────────────────────────────────────────

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None


@app.post("/items")
async def create_item(item: Item):
    """FastAPI validate JSON body อัตโนมัติ"""
    item_dict = item.dict()
    
    if item.tax is not None:
        price_with_tax = item.price + item.tax
        item_dict.update({"price_with_tax": price_with_tax})
    
    return item_dict


# ─────────────────────────────────────────
# Body พร้อม Field validation
# ─────────────────────────────────────────

class UserCreate(BaseModel):
    username: str = Field(
        ...,
        min_length=3,
        max_length=50,
        pattern=r'^[a-zA-Z0-9_]+$',
        example="alice123"
    )
    email: str = Field(..., example="alice@example.com")
    password: str = Field(..., min_length=8, example="secretpass")
    full_name: Optional[str] = Field(None, max_length=100)
    age: Optional[int] = Field(None, ge=0, le=120)
    birth_date: Optional[date] = None
    interests: List[str] = Field(default_factory=list)
    
    class Config:
        # ตัวอย่างสำหรับ docs
        schema_extra = {
            "example": {
                "username": "alice123",
                "email": "alice@example.com",
                "password": "securepass",
                "full_name": "Alice Smith",
                "age": 25
            }
        }


@app.post("/users")
async def create_user(user: UserCreate):
    # user.dict() แปลงเป็น dict
    user_data = user.dict()
    # ลบ password ก่อน return (ไม่ควรส่ง password กลับ)
    user_data.pop('password', None)
    return {"id": 1, **user_data}


# ─────────────────────────────────────────
# Nested Models
# ─────────────────────────────────────────

class Address(BaseModel):
    street: str
    city: str
    country: str = "Thailand"
    zipcode: Optional[str] = None


class OrderItem(BaseModel):
    product_id: int
    quantity: int = Field(gt=0)
    unit_price: float = Field(gt=0)


class Order(BaseModel):
    customer_name: str
    customer_email: str
    shipping_address: Address
    items: List[OrderItem] = Field(min_items=1)
    notes: Optional[str] = None


@app.post("/orders", status_code=201)
async def create_order(order: Order):
    """Nested model validation"""
    total = sum(item.quantity * item.unit_price for item in order.items)
    
    return {
        "order_id": "ORD-001",
        "customer": order.customer_name,
        "shipping_to": f"{order.shipping_address.city}, {order.shipping_address.country}",
        "items_count": len(order.items),
        "total": round(total, 2),
        "status": "pending"
    }


# ─────────────────────────────────────────
# Path + Body + Query ร่วมกัน
# ─────────────────────────────────────────

class ItemUpdate(BaseModel):
    name: Optional[str] = None
    price: Optional[float] = Field(None, gt=0)
    description: Optional[str] = None


@app.put("/items/{item_id}")
async def update_item(
    item_id: int,           # path parameter
    item: ItemUpdate,       # request body
    notify: bool = False,   # query parameter
):
    """ใช้ทั้ง path, body, และ query ร่วมกัน"""
    return {
        "item_id": item_id,
        "item": item.dict(exclude_unset=True),
        "notify": notify
    }
```

---

## 4. Response Models

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse, HTMLResponse, RedirectResponse
from pydantic import BaseModel
from typing import Optional, List

app = FastAPI()


# ─────────────────────────────────────────
# Response Model
# ─────────────────────────────────────────

class UserCreate(BaseModel):
    username: str
    email: str
    password: str  # รับ input นี้


class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    # ไม่มี password field - จะไม่ถูก return


@app.post(
    "/users",
    response_model=UserResponse,  # กำหนด response schema
    status_code=201,
    summary="สร้าง user ใหม่",
    description="สร้าง user และส่งข้อมูลกลับโดยไม่มี password"
)
async def create_user(user: UserCreate):
    """FastAPI จะ filter response ตาม UserResponse model"""
    # return ทั้งหมดรวม password
    return {
        "id": 1,
        "username": user.username,
        "email": user.email,
        "password": user.password,  # จะถูก filter ออก!
        "internal_data": "hidden"   # จะถูก filter ออก!
    }


# ─────────────────────────────────────────
# Response Model Options
# ─────────────────────────────────────────

class Item(BaseModel):
    name: str
    price: float
    description: Optional[str] = None
    internal_id: Optional[str] = None


@app.get(
    "/items/{item_id}",
    response_model=Item,
    response_model_exclude={"internal_id"},      # ซ่อน specific fields
    response_model_exclude_none=True,             # ซ่อน None fields
    response_model_exclude_unset=True,            # ซ่อน fields ที่ไม่ได้ set
)
async def get_item(item_id: int):
    return {
        "name": "Notebook",
        "price": 59.0,
        "description": None,    # จะถูกซ่อน (exclude_none=True)
        "internal_id": "INT-001"  # จะถูกซ่อน (exclude)
    }


# ─────────────────────────────────────────
# Response Status Codes
# ─────────────────────────────────────────

from fastapi import status

@app.post("/items", status_code=status.HTTP_201_CREATED)
async def create_item():
    return {"id": 1}

@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: int):
    return None  # 204 ไม่ส่งข้อมูลกลับ


# ─────────────────────────────────────────
# Custom Response Types
# ─────────────────────────────────────────

@app.get("/html", response_class=HTMLResponse)
async def html_page():
    """ส่ง HTML response"""
    return """
    <html>
    <body>
        <h1>Hello from FastAPI!</h1>
    </body>
    </html>
    """


@app.get("/redirect")
async def redirect():
    """Redirect"""
    return RedirectResponse(url="/docs", status_code=302)


@app.get("/custom-response")
async def custom_response():
    """Custom JSONResponse พร้อม custom headers"""
    data = {"message": "Hello!"}
    return JSONResponse(
        content=data,
        status_code=200,
        headers={
            "X-Custom-Header": "my-value",
            "Cache-Control": "no-cache"
        }
    )
```

---

## 5. File Upload

```python
from fastapi import FastAPI, File, UploadFile, HTTPException
from fastapi.responses import JSONResponse
from typing import List
import os
import uuid
import aiofiles

app = FastAPI()

UPLOAD_DIR = "uploads"
ALLOWED_IMAGE_TYPES = {"image/jpeg", "image/png", "image/gif", "image/webp"}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10 MB


# ─────────────────────────────────────────
# Single File Upload
# ─────────────────────────────────────────

@app.post("/upload/image")
async def upload_image(
    file: UploadFile = File(..., description="ไฟล์รูปภาพ")
):
    """อัปโหลดรูปภาพ"""
    # ตรวจสอบ content type
    if file.content_type not in ALLOWED_IMAGE_TYPES:
        raise HTTPException(
            status_code=400,
            detail=f"ไม่รองรับไฟล์ประเภท {file.content_type}"
        )
    
    # อ่านไฟล์
    contents = await file.read()
    
    # ตรวจสอบขนาดไฟล์
    if len(contents) > MAX_FILE_SIZE:
        raise HTTPException(
            status_code=413,
            detail=f"ไฟล์ใหญ่เกิน {MAX_FILE_SIZE // (1024*1024)} MB"
        )
    
    # สร้างชื่อไฟล์ unique
    ext = os.path.splitext(file.filename)[1].lower()
    unique_filename = f"{uuid.uuid4().hex}{ext}"
    
    # บันทึกไฟล์ (async)
    os.makedirs(UPLOAD_DIR, exist_ok=True)
    filepath = os.path.join(UPLOAD_DIR, unique_filename)
    
    async with aiofiles.open(filepath, 'wb') as f:
        await f.write(contents)
    
    return {
        "filename": unique_filename,
        "original_name": file.filename,
        "content_type": file.content_type,
        "size": len(contents),
        "url": f"/static/uploads/{unique_filename}"
    }


# ─────────────────────────────────────────
# Multiple Files Upload
# ─────────────────────────────────────────

@app.post("/upload/multiple")
async def upload_multiple(
    files: List[UploadFile] = File(..., description="หลายไฟล์"),
):
    """อัปโหลดหลายไฟล์"""
    if len(files) > 10:
        raise HTTPException(status_code=400, detail="อัปโหลดได้สูงสุด 10 ไฟล์")
    
    uploaded = []
    errors = []
    
    for file in files:
        try:
            if file.content_type not in ALLOWED_IMAGE_TYPES:
                errors.append({"file": file.filename, "error": "ประเภทไฟล์ไม่รองรับ"})
                continue
            
            contents = await file.read()
            ext = os.path.splitext(file.filename)[1].lower()
            unique_name = f"{uuid.uuid4().hex}{ext}"
            
            filepath = os.path.join(UPLOAD_DIR, unique_name)
            async with aiofiles.open(filepath, 'wb') as f:
                await f.write(contents)
            
            uploaded.append({
                "filename": unique_name,
                "original_name": file.filename,
                "size": len(contents)
            })
        except Exception as e:
            errors.append({"file": file.filename, "error": str(e)})
    
    return {
        "uploaded": uploaded,
        "errors": errors,
        "total_uploaded": len(uploaded)
    }
```

---

## 6. Form Data

```python
from fastapi import FastAPI, Form, File, UploadFile
from typing import Optional

app = FastAPI()


# ─────────────────────────────────────────
# Basic Form
# ─────────────────────────────────────────

@app.post("/login")
async def login(
    username: str = Form(...),
    password: str = Form(...)
):
    """รับ Form data (application/x-www-form-urlencoded)"""
    # ตรวจสอบ username/password
    if username == "admin" and password == "secret":
        return {"access_token": "fake-token", "token_type": "bearer"}
    
    raise HTTPException(status_code=401, detail="ข้อมูลไม่ถูกต้อง")


# ─────────────────────────────────────────
# Form + File Upload ร่วมกัน
# ─────────────────────────────────────────

@app.post("/profile/update")
async def update_profile(
    # Form fields
    full_name: str = Form(...),
    bio: Optional[str] = Form(None),
    website: Optional[str] = Form(None),
    
    # File (optional)
    avatar: Optional[UploadFile] = File(None),
):
    """อัปเดตโปรไฟล์พร้อม avatar"""
    result = {
        "full_name": full_name,
        "bio": bio,
        "website": website,
    }
    
    if avatar:
        # บันทึก avatar
        contents = await avatar.read()
        result["avatar_filename"] = avatar.filename
        result["avatar_size"] = len(contents)
    
    return result
```

---

## 7. Headers และ Cookies

```python
from fastapi import FastAPI, Header, Cookie, Response
from typing import Optional

app = FastAPI()


# ─────────────────────────────────────────
# Request Headers
# ─────────────────────────────────────────

@app.get("/items")
async def get_items(
    # Header parameter (FastAPI แปลง - เป็น _ อัตโนมัติ)
    # x-api-key -> x_api_key
    x_api_key: Optional[str] = Header(None, alias="X-API-Key"),
    user_agent: Optional[str] = Header(None),
    accept_language: Optional[str] = Header(None),
    authorization: Optional[str] = Header(None)
):
    """รับ request headers"""
    return {
        "api_key": x_api_key,
        "user_agent": user_agent,
        "language": accept_language
    }


# Require specific header
@app.get("/secure")
async def secure_endpoint(
    api_key: str = Header(..., alias="X-API-Key")
):
    """ต้องมี X-API-Key header"""
    if api_key != "valid-key":
        raise HTTPException(status_code=403, detail="Invalid API key")
    return {"data": "secret"}


# ─────────────────────────────────────────
# Response Headers
# ─────────────────────────────────────────

@app.get("/download")
async def download(response: Response):
    """ตั้งค่า response headers"""
    response.headers["Content-Disposition"] = 'attachment; filename="data.json"'
    response.headers["X-Custom"] = "my-value"
    response.headers["Cache-Control"] = "max-age=3600"
    return {"data": "file content"}


# ─────────────────────────────────────────
# Cookies
# ─────────────────────────────────────────

@app.post("/set-cookie")
async def set_cookie(response: Response):
    """ตั้งค่า cookie"""
    response.set_cookie(
        key="session_id",
        value="abc123",
        max_age=3600,
        httponly=True,
        secure=True,
        samesite="lax"
    )
    return {"message": "Cookie set!"}


@app.get("/read-cookie")
async def read_cookie(
    session_id: Optional[str] = Cookie(None),
    user_pref: Optional[str] = Cookie(None)
):
    """อ่าน cookies"""
    return {
        "session_id": session_id,
        "user_pref": user_pref
    }


@app.delete("/clear-cookie")
async def clear_cookie(response: Response):
    """ลบ cookie"""
    response.delete_cookie(key="session_id")
    return {"message": "Cookie deleted"}
```

---

## 8. Request Object

```python
from fastapi import FastAPI, Request

app = FastAPI()


@app.get("/request-info")
async def request_info(request: Request):
    """ดูข้อมูล request ทั้งหมด"""
    return {
        "method": request.method,
        "url": str(request.url),
        "path": request.url.path,
        "query_params": dict(request.query_params),
        "headers": dict(request.headers),
        "client_host": request.client.host,
        "client_port": request.client.port,
    }


@app.post("/raw-body")
async def raw_body(request: Request):
    """รับ raw request body"""
    body = await request.body()
    json_data = await request.json()
    form_data = await request.form()
    
    return {
        "body_length": len(body),
        "json": json_data,
        "form": dict(form_data)
    }
```

---

## Exercises

### Exercise 1: Product Catalog API
สร้าง API ที่รับ:
- Path: `/products/{category}/{product_id}`
- Query: `?sort=price&order=asc&in_stock=true`
- Body: `{name, price, description, tags, images[]}`
- Header: `X-Store-ID`

### Exercise 2: Image Upload Service
สร้าง image upload service:
- อัปโหลด image + metadata (form + file)
- ตรวจสอบ file type และ size
- Resize image ด้วย Pillow
- ส่ง URL กลับ

### Exercise 3: User Registration with Avatar
สร้าง registration endpoint ที่รับ:
- Form data: username, email, password, bio
- File: avatar image (optional)
- Validate ทุก fields
- Return user data (ไม่มี password)

---

## สรุป

สิ่งที่เรียนรู้ใน Part นี้:
- **Path Parameters** พร้อม validation (gt, le, pattern)
- **Query Parameters** optional/required พร้อม defaults
- **Request Body** ด้วย Pydantic models
- **Response Models** filter sensitive data
- **File Upload** single และ multiple files
- **Form Data** พร้อมและไม่พร้อม files
- **Headers และ Cookies** อ่านและเขียน

---

## ลิงก์ Part ถัดไป

➡️ [Part 093 - FastAPI Pydantic Validation](./part-093.md)
