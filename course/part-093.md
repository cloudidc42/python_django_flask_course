# Part 093 - FastAPI Pydantic Validation

## เป้าหมายการเรียนรู้

- ใช้ Pydantic models ขั้นสูง
- สร้าง Field validators
- ทำงานกับ Nested models
- สร้าง Custom validators
- จัดการ Settings ด้วย Pydantic Settings

---

## 1. Pydantic v2 พื้นฐาน

Pydantic เป็น library สำหรับ data validation และ settings management โดยใช้ Python type annotations

```bash
pip install pydantic[email] pydantic-settings
```

```python
from pydantic import BaseModel, Field, validator, field_validator, model_validator
from typing import Optional, List, Dict, Any
from datetime import datetime, date
from enum import Enum
import re


# ─────────────────────────────────────────
# Basic Model
# ─────────────────────────────────────────

class User(BaseModel):
    id: int
    username: str
    email: str
    is_active: bool = True
    created_at: datetime = Field(default_factory=datetime.utcnow)


# สร้าง instance
user = User(id=1, username="alice", email="alice@example.com")
print(user)
# id=1 username='alice' email='alice@example.com' is_active=True

# แปลงเป็น dict
user_dict = user.model_dump()
# หรือ Pydantic v1: user.dict()

# แปลงเป็น JSON string
user_json = user.model_dump_json()

# สร้างจาก dict
user2 = User.model_validate({"id": 2, "username": "bob", "email": "bob@test.com"})
```

---

## 2. Field Validators

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from typing import Optional


class Product(BaseModel):
    name: str = Field(
        ...,
        min_length=1,
        max_length=200,
        strip_whitespace=True,  # ตัด whitespace อัตโนมัติ
        description="ชื่อสินค้า"
    )
    price: float = Field(..., gt=0, le=1_000_000)
    discount_percent: Optional[float] = Field(None, ge=0, le=100)
    sku: Optional[str] = Field(None, pattern=r'^[A-Z]{3}-\d{4}$')
    tags: List[str] = Field(default_factory=list, max_items=10)
    
    # ─────────────────────────────────────────
    # Field Validators (Pydantic v2)
    # ─────────────────────────────────────────
    
    @field_validator('name')
    @classmethod
    def name_must_not_be_profanity(cls, v):
        """ตรวจสอบว่าชื่อไม่มี bad words"""
        bad_words = ['spam', 'scam']
        for word in bad_words:
            if word.lower() in v.lower():
                raise ValueError(f'ชื่อสินค้าไม่ควรมีคำว่า "{word}"')
        return v.title()  # แปลง Title Case
    
    @field_validator('price')
    @classmethod
    def round_price(cls, v):
        """ปัดราคาเป็น 2 ตำแหน่งทศนิยม"""
        return round(v, 2)
    
    @field_validator('tags', mode='before')
    @classmethod
    def clean_tags(cls, v):
        """ทำ tags เป็น lowercase และ trim"""
        if isinstance(v, list):
            return [tag.lower().strip() for tag in v if tag.strip()]
        return v
    
    # ─────────────────────────────────────────
    # Model Validator (cross-field validation)
    # ─────────────────────────────────────────
    
    @model_validator(mode='after')
    def validate_discount_vs_price(self):
        """ตรวจสอบว่า discount ไม่สูงเกินไปสำหรับสินค้าถูก"""
        if self.discount_percent is not None:
            if self.price < 100 and self.discount_percent > 50:
                raise ValueError('สินค้าราคาต่ำกว่า 100 บาทลด discount สูงสุดได้ 50%')
        return self
    
    # คำนวณ final price
    @property
    def final_price(self) -> float:
        if self.discount_percent:
            return self.price * (1 - self.discount_percent / 100)
        return self.price


# Test validation
try:
    product = Product(
        name="  Python Book  ",
        price=250.5678,
        discount_percent=10,
        sku="PYT-0001",
        tags=["Python", "Programming", "  BOOK  "]
    )
    print(product.name)         # "Python Book" (stripped + title case)
    print(product.price)        # 250.57 (rounded)
    print(product.tags)         # ['python', 'programming', 'book'] (lowercase)
    print(product.final_price)  # 225.51
except Exception as e:
    print(f"Validation error: {e}")
```

---

## 3. Custom Validators

```python
# validators.py - Custom validators ที่ใช้ซ้ำได้

from pydantic import BaseModel, field_validator, EmailStr
import re
from datetime import date


def validate_thai_phone(v: str) -> str:
    """ตรวจสอบเบอร์โทรศัพท์ไทย"""
    # ลบ spaces, dashes, parentheses
    cleaned = re.sub(r'[\s\-\(\)]', '', v)
    
    # เบอร์ไทย: 0XX-XXX-XXXX หรือ +66XX-XXX-XXXX
    thai_phone = re.compile(r'^(0[689]\d{8}|(\+66|0066)[689]\d{8})$')
    
    if not thai_phone.match(cleaned):
        raise ValueError('เบอร์โทรศัพท์ไม่ถูกต้อง (รูปแบบ: 0XX-XXX-XXXX)')
    
    return cleaned


def validate_thai_id(v: str) -> str:
    """ตรวจสอบเลขบัตรประชาชนไทย"""
    # ลบ dashes
    cleaned = re.sub(r'[-\s]', '', v)
    
    if len(cleaned) != 13 or not cleaned.isdigit():
        raise ValueError('เลขบัตรประชาชนต้องมี 13 หลัก')
    
    # Luhn algorithm สำหรับเลขบัตรประชาชนไทย
    digits = [int(d) for d in cleaned]
    total = sum(digits[i] * (13 - i) for i in range(12))
    check = (11 - (total % 11)) % 10
    
    if check != digits[12]:
        raise ValueError('เลขบัตรประชาชนไม่ถูกต้อง')
    
    return cleaned


class CustomerProfile(BaseModel):
    full_name: str
    email: str
    phone: str
    national_id: Optional[str] = None
    birth_date: Optional[date] = None
    
    @field_validator('phone')
    @classmethod
    def validate_phone(cls, v):
        return validate_thai_phone(v)
    
    @field_validator('national_id')
    @classmethod
    def validate_national_id(cls, v):
        if v is not None:
            return validate_thai_id(v)
        return v
    
    @field_validator('birth_date')
    @classmethod
    def validate_age(cls, v):
        if v:
            today = date.today()
            age = (today - v).days // 365
            if age < 18:
                raise ValueError('ต้องมีอายุ 18 ปีขึ้นไป')
            if age > 120:
                raise ValueError('วันเกิดไม่ถูกต้อง')
        return v
    
    @field_validator('full_name')
    @classmethod
    def validate_name(cls, v):
        v = v.strip()
        if len(v.split()) < 2:
            raise ValueError('กรุณากรอกชื่อและนามสกุล')
        return v
```

---

## 4. Nested Models

```python
# nested_models.py

from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime
from enum import Enum


class OrderStatus(str, Enum):
    pending = "pending"
    confirmed = "confirmed"
    shipped = "shipped"
    delivered = "delivered"
    cancelled = "cancelled"


class Address(BaseModel):
    """ที่อยู่"""
    line1: str = Field(..., min_length=5)
    line2: Optional[str] = None
    city: str
    province: str
    zipcode: str = Field(..., pattern=r'^\d{5}$')
    country: str = "TH"


class ProductSnapshot(BaseModel):
    """ข้อมูลสินค้า ณ เวลาซื้อ"""
    product_id: int
    name: str
    sku: Optional[str] = None
    price: float


class OrderItem(BaseModel):
    """รายการสินค้าในคำสั่งซื้อ"""
    product: ProductSnapshot
    quantity: int = Field(..., ge=1, le=999)
    unit_price: float = Field(..., ge=0)
    discount: float = Field(0, ge=0, le=100)  # discount %
    
    @property
    def subtotal(self) -> float:
        return self.unit_price * self.quantity * (1 - self.discount / 100)


class CustomerInfo(BaseModel):
    """ข้อมูลลูกค้า"""
    name: str
    email: str
    phone: str
    is_member: bool = False


class ShippingMethod(str, Enum):
    standard = "standard"
    express = "express"
    same_day = "same_day"


class Order(BaseModel):
    """คำสั่งซื้อ"""
    id: Optional[int] = None
    customer: CustomerInfo
    shipping_address: Address
    billing_address: Optional[Address] = None  # None = same as shipping
    items: List[OrderItem] = Field(..., min_items=1)
    shipping_method: ShippingMethod = ShippingMethod.standard
    notes: Optional[str] = Field(None, max_length=500)
    status: OrderStatus = OrderStatus.pending
    created_at: datetime = Field(default_factory=datetime.utcnow)
    
    @property
    def subtotal(self) -> float:
        return sum(item.subtotal for item in self.items)
    
    @property
    def shipping_cost(self) -> float:
        costs = {
            ShippingMethod.standard: 50,
            ShippingMethod.express: 150,
            ShippingMethod.same_day: 300
        }
        return costs[self.shipping_method]
    
    @property
    def total(self) -> float:
        return self.subtotal + self.shipping_cost
    
    def to_response(self) -> dict:
        """แปลงเป็น dict สำหรับ API response"""
        return {
            **self.model_dump(),
            "subtotal": self.subtotal,
            "shipping_cost": self.shipping_cost,
            "total": self.total
        }


# ─────────────────────────────────────────
# ใช้ใน FastAPI
# ─────────────────────────────────────────

from fastapi import FastAPI

app = FastAPI()


@app.post("/orders", status_code=201)
async def create_order(order: Order):
    """สร้างคำสั่งซื้อ"""
    # Pydantic validate nested models อัตโนมัติ
    # order.customer, order.shipping_address, order.items ล้วน validated
    
    order_id = 1001
    order_response = order.to_response()
    order_response["id"] = order_id
    
    return order_response
```

---

## 5. Settings Management

```python
# config.py - Settings ด้วย Pydantic Settings

from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field, field_validator
from typing import Optional, List
from functools import lru_cache


class Settings(BaseSettings):
    """Application settings"""
    
    model_config = SettingsConfigDict(
        env_file=".env",             # โหลดจาก .env
        env_file_encoding="utf-8",
        case_sensitive=False,         # KEY=value หรือ key=value เหมือนกัน
        extra="ignore",               # ไม่ error ถ้า env var เพิ่มเติม
    )
    
    # App settings
    app_name: str = "My FastAPI App"
    app_version: str = "1.0.0"
    debug: bool = False
    
    # Server
    host: str = "0.0.0.0"
    port: int = 8000
    workers: int = Field(default=1, ge=1, le=32)
    
    # Database
    database_url: str = Field(..., description="Database connection URL")
    db_pool_size: int = 5
    db_max_overflow: int = 10
    db_echo: bool = False
    
    # Security
    secret_key: str = Field(..., min_length=32)
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 60
    refresh_token_expire_days: int = 30
    
    # CORS
    allowed_origins: List[str] = ["http://localhost:3000"]
    allowed_methods: List[str] = ["*"]
    allowed_headers: List[str] = ["*"]
    
    # Email
    mail_server: Optional[str] = None
    mail_port: int = 587
    mail_tls: bool = True
    mail_username: Optional[str] = None
    mail_password: Optional[str] = None
    mail_from: Optional[str] = None
    
    # Redis
    redis_url: Optional[str] = None
    
    # File Upload
    max_upload_size_mb: int = Field(default=16, ge=1, le=100)
    upload_dir: str = "uploads"
    allowed_file_types: List[str] = ["image/jpeg", "image/png", "image/gif"]
    
    # Validators
    @field_validator('database_url')
    @classmethod
    def validate_db_url(cls, v):
        """ตรวจสอบว่า database_url ถูกรูปแบบ"""
        if not v.startswith(('sqlite', 'postgresql', 'mysql', 'mongodb')):
            raise ValueError('database_url รูปแบบไม่รองรับ')
        # แปลง postgres:// เป็น postgresql://
        if v.startswith('postgres://'):
            return v.replace('postgres://', 'postgresql://', 1)
        return v
    
    @field_validator('secret_key')
    @classmethod
    def validate_secret_key(cls, v):
        if v in ['secret', 'change-me', 'my-secret']:
            raise ValueError('กรุณาเปลี่ยน secret_key เป็นค่าที่ปลอดภัย')
        return v
    
    @property
    def max_upload_size_bytes(self) -> int:
        return self.max_upload_size_mb * 1024 * 1024
    
    @property
    def is_development(self) -> bool:
        return self.debug
    
    @property
    def database_settings(self) -> dict:
        return {
            "url": self.database_url,
            "pool_size": self.db_pool_size,
            "max_overflow": self.db_max_overflow,
            "echo": self.db_echo
        }


# Singleton pattern ด้วย lru_cache
@lru_cache()
def get_settings() -> Settings:
    """
    สร้าง Settings instance ครั้งเดียว
    lru_cache ทำให้ function รัน 1 ครั้ง และ cache ผล
    """
    return Settings()


# ใช้ใน FastAPI
from fastapi import FastAPI, Depends

app = FastAPI()


@app.get("/info")
async def app_info(settings: Settings = Depends(get_settings)):
    """ดูข้อมูล app จาก settings"""
    return {
        "app_name": settings.app_name,
        "version": settings.app_version,
        "debug": settings.debug
    }
```

### 5.1 .env ไฟล์

```bash
# .env

# App
APP_NAME="My FastAPI App"
DEBUG=false

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/myapp

# Security  
SECRET_KEY=your-very-long-and-random-secret-key-here-minimum-32-chars

# CORS (comma separated)
ALLOWED_ORIGINS=["http://localhost:3000","https://app.example.com"]

# Email
MAIL_SERVER=smtp.gmail.com
MAIL_USERNAME=myapp@gmail.com
MAIL_PASSWORD=app-specific-password

# Redis
REDIS_URL=redis://localhost:6379/0
```

---

## 6. Advanced Pydantic Features

```python
# advanced.py

from pydantic import BaseModel, Field, computed_field
from typing import Optional, ClassVar
from datetime import datetime


class BlogPost(BaseModel):
    """Blog post พร้อม computed fields"""
    
    title: str
    content: str
    tags: list[str] = []
    is_published: bool = False
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: Optional[datetime] = None
    
    # Class variable (ไม่ใช่ field)
    MAX_TITLE_LENGTH: ClassVar[int] = 200
    
    @computed_field  # Pydantic v2
    @property
    def word_count(self) -> int:
        """คำนวณจำนวนคำ"""
        return len(self.content.split())
    
    @computed_field
    @property
    def reading_time_minutes(self) -> int:
        """เวลาอ่าน (200 words/minute)"""
        return max(1, self.word_count // 200)
    
    @computed_field
    @property
    def excerpt(self) -> str:
        """ตัวอย่างเนื้อหา 150 ตัวอักษรแรก"""
        return self.content[:150] + "..." if len(self.content) > 150 else self.content


# ─────────────────────────────────────────
# Model Config
# ─────────────────────────────────────────

class UserDB(BaseModel):
    """User model สำหรับ database"""
    
    model_config = {
        "from_attributes": True,   # อ่านจาก ORM objects (Pydantic v2)
        "populate_by_name": True,  # อนุญาตใช้ชื่อ field หรือ alias
        "str_strip_whitespace": True,  # strip whitespace อัตโนมัติ
        "str_max_length": 200,    # max length สำหรับทุก str fields
        "frozen": False,           # อนุญาตแก้ไข
    }
    
    id: int
    username: str
    email: str
    
    # Field alias - ใช้ "user_name" ใน input, "username" ใน code
    display_name: Optional[str] = Field(None, alias="name")


# ─────────────────────────────────────────
# Discriminated Unions
# ─────────────────────────────────────────

from typing import Union, Literal


class PaymentCard(BaseModel):
    payment_type: Literal["card"]
    card_number: str
    expiry: str
    cvv: str


class PaymentQR(BaseModel):
    payment_type: Literal["qr"]
    qr_ref: str


class PaymentTransfer(BaseModel):
    payment_type: Literal["transfer"]
    bank: str
    account: str


# Union ที่ discriminate ด้วย payment_type
Payment = Union[PaymentCard, PaymentQR, PaymentTransfer]


class Order(BaseModel):
    product_id: int
    quantity: int
    payment: Payment = Field(..., discriminator="payment_type")


# ─────────────────────────────────────────
# Generic Models
# ─────────────────────────────────────────

from typing import TypeVar, Generic

T = TypeVar('T')


class APIResponse(BaseModel, Generic[T]):
    """Generic response wrapper"""
    success: bool = True
    data: Optional[T] = None
    error: Optional[str] = None
    code: int = 200


class PaginatedResponse(BaseModel, Generic[T]):
    """Generic paginated response"""
    items: list[T]
    total: int
    page: int
    per_page: int
    pages: int
    has_next: bool
    has_prev: bool


# ใช้งาน
from fastapi import FastAPI

app = FastAPI()


class Product(BaseModel):
    id: int
    name: str
    price: float


@app.get("/products", response_model=PaginatedResponse[Product])
async def list_products():
    products = [
        Product(id=1, name="Book", price=59.0),
        Product(id=2, name="Pen", price=25.0),
    ]
    return PaginatedResponse(
        items=products,
        total=2,
        page=1,
        per_page=10,
        pages=1,
        has_next=False,
        has_prev=False
    )
```

---

## Exercises

### Exercise 1: E-commerce Models
สร้าง Pydantic models สำหรับ:
- `Product` พร้อม validators สำหรับ price, sku
- `CartItem` (product_id, quantity, options)
- `Cart` (items, coupon_code)
- Computed: subtotal, discount, total
- Cross-field: validate ว่า total > 0 เมื่อมี items

### Exercise 2: Settings สำหรับ Production
สร้าง Settings class ที่:
- รับจาก .env
- Required: DATABASE_URL, SECRET_KEY
- Optional: EMAIL settings, REDIS_URL
- Validators สำหรับทุก required fields
- Test ว่า settings โหลดถูกต้อง

### Exercise 3: Custom Validators
สร้าง validators สำหรับ:
- เลขบัตรเครดิต (Luhn algorithm)
- URL ต้องเป็น https
- Password strength (uppercase, number, special char)
- Date range (start_date < end_date)

---

## สรุป

สิ่งที่เรียนรู้ใน Part นี้:
- **Field validators** ตรวจสอบและแปลง field เดียว
- **Model validators** cross-field validation
- **Nested models** จัดการ complex data structures
- **Custom validators** logic ที่ซับซ้อนและ reusable
- **Pydantic Settings** จัดการ configuration จาก env vars
- **Computed fields** ค่าที่คำนวณจาก fields อื่น
- **Generic models** API response templates

---

## ลิงก์ Part ถัดไป

➡️ [Part 094 - FastAPI Database (SQLAlchemy Async)](./part-094.md)
