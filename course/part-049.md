# Part 049: Data Validation with Pydantic
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- สร้าง data models ด้วย BaseModel
- ใช้ field validators ตรวจสอบข้อมูล
- สร้าง model validators (cross-field)
- กำหนด custom types
- สร้าง nested models
- Serialization และ deserialization
- Settings management ด้วย pydantic-settings

---

## 1. BaseModel พื้นฐาน

```bash
pip install pydantic email-validator
```

```python
from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List, Dict, Any
from datetime import datetime, date
from decimal import Decimal
from enum import Enum
from uuid import UUID

# === Basic Model ===
class User(BaseModel):
    id: int
    name: str
    email: EmailStr  # pip install email-validator
    age: int
    is_active: bool = True
    created_at: datetime = Field(default_factory=datetime.now)

# สร้าง instance
user = User(id=1, name="Alice", email="alice@example.com", age=25)
print(user)
print(f"Email: {user.email}")
print(f"Active: {user.is_active}")
print(f"Created: {user.created_at}")

# Pydantic จะ validate types อัตโนมัติ
try:
    invalid = User(id="not-an-int", name="Bob", email="bob@example.com", age="thirty")
except Exception as e:
    print(f"\nValidation errors: {e}")

# Coercion - แปลง type อัตโนมัติ
coerced = User(id="42", name="Charlie", email="charlie@example.com", age="30")
print(f"\nCoerced id type: {type(coerced.id)}")  # int


# === Model ที่ซับซ้อนขึ้น ===
class Gender(str, Enum):
    MALE = "male"
    FEMALE = "female"
    OTHER = "other"


class Address(BaseModel):
    street: str
    city: str
    state: str
    zip_code: str
    country: str = "TH"


class UserProfile(BaseModel):
    # Required fields
    id: UUID
    username: str
    email: EmailStr
    
    # Optional fields
    full_name: Optional[str] = None
    gender: Optional[Gender] = None
    birth_date: Optional[date] = None
    phone: Optional[str] = None
    
    # Nested model
    address: Optional[Address] = None
    
    # Collections
    tags: List[str] = []
    preferences: Dict[str, Any] = {}
    
    # Computed field
    @property
    def is_adult(self) -> bool:
        if not self.birth_date:
            return None
        today = date.today()
        age = today.year - self.birth_date.year
        return age >= 18


# ทดสอบ nested model
import uuid

profile = UserProfile(
    id=uuid.uuid4(),
    username="alice_th",
    email="alice@example.com",
    full_name="Alice Smith",
    gender=Gender.FEMALE,
    birth_date=date(1995, 6, 15),
    address=Address(
        street="123 Main St",
        city="Bangkok",
        state="BKK",
        zip_code="10110"
    ),
    tags=["python", "developer"],
    preferences={"theme": "dark", "language": "th"}
)

print(f"\nProfile: {profile.username}")
print(f"Address: {profile.address.city}, {profile.address.country}")
print(f"Is adult: {profile.is_adult}")
```

---

## 2. Field Validators

```python
from pydantic import BaseModel, Field, field_validator, ValidationInfo
from typing import Optional
import re

# === Field ที่มี Constraints ===
class ProductCreate(BaseModel):
    name: str = Field(
        min_length=2,
        max_length=200,
        description="Product name"
    )
    price: Decimal = Field(
        gt=0,           # greater than
        le=1000000,     # less than or equal
        decimal_places=2,
        description="Price in THB"
    )
    quantity: int = Field(
        ge=0,           # greater than or equal
        description="Stock quantity"
    )
    sku: str = Field(
        pattern=r"^[A-Z]{3}-\d{6}$",  # regex pattern
        description="SKU format: ABC-123456"
    )
    category: str = Field(min_length=1)
    discount: float = Field(default=0.0, ge=0, le=100)


# === @field_validator ===
class UserRegistration(BaseModel):
    username: str
    email: EmailStr
    password: str
    confirm_password: str
    age: int
    phone: Optional[str] = None
    website: Optional[str] = None
    
    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        """Username: 3-50 chars, alphanumeric + underscore"""
        v = v.strip().lower()
        
        if not re.match(r'^[a-z0-9_]{3,50}$', v):
            raise ValueError(
                "Username must be 3-50 characters, "
                "lowercase letters, numbers, and underscores only"
            )
        
        # ตรวจสอบ reserved words
        reserved = {"admin", "root", "system", "null", "undefined"}
        if v in reserved:
            raise ValueError(f"Username '{v}' is reserved")
        
        return v
    
    @field_validator("password")
    @classmethod
    def validate_password(cls, v: str) -> str:
        """Password ต้องมี uppercase, lowercase, digit, special char"""
        errors = []
        
        if len(v) < 8:
            errors.append("at least 8 characters")
        if not re.search(r'[A-Z]', v):
            errors.append("at least one uppercase letter")
        if not re.search(r'[a-z]', v):
            errors.append("at least one lowercase letter")
        if not re.search(r'\d', v):
            errors.append("at least one digit")
        if not re.search(r'[!@#$%^&*(),.?":{}|<>]', v):
            errors.append("at least one special character")
        
        if errors:
            raise ValueError(f"Password must contain: {', '.join(errors)}")
        
        return v
    
    @field_validator("age")
    @classmethod
    def validate_age(cls, v: int) -> int:
        if v < 13:
            raise ValueError("Must be at least 13 years old")
        if v > 120:
            raise ValueError("Invalid age")
        return v
    
    @field_validator("phone")
    @classmethod
    def validate_phone(cls, v: Optional[str]) -> Optional[str]:
        if v is None:
            return v
        
        # ลบ spaces, dashes
        cleaned = re.sub(r'[\s\-\(\)]', '', v)
        
        # ตรวจสอบเบอร์ไทย
        if not re.match(r'^(?:\+66|0)[689]\d{8}$', cleaned):
            raise ValueError("Invalid Thai phone number format")
        
        # Format: 0XX-XXX-XXXX
        if cleaned.startswith('0'):
            return f"{cleaned[:3]}-{cleaned[3:6]}-{cleaned[6:]}"
        elif cleaned.startswith('+66'):
            local = '0' + cleaned[3:]
            return f"{local[:3]}-{local[3:6]}-{local[6:]}"
        
        return cleaned
    
    @field_validator("website")
    @classmethod
    def validate_website(cls, v: Optional[str]) -> Optional[str]:
        if v is None:
            return v
        
        if not v.startswith(("http://", "https://")):
            v = f"https://{v}"
        
        url_pattern = r'^https?://[\w\-]+(\.[\w\-]+)+([\w.,@?^=%&:/~+#\-]*[\w@?^=%&/~+#\-])?$'
        if not re.match(url_pattern, v):
            raise ValueError("Invalid URL format")
        
        return v


# ทดสอบ validators
from pydantic import ValidationError

def test_registration():
    # Valid registration
    try:
        reg = UserRegistration(
            username="alice_th",
            email="alice@example.com",
            password="SecurePass@123",
            confirm_password="SecurePass@123",
            age=25,
            phone="089-123-4567",
            website="mysite.com"
        )
        print(f"Valid: {reg.username}, phone: {reg.phone}, website: {reg.website}")
    except ValidationError as e:
        print(f"Error: {e}")
    
    # Invalid registration
    try:
        reg = UserRegistration(
            username="admin",  # Reserved
            email="not-an-email",
            password="weak",
            confirm_password="weak",
            age=10,  # Too young
        )
    except ValidationError as e:
        print(f"\nValidation errors ({e.error_count()}):")
        for error in e.errors():
            print(f"  [{error['loc']}] {error['msg']}")


test_registration()
```

---

## 3. Model Validators (Cross-field)

```python
from pydantic import BaseModel, field_validator, model_validator
from typing import Optional
from datetime import date
from decimal import Decimal

class OrderCreate(BaseModel):
    product_id: int
    quantity: int = Field(gt=0)
    unit_price: Decimal = Field(gt=0)
    discount_amount: Decimal = Field(default=Decimal("0"), ge=0)
    total_amount: Optional[Decimal] = None
    ship_date: Optional[date] = None
    order_date: date = Field(default_factory=date.today)
    
    @model_validator(mode="before")
    @classmethod
    def pre_validate(cls, data: dict) -> dict:
        """ทำงานก่อน validate ทุก fields"""
        # Normalize data
        if "quantity" in data and isinstance(data["quantity"], str):
            data["quantity"] = int(data["quantity"].replace(",", ""))
        return data
    
    @model_validator(mode="after")
    def post_validate(self) -> "OrderCreate":
        """ทำงานหลัง validate ทุก fields (cross-field validation)"""
        
        # คำนวณ total_amount ถ้าไม่ระบุ
        if self.total_amount is None:
            self.total_amount = (
                Decimal(str(self.quantity)) * self.unit_price - self.discount_amount
            )
        
        # ตรวจสอบ total ต้อง > 0
        if self.total_amount <= 0:
            raise ValueError("Total amount must be positive")
        
        # ตรวจสอบ discount ต้องไม่เกิน subtotal
        subtotal = Decimal(str(self.quantity)) * self.unit_price
        if self.discount_amount > subtotal:
            raise ValueError(
                f"Discount ({self.discount_amount}) cannot exceed "
                f"subtotal ({subtotal})"
            )
        
        # ตรวจสอบ ship_date ต้องหลัง order_date
        if self.ship_date and self.ship_date < self.order_date:
            raise ValueError("Ship date cannot be before order date")
        
        return self


class DateRange(BaseModel):
    start_date: date
    end_date: date
    max_days: int = 365
    
    @model_validator(mode="after")
    def validate_date_range(self) -> "DateRange":
        if self.end_date < self.start_date:
            raise ValueError("end_date must be after start_date")
        
        days = (self.end_date - self.start_date).days
        if days > self.max_days:
            raise ValueError(
                f"Date range too large: {days} days (max: {self.max_days})"
            )
        
        return self
    
    @property
    def duration_days(self) -> int:
        return (self.end_date - self.start_date).days


# ทดสอบ Model Validators
def test_model_validators():
    print("=== Model Validators ===")
    
    # Valid order
    order = OrderCreate(
        product_id=1,
        quantity=5,
        unit_price=Decimal("100.00"),
        discount_amount=Decimal("50.00")
    )
    print(f"Order total: {order.total_amount}")
    
    # Discount เกิน subtotal
    try:
        OrderCreate(
            product_id=1,
            quantity=2,
            unit_price=Decimal("100.00"),
            discount_amount=Decimal("300.00")  # เกิน 2*100=200
        )
    except ValidationError as e:
        print(f"\nDiscount error: {e.errors()[0]['msg']}")
    
    # Date range
    dr = DateRange(
        start_date=date(2024, 1, 1),
        end_date=date(2024, 12, 31)
    )
    print(f"\nDate range: {dr.duration_days} days")
    
    try:
        DateRange(
            start_date=date(2024, 12, 31),
            end_date=date(2024, 1, 1)  # Before start
        )
    except ValidationError as e:
        print(f"Date range error: {e.errors()[0]['msg']}")


test_model_validators()
```

---

## 4. Custom Types

```python
from pydantic import BaseModel, GetCoreSchemaHandler
from pydantic_core import core_schema
from typing import Annotated, Any
import re

# === Annotated สำหรับ Custom Constraints ===
from pydantic import Field
from typing import Annotated

# Custom type aliases
ThaiPhone = Annotated[str, Field(pattern=r'^0[689]\d{8}$')]
PositiveDecimal = Annotated[Decimal, Field(gt=0)]
NonEmptyStr = Annotated[str, Field(min_length=1, strip_whitespace=True)]
PercentFloat = Annotated[float, Field(ge=0, le=100)]


# === Custom Type ด้วย __get_validators__ ===
class ThaiIDCard:
    """เลขบัตรประชาชนไทย (13 หลัก พร้อม validate checksum)"""
    
    def __init__(self, value: str):
        cleaned = re.sub(r'\D', '', value)
        if not self._validate(cleaned):
            raise ValueError(f"Invalid Thai ID card number: {value}")
        self.value = cleaned
    
    @classmethod
    def _validate(cls, id_number: str) -> bool:
        """ตรวจสอบ checksum ของเลขบัตรประชาชน"""
        if len(id_number) != 13:
            return False
        
        total = sum(
            int(id_number[i]) * (13 - i) 
            for i in range(12)
        )
        
        checksum = (11 - (total % 11)) % 10
        return checksum == int(id_number[12])
    
    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type: Any, handler: GetCoreSchemaHandler
    ) -> core_schema.CoreSchema:
        return core_schema.no_info_plain_validator_function(
            lambda v: cls(v) if isinstance(v, str) else v,
            serialization=core_schema.to_string_ser_schema(),
        )
    
    def __str__(self) -> str:
        v = self.value
        return f"{v[0]}-{v[1:5]}-{v[5:10]}-{v[10:12]}-{v[12]}"
    
    def __repr__(self) -> str:
        return f"ThaiIDCard('{self}')"


# ทดสอบ Custom Type
class ThaiCitizen(BaseModel):
    name: str
    # id_card: ThaiIDCard  # Custom type
    phone: Optional[str] = None


citizen = ThaiCitizen(name="สมชาย ใจดี")
print(f"Citizen: {citizen}")
```

---

## 5. Serialization

```python
from pydantic import BaseModel, Field, computed_field, field_serializer
from typing import Optional
import json
from datetime import datetime

# === Model Serialization ===
class Article(BaseModel):
    id: int
    title: str
    content: str
    author_id: int
    tags: list[str] = []
    published_at: Optional[datetime] = None
    is_published: bool = False
    
    # Computed field (ไม่ต้อง pass ตอนสร้าง)
    @computed_field
    @property
    def word_count(self) -> int:
        return len(self.content.split())
    
    @computed_field
    @property
    def reading_time_minutes(self) -> int:
        return max(1, self.word_count // 200)  # 200 words per minute
    
    @field_serializer("published_at")
    def serialize_published_at(self, dt: Optional[datetime]) -> Optional[str]:
        """Custom serializer สำหรับ datetime"""
        if dt is None:
            return None
        return dt.strftime("%Y-%m-%d %H:%M:%S")


# ทดสอบ Serialization
article = Article(
    id=1,
    title="Python Tips",
    content="Python is great. " * 100,
    author_id=42,
    tags=["python", "tutorial"],
    published_at=datetime.now(),
    is_published=True
)

print(f"Word count: {article.word_count}")
print(f"Reading time: {article.reading_time_minutes} min")

# Serialize เป็น dict
article_dict = article.model_dump()
print(f"\nDict keys: {list(article_dict.keys())}")

# Serialize เป็น JSON
article_json = article.model_dump_json(indent=2)
print(f"\nJSON (first 200 chars): {article_json[:200]}...")

# Exclude fields
minimal_dict = article.model_dump(
    exclude={"content"},
    exclude_none=True
)
print(f"\nMinimal dict: {minimal_dict}")

# Include only specified fields
summary_dict = article.model_dump(include={"id", "title", "word_count", "reading_time_minutes"})
print(f"\nSummary: {summary_dict}")


# === Deserialization ===
# จาก dict
data = {
    "id": 2,
    "title": "Advanced Python",
    "content": "Deep dive into Python internals.",
    "author_id": 1,
    "published_at": "2024-01-15 10:00:00"
}

article2 = Article.model_validate(data)
print(f"\nFrom dict: {article2.title}")

# จาก JSON string
json_str = '{"id": 3, "title": "FastAPI Guide", "content": "Building APIs with FastAPI.", "author_id": 1}'
article3 = Article.model_validate_json(json_str)
print(f"From JSON: {article3.title}")
```

---

## 6. Nested Models และ Relationships

```python
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime
from decimal import Decimal
from enum import Enum

# === E-commerce Models ===
class ProductCategory(str, Enum):
    ELECTRONICS = "electronics"
    CLOTHING = "clothing"
    BOOKS = "books"
    FOOD = "food"

class Address(BaseModel):
    street: str
    city: str
    postal_code: str
    country: str = "TH"

class Product(BaseModel):
    id: int
    name: str
    sku: str
    price: Decimal = Field(gt=0)
    category: ProductCategory
    in_stock: bool = True
    
    @computed_field
    @property
    def price_with_vat(self) -> Decimal:
        return self.price * Decimal("1.07")

class OrderItem(BaseModel):
    product: Product
    quantity: int = Field(gt=0)
    unit_price: Decimal
    
    @computed_field
    @property
    def subtotal(self) -> Decimal:
        return self.unit_price * Decimal(str(self.quantity))

class Customer(BaseModel):
    id: int
    name: str
    email: EmailStr
    phone: Optional[str] = None
    shipping_address: Address
    billing_address: Optional[Address] = None
    
    def get_billing_address(self) -> Address:
        return self.billing_address or self.shipping_address

class OrderStatus(str, Enum):
    PENDING = "pending"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class Order(BaseModel):
    id: int
    customer: Customer
    items: List[OrderItem] = Field(min_length=1)
    status: OrderStatus = OrderStatus.PENDING
    created_at: datetime = Field(default_factory=datetime.now)
    notes: Optional[str] = None
    
    @computed_field
    @property
    def subtotal(self) -> Decimal:
        return sum(item.subtotal for item in self.items)
    
    @computed_field
    @property
    def vat_amount(self) -> Decimal:
        return self.subtotal * Decimal("0.07")
    
    @computed_field
    @property
    def total(self) -> Decimal:
        return self.subtotal + self.vat_amount
    
    @computed_field
    @property
    def item_count(self) -> int:
        return sum(item.quantity for item in self.items)
    
    def can_cancel(self) -> bool:
        return self.status in [OrderStatus.PENDING, OrderStatus.PAID]
    
    def get_shipping_address(self) -> Address:
        return self.customer.get_billing_address()


# ทดสอบ Nested Models
def test_order():
    customer = Customer(
        id=1,
        name="สมชาย ใจดี",
        email="somchai@example.com",
        phone="089-123-4567",
        shipping_address=Address(
            street="123 ถ.สุขุมวิท",
            city="กรุงเทพฯ",
            postal_code="10110"
        )
    )
    
    products = [
        Product(id=1, name="Laptop", sku="LAP-001", price=Decimal("35000"), 
                category=ProductCategory.ELECTRONICS),
        Product(id=2, name="Mouse", sku="MOU-001", price=Decimal("800"),
                category=ProductCategory.ELECTRONICS),
    ]
    
    order = Order(
        id=1001,
        customer=customer,
        items=[
            OrderItem(product=products[0], quantity=1, unit_price=Decimal("35000")),
            OrderItem(product=products[1], quantity=2, unit_price=Decimal("800")),
        ]
    )
    
    print(f"Order #{order.id}")
    print(f"Customer: {order.customer.name}")
    print(f"Items: {order.item_count}")
    print(f"Subtotal: {order.subtotal:,.2f}")
    print(f"VAT: {order.vat_amount:,.2f}")
    print(f"Total: {order.total:,.2f}")
    print(f"Can cancel: {order.can_cancel()}")
    
    # Serialize
    order_dict = order.model_dump()
    print(f"\nOrder JSON fields: {list(order_dict.keys())}")
    
    # ไม่รวม computed fields
    order_dict_no_computed = order.model_dump(exclude={"subtotal", "vat_amount", "total", "item_count"})
    
    return order


order = test_order()
```

---

## 7. Response Models สำหรับ API

```python
from pydantic import BaseModel, Field
from typing import Optional, List, Generic, TypeVar, Any
from datetime import datetime

T = TypeVar("T")

# === Generic Response Models ===
class APIResponse(BaseModel, Generic[T]):
    """Standard API response wrapper"""
    success: bool = True
    message: str = "OK"
    data: Optional[T] = None
    errors: Optional[List[str]] = None
    timestamp: datetime = Field(default_factory=datetime.now)
    
    @classmethod
    def ok(cls, data: T, message: str = "OK") -> "APIResponse[T]":
        return cls(success=True, message=message, data=data)
    
    @classmethod
    def error(cls, errors: List[str], message: str = "Error") -> "APIResponse[T]":
        return cls(success=False, message=message, errors=errors)


class PaginatedResponse(BaseModel, Generic[T]):
    """Paginated list response"""
    items: List[T]
    total: int
    page: int
    per_page: int
    total_pages: int
    has_next: bool
    has_prev: bool
    
    @classmethod
    def create(
        cls,
        items: List[T],
        total: int,
        page: int,
        per_page: int
    ) -> "PaginatedResponse[T]":
        total_pages = (total + per_page - 1) // per_page
        return cls(
            items=items,
            total=total,
            page=page,
            per_page=per_page,
            total_pages=total_pages,
            has_next=page < total_pages,
            has_prev=page > 1
        )


# === Request/Response DTOs ===
class UserCreate(BaseModel):
    """Request body สำหรับ create user"""
    username: str = Field(min_length=3, max_length=50)
    email: EmailStr
    password: str = Field(min_length=8)
    full_name: Optional[str] = None

class UserUpdate(BaseModel):
    """Request body สำหรับ update user (all optional)"""
    full_name: Optional[str] = None
    email: Optional[EmailStr] = None
    phone: Optional[str] = None

class UserResponse(BaseModel):
    """Response ที่ไม่มี sensitive data"""
    id: int
    username: str
    email: str
    full_name: Optional[str] = None
    is_active: bool
    created_at: datetime
    
    model_config = {"from_attributes": True}  # สำหรับ ORM objects


# ทดสอบ Response Models
def demo_responses():
    # Single item response
    user_data = UserResponse(
        id=1,
        username="alice",
        email="alice@example.com",
        full_name="Alice Smith",
        is_active=True,
        created_at=datetime.now()
    )
    
    response = APIResponse.ok(data=user_data, message="User retrieved")
    print(f"Response: {response.model_dump_json(indent=2)[:300]}")
    
    # Paginated response
    users = [
        UserResponse(id=i, username=f"user{i}", email=f"user{i}@example.com",
                    is_active=True, created_at=datetime.now())
        for i in range(1, 6)
    ]
    
    paginated = PaginatedResponse.create(
        items=users,
        total=47,
        page=1,
        per_page=5
    )
    
    print(f"\nPaginated: page {paginated.page}/{paginated.total_pages}")
    print(f"Has next: {paginated.has_next}, Has prev: {paginated.has_prev}")
    
    # Error response
    error_response = APIResponse[UserResponse].error(
        errors=["Email already exists", "Username taken"],
        message="Validation failed"
    )
    print(f"\nError: {error_response.success}, {error_response.errors}")


demo_responses()
```

---

## 8. Model Configuration

```python
from pydantic import BaseModel, ConfigDict, field_validator
from typing import Optional

# === Model Config ===
class StrictModel(BaseModel):
    model_config = ConfigDict(
        # Validation
        strict=True,              # ไม่ coerce types (ส่ง str เพื่อ int จะ error)
        
        # Extra fields
        extra="forbid",           # ไม่อนุญาต extra fields (error)
        # extra="ignore"          # ไม่สนใจ extra fields (ลบทิ้ง)
        # extra="allow"           # อนุญาต extra fields
        
        # ORM
        from_attributes=True,     # สร้างจาก ORM objects
        
        # Serialization
        populate_by_name=True,    # ใช้ field name หรือ alias ก็ได้
        
        # String options
        str_strip_whitespace=True,  # Strip whitespace จาก strings
        str_min_length=1,           # Minimum string length
        
        # Validation
        validate_default=True,   # Validate default values ด้วย
        validate_assignment=True, # Validate เมื่อ assign ค่า
        
        # Frozen (immutable)
        frozen=False,             # True = immutable model
    )
    
    name: str
    value: int


# ทดสอบ strict mode
try:
    m = StrictModel(name="test", value=42)
    print(f"Valid: {m}")
    
    # Extra field ถูก forbid
    m2 = StrictModel(name="test", value=42, extra_field="not allowed")
except Exception as e:
    print(f"Error: {e}")


# === Frozen Model (Immutable) ===
class ImmutablePoint(BaseModel):
    model_config = ConfigDict(frozen=True)
    
    x: float
    y: float
    
    def distance_to_origin(self) -> float:
        import math
        return math.sqrt(self.x ** 2 + self.y ** 2)
    
    def translate(self, dx: float, dy: float) -> "ImmutablePoint":
        """คืน point ใหม่แทนที่จะแก้ไขของเดิม"""
        return ImmutablePoint(x=self.x + dx, y=self.y + dy)


p1 = ImmutablePoint(x=3, y=4)
print(f"\nPoint: ({p1.x}, {p1.y})")
print(f"Distance: {p1.distance_to_origin()}")

p2 = p1.translate(1, 1)
print(f"Translated: ({p2.x}, {p2.y})")
print(f"Original unchanged: ({p1.x}, {p1.y})")

try:
    p1.x = 10  # จะ error
except Exception as e:
    print(f"Immutable error: {e}")
```

---

## 9. สรุป Part 049

✅ **BaseModel** - type validation, coercion, nested models  
✅ **Field** - constraints, min/max, regex, descriptions  
✅ **field_validator** - custom validation logic  
✅ **model_validator** - cross-field validation  
✅ **Custom Types** - Annotated, custom classes  
✅ **computed_field** - computed properties  
✅ **Serialization** - model_dump, model_dump_json  
✅ **Deserialization** - model_validate, model_validate_json  
✅ **Generic Models** - APIResponse, PaginatedResponse  
✅ **Model Config** - strict mode, frozen, ORM mode  

**Use Cases:**
- API request/response validation
- Configuration management
- Data parsing and transformation
- Form validation
- Database model serialization

## ➡️ ถัดไป: Part 050 - Python Best Practices Review
*Part 049/100+ | Python Course - Beginner to World-Class*
