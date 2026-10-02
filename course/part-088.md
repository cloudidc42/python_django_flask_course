# Part 088: FastAPI Request Body
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- รับ JSON request body ด้วย Pydantic models
- สร้าง nested models
- ใช้ Field() สำหรับ validation
- รับ Form data
- อัปโหลดไฟล์

---

## 1. Request Body พื้นฐาน

```python
# request_body.py

from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()


class Item(BaseModel):
    """Model สำหรับรับ request body"""
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None


@app.post("/items")
def create_item(item: Item):
    """
    Request body (JSON):
    {
        "name": "Laptop",
        "description": "Gaming laptop",
        "price": 45000.0,
        "tax": 3150.0
    }
    
    FastAPI จะ:
    1. อ่าน request body เป็น JSON
    2. Convert ตาม type
    3. Validate ข้อมูล
    4. ส่งเป็น item parameter
    """
    # คำนวณราคารวม tax
    item_dict = item.model_dump()
    if item.tax:
        price_with_tax = item.price + item.tax
        item_dict.update({"price_with_tax": price_with_tax})
    return item_dict


@app.put("/items/{item_id}")
def update_item(item_id: int, item: Item):
    """
    ผสม path parameter + request body
    """
    return {"item_id": item_id, **item.model_dump()}
```

---

## 2. Field() Validation

```python
# field_validation.py

from fastapi import FastAPI
from pydantic import BaseModel, Field, field_validator, model_validator
from typing import Optional, List
from datetime import date

app = FastAPI()


class Product(BaseModel):
    """Product model พร้อม validation"""
    
    name: str = Field(
        ...,                      # required
        min_length=1,
        max_length=200,
        title="ชื่อสินค้า",
        description="ชื่อของสินค้า ความยาว 1-200 ตัวอักษร",
        examples=["MacBook Pro"]  # Pydantic v2
    )
    
    description: Optional[str] = Field(
        None,
        max_length=2000,
        description="คำอธิบายสินค้า"
    )
    
    price: float = Field(
        ...,
        gt=0,           # strictly greater than 0
        le=9999999.0,
        description="ราคา (บาท)"
    )
    
    discount_price: Optional[float] = Field(
        None,
        gt=0,
        description="ราคาส่วนลด (ต้องน้อยกว่าราคาปกติ)"
    )
    
    stock: int = Field(
        ...,
        ge=0,           # >= 0
        description="จำนวนในสต็อก"
    )
    
    weight_kg: Optional[float] = Field(
        None,
        gt=0,
        lt=1000,
        description="น้ำหนัก (กิโลกรัม)"
    )
    
    tags: List[str] = Field(
        default=[],
        max_length=10,   # ไม่เกิน 10 tags
        description="Tags ของสินค้า"
    )
    
    launch_date: Optional[date] = Field(
        None,
        description="วันที่เริ่มขาย"
    )
    
    # Pydantic v2 validators
    @field_validator('name')
    @classmethod
    def name_must_not_be_empty_spaces(cls, v):
        """ชื่อสินค้าต้องไม่เป็นแค่ spaces"""
        if not v.strip():
            raise ValueError('ชื่อสินค้าต้องไม่เป็น whitespace เท่านั้น')
        return v.strip()
    
    @model_validator(mode='after')
    def check_discount_less_than_price(self):
        """discount_price ต้องน้อยกว่า price"""
        if self.discount_price and self.discount_price >= self.price:
            raise ValueError('discount_price ต้องน้อยกว่า price')
        return self


@app.post("/products")
def create_product(product: Product):
    return product.model_dump()
```

---

## 3. Nested Models

```python
# nested_models.py

from fastapi import FastAPI
from pydantic import BaseModel, EmailStr
from typing import Optional, List
from datetime import datetime

app = FastAPI()


# ---- Nested Models ----

class Address(BaseModel):
    street: str
    city: str
    province: str
    postal_code: str = Field(..., regex=r'^\d{5}$')
    country: str = "Thailand"


class ContactInfo(BaseModel):
    email: EmailStr
    phone: Optional[str] = None
    website: Optional[str] = None


class SocialLinks(BaseModel):
    facebook: Optional[str] = None
    twitter: Optional[str] = None
    instagram: Optional[str] = None
    linkedin: Optional[str] = None


class UserProfile(BaseModel):
    """User profile ที่มี nested models"""
    
    # Basic info
    username: str
    display_name: str
    bio: Optional[str] = None
    
    # Nested models
    contact: ContactInfo
    address: Optional[Address] = None
    social: Optional[SocialLinks] = None
    
    # List of nested models
    skills: List[str] = []
    
    class Config:
        json_schema_extra = {
            "example": {
                "username": "john_doe",
                "display_name": "John Doe",
                "bio": "Python Developer",
                "contact": {
                    "email": "john@example.com",
                    "phone": "0812345678"
                },
                "address": {
                    "street": "123 Main St",
                    "city": "Bangkok",
                    "province": "Bangkok",
                    "postal_code": "10100"
                },
                "skills": ["Python", "FastAPI", "SQL"]
            }
        }


@app.post("/users/profile")
def create_profile(profile: UserProfile):
    """สร้าง user profile"""
    return {
        "message": "สร้าง profile สำเร็จ",
        "profile": profile.model_dump()
    }


# ---- Order System ----

class ProductItem(BaseModel):
    product_id: int
    quantity: int = Field(..., ge=1)
    unit_price: float = Field(..., gt=0)
    
    @property
    def subtotal(self) -> float:
        return self.quantity * self.unit_price


class PaymentMethod(BaseModel):
    type: str  # "credit_card", "promptpay", "cod"
    details: dict = {}


class Order(BaseModel):
    """Order ที่มี nested models หลายชั้น"""
    
    customer_name: str
    customer_email: EmailStr
    
    # List ของ nested models
    items: List[ProductItem]
    
    # Nested model
    shipping_address: Address
    payment: PaymentMethod
    
    notes: Optional[str] = None
    
    @property
    def total(self) -> float:
        return sum(item.unit_price * item.quantity for item in self.items)
    
    @field_validator('items')
    @classmethod
    def must_have_items(cls, v):
        if not v:
            raise ValueError('ต้องมีสินค้าอย่างน้อย 1 รายการ')
        return v


@app.post("/orders")
def create_order(order: Order):
    """สร้าง order"""
    order_dict = order.model_dump()
    order_dict['total'] = order.total
    order_dict['order_id'] = 'ORD-001'
    return order_dict
```

---

## 4. Multiple Request Bodies

```python
# multiple_bodies.py

from fastapi import FastAPI, Body
from pydantic import BaseModel
from typing import Optional

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float


class User(BaseModel):
    username: str
    email: str


# หลาย request body
@app.put("/items/{item_id}")
def update_item(
    item_id: int,
    item: Item,
    user: User,
    importance: int = Body(...)  # single value ใน body
):
    """
    Request body:
    {
        "item": {"name": "Laptop", "price": 45000},
        "user": {"username": "john", "email": "john@example.com"},
        "importance": 5
    }
    """
    return {
        "item_id": item_id,
        "item": item,
        "user": user,
        "importance": importance
    }


# Body() สำหรับ single values
@app.post("/review")
def create_review(
    item_id: int = Body(..., gt=0),
    rating: int = Body(..., ge=1, le=5),
    comment: Optional[str] = Body(None, max_length=500)
):
    """
    Request body:
    {
        "item_id": 1,
        "rating": 5,
        "comment": "ดีมาก!"
    }
    """
    return {"item_id": item_id, "rating": rating, "comment": comment}


# embed=True: ห่อ model ใน key ชื่อ model
@app.post("/items-embedded")
def create_item_embedded(item: Item = Body(..., embed=True)):
    """
    Request body (embedded):
    {
        "item": {
            "name": "Laptop",
            "price": 45000
        }
    }
    """
    return item
```

---

## 5. Form Data

```python
# form_data.py

from fastapi import FastAPI, Form, File, UploadFile, HTTPException
from typing import Optional, List
import os

app = FastAPI()


# รับ Form data
@app.post("/login")
def login(
    username: str = Form(...),
    password: str = Form(...)
):
    """
    Content-Type: application/x-www-form-urlencoded
    
    username=john&password=secret123
    """
    if username == "admin" and password == "admin123":
        return {"access_token": "fake-token", "token_type": "bearer"}
    raise HTTPException(401, "ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง")


@app.post("/register")
def register(
    username: str = Form(..., min_length=3),
    email: str = Form(...),
    password: str = Form(..., min_length=8),
    bio: Optional[str] = Form(None)
):
    """รับ Form data สำหรับสมัครสมาชิก"""
    return {
        "message": "สมัครสมาชิกสำเร็จ",
        "username": username,
        "email": email
    }
```

---

## 6. File Upload

```python
# file_upload.py

from fastapi import FastAPI, File, UploadFile, HTTPException
from typing import Optional, List
import os
import uuid
import shutil

app = FastAPI()

UPLOAD_DIR = "uploads"
os.makedirs(UPLOAD_DIR, exist_ok=True)

ALLOWED_IMAGE_TYPES = {"image/jpeg", "image/png", "image/gif", "image/webp"}
MAX_FILE_SIZE = 5 * 1024 * 1024  # 5 MB


@app.post("/upload/image")
async def upload_image(
    file: UploadFile = File(...),
    description: Optional[str] = None
):
    """อัปโหลดรูปภาพ"""
    
    # ตรวจสอบ content type
    if file.content_type not in ALLOWED_IMAGE_TYPES:
        raise HTTPException(
            400,
            f"ไม่รองรับ file type: {file.content_type}. รองรับ: {ALLOWED_IMAGE_TYPES}"
        )
    
    # อ่านไฟล์
    contents = await file.read()
    
    # ตรวจสอบขนาด
    if len(contents) > MAX_FILE_SIZE:
        raise HTTPException(
            400,
            f"ไฟล์ใหญ่เกินไป (max {MAX_FILE_SIZE // 1024 // 1024} MB)"
        )
    
    # สร้างชื่อไฟล์ unique
    ext = file.filename.split('.')[-1].lower() if '.' in file.filename else 'bin'
    unique_filename = f"{uuid.uuid4().hex}.{ext}"
    file_path = os.path.join(UPLOAD_DIR, unique_filename)
    
    # บันทึกไฟล์
    with open(file_path, "wb") as f:
        f.write(contents)
    
    return {
        "filename": unique_filename,
        "original_filename": file.filename,
        "content_type": file.content_type,
        "size_bytes": len(contents),
        "url": f"/static/{unique_filename}",
        "description": description
    }


@app.post("/upload/multiple")
async def upload_multiple_files(
    files: List[UploadFile] = File(...)
):
    """อัปโหลดหลายไฟล์พร้อมกัน"""
    
    results = []
    for file in files:
        contents = await file.read()
        
        ext = file.filename.split('.')[-1].lower() if '.' in file.filename else 'bin'
        unique_filename = f"{uuid.uuid4().hex}.{ext}"
        file_path = os.path.join(UPLOAD_DIR, unique_filename)
        
        with open(file_path, "wb") as f:
            f.write(contents)
        
        results.append({
            "filename": unique_filename,
            "original_filename": file.filename,
            "size": len(contents)
        })
    
    return {"uploaded": len(results), "files": results}


@app.post("/upload/profile")
async def upload_profile(
    username: str = Form(...),
    bio: Optional[str] = Form(None),
    avatar: Optional[UploadFile] = File(None)
):
    """ผสม Form data + File upload"""
    
    avatar_url = None
    
    if avatar:
        if avatar.content_type not in ALLOWED_IMAGE_TYPES:
            raise HTTPException(400, "ต้องเป็นรูปภาพเท่านั้น")
        
        contents = await avatar.read()
        ext = avatar.filename.split('.')[-1].lower()
        filename = f"{uuid.uuid4().hex}.{ext}"
        
        with open(os.path.join(UPLOAD_DIR, filename), "wb") as f:
            f.write(contents)
        
        avatar_url = f"/static/{filename}"
    
    return {
        "username": username,
        "bio": bio,
        "avatar_url": avatar_url,
        "message": "อัปเดต profile สำเร็จ"
    }
```

---

## 7. สรุป Part 088

✅ **Request body** รับด้วย Pydantic model parameter  
✅ **Field()** ใช้กำหนด validation rules เพิ่มเติม  
✅ **Nested models** รองรับ models ซ้อนกัน  
✅ **field_validator** สำหรับ custom field validation  
✅ **model_validator** สำหรับ validate ระหว่าง fields  
✅ **Form data** รับด้วย `Form(...)` (ต้อง import)  
✅ **File upload** รับด้วย `UploadFile` parameter  

---

## ➡️ ถัดไป: Part 089 - FastAPI Dependencies

*Part 088/100+ | Python Course - Beginner to World-Class*
