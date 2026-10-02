# Part 088: FastAPI Request Body and Validation
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ Pydantic BaseModel สร้าง request body
- สร้าง nested models สำหรับข้อมูลซับซ้อน
- Validate ข้อมูลด้วย Field()
- รับ request body ร่วมกับ path และ query params
- รับ form data ด้วย Form()
- อัปโหลดไฟล์ด้วย File() และ UploadFile
- อัปโหลดหลายไฟล์พร้อมกัน
- ใช้ model_config สำหรับ extra fields
- กำหนด response model และ response_model_exclude

---

## 1. Pydantic BaseModel สำหรับ Request Body

เมื่อ client ส่งข้อมูล JSON มาใน request body เราใช้ Pydantic `BaseModel` ในการ define โครงสร้างและ validate อัตโนมัติ

```python
# main.py - Request Body พื้นฐาน

from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()


# กำหนดโครงสร้าง request body ด้วย BaseModel
class Item(BaseModel):
    name: str                      # จำเป็นต้องมี (required)
    description: Optional[str] = None  # ไม่จำเป็น (optional)
    price: float                   # จำเป็นต้องมี
    tax: Optional[float] = None    # ไม่จำเป็น default = None


# POST endpoint รับ request body
@app.post("/items/")
def create_item(item: Item):
    """
    FastAPI จะ:
    1. อ่าน request body เป็น JSON
    2. Convert เป็น Item object
    3. Validate ว่า name และ price มีค่า
    4. คืน error 422 ถ้า validation ไม่ผ่าน
    """
    # เข้าถึง field ได้เหมือน attribute ปกติ
    item_data = item.dict()

    # คำนวณราคารวม
    if item.tax:
        price_with_tax = item.price + item.tax
        item_data["price_with_tax"] = price_with_tax

    return item_data


# PUT endpoint สำหรับ update
@app.put("/items/{item_id}")
def update_item(item_id: int, item: Item):
    """อัปเดต item ตาม id"""
    return {"item_id": item_id, **item.dict()}
```

ตัวอย่าง request:
```bash
# สร้าง item ใหม่
curl -X POST "http://localhost:8000/items/" \
     -H "Content-Type: application/json" \
     -d '{"name": "มะม่วง", "price": 50.0, "tax": 5.0}'

# Response:
# {
#   "name": "มะม่วง",
#   "description": null,
#   "price": 50.0,
#   "tax": 5.0,
#   "price_with_tax": 55.0
# }
```

---

## 2. Nested Models (โมเดลซ้อนกัน)

ใน real-world applications ข้อมูลมักมีโครงสร้างซับซ้อน เช่น สินค้ามีที่อยู่จัดส่ง

```python
# nested_models.py

from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional, List, Set

app = FastAPI()


# โมเดลสำหรับที่อยู่
class Address(BaseModel):
    street: str        # ถนน
    city: str          # เมือง
    province: str      # จังหวัด
    postal_code: str   # รหัสไปรษณีย์


# โมเดลสำหรับรายการสินค้าใน order
class OrderItem(BaseModel):
    product_id: int    # รหัสสินค้า
    name: str          # ชื่อสินค้า
    quantity: int      # จำนวน
    price: float       # ราคาต่อชิ้น


# โมเดลหลักสำหรับ order ที่มี nested models
class Order(BaseModel):
    customer_name: str              # ชื่อลูกค้า
    customer_email: str             # อีเมล
    shipping_address: Address       # ที่อยู่จัดส่ง (nested model)
    items: List[OrderItem]          # รายการสินค้า (list of nested models)
    notes: Optional[str] = None     # หมายเหตุ


@app.post("/orders/")
def create_order(order: Order):
    """สร้าง order ใหม่พร้อมที่อยู่และรายการสินค้า"""

    # คำนวณราคารวม
    total_price = sum(item.price * item.quantity for item in order.items)

    return {
        "message": "สร้าง order สำเร็จ",
        "customer": order.customer_name,
        "shipping_to": f"{order.shipping_address.city}, {order.shipping_address.province}",
        "item_count": len(order.items),
        "total_price": total_price,
        "order_detail": order.dict()
    }


# โมเดลที่มี List และ Set
class Product(BaseModel):
    name: str
    tags: List[str] = []         # list of strings
    images: Set[str] = set()     # set (ไม่มีซ้ำ)
    attributes: dict = {}        # dict อิสระ


@app.post("/products/")
def create_product(product: Product):
    """สร้างสินค้าพร้อม tags และ images"""
    return {
        "name": product.name,
        "tags": product.tags,
        "unique_images": list(product.images),
        "attributes": product.attributes
    }
```

ตัวอย่าง request สำหรับ nested model:
```bash
curl -X POST "http://localhost:8000/orders/" \
     -H "Content-Type: application/json" \
     -d '{
       "customer_name": "สมชาย ใจดี",
       "customer_email": "somchai@example.com",
       "shipping_address": {
         "street": "123 ถนนสุขุมวิท",
         "city": "กรุงเทพ",
         "province": "กรุงเทพมหานคร",
         "postal_code": "10110"
       },
       "items": [
         {"product_id": 1, "name": "เสื้อยืด", "quantity": 2, "price": 299.0},
         {"product_id": 2, "name": "กางเกง", "quantity": 1, "price": 599.0}
       ]
     }'
```

---

## 3. Field Validation ด้วย Field()

`Field()` ใช้สำหรับกำหนด validation rules และ metadata ให้กับแต่ละ field

```python
# field_validation.py

from fastapi import FastAPI
from pydantic import BaseModel, Field, field_validator
from typing import Optional
from decimal import Decimal

app = FastAPI()


class UserRegistration(BaseModel):
    # กำหนด validation ด้วย Field()
    username: str = Field(
        min_length=3,          # ความยาวขั้นต่ำ 3 ตัวอักษร
        max_length=20,         # ความยาวสูงสุด 20 ตัวอักษร
        pattern=r"^[a-zA-Z0-9_]+$",  # เฉพาะตัวอักษรและตัวเลข
        description="ชื่อผู้ใช้ (a-z, 0-9, _)",
        examples=["john_doe"]
    )
    email: str = Field(
        description="อีเมลของผู้ใช้",
        examples=["user@example.com"]
    )
    age: int = Field(
        ge=13,    # greater than or equal (อายุอย่างน้อย 13 ปี)
        le=120,   # less than or equal (อายุไม่เกิน 120 ปี)
        description="อายุของผู้ใช้"
    )
    password: str = Field(
        min_length=8,
        description="รหัสผ่าน (อย่างน้อย 8 ตัวอักษร)"
    )
    bio: Optional[str] = Field(
        default=None,
        max_length=500,
        description="ประวัติย่อ"
    )


class ProductPrice(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(
        gt=0,      # greater than (ราคาต้องมากกว่า 0)
        lt=1000000 # less than (ราคาไม่เกิน 1 ล้าน)
    )
    discount: float = Field(
        default=0.0,
        ge=0.0,   # ส่วนลดไม่ติดลบ
        le=100.0  # ส่วนลดไม่เกิน 100%
    )
    stock: int = Field(
        default=0,
        ge=0,     # สต็อกไม่ติดลบ
        description="จำนวนสินค้าในสต็อก"
    )

    # Custom validator ด้วย @field_validator
    @field_validator("price")
    @classmethod
    def price_must_be_reasonable(cls, v):
        """ราคาต้องไม่เป็นทศนิยมเกิน 2 ตำแหน่ง"""
        if round(v, 2) != v:
            raise ValueError("ราคาต้องมีทศนิยมไม่เกิน 2 ตำแหน่ง")
        return v


@app.post("/register/")
def register_user(user: UserRegistration):
    """ลงทะเบียนผู้ใช้ใหม่พร้อม validation"""
    return {
        "message": "ลงทะเบียนสำเร็จ",
        "username": user.username,
        "email": user.email,
        "age": user.age
        # ไม่คืน password กลับไป
    }


@app.post("/products/")
def create_product(product: ProductPrice):
    """สร้างสินค้าพร้อม validation ราคา"""
    # คำนวณราคาหลังหักส่วนลด
    final_price = product.price * (1 - product.discount / 100)
    return {
        "name": product.name,
        "original_price": product.price,
        "discount": f"{product.discount}%",
        "final_price": round(final_price, 2)
    }
```

---

## 4. Request Body + Path + Query Params ร่วมกัน

FastAPI สามารถรับ parameters จากหลายแหล่งพร้อมกัน

```python
# mixed_params.py

from fastapi import FastAPI, Path, Query, Body
from pydantic import BaseModel
from typing import Optional, List

app = FastAPI()


class ItemUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    price: Optional[float] = None
    in_stock: Optional[bool] = None


class SearchFilter(BaseModel):
    min_price: Optional[float] = None
    max_price: Optional[float] = None
    categories: Optional[List[str]] = None


# รับ path + query + body พร้อมกัน
@app.put("/items/{item_id}")
def update_item(
    item_id: int = Path(ge=1, description="รหัส item"),        # จาก path
    notify: bool = Query(default=False, description="ส่งแจ้งเตือน"),  # จาก query
    item: ItemUpdate = Body(..., description="ข้อมูลที่ต้องการอัปเดต"),  # จาก body
):
    """
    อัปเดต item:
    - item_id มาจาก URL path
    - notify มาจาก query string (?notify=true)
    - item มาจาก request body (JSON)
    """
    update_data = item.dict(exclude_none=True)  # เอาเฉพาะ field ที่มีค่า

    result = {
        "item_id": item_id,
        "updated_fields": update_data,
        "notification_sent": notify
    }

    if notify:
        result["message"] = f"อัปเดต item {item_id} และส่งแจ้งเตือนแล้ว"
    else:
        result["message"] = f"อัปเดต item {item_id} สำเร็จ"

    return result


# รับหลาย body parameters (ต้องใช้ Body())
@app.post("/compare/")
def compare_items(
    item1: ItemUpdate = Body(..., description="สินค้าตัวที่ 1"),
    item2: ItemUpdate = Body(..., description="สินค้าตัวที่ 2"),
    currency: str = Query(default="THB", description="สกุลเงิน")
):
    """
    เปรียบเทียบ 2 items
    Request body ต้องส่งเป็น:
    {
        "item1": {...},
        "item2": {...}
    }
    """
    return {
        "item1": item1.dict(exclude_none=True),
        "item2": item2.dict(exclude_none=True),
        "currency": currency
    }


# ใช้ Body(embed=True) เมื่อต้องการ wrapper
class UserProfile(BaseModel):
    name: str
    email: str


@app.post("/profile/update/{user_id}")
def update_profile(
    user_id: int = Path(ge=1),
    version: str = Query(default="v1"),
    profile: UserProfile = Body(..., embed=True)  # embed=True ทำให้ body เป็น {"profile": {...}}
):
    """อัปเดต profile ด้วย path + query + embedded body"""
    return {
        "user_id": user_id,
        "api_version": version,
        "profile": profile.dict()
    }
```

ตัวอย่าง request:
```bash
# อัปเดต item พร้อมแจ้งเตือน
curl -X PUT "http://localhost:8000/items/42?notify=true" \
     -H "Content-Type: application/json" \
     -d '{"name": "สินค้าใหม่", "price": 199.0}'

# เปรียบเทียบ 2 items
curl -X POST "http://localhost:8000/compare/?currency=USD" \
     -H "Content-Type: application/json" \
     -d '{
       "item1": {"name": "สินค้า A", "price": 100.0},
       "item2": {"name": "สินค้า B", "price": 150.0}
     }'
```

---

## 5. Form Data ด้วย Form()

สำหรับ HTML form submissions ที่ส่งข้อมูลเป็น `application/x-www-form-urlencoded`

```python
# form_data.py

from fastapi import FastAPI, Form, HTTPException
from typing import Optional

app = FastAPI()


# รับ form data พื้นฐาน
@app.post("/login/")
def login(
    username: str = Form(..., description="ชื่อผู้ใช้"),
    password: str = Form(..., description="รหัสผ่าน")
):
    """
    Login ด้วย form data
    Content-Type: application/x-www-form-urlencoded
    
    หมายเหตุ: ไม่สามารถใช้ Form() และ JSON body ร่วมกันได้
    """
    # ตรวจสอบ credentials (ตัวอย่างเท่านั้น)
    if username == "admin" and password == "secret":
        return {"message": "เข้าสู่ระบบสำเร็จ", "username": username}
    raise HTTPException(status_code=401, detail="ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง")


# Form data ที่มี optional fields
@app.post("/register/")
def register(
    username: str = Form(..., min_length=3, max_length=20),
    email: str = Form(...),
    password: str = Form(..., min_length=8),
    first_name: str = Form(...),
    last_name: str = Form(...),
    phone: Optional[str] = Form(default=None),
    newsletter: bool = Form(default=False)
):
    """ลงทะเบียนผู้ใช้ใหม่ด้วย form data"""
    return {
        "message": "ลงทะเบียนสำเร็จ",
        "user": {
            "username": username,
            "email": email,
            "full_name": f"{first_name} {last_name}",
            "phone": phone,
            "newsletter_subscribed": newsletter
        }
    }


# Form data สำหรับแก้ไขโปรไฟล์
@app.put("/users/{user_id}/profile/")
def update_profile_form(
    user_id: int,
    display_name: str = Form(...),
    bio: Optional[str] = Form(default=None, max_length=500),
    website: Optional[str] = Form(default=None)
):
    """อัปเดตโปรไฟล์ด้วย form data"""
    return {
        "user_id": user_id,
        "updated": {
            "display_name": display_name,
            "bio": bio,
            "website": website
        }
    }
```

ทดสอบด้วย curl:
```bash
# Login ด้วย form data
curl -X POST "http://localhost:8000/login/" \
     -d "username=admin&password=secret"

# หรือใช้ -F flag
curl -X POST "http://localhost:8000/login/" \
     -F "username=admin" \
     -F "password=secret"
```

---

## 6. File Upload ด้วย File() และ UploadFile

```python
# file_upload.py

from fastapi import FastAPI, File, UploadFile, HTTPException
from fastapi.responses import JSONResponse
import shutil
import os
from pathlib import Path
from typing import Optional

app = FastAPI()

# กำหนด directory สำหรับเก็บไฟล์
UPLOAD_DIR = Path("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)


# อัปโหลดไฟล์พื้นฐาน (bytes)
@app.post("/upload/bytes/")
def upload_bytes(file: bytes = File(...)):
    """
    รับไฟล์เป็น bytes โดยตรง
    ข้อเสีย: ไฟล์ทั้งหมดถูกโหลดเข้า memory
    เหมาะกับไฟล์ขนาดเล็กเท่านั้น
    """
    return {
        "file_size": len(file),
        "message": "รับไฟล์เป็น bytes สำเร็จ"
    }


# อัปโหลดด้วย UploadFile (แนะนำ)
@app.post("/upload/")
async def upload_file(file: UploadFile = File(...)):
    """
    UploadFile ดีกว่า bytes เพราะ:
    - ไม่โหลดทั้งไฟล์เข้า memory ทันที
    - มี metadata (filename, content_type)
    - สามารถอ่านแบบ async ได้
    """
    # ตรวจสอบประเภทไฟล์
    allowed_types = ["image/jpeg", "image/png", "image/gif", "application/pdf"]
    if file.content_type not in allowed_types:
        raise HTTPException(
            status_code=400,
            detail=f"ประเภทไฟล์ {file.content_type} ไม่ได้รับอนุญาต"
        )

    # บันทึกไฟล์
    file_path = UPLOAD_DIR / file.filename
    with open(file_path, "wb") as buffer:
        shutil.copyfileobj(file.file, buffer)

    # ดึงขนาดไฟล์
    file_size = os.path.getsize(file_path)

    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "file_size": file_size,
        "saved_to": str(file_path),
        "message": "อัปโหลดไฟล์สำเร็จ"
    }


# อัปโหลดรูปภาพพร้อม form fields
@app.post("/upload/product-image/")
async def upload_product_image(
    product_id: int,
    image: UploadFile = File(...),
    description: Optional[str] = None
):
    """อัปโหลดรูปภาพสินค้าพร้อม metadata"""

    # ตรวจสอบว่าเป็นรูปภาพ
    if not image.content_type.startswith("image/"):
        raise HTTPException(
            status_code=400,
            detail="กรุณาอัปโหลดรูปภาพเท่านั้น"
        )

    # ตรวจสอบขนาดไฟล์ (ไม่เกิน 5MB)
    MAX_SIZE = 5 * 1024 * 1024  # 5MB in bytes
    content = await image.read()
    if len(content) > MAX_SIZE:
        raise HTTPException(
            status_code=400,
            detail="ขนาดไฟล์ต้องไม่เกิน 5MB"
        )

    # บันทึกไฟล์
    file_ext = image.filename.split(".")[-1]
    new_filename = f"product_{product_id}.{file_ext}"
    file_path = UPLOAD_DIR / new_filename

    with open(file_path, "wb") as f:
        f.write(content)

    return {
        "product_id": product_id,
        "image_filename": new_filename,
        "original_filename": image.filename,
        "content_type": image.content_type,
        "file_size": len(content),
        "description": description
    }
```

---

## 7. Multiple Files Upload

```python
# multiple_files.py

from fastapi import FastAPI, File, UploadFile, Form, HTTPException
from typing import List
import os
from pathlib import Path
import hashlib

app = FastAPI()

UPLOAD_DIR = Path("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)


# อัปโหลดหลายไฟล์พร้อมกัน
@app.post("/upload/multiple/")
async def upload_multiple_files(
    files: List[UploadFile] = File(..., description="ไฟล์หลายไฟล์")
):
    """อัปโหลดหลายไฟล์พร้อมกัน"""
    results = []

    for file in files:
        # อ่านเนื้อหาไฟล์
        content = await file.read()

        # คำนวณ hash เพื่อตรวจสอบความสมบูรณ์
        file_hash = hashlib.md5(content).hexdigest()

        # บันทึกไฟล์
        file_path = UPLOAD_DIR / file.filename
        with open(file_path, "wb") as f:
            f.write(content)

        results.append({
            "filename": file.filename,
            "content_type": file.content_type,
            "size": len(content),
            "md5_hash": file_hash
        })

    return {
        "uploaded_count": len(results),
        "files": results
    }


# อัปโหลดหลายไฟล์พร้อม form fields
@app.post("/upload/album/")
async def upload_album(
    album_name: str = Form(...),
    description: str = Form(default=""),
    images: List[UploadFile] = File(...),
    cover_index: int = Form(default=0)  # index ของรูป cover
):
    """สร้าง album พร้อมรูปภาพหลายรูป"""

    # ตรวจสอบว่าทุกไฟล์เป็นรูปภาพ
    for image in images:
        if not image.content_type.startswith("image/"):
            raise HTTPException(
                status_code=400,
                detail=f"ไฟล์ {image.filename} ไม่ใช่รูปภาพ"
            )

    # ตรวจสอบ cover_index
    if cover_index >= len(images):
        raise HTTPException(
            status_code=400,
            detail=f"cover_index ต้องไม่เกิน {len(images) - 1}"
        )

    # บันทึกรูปภาพทั้งหมด
    album_dir = UPLOAD_DIR / album_name.replace(" ", "_")
    album_dir.mkdir(exist_ok=True)

    saved_files = []
    for i, image in enumerate(images):
        content = await image.read()
        file_path = album_dir / image.filename
        with open(file_path, "wb") as f:
            f.write(content)
        saved_files.append(image.filename)

    return {
        "album_name": album_name,
        "description": description,
        "cover_image": saved_files[cover_index],
        "total_images": len(saved_files),
        "images": saved_files
    }


# อัปโหลดไฟล์หลายประเภทพร้อมกัน (mixed upload)
@app.post("/upload/document-with-attachments/")
async def upload_document(
    title: str = Form(...),
    content: str = Form(...),
    main_document: UploadFile = File(..., description="เอกสารหลัก"),
    attachments: List[UploadFile] = File(default=[], description="ไฟล์แนบ (optional)")
):
    """อัปโหลดเอกสารหลักพร้อมไฟล์แนบ"""
    results = {
        "title": title,
        "content_preview": content[:100] + "..." if len(content) > 100 else content,
        "main_document": {
            "filename": main_document.filename,
            "content_type": main_document.content_type
        },
        "attachments": []
    }

    for attachment in attachments:
        results["attachments"].append({
            "filename": attachment.filename,
            "content_type": attachment.content_type
        })

    return results
```

ทดสอบ multiple upload:
```bash
# อัปโหลดหลายไฟล์
curl -X POST "http://localhost:8000/upload/multiple/" \
     -F "files=@image1.jpg" \
     -F "files=@image2.png" \
     -F "files=@document.pdf"
```

---

## 8. Model Config และ Extra Fields

```python
# model_config.py

from fastapi import FastAPI
from pydantic import BaseModel, ConfigDict
from typing import Optional, Any

app = FastAPI()


# โมเดลที่ reject extra fields (default behavior ใน Pydantic v2)
class StrictItem(BaseModel):
    model_config = ConfigDict(extra="forbid")  # ห้าม extra fields

    name: str
    price: float


# โมเดลที่อนุญาต extra fields
class FlexibleItem(BaseModel):
    model_config = ConfigDict(extra="allow")  # อนุญาต extra fields

    name: str
    price: float
    # Extra fields จะถูกเก็บโดยอัตโนมัติ


# โมเดลที่ ignore extra fields
class IgnoreExtraItem(BaseModel):
    model_config = ConfigDict(extra="ignore")  # ละเว้น extra fields

    name: str
    price: float


# โมเดลที่ใช้ alias (ชื่อ field ในภาษาอื่น)
class ThaiProduct(BaseModel):
    model_config = ConfigDict(
        populate_by_name=True  # อนุญาตใช้ทั้ง alias และชื่อจริง
    )

    # ใช้ alias เพื่อรองรับ JSON ที่มีชื่อ field ต่างกัน
    from pydantic import Field
    product_name: str = Field(alias="ชื่อสินค้า")
    product_price: float = Field(alias="ราคา")
    quantity: int = Field(alias="จำนวน", default=1)


@app.post("/items/strict/")
def create_strict_item(item: StrictItem):
    """ไม่อนุญาต extra fields - จะ error ถ้าส่ง field เพิ่ม"""
    return item.dict()


@app.post("/items/flexible/")
def create_flexible_item(item: FlexibleItem):
    """อนุญาต extra fields - รับค่าพิเศษได้"""
    return {
        "known_fields": {"name": item.name, "price": item.price},
        "extra_fields": item.model_extra  # เข้าถึง extra fields
    }


@app.post("/items/ignore-extra/")
def create_ignore_extra_item(item: IgnoreExtraItem):
    """ละเว้น extra fields - ไม่ error แต่ไม่เก็บค่า"""
    return item.dict()


# Immutable model (frozen)
class ImmutableConfig(BaseModel):
    model_config = ConfigDict(frozen=True)

    host: str
    port: int = 8080
    debug: bool = False


@app.post("/config/")
def create_config(config: ImmutableConfig):
    """Config ที่แก้ไขค่าไม่ได้หลังสร้าง"""
    # config.host = "new_host"  # นี้จะ raise error ถ้า frozen=True
    return config.dict()
```

---

## 9. Response Models และ response_model_exclude

Response model ช่วย filter ข้อมูลที่ส่งกลับ ป้องกันการ leak ข้อมูลสำคัญ

```python
# response_models.py

from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional, List, Set

app = FastAPI()


# Model สำหรับ input (รับข้อมูลจาก client)
class UserCreate(BaseModel):
    username: str
    email: str
    password: str          # sensitive - ไม่ควรส่งกลับ
    full_name: str
    is_admin: bool = False  # ไม่ควรให้ client กำหนดเองได้


# Model สำหรับ output (ส่งข้อมูลกลับ client)
class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    full_name: str
    is_active: bool = True
    # ไม่มี password และ is_admin - ปลอดภัย!


# Model สำหรับ list
class UserListResponse(BaseModel):
    id: int
    username: str
    email: str
    # เฉพาะข้อมูลที่จำเป็นสำหรับ list view


# ใช้ response_model เพื่อกำหนด output schema
@app.post("/users/", response_model=UserResponse)
def create_user(user: UserCreate):
    """
    สร้าง user ใหม่
    FastAPI จะ filter response ให้ตรงกับ UserResponse โดยอัตโนมัติ
    - password จะไม่ถูกส่งกลับ
    - is_admin จะไม่ถูกส่งกลับ
    """
    # สร้าง user ในฐานข้อมูล (ตัวอย่าง)
    new_user = {
        "id": 1,
        "username": user.username,
        "email": user.email,
        "password": user.password,       # มีใน database
        "full_name": user.full_name,
        "is_admin": False,               # ไม่ให้ client กำหนด
        "is_active": True,
        "secret_token": "abc123"         # มีใน database แต่ไม่ส่งกลับ
    }
    # response_model จะ filter ให้เหลือเฉพาะ field ใน UserResponse
    return new_user


# response_model_exclude - ระบุ field ที่ไม่ต้องการส่ง
@app.get("/users/{user_id}", response_model=UserResponse, response_model_exclude={"is_active"})
def get_user(user_id: int):
    """ดึง user - ไม่แสดง is_active"""
    return {
        "id": user_id,
        "username": "johndoe",
        "email": "john@example.com",
        "full_name": "John Doe",
        "is_active": True  # จะถูก exclude ออก
    }


# response_model_include - ส่งเฉพาะ field ที่ระบุ
@app.get("/users/", response_model=List[UserResponse], response_model_include={"id", "username", "email"})
def list_users():
    """ดึง user list - แสดงเฉพาะ id, username, email"""
    users = [
        {"id": 1, "username": "alice", "email": "alice@example.com", "full_name": "Alice Smith", "is_active": True},
        {"id": 2, "username": "bob", "email": "bob@example.com", "full_name": "Bob Jones", "is_active": False},
    ]
    return users


# response_model_exclude_none - ไม่ส่ง field ที่เป็น None
class ProductResponse(BaseModel):
    id: int
    name: str
    description: Optional[str] = None
    image_url: Optional[str] = None
    discount: Optional[float] = None


@app.get("/products/{product_id}", response_model=ProductResponse, response_model_exclude_none=True)
def get_product(product_id: int):
    """ดึงสินค้า - ไม่แสดง field ที่เป็น None"""
    return {
        "id": product_id,
        "name": "มะม่วงน้ำดอกไม้",
        "description": None,   # จะไม่แสดงใน response
        "image_url": None,     # จะไม่แสดงใน response
        "discount": None       # จะไม่แสดงใน response
    }
    # Response จะเป็น: {"id": 1, "name": "มะม่วงน้ำดอกไม้"}


# Custom response model สำหรับ pagination
class PaginatedResponse(BaseModel):
    total: int
    page: int
    per_page: int
    items: List[UserListResponse]


@app.get("/users/paginated/", response_model=PaginatedResponse)
def get_users_paginated(page: int = 1, per_page: int = 10):
    """ดึง users แบบ pagination"""
    # Mock data
    all_users = [
        {"id": i, "username": f"user{i}", "email": f"user{i}@example.com"}
        for i in range(1, 51)
    ]

    start = (page - 1) * per_page
    end = start + per_page
    paginated = all_users[start:end]

    return {
        "total": len(all_users),
        "page": page,
        "per_page": per_page,
        "items": paginated
    }
```

---

## 10. ตัวอย่างสมบูรณ์: Product API

รวม concepts ทั้งหมดในตัวอย่างเดียว

```python
# product_api.py - ตัวอย่างสมบูรณ์

from fastapi import FastAPI, File, UploadFile, Form, Path, Query, HTTPException, Depends
from pydantic import BaseModel, Field, field_validator, ConfigDict
from typing import Optional, List
from pathlib import Path as PathLib
import shutil

app = FastAPI(title="Product API", version="1.0.0")

UPLOAD_DIR = PathLib("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)


# === Models ===

class CategoryBase(BaseModel):
    name: str = Field(min_length=1, max_length=50)
    slug: str = Field(min_length=1, max_length=50, pattern=r"^[a-z0-9-]+$")


class ProductCreate(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True)  # ตัด whitespace อัตโนมัติ

    name: str = Field(min_length=1, max_length=200, description="ชื่อสินค้า")
    sku: str = Field(min_length=3, max_length=50, description="รหัสสินค้า")
    price: float = Field(gt=0, description="ราคา (บาท)")
    sale_price: Optional[float] = Field(default=None, gt=0, description="ราคาลด")
    stock: int = Field(ge=0, default=0, description="จำนวนสต็อก")
    category: CategoryBase
    tags: List[str] = Field(default=[], description="แท็กสินค้า")
    is_active: bool = Field(default=True, description="เปิดขายหรือไม่")

    @field_validator("sale_price")
    @classmethod
    def sale_price_less_than_price(cls, v, info):
        """ราคาลดต้องน้อยกว่าราคาปกติ"""
        if v is not None and "price" in info.data and v >= info.data["price"]:
            raise ValueError("ราคาลดต้องน้อยกว่าราคาปกติ")
        return v

    @field_validator("sku")
    @classmethod
    def sku_uppercase(cls, v):
        """SKU ต้องเป็นตัวพิมพ์ใหญ่"""
        return v.upper()


class ProductResponse(BaseModel):
    id: int
    name: str
    sku: str
    price: float
    sale_price: Optional[float]
    stock: int
    category_name: str
    tags: List[str]
    is_active: bool
    image_url: Optional[str] = None


class ProductUpdate(BaseModel):
    name: Optional[str] = Field(default=None, min_length=1, max_length=200)
    price: Optional[float] = Field(default=None, gt=0)
    sale_price: Optional[float] = Field(default=None, gt=0)
    stock: Optional[int] = Field(default=None, ge=0)
    is_active: Optional[bool] = None


# Mock database
products_db: dict = {}
product_counter = 0


# === Endpoints ===

@app.post("/api/v1/products/", response_model=ProductResponse, status_code=201)
def create_product(product: ProductCreate):
    """สร้างสินค้าใหม่"""
    global product_counter
    product_counter += 1

    new_product = {
        "id": product_counter,
        "name": product.name,
        "sku": product.sku,
        "price": product.price,
        "sale_price": product.sale_price,
        "stock": product.stock,
        "category_name": product.category.name,
        "tags": product.tags,
        "is_active": product.is_active,
        "image_url": None
    }
    products_db[product_counter] = new_product
    return new_product


@app.post("/api/v1/products/{product_id}/image/")
async def upload_product_image(
    product_id: int = Path(ge=1),
    image: UploadFile = File(...),
    alt_text: str = Form(default="")
):
    """อัปโหลดรูปภาพสินค้า"""
    if product_id not in products_db:
        raise HTTPException(status_code=404, detail="ไม่พบสินค้า")

    if not image.content_type.startswith("image/"):
        raise HTTPException(status_code=400, detail="กรุณาอัปโหลดรูปภาพเท่านั้น")

    # บันทึกรูปภาพ
    file_ext = image.filename.split(".")[-1]
    filename = f"product_{product_id}.{file_ext}"
    file_path = UPLOAD_DIR / filename

    content = await image.read()
    with open(file_path, "wb") as f:
        f.write(content)

    # อัปเดต database
    products_db[product_id]["image_url"] = f"/uploads/{filename}"

    return {
        "product_id": product_id,
        "image_url": f"/uploads/{filename}",
        "alt_text": alt_text,
        "file_size": len(content)
    }


@app.get("/api/v1/products/", response_model=List[ProductResponse])
def list_products(
    page: int = Query(default=1, ge=1),
    per_page: int = Query(default=10, ge=1, le=100),
    active_only: bool = Query(default=True),
    min_price: Optional[float] = Query(default=None, gt=0),
    max_price: Optional[float] = Query(default=None, gt=0)
):
    """ดึงรายการสินค้าพร้อม filtering และ pagination"""
    products = list(products_db.values())

    # Filter
    if active_only:
        products = [p for p in products if p["is_active"]]
    if min_price:
        products = [p for p in products if p["price"] >= min_price]
    if max_price:
        products = [p for p in products if p["price"] <= max_price]

    # Pagination
    start = (page - 1) * per_page
    return products[start:start + per_page]


@app.patch("/api/v1/products/{product_id}/", response_model=ProductResponse)
def update_product(
    product_id: int = Path(ge=1),
    update: ProductUpdate = ...
):
    """อัปเดตสินค้า (partial update)"""
    if product_id not in products_db:
        raise HTTPException(status_code=404, detail="ไม่พบสินค้า")

    product = products_db[product_id]
    update_data = update.dict(exclude_none=True)
    product.update(update_data)

    return product
```

---

## 11. สรุป Part 088

✅ **BaseModel** - ใช้ Pydantic define โครงสร้าง request body และ validate อัตโนมัติ

✅ **Nested Models** - สร้างโมเดลซ้อนกันสำหรับข้อมูลซับซ้อน (Address, OrderItem ใน Order)

✅ **Field()** - กำหนด validation rules (min_length, ge, le, pattern), description, examples

✅ **Mixed Params** - รับ path + query + body ร่วมกันในฟังก์ชันเดียว

✅ **Form()** - รับ HTML form data (application/x-www-form-urlencoded)

✅ **UploadFile** - อัปโหลดไฟล์ที่มี metadata และอ่านแบบ async ได้

✅ **Multiple Files** - `List[UploadFile]` สำหรับอัปโหลดหลายไฟล์พร้อมกัน

✅ **model_config** - ควบคุม extra fields (forbid/allow/ignore) และ behavior ของ model

✅ **response_model** - filter ข้อมูลที่ส่งกลับ, ป้องกัน sensitive data leak

✅ **response_model_exclude/include** - ระบุ field ที่ต้องการ/ไม่ต้องการในการ response

---

## ➡️ ถัดไป: Part 089 - FastAPI Dependencies and Security

ใน Part ถัดไปเราจะเรียนรู้:
- Dependency injection ด้วย `Depends()`
- การสร้างและ verify JWT token
- Password hashing ด้วย passlib
- OAuth2 authentication
- Role-based access control

*Part 088/100+ | Python Course - Beginner to World-Class*
