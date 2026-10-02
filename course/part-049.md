# Part 049: Data Validation with Pydantic v2
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ `BaseModel`, `Field`, `model_config` ใน Pydantic v2
- ใช้ `field_validator` และ `model_validator` สร้าง custom validation
- สร้าง Custom Types และ Annotated validators
- สร้าง Nested models และ relationships
- ใช้ `model_dump()` และ `model_validate()` อย่างถูกต้อง
- จัดการ Configuration ด้วย `pydantic-settings`
- ใช้งานร่วมกับ FastAPI
- Pattern การตรวจสอบ email, phone, URL

---

## 1. BaseModel และ Field พื้นฐาน

```bash
# ติดตั้ง Pydantic v2
pip install pydantic

# สำหรับ Settings
pip install pydantic-settings

# สำหรับ email validation
pip install "pydantic[email]"
```

### 1.1 BaseModel พื้นฐาน

```python
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime, date

# === BaseModel พื้นฐาน ===
class User(BaseModel):
    """โมเดลผู้ใช้งาน"""
    id: int
    username: str
    email: str
    age: int
    is_active: bool = True   # ค่า default

# สร้าง instance
user = User(
    id=1,
    username="somchai",
    email="somchai@example.com",
    age=30
)

print(user)                          # User(id=1, username='somchai', ...)
print(user.model_dump())             # {'id': 1, 'username': 'somchai', ...}
print(user.model_dump_json())        # '{"id": 1, "username": "somchai", ...}'

# Type coercion อัตโนมัติ
user2 = User(
    id="5",          # str → int (coerce)
    username="test",
    email="test@example.com",
    age="25"         # str → int (coerce)
)
print(f"id type: {type(user2.id)}")   # <class 'int'>
```

### 1.2 Field สำหรับ Validation และ Metadata

```python
from pydantic import BaseModel, Field
from typing import Optional, List
from decimal import Decimal

class Product(BaseModel):
    """โมเดลสินค้าพร้อม Field constraints"""
    
    id: int = Field(
        gt=0,                      # greater than 0
        description="Product ID"
    )
    name: str = Field(
        min_length=2,              # ความยาวขั้นต่ำ
        max_length=200,            # ความยาวสูงสุด
        description="ชื่อสินค้า"
    )
    description: Optional[str] = Field(
        default=None,
        max_length=2000
    )
    price: Decimal = Field(
        gt=0,                      # ราคาต้องมากกว่า 0
        decimal_places=2,          # ทศนิยม 2 ตำแหน่ง
        description="ราคาสินค้า"
    )
    stock: int = Field(
        ge=0,                      # >= 0
        default=0,
        description="จำนวนสินค้าคงคลัง"
    )
    discount_percent: float = Field(
        default=0.0,
        ge=0.0,                    # >= 0
        le=100.0,                  # <= 100
        description="เปอร์เซ็นต์ส่วนลด"
    )
    tags: List[str] = Field(
        default_factory=list,      # ใช้ factory สำหรับ mutable defaults
        max_length=20,             # สูงสุด 20 tags
        description="Tags ของสินค้า"
    )
    sku: str = Field(
        pattern=r"^[A-Z]{2}-\d{6}$",  # regex pattern
        description="SKU (เช่น AB-123456)"
    )
    
    # Field aliases
    internal_code: str = Field(
        alias="internalCode",      # ชื่อที่ใช้รับข้อมูลจาก JSON
        default="",
        exclude=True               # ไม่รวมใน model_dump()
    )

# ทดสอบ
try:
    product = Product(
        id=1,
        name="สินค้า A",
        price="99.99",
        sku="AB-123456",
        internalCode="INT-001"
    )
    print(product.model_dump())
except Exception as e:
    print(f"Validation error: {e}")
```

### 1.3 model_config

```python
from pydantic import BaseModel, ConfigDict, Field
from typing import Optional

# === model_config ใน Pydantic v2 ===
class UserConfig(BaseModel):
    model_config = ConfigDict(
        # Strict mode — ไม่อนุญาต coercion
        strict=False,
        
        # ตรวจสอบ assignment
        validate_assignment=True,
        
        # populate_by_name — ใช้ชื่อจริงได้แม้มี alias
        populate_by_name=True,
        
        # frozen — ทำให้ immutable
        frozen=False,
        
        # extra fields behavior
        extra="ignore",            # "ignore", "allow", "forbid"
        
        # การแปลง str
        str_strip_whitespace=True, # trim whitespace อัตโนมัติ
        str_min_length=1,          # min length สำหรับทุก str fields
        
        # JSON encoding
        json_encoders={            # custom serializers
            # datetime: lambda v: v.isoformat()
        },
        
        # Schema metadata
        title="User Configuration",
        description="โมเดลสำหรับ user configuration",
    )
    
    user_id: int = Field(alias="userId")
    full_name: str
    email: str

# ทดสอบ validate_assignment
user = UserConfig(userId=1, full_name="สมชาย", email="test@example.com")
user.full_name = "   สมหญิง   "  # whitespace จะถูก strip
print(user.full_name)  # "สมหญิง"

# extra="forbid" — ห้ามส่ง field ที่ไม่รู้จัก
class StrictModel(BaseModel):
    model_config = ConfigDict(extra="forbid")
    name: str
    age: int

try:
    obj = StrictModel(name="test", age=25, extra_field="ห้ามใส่")
except Exception as e:
    print(f"Error: {e}")  # จะ error เพราะ extra_field ไม่ได้ defined

# frozen=True — ทำให้ immutable (hashable)
class ImmutablePoint(BaseModel):
    model_config = ConfigDict(frozen=True)
    x: float
    y: float

point = ImmutablePoint(x=1.0, y=2.0)
try:
    point.x = 3.0  # จะ error
except Exception as e:
    print(f"Immutable error: {e}")

# สามารถใช้เป็น dict key ได้
point_dict = {point: "ตำแหน่ง A"}
print(point_dict[ImmutablePoint(x=1.0, y=2.0)])
```

---

## 2. Validators

### 2.1 field_validator

```python
from pydantic import BaseModel, field_validator, Field
from typing import Optional
import re

class UserRegistration(BaseModel):
    """โมเดล user registration พร้อม validators"""
    
    username: str = Field(min_length=3, max_length=50)
    email: str
    password: str
    confirm_password: str
    age: int = Field(gt=0, lt=150)
    phone: Optional[str] = None
    website: Optional[str] = None
    
    # === @field_validator ===
    @field_validator("username")
    @classmethod
    def username_must_be_alphanumeric(cls, v: str) -> str:
        """Username ต้องเป็น alphanumeric เท่านั้น"""
        if not re.match(r"^[a-zA-Z0-9_]+$", v):
            raise ValueError("Username ต้องประกอบด้วยตัวอักษร ตัวเลข และ _ เท่านั้น")
        return v.lower()  # บังคับ lowercase
    
    @field_validator("email")
    @classmethod
    def email_must_be_valid(cls, v: str) -> str:
        """ตรวจสอบ email format"""
        v = v.lower().strip()
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        if not re.match(pattern, v):
            raise ValueError(f"'{v}' ไม่ใช่ email ที่ถูกต้อง")
        return v
    
    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        """ตรวจสอบความซับซ้อนของรหัสผ่าน"""
        errors = []
        if len(v) < 8:
            errors.append("ต้องมีอย่างน้อย 8 ตัวอักษร")
        if not re.search(r"[A-Z]", v):
            errors.append("ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว")
        if not re.search(r"[a-z]", v):
            errors.append("ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว")
        if not re.search(r"\d", v):
            errors.append("ต้องมีตัวเลขอย่างน้อย 1 ตัว")
        if not re.search(r"[!@#$%^&*(),.?\":{}|<>]", v):
            errors.append("ต้องมีอักขระพิเศษอย่างน้อย 1 ตัว")
        
        if errors:
            raise ValueError("รหัสผ่านไม่ผ่านเงื่อนไข: " + ", ".join(errors))
        return v
    
    @field_validator("phone")
    @classmethod
    def phone_format(cls, v: Optional[str]) -> Optional[str]:
        """ตรวจสอบ format เบอร์โทรศัพท์ไทย"""
        if v is None:
            return v
        
        # ลบ -, (, ), space
        cleaned = re.sub(r"[\s\-\(\)]", "", v)
        
        # เบอร์มือถือไทย: 08x, 09x หรือ +668x, +669x
        patterns = [
            r"^0[689]\d{8}$",          # 10 หลัก เริ่มต้น 06,07,08,09
            r"^\+66[689]\d{8}$",       # +66 format
            r"^66[689]\d{8}$",         # 66 format (ไม่มี +)
        ]
        
        if not any(re.match(p, cleaned) for p in patterns):
            raise ValueError(f"'{v}' ไม่ใช่เบอร์โทรศัพท์ไทยที่ถูกต้อง")
        
        # Normalize เป็น +66 format
        if cleaned.startswith("0"):
            return "+66" + cleaned[1:]
        elif cleaned.startswith("66"):
            return "+" + cleaned
        return cleaned
    
    # mode="before" — validator รันก่อน type coercion
    @field_validator("age", mode="before")
    @classmethod
    def parse_age(cls, v) -> int:
        """รับ string "25" หรือ "25 ปี" แล้วแปลงเป็น int"""
        if isinstance(v, str):
            # ลบ "ปี", whitespace, etc.
            cleaned = re.sub(r"[^\d]", "", v)
            if not cleaned:
                raise ValueError("อายุต้องเป็นตัวเลข")
            return int(cleaned)
        return v

# ทดสอบ
try:
    user = UserRegistration(
        username="Somchai_123",
        email="  SOMCHAI@EXAMPLE.COM  ",
        password="SecurePass1!",
        confirm_password="SecurePass1!",
        age="30 ปี",
        phone="081-234-5678"
    )
    print("สร้าง user สำเร็จ:")
    print(f"  username: {user.username}")
    print(f"  email:    {user.email}")
    print(f"  age:      {user.age}")
    print(f"  phone:    {user.phone}")
except Exception as e:
    print(f"Validation error:\n{e}")
```

### 2.2 model_validator

```python
from pydantic import BaseModel, model_validator, field_validator, Field
from typing import Optional, Self
from datetime import date

class DateRange(BaseModel):
    """โมเดลช่วงวันที่ — ต้องตรวจสอบ cross-field"""
    
    start_date: date
    end_date: date
    name: str
    
    # model_validator รันหลังจาก fields ทั้งหมดถูก validate
    @model_validator(mode="after")
    def check_date_range(self) -> "DateRange":
        """ตรวจสอบว่า start_date < end_date"""
        if self.start_date >= self.end_date:
            raise ValueError(
                f"start_date ({self.start_date}) ต้องน้อยกว่า end_date ({self.end_date})"
            )
        return self
    
    @property
    def duration_days(self) -> int:
        return (self.end_date - self.start_date).days

class OrderModel(BaseModel):
    """Order ที่ต้องตรวจสอบหลาย fields ร่วมกัน"""
    
    order_id: str
    customer_name: str
    items: list[dict]
    discount_amount: float = 0.0
    coupon_code: Optional[str] = None
    payment_method: str  # "credit_card", "bank_transfer", "cash_on_delivery"
    credit_card_last4: Optional[str] = None
    
    # mode="before" — validator รันก่อน fields ถูก validate
    @model_validator(mode="before")
    @classmethod
    def set_defaults(cls, data: dict) -> dict:
        """ตั้งค่า defaults ก่อน validation"""
        if isinstance(data, dict):
            # Generate order_id ถ้าไม่มี
            if "order_id" not in data or not data["order_id"]:
                import uuid
                data["order_id"] = f"ORD-{uuid.uuid4().hex[:8].upper()}"
        return data
    
    @model_validator(mode="after")
    def validate_payment_details(self) -> "OrderModel":
        """ตรวจสอบว่า payment method มีข้อมูลที่ต้องการ"""
        if self.payment_method == "credit_card":
            if not self.credit_card_last4:
                raise ValueError("Credit card payment ต้องระบุ credit_card_last4")
            if not re.match(r"^\d{4}$", self.credit_card_last4):
                raise ValueError("credit_card_last4 ต้องเป็นตัวเลข 4 หลัก")
        
        if self.coupon_code and self.discount_amount <= 0:
            raise ValueError("ถ้ามี coupon_code ต้องมี discount_amount > 0")
        
        if self.items and len(self.items) == 0:
            raise ValueError("Order ต้องมีอย่างน้อย 1 item")
        
        return self

# ทดสอบ
import re

# ทดสอบ DateRange
try:
    dr = DateRange(
        start_date=date(2024, 1, 1),
        end_date=date(2024, 12, 31),
        name="ปี 2024"
    )
    print(f"Duration: {dr.duration_days} วัน")
except Exception as e:
    print(f"Error: {e}")

# ทดสอบ OrderModel
try:
    order = OrderModel(
        customer_name="สมชาย",
        items=[{"product": "สินค้า A", "qty": 2}],
        payment_method="credit_card",
        credit_card_last4="1234"
    )
    print(f"Order ID: {order.order_id}")
except Exception as e:
    print(f"Error: {e}")
```

---

## 3. Custom Types และ Annotated Validators

### 3.1 Custom Types ด้วย Annotated

```python
from pydantic import BaseModel, field_validator, GetCoreSchemaHandler
from pydantic_core import core_schema
from typing import Annotated, Any
import re

# === วิธีที่ 1: Annotated + AfterValidator ===
from pydantic.functional_validators import AfterValidator, BeforeValidator, PlainValidator

def validate_thai_id(v: str) -> str:
    """ตรวจสอบเลขบัตรประชาชนไทย 13 หลัก"""
    # ลบ - และ space
    cleaned = re.sub(r"[\s\-]", "", v)
    
    if not re.match(r"^\d{13}$", cleaned):
        raise ValueError("เลขบัตรประชาชนต้องเป็นตัวเลข 13 หลัก")
    
    # ตรวจสอบ check digit
    total = 0
    for i, digit in enumerate(cleaned[:12]):
        total += int(digit) * (13 - i)
    
    check_digit = (11 - (total % 11)) % 10
    if int(cleaned[12]) != check_digit:
        raise ValueError("เลขบัตรประชาชนไม่ถูกต้อง (check digit ไม่ตรง)")
    
    return cleaned

def validate_url(v: str) -> str:
    """ตรวจสอบ URL"""
    from urllib.parse import urlparse
    parsed = urlparse(v)
    if not all([parsed.scheme in ("http", "https"), parsed.netloc]):
        raise ValueError(f"'{v}' ไม่ใช่ URL ที่ถูกต้อง")
    return v.lower()

def normalize_email(v: str) -> str:
    """ทำความสะอาด email"""
    return v.lower().strip()

# สร้าง Custom Types
ThaiNationalID = Annotated[str, AfterValidator(validate_thai_id)]
ValidURL = Annotated[str, AfterValidator(validate_url)]
NormalizedEmail = Annotated[str, BeforeValidator(normalize_email)]

# === วิธีที่ 2: Custom Type Class ===
class ThaiPhoneNumber(str):
    """Custom type สำหรับเบอร์โทรศัพท์ไทย"""
    
    @classmethod
    def __get_validators__(cls):
        yield cls.validate
    
    @classmethod
    def validate(cls, v: Any) -> "ThaiPhoneNumber":
        if not isinstance(v, str):
            raise TypeError("Phone number ต้องเป็น string")
        
        cleaned = re.sub(r"[\s\-\(\)]", "", v)
        patterns = [
            r"^0[689]\d{8}$",
            r"^\+66[689]\d{8}$",
        ]
        
        if not any(re.match(p, cleaned) for p in patterns):
            raise ValueError(f"'{v}' ไม่ใช่เบอร์โทรศัพท์ไทยที่ถูกต้อง")
        
        # Normalize
        if cleaned.startswith("0"):
            return cls("+66" + cleaned[1:])
        return cls(cleaned)
    
    @classmethod
    def __get_pydantic_core_schema__(
        cls,
        source_type: Any,
        handler: GetCoreSchemaHandler
    ) -> core_schema.CoreSchema:
        return core_schema.no_info_plain_validator_function(
            cls.validate,
            serialization=core_schema.to_string_ser_schema(),
        )

# ใช้งาน Custom Types
class PersonProfile(BaseModel):
    """Profile ที่ใช้ custom types"""
    
    name: str
    email: NormalizedEmail
    phone: ThaiPhoneNumber
    national_id: ThaiNationalID
    website: Optional[ValidURL] = None
    
    model_config = ConfigDict(arbitrary_types_allowed=True)

from pydantic import ConfigDict

class PersonProfile(BaseModel):
    model_config = ConfigDict(arbitrary_types_allowed=True)
    
    name: str
    email: NormalizedEmail
    phone: ThaiPhoneNumber

# ทดสอบ
try:
    person = PersonProfile(
        name="สมชาย ใจดี",
        email="  SOMCHAI@GMAIL.COM  ",
        phone="081-234-5678",
    )
    print(f"Email: {person.email}")   # somchai@gmail.com
    print(f"Phone: {person.phone}")   # +66812345678
except Exception as e:
    print(f"Error: {e}")
```

### 3.2 Pydantic Validators สำหรับ Common Patterns

```python
from pydantic import BaseModel, EmailStr, HttpUrl, AnyUrl
from pydantic import field_validator, model_validator
from typing import Optional, Annotated
import re

# ใช้ pydantic[email] สำหรับ EmailStr
# pip install "pydantic[email]"

class ContactInfo(BaseModel):
    """ข้อมูลติดต่อพร้อม built-in validators"""
    
    # EmailStr ตรวจสอบ email format โดยอัตโนมัติ
    email: EmailStr
    
    # HttpUrl ตรวจสอบ HTTP/HTTPS URL
    website: Optional[HttpUrl] = None
    
    # AnyUrl รับทุก URL scheme
    profile_url: Optional[AnyUrl] = None

# === Regex-based validators ===
PostalCode = Annotated[
    str,
    Field(pattern=r"^\d{5}$", description="รหัสไปรษณีย์ไทย 5 หลัก")
]

CreditCardNumber = Annotated[
    str,
    Field(pattern=r"^\d{16}$", description="หมายเลขบัตร 16 หลัก")
]

class PaymentInfo(BaseModel):
    """ข้อมูลการชำระเงิน"""
    
    card_number: str
    cvv: str = Field(pattern=r"^\d{3,4}$")
    expiry_month: int = Field(ge=1, le=12)
    expiry_year: int = Field(ge=2024, le=2040)
    cardholder_name: str = Field(min_length=2, max_length=100)
    billing_postal: PostalCode
    
    @field_validator("card_number")
    @classmethod
    def validate_card_number(cls, v: str) -> str:
        """ตรวจสอบด้วย Luhn algorithm"""
        v = re.sub(r"\s", "", v)  # ลบ space
        
        if not v.isdigit():
            raise ValueError("หมายเลขบัตรต้องเป็นตัวเลขเท่านั้น")
        
        if len(v) not in (13, 14, 15, 16):
            raise ValueError("หมายเลขบัตรต้องมี 13-16 หลัก")
        
        # Luhn check
        total = 0
        for i, digit in enumerate(reversed(v)):
            n = int(digit)
            if i % 2 == 1:
                n *= 2
                if n > 9:
                    n -= 9
            total += n
        
        if total % 10 != 0:
            raise ValueError("หมายเลขบัตรไม่ผ่าน Luhn check")
        
        return v
    
    @model_validator(mode="after")
    def validate_expiry(self) -> "PaymentInfo":
        """ตรวจสอบวันหมดอายุ"""
        from datetime import datetime
        now = datetime.now()
        
        if (self.expiry_year < now.year or 
            (self.expiry_year == now.year and self.expiry_month < now.month)):
            raise ValueError("บัตรหมดอายุแล้ว")
        
        return self
    
    def mask_card_number(self) -> str:
        """แสดงหมายเลขบัตรแบบซ่อน"""
        return f"****-****-****-{self.card_number[-4:]}"
```

---

## 4. Nested Models และ Relationships

```python
from pydantic import BaseModel, Field, model_validator
from typing import Optional, List, Dict, Any
from datetime import datetime
from enum import Enum

# === Enums ===
class OrderStatus(str, Enum):
    pending = "pending"
    confirmed = "confirmed"
    processing = "processing"
    shipped = "shipped"
    delivered = "delivered"
    cancelled = "cancelled"

class PaymentStatus(str, Enum):
    pending = "pending"
    paid = "paid"
    failed = "failed"
    refunded = "refunded"

# === Nested Models ===
class Address(BaseModel):
    """ที่อยู่"""
    street: str = Field(min_length=5, description="ที่อยู่บ้าน/ถนน")
    district: str = Field(description="แขวง/ตำบล")
    city: str = Field(description="เขต/อำเภอ")
    province: str = Field(description="จังหวัด")
    postal_code: str = Field(pattern=r"^\d{5}$", description="รหัสไปรษณีย์")
    country: str = Field(default="Thailand")
    
    def format(self) -> str:
        """แสดงที่อยู่แบบ formatted"""
        return f"{self.street} {self.district} {self.city} {self.province} {self.postal_code}"

class ProductItem(BaseModel):
    """รายการสินค้าใน order"""
    product_id: int
    product_name: str
    quantity: int = Field(gt=0)
    unit_price: float = Field(gt=0)
    discount: float = Field(default=0.0, ge=0.0, le=100.0)
    
    @property
    def subtotal(self) -> float:
        """ราคาหลังหักส่วนลด"""
        return self.quantity * self.unit_price * (1 - self.discount / 100)

class CustomerInfo(BaseModel):
    """ข้อมูลลูกค้า"""
    customer_id: Optional[int] = None
    first_name: str = Field(min_length=1)
    last_name: str = Field(min_length=1)
    email: str
    phone: Optional[str] = None
    shipping_address: Address         # Nested model
    billing_address: Optional[Address] = None  # Optional nested model
    
    @property
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"
    
    @model_validator(mode="after")
    def set_billing_address(self) -> "CustomerInfo":
        """ถ้าไม่มี billing address ให้ใช้ shipping address"""
        if self.billing_address is None:
            self.billing_address = self.shipping_address
        return self

class Order(BaseModel):
    """Order หลัก — มี nested models หลายชั้น"""
    
    order_id: str
    created_at: datetime = Field(default_factory=datetime.now)
    status: OrderStatus = OrderStatus.pending
    payment_status: PaymentStatus = PaymentStatus.pending
    
    # Nested models
    customer: CustomerInfo
    items: List[ProductItem] = Field(min_length=1)
    
    # Optional nested
    notes: Optional[str] = None
    metadata: Dict[str, Any] = Field(default_factory=dict)
    
    @property
    def subtotal(self) -> float:
        return sum(item.subtotal for item in self.items)
    
    @property
    def item_count(self) -> int:
        return sum(item.quantity for item in self.items)
    
    @model_validator(mode="after")
    def validate_order(self) -> "Order":
        if len(self.items) == 0:
            raise ValueError("Order ต้องมีอย่างน้อย 1 item")
        return self

# === ตัวอย่างการสร้าง nested models ===
def create_sample_order():
    """สร้าง order ตัวอย่าง"""
    address_data = {
        "street": "123 ถนนสุขุมวิท",
        "district": "แขวงคลองเตย",
        "city": "เขตคลองเตย",
        "province": "กรุงเทพมหานคร",
        "postal_code": "10110"
    }
    
    order = Order(
        order_id="ORD-2024-001",
        customer={
            "first_name": "สมชาย",
            "last_name": "ใจดี",
            "email": "somchai@example.com",
            "phone": "0812345678",
            "shipping_address": address_data
        },
        items=[
            {
                "product_id": 1,
                "product_name": "Python Book",
                "quantity": 2,
                "unit_price": 450.00,
                "discount": 10.0
            },
            {
                "product_id": 2,
                "product_name": "Django T-Shirt",
                "quantity": 1,
                "unit_price": 299.00
            }
        ]
    )
    
    return order

order = create_sample_order()
print(f"Order: {order.order_id}")
print(f"Customer: {order.customer.full_name}")
print(f"Items: {order.item_count}")
print(f"Subtotal: {order.subtotal:.2f} บาท")
print(f"Shipping to: {order.customer.shipping_address.format()}")
```

---

## 5. model_dump() และ model_validate()

```python
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime
from decimal import Decimal
import json

class Product(BaseModel):
    id: int
    name: str
    price: Decimal
    tags: List[str] = []
    created_at: datetime = Field(default_factory=datetime.now)
    internal_notes: Optional[str] = Field(default=None, exclude=True)

product = Product(
    id=1,
    name="Python Book",
    price=Decimal("450.00"),
    tags=["python", "programming"],
    internal_notes="หมายเหตุภายใน — ไม่แสดงใน output"
)

# === model_dump() ===
# ทั้งหมด
dump_all = product.model_dump()
print("All fields:", dump_all)

# เฉพาะ fields ที่ระบุ
dump_include = product.model_dump(include={"id", "name", "price"})
print("Include:", dump_include)

# ยกเว้น fields ที่ระบุ
dump_exclude = product.model_dump(exclude={"tags", "created_at"})
print("Exclude:", dump_exclude)

# ยกเว้น None values
dump_no_none = product.model_dump(exclude_none=True)

# ยกเว้น default values
dump_no_default = product.model_dump(exclude_defaults=True)

# ยกเว้น unset values (fields ที่ไม่ได้ระบุตอนสร้าง)
dump_no_unset = product.model_dump(exclude_unset=True)

# nested — แบบ nested dict (default)
dump_nested = product.model_dump(mode="python")

# JSON serialization — แปลงเป็น JSON-compatible types
dump_json = product.model_dump(mode="json")
print("JSON mode:", dump_json)  # Decimal จะถูกแปลงเป็น string

# === model_dump_json() ===
json_str = product.model_dump_json()
print("JSON string:", json_str)

# Custom serialization options
json_str2 = product.model_dump_json(
    exclude={"internal_notes"},
    indent=2
)

# === model_validate() ===
# จาก dict
data = {"id": 2, "name": "Flask Book", "price": "350.00"}
product2 = Product.model_validate(data)
print(f"Validated: {product2.name}, price={product2.price}")

# จาก JSON string
json_data = '{"id": 3, "name": "Django Book", "price": "500.00"}'
product3 = Product.model_validate_json(json_data)
print(f"From JSON: {product3.name}")

# จาก ORM object (with from_attributes)
class ProductORM:
    """จำลอง ORM model"""
    def __init__(self):
        self.id = 4
        self.name = "FastAPI Book"
        self.price = Decimal("600.00")
        self.tags = ["fastapi", "async"]
        self.created_at = datetime.now()

from pydantic import ConfigDict

class ProductFromORM(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    
    id: int
    name: str
    price: Decimal
    tags: List[str] = []

orm_obj = ProductORM()
product_from_orm = ProductFromORM.model_validate(orm_obj)
print(f"From ORM: {product_from_orm.name}")
```

---

## 6. Pydantic Settings สำหรับ Configuration Management

```bash
pip install pydantic-settings
```

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field, SecretStr, AnyHttpUrl, PostgresDsn
from typing import Optional, List
from pathlib import Path

# === BaseSettings พื้นฐาน ===
class AppSettings(BaseSettings):
    """Application settings — อ่านจาก .env file และ environment variables"""
    
    model_config = SettingsConfigDict(
        env_file=".env",                # อ่านจากไฟล์นี้
        env_file_encoding="utf-8",
        env_prefix="APP_",             # prefix สำหรับ env vars
        case_sensitive=False,          # ไม่สนใจ uppercase/lowercase
        extra="ignore",                # ไม่สนใจ env vars ที่ไม่รู้จัก
    )
    
    # Application
    app_name: str = Field(default="My Application", description="ชื่อ app")
    app_version: str = Field(default="1.0.0")
    debug: bool = Field(default=False)
    secret_key: SecretStr = Field(description="Secret key สำหรับ JWT")
    
    # Server
    host: str = Field(default="0.0.0.0")
    port: int = Field(default=8000, ge=1, le=65535)
    workers: int = Field(default=4, ge=1)
    
    # Database
    database_url: Optional[str] = Field(default=None)
    db_pool_size: int = Field(default=10)
    db_max_overflow: int = Field(default=20)
    
    # Redis
    redis_url: str = Field(default="redis://localhost:6379/0")
    redis_ttl: int = Field(default=300)
    
    # Email
    smtp_host: Optional[str] = None
    smtp_port: int = Field(default=587)
    smtp_username: Optional[str] = None
    smtp_password: Optional[SecretStr] = None
    
    # CORS
    allowed_origins: List[str] = Field(default=["http://localhost:3000"])
    
    # Logging
    log_level: str = Field(default="INFO")
    log_file: Optional[Path] = None

# === การใช้งาน ===
def get_settings() -> AppSettings:
    """Factory function สำหรับ settings"""
    return AppSettings()

# Singleton pattern
_settings: Optional[AppSettings] = None

def settings() -> AppSettings:
    global _settings
    if _settings is None:
        _settings = AppSettings()
    return _settings

# === Nested Settings ===
class DatabaseSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="DB_")
    
    host: str = "localhost"
    port: int = 5432
    name: str = "mydb"
    user: str = "postgres"
    password: SecretStr = SecretStr("password")
    pool_size: int = 10
    
    @property
    def url(self) -> str:
        return (
            f"postgresql://{self.user}:{self.password.get_secret_value()}"
            f"@{self.host}:{self.port}/{self.name}"
        )

class RedisSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="REDIS_")
    
    host: str = "localhost"
    port: int = 6379
    db: int = 0
    password: Optional[SecretStr] = None
    
    @property
    def url(self) -> str:
        auth = f":{self.password.get_secret_value()}@" if self.password else ""
        return f"redis://{auth}{self.host}:{self.port}/{self.db}"

class Settings(BaseSettings):
    """Main settings ที่รวม nested settings"""
    
    app_name: str = "My App"
    debug: bool = False
    
    # Nested settings (อ่านแยกกัน)
    database: DatabaseSettings = DatabaseSettings()
    redis: RedisSettings = RedisSettings()

# ใช้งาน
app_settings = AppSettings()
print(f"App: {app_settings.app_name}")
print(f"Debug: {app_settings.debug}")
print(f"Port: {app_settings.port}")

# ตัวอย่าง .env file:
ENV_FILE_EXAMPLE = """
APP_APP_NAME=My Python App
APP_DEBUG=true
APP_SECRET_KEY=my-super-secret-key-here
APP_PORT=8080
APP_DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
APP_ALLOWED_ORIGINS=["http://localhost:3000","https://myapp.com"]
DB_HOST=db.example.com
DB_NAME=production_db
REDIS_HOST=redis.example.com
"""
```

---

## 7. Integration กับ FastAPI

```python
from fastapi import FastAPI, HTTPException, Depends, status
from pydantic import BaseModel, Field, field_validator, model_validator
from typing import Optional, List
from datetime import datetime

app = FastAPI(title="Task API", version="1.0.0")

# === Request/Response Models ===

class TaskCreate(BaseModel):
    """Schema สำหรับสร้าง task ใหม่"""
    title: str = Field(min_length=1, max_length=200, description="ชื่องาน")
    description: Optional[str] = Field(default=None, max_length=2000)
    priority: str = Field(default="medium")
    due_date: Optional[datetime] = None
    tags: List[str] = Field(default_factory=list)
    
    @field_validator("priority")
    @classmethod
    def validate_priority(cls, v: str) -> str:
        allowed = ["low", "medium", "high", "urgent"]
        if v not in allowed:
            raise ValueError(f"priority ต้องเป็นหนึ่งใน: {allowed}")
        return v
    
    @field_validator("tags")
    @classmethod
    def validate_tags(cls, v: list) -> list:
        if len(v) > 10:
            raise ValueError("tags สูงสุด 10 อัน")
        return [tag.lower().strip() for tag in v]

class TaskUpdate(BaseModel):
    """Schema สำหรับอัพเดต task — ทุก field เป็น optional"""
    title: Optional[str] = Field(default=None, min_length=1, max_length=200)
    description: Optional[str] = Field(default=None, max_length=2000)
    priority: Optional[str] = None
    status: Optional[str] = None
    due_date: Optional[datetime] = None
    tags: Optional[List[str]] = None
    
    @model_validator(mode="after")
    def check_at_least_one_field(self) -> "TaskUpdate":
        """ต้องส่งอย่างน้อย 1 field"""
        values = self.model_dump(exclude_none=True)
        if not values:
            raise ValueError("ต้องส่งอย่างน้อย 1 field สำหรับอัพเดต")
        return self

class TaskResponse(BaseModel):
    """Schema สำหรับ response"""
    id: int
    title: str
    description: Optional[str] = None
    priority: str
    status: str
    due_date: Optional[datetime] = None
    tags: List[str] = []
    created_at: datetime
    updated_at: datetime
    
    model_config = ConfigDict(from_attributes=True)

class TaskListResponse(BaseModel):
    """Schema สำหรับ list response"""
    items: List[TaskResponse]
    total: int
    page: int
    page_size: int
    has_next: bool

# === API Endpoints ===

# Fake database
tasks_db: dict[int, dict] = {}
task_counter = 0

@app.post("/tasks/", response_model=TaskResponse, status_code=status.HTTP_201_CREATED)
async def create_task(task: TaskCreate):
    """
    สร้าง task ใหม่
    
    - **title**: ชื่องาน (บังคับ)
    - **priority**: low, medium, high, urgent
    """
    global task_counter
    task_counter += 1
    
    now = datetime.now()
    task_data = {
        "id": task_counter,
        **task.model_dump(),
        "status": "todo",
        "created_at": now,
        "updated_at": now,
    }
    tasks_db[task_counter] = task_data
    
    return TaskResponse(**task_data)

@app.get("/tasks/", response_model=TaskListResponse)
async def list_tasks(
    page: int = 1,
    page_size: int = 10,
    priority: Optional[str] = None,
    status_filter: Optional[str] = None,
):
    """แสดงรายการ tasks พร้อม pagination"""
    items = list(tasks_db.values())
    
    # Filter
    if priority:
        items = [t for t in items if t["priority"] == priority]
    if status_filter:
        items = [t for t in items if t["status"] == status_filter]
    
    total = len(items)
    start = (page - 1) * page_size
    end = start + page_size
    page_items = items[start:end]
    
    return TaskListResponse(
        items=[TaskResponse(**t) for t in page_items],
        total=total,
        page=page,
        page_size=page_size,
        has_next=end < total,
    )

@app.get("/tasks/{task_id}", response_model=TaskResponse)
async def get_task(task_id: int):
    """ดู task ตาม ID"""
    if task_id not in tasks_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"ไม่พบ task #{task_id}"
        )
    return TaskResponse(**tasks_db[task_id])

@app.patch("/tasks/{task_id}", response_model=TaskResponse)
async def update_task(task_id: int, updates: TaskUpdate):
    """อัพเดต task"""
    if task_id not in tasks_db:
        raise HTTPException(status_code=404, detail=f"ไม่พบ task #{task_id}")
    
    task = tasks_db[task_id]
    
    # อัพเดตเฉพาะ fields ที่ส่งมา
    update_data = updates.model_dump(exclude_none=True)
    task.update(update_data)
    task["updated_at"] = datetime.now()
    
    return TaskResponse(**task)

@app.delete("/tasks/{task_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_task(task_id: int):
    """ลบ task"""
    if task_id not in tasks_db:
        raise HTTPException(status_code=404, detail=f"ไม่พบ task #{task_id}")
    del tasks_db[task_id]

# === Dependency Injection ด้วย Pydantic ===
class PaginationParams(BaseModel):
    """Common pagination parameters"""
    page: int = Field(default=1, ge=1, description="หน้าที่ต้องการ")
    page_size: int = Field(default=10, ge=1, le=100, description="จำนวนต่อหน้า")
    
    @property
    def offset(self) -> int:
        return (self.page - 1) * self.page_size

async def get_pagination(page: int = 1, page_size: int = 10) -> PaginationParams:
    """Dependency สำหรับ pagination"""
    return PaginationParams(page=page, page_size=page_size)

@app.get("/tasks/v2/", response_model=TaskListResponse)
async def list_tasks_v2(
    pagination: PaginationParams = Depends(get_pagination),
    priority: Optional[str] = None,
):
    """แสดงรายการ tasks ด้วย dependency injection"""
    items = list(tasks_db.values())
    
    if priority:
        items = [t for t in items if t["priority"] == priority]
    
    total = len(items)
    start = pagination.offset
    end = start + pagination.page_size
    page_items = items[start:end]
    
    return TaskListResponse(
        items=[TaskResponse(**t) for t in page_items],
        total=total,
        page=pagination.page,
        page_size=pagination.page_size,
        has_next=end < total,
    )

# import Config from pydantic
from pydantic import ConfigDict
```

---

## 8. Common Validation Patterns

### 8.1 การ Handle Validation Errors

```python
from pydantic import BaseModel, ValidationError, Field
from typing import Optional
import json

class UserInput(BaseModel):
    name: str = Field(min_length=2)
    age: int = Field(gt=0, lt=150)
    email: str

def process_user_input(data: dict) -> dict:
    """จัดการ validation error อย่างเป็นมิตร"""
    try:
        user = UserInput(**data)
        return {"success": True, "data": user.model_dump()}
    
    except ValidationError as e:
        # แปลง errors เป็น format ที่ใช้งานง่าย
        errors = {}
        for error in e.errors():
            # loc เป็น tuple ของ path เช่น ("address", "postal_code")
            field = ".".join(str(loc) for loc in error["loc"])
            msg = error["msg"]
            
            # แปล error type
            error_type = error["type"]
            if error_type == "string_too_short":
                msg = f"ต้องมีอย่างน้อย {error['ctx']['min_length']} ตัวอักษร"
            elif error_type == "greater_than":
                msg = f"ต้องมากกว่า {error['ctx']['gt']}"
            elif error_type == "less_than":
                msg = f"ต้องน้อยกว่า {error['ctx']['lt']}"
            
            errors[field] = msg
        
        return {"success": False, "errors": errors}

# ทดสอบ
result = process_user_input({"name": "A", "age": 200, "email": "invalid"})
print(json.dumps(result, ensure_ascii=False, indent=2))
```

### 8.2 Partial Updates (PATCH Pattern)

```python
from pydantic import BaseModel, Field
from typing import Optional, Any
import copy

class UserBase(BaseModel):
    """Fields พื้นฐาน"""
    name: str
    email: str
    age: int = Field(gt=0)
    bio: Optional[str] = None

class UserCreate(UserBase):
    """สำหรับ POST — ต้องการ fields ทั้งหมด"""
    password: str = Field(min_length=8)

def make_partial(model_class):
    """สร้าง partial version ของ model (ทุก field optional)"""
    fields = {}
    for name, field_info in model_class.model_fields.items():
        # ทำให้ทุก field เป็น Optional
        fields[name] = (Optional[field_info.annotation], None)
    
    return type(f"Partial{model_class.__name__}", (BaseModel,), {
        "__annotations__": {k: v[0] for k, v in fields.items()},
        **{k: v[1] for k, v in fields.items()}
    })

# สร้าง PartialUser สำหรับ PATCH
PartialUser = make_partial(UserBase)

def apply_patch(existing: dict, patch: PartialUser) -> dict:
    """Apply patch ไปยัง existing data"""
    updated = copy.deepcopy(existing)
    
    # อัพเดตเฉพาะ fields ที่ส่งมา (ไม่ใช่ None)
    patch_data = patch.model_dump(exclude_none=True)
    updated.update(patch_data)
    
    return updated

# ทดสอบ
current_user = {
    "id": 1,
    "name": "สมชาย",
    "email": "somchai@example.com",
    "age": 30,
    "bio": None
}

patch = PartialUser(name="สมชาย ใจดี", age=31)
updated_user = apply_patch(current_user, patch)
print(updated_user)
```

---

## 9. สรุป Part 049

✅ **BaseModel + Field** — สร้าง data models พร้อม constraints ครบถ้วน

✅ **model_config** — ตั้งค่า strict mode, extra fields, frozen, str stripping

✅ **field_validator** — ตรวจสอบ field เดียวพร้อมแปลงค่า

✅ **model_validator** — ตรวจสอบหลาย fields ร่วมกัน (cross-field validation)

✅ **Custom Types** — สร้าง reusable types ด้วย Annotated และ class

✅ **Nested Models** — สร้าง complex data structures หลายชั้น

✅ **model_dump() / model_validate()** — serialize/deserialize อย่างยืดหยุ่น

✅ **Pydantic Settings** — จัดการ config จาก .env files และ environment variables

✅ **FastAPI Integration** — สร้าง Request/Response schemas ที่ถูกต้อง

## ➡️ ถัดไป: Part 050 - Python Best Practices and Clean Code

*Part 049/100+ | Python Course - Beginner to World-Class*
