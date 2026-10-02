# Part 028: Dataclasses
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ @dataclass decorator และประโยชน์ของมัน
- กำหนด fields, default values และ field metadata
- ใช้ __post_init__ สำหรับ initialization logic
- สร้าง frozen dataclasses (immutable)
- เปรียบเทียบ dataclass กับ namedtuple และ dict
- ใช้ dataclass inheritance
- ทำความเข้าใจ __repr__, __eq__, __hash__

---

## 1. ทำไมต้องใช้ Dataclasses?

```python
# ❌ แบบดั้งเดิม - เขียนเยอะ, boilerplate มาก
class UserOld:
    def __init__(self, name: str, age: int, email: str):
        self.name = name
        self.age = age
        self.email = email
    
    def __repr__(self):
        return f"User(name={self.name!r}, age={self.age!r}, email={self.email!r})"
    
    def __eq__(self, other):
        if not isinstance(other, UserOld):
            return NotImplemented
        return (self.name, self.age, self.email) == (other.name, other.age, other.email)

# ✅ แบบ dataclass - กระชับ, อ่านง่าย
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
    email: str

# Python สร้าง __init__, __repr__, __eq__ ให้อัตโนมัติ!

u1 = User("Alice", 30, "alice@example.com")
u2 = User("Alice", 30, "alice@example.com")
u3 = User("Bob", 25, "bob@example.com")

print(u1)          # User(name='Alice', age=30, email='alice@example.com')
print(u1 == u2)    # True (เปรียบเทียบ field by field)
print(u1 == u3)    # False
print(u1.name)     # Alice
```

---

## 2. Fields และ Default Values

```python
from dataclasses import dataclass, field
from typing import List, Optional
from datetime import datetime

@dataclass
class Product:
    # Required fields (ไม่มี default) ต้องมาก่อน
    name: str
    price: float
    
    # Optional fields กับ default values
    category: str = "general"
    in_stock: bool = True
    discount: float = 0.0
    
    # Mutable defaults ต้องใช้ field(default_factory=...)
    # ❌ tags: list = []  # Error! mutable default
    tags: List[str] = field(default_factory=list)
    
    # Optional ที่ default เป็น None
    description: Optional[str] = None

# สร้าง Product
laptop = Product(
    name="MacBook Pro",
    price=59900.0,
    category="electronics",
    tags=["laptop", "apple"]
)
print(laptop)
# Product(name='MacBook Pro', price=59900.0, category='electronics',
#         in_stock=True, discount=0.0, tags=['laptop', 'apple'], description=None)

# Product ด้วย defaults
basic = Product("Basic Item", 99.0)
print(basic)
# Product(name='Basic Item', price=99.0, category='general', ...)

# field() options
@dataclass
class Employee:
    name: str
    department: str
    
    # repr=False - ไม่แสดงใน __repr__
    password_hash: str = field(default="", repr=False)
    
    # compare=False - ไม่ใช้ใน __eq__ และ __lt__
    internal_id: int = field(default=0, compare=False)
    
    # init=False - ไม่รับใน __init__ (ต้องกำหนดใน __post_init__)
    created_at: str = field(default="", init=False)
    
    # hash=False - ไม่ใช้ใน __hash__
    notes: str = field(default="", hash=False)

emp = Employee("Alice", "Engineering", password_hash="abc123")
print(emp)  # Employee(name='Alice', department='Engineering', internal_id=0, notes='')
# password_hash ไม่แสดงเพราะ repr=False
```

---

## 3. __post_init__: Custom Initialization

```python
from dataclasses import dataclass, field
from typing import List
import hashlib
import re

@dataclass
class UserAccount:
    username: str
    email: str
    password: str
    age: int
    
    # Computed fields (init=False)
    password_hash: str = field(default="", init=False, repr=False)
    display_name: str = field(default="", init=False)
    
    def __post_init__(self):
        """เรียกหลังจาก __init__ - ทำ validation และ computed fields"""
        # Validation
        if not re.match(r'^[a-zA-Z0-9_]{3,20}$', self.username):
            raise ValueError(f"Invalid username: {self.username!r}")
        
        if '@' not in self.email:
            raise ValueError(f"Invalid email: {self.email!r}")
        
        if self.age < 18:
            raise ValueError(f"User must be 18+, got {self.age}")
        
        if len(self.password) < 8:
            raise ValueError("Password must be at least 8 characters")
        
        # Computed fields
        self.password_hash = hashlib.sha256(self.password.encode()).hexdigest()
        self.display_name = self.username.capitalize()

try:
    # สร้าง valid user
    user = UserAccount(
        username="alice_dev",
        email="alice@example.com",
        password="secure123",
        age=25
    )
    print(user)
    # UserAccount(username='alice_dev', email='alice@example.com',
    #             age=25, display_name='Alice_dev')
    print(f"Password hash: {user.password_hash[:16]}...")
    # Password hash: 2bd45c8b7a4e9f3a...

    # Invalid user
    bad_user = UserAccount("ab", "notanemail", "short", 15)
except ValueError as e:
    print(f"Error: {e}")
    # Error: Invalid username: 'ab'

# __post_init__ กับ type conversion
@dataclass
class Temperature:
    celsius: float
    
    # Computed
    fahrenheit: float = field(default=0.0, init=False)
    kelvin: float = field(default=0.0, init=False)
    
    def __post_init__(self):
        if self.celsius < -273.15:
            raise ValueError(f"Temperature below absolute zero: {self.celsius}°C")
        self.fahrenheit = self.celsius * 9/5 + 32
        self.kelvin = self.celsius + 273.15
    
    def __str__(self) -> str:
        return f"{self.celsius}°C = {self.fahrenheit}°F = {self.kelvin}K"

temps = [Temperature(0), Temperature(100), Temperature(-40)]
for t in temps:
    print(t)
# 0°C = 32.0°F = 273.15K
# 100°C = 212.0°F = 373.15K
# -40°C = -40.0°F = 233.15K
```

---

## 4. Frozen Dataclasses (Immutable)

```python
from dataclasses import dataclass
from typing import Tuple

# frozen=True - ทำให้ instances เป็น immutable
@dataclass(frozen=True)
class Point:
    x: float
    y: float
    
    def distance_to(self, other: 'Point') -> float:
        return ((self.x - other.x)**2 + (self.y - other.y)**2)**0.5
    
    def translate(self, dx: float, dy: float) -> 'Point':
        """คืน Point ใหม่แทนที่จะ modify ตัวเอง"""
        return Point(self.x + dx, self.y + dy)

p1 = Point(0.0, 0.0)
p2 = Point(3.0, 4.0)

print(p1.distance_to(p2))  # 5.0

# ไม่สามารถ modify ได้
try:
    p1.x = 10.0  # FrozenInstanceError!
except Exception as e:
    print(f"Error: {e}")
    # Error: cannot assign to field 'x'

# สร้าง point ใหม่แทน
p3 = p1.translate(1.0, 2.0)
print(p3)  # Point(x=1.0, y=2.0)
print(p1)  # Point(x=0.0, y=0.0) - ไม่เปลี่ยน

# frozen dataclass สามารถใช้เป็น dict key และใส่ใน set ได้
point_set = {p1, p2, p3}
point_dict = {p1: "origin", p2: "3-4-5 point"}
print(point_dict[p1])  # origin

# ตัวอย่าง: Immutable configuration
@dataclass(frozen=True)
class DatabaseConfig:
    host: str
    port: int
    database: str
    username: str
    password: str = ""
    
    @property
    def connection_string(self) -> str:
        return f"postgresql://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"
    
    def with_password(self, password: str) -> 'DatabaseConfig':
        """สร้าง config ใหม่ที่มี password"""
        # ใช้ dataclasses.replace สำหรับ copy + change
        from dataclasses import replace
        return replace(self, password=password)

config = DatabaseConfig(
    host="localhost",
    port=5432,
    database="myapp",
    username="admin"
)

config_with_pw = config.with_password("secret123")
print(config_with_pw.connection_string)
# postgresql://admin:secret123@localhost:5432/myapp
```

---

## 5. Comparison และ Ordering

```python
from dataclasses import dataclass

# eq=True (default) - สร้าง __eq__ และ __ne__
# order=True - สร้าง __lt__, __le__, __gt__, __ge__

@dataclass(order=True)
class Student:
    # Python เปรียบเทียบ fields ตาม order ที่กำหนด
    # (เหมือนเปรียบเทียบ tuple)
    grade: float    # เปรียบเทียบ grade ก่อน
    name: str       # แล้วค่อยเปรียบเทียบ name

students = [
    Student(3.8, "Charlie"),
    Student(3.9, "Alice"),
    Student(3.7, "Bob"),
    Student(3.9, "Dave"),
]

# sort โดยอัตโนมัติ
sorted_students = sorted(students)
for s in sorted_students:
    print(f"{s.name}: {s.grade}")
# Bob: 3.7
# Charlie: 3.8
# Alice: 3.9
# Dave: 3.9

# min, max
best = max(students)
print(f"Top student: {best.name} ({best.grade})")  # Top student: Dave (3.9)

# กำหนด sort_index เอง ด้วย field(compare=...)
from dataclasses import field

@dataclass(order=True)
class Task:
    # sort_index ใช้สำหรับ comparison เท่านั้น
    sort_index: int = field(init=False, repr=False)
    
    priority: int  # 1=high, 2=medium, 3=low
    name: str
    
    def __post_init__(self):
        # priority ต่ำ = urgent = sort ก่อน
        self.sort_index = self.priority

tasks = [
    Task(priority=3, name="Clean desk"),
    Task(priority=1, name="Fix critical bug"),
    Task(priority=2, name="Write tests"),
    Task(priority=1, name="Deploy hotfix"),
]

for t in sorted(tasks):
    print(f"[P{t.priority}] {t.name}")
# [P1] Deploy hotfix
# [P1] Fix critical bug
# [P2] Write tests
# [P3] Clean desk
```

---

## 6. Dataclass Inheritance

```python
from dataclasses import dataclass, field
from typing import Optional, List
from datetime import datetime

@dataclass
class Animal:
    name: str
    species: str
    age: int
    
    def describe(self) -> str:
        return f"{self.name} is a {self.age}-year-old {self.species}"

@dataclass
class Pet(Animal):
    """Pet extends Animal โดย dataclass inheritance"""
    owner: str
    vaccinated: bool = False
    
    def describe(self) -> str:
        base = super().describe()
        vacc_status = "vaccinated" if self.vaccinated else "not vaccinated"
        return f"{base}, owned by {self.owner} ({vacc_status})"

@dataclass
class Dog(Pet):
    breed: str = "Mixed"
    trained: bool = False
    
    def describe(self) -> str:
        base = super().describe()
        trained_status = "trained" if self.trained else "not trained"
        return f"{base}, {self.breed} breed, {trained_status}"

# สร้าง instances
cat = Pet("Whiskers", "Cat", 3, owner="Alice", vaccinated=True)
dog = Dog("Buddy", "Dog", 5, owner="Bob", breed="Golden Retriever", trained=True)

print(cat.describe())
# Whiskers is a 3-year-old Cat, owned by Alice (vaccinated)

print(dog.describe())
# Buddy is a 5-year-old Dog, owned by Bob (not vaccinated),
# Golden Retriever breed, trained

# ข้อควรระวัง: fields ที่มี default values ต้องมาหลัง fields ที่ไม่มี default
# Parent fields ไม่มี default → Child fields ที่มี default ทำได้

# ⚠️ ปัญหา: ถ้า parent มี field กับ default แล้ว child จะมี field ไม่มี default ไม่ได้
@dataclass
class Base:
    x: int = 0  # มี default

# ❌ นี้จะ error
# @dataclass
# class Child(Base):
#     y: int  # ไม่มี default แต่ base มี - TypeError!

# ✅ ต้องให้ child field มี default ด้วย
@dataclass
class ChildOK(Base):
    y: int = 1  # มี default ด้วย

print(ChildOK())      # ChildOK(x=0, y=1)
print(ChildOK(5, 10)) # ChildOK(x=5, y=10)
```

---

## 7. Utility Functions

```python
from dataclasses import dataclass, field, fields, asdict, astuple, replace
from typing import List, Dict

@dataclass
class Address:
    street: str
    city: str
    country: str = "Thailand"
    postal_code: str = ""

@dataclass
class Contact:
    name: str
    email: str
    phone: str
    address: Address
    tags: List[str] = field(default_factory=list)

contact = Contact(
    name="Alice",
    email="alice@example.com",
    phone="081-234-5678",
    address=Address("123 Main St", "Bangkok"),
    tags=["vip", "regular"]
)

# fields() - ดู field metadata
for f in fields(contact):
    print(f"  {f.name}: {f.type}")
# name: str
# email: str
# phone: str
# address: Address
# tags: List[str]

# asdict() - แปลงเป็น dict (nested ด้วย)
contact_dict = asdict(contact)
print(contact_dict)
# {'name': 'Alice', 'email': 'alice@example.com', 'phone': '081-234-5678',
#  'address': {'street': '123 Main St', 'city': 'Bangkok', 'country': 'Thailand', 'postal_code': ''},
#  'tags': ['vip', 'regular']}

# astuple() - แปลงเป็น tuple (nested ด้วย)
contact_tuple = astuple(contact)
print(contact_tuple[0])  # Alice (ชื่อ)

# replace() - สร้าง copy พร้อม overrides บาง fields
updated_contact = replace(contact, email="alice_new@example.com", phone="089-999-8888")
print(updated_contact.email)   # alice_new@example.com
print(contact.email)           # alice@example.com (ไม่เปลี่ยน)

# Nested replace
updated_address = replace(contact.address, city="Chiang Mai")
updated_with_new_city = replace(contact, address=updated_address)
print(updated_with_new_city.address.city)  # Chiang Mai

# JSON serialization
import json

def to_json(obj) -> str:
    return json.dumps(asdict(obj), ensure_ascii=False, indent=2)

print(to_json(contact))
```

---

## 8. Dataclass vs NamedTuple vs Dict

```python
from dataclasses import dataclass
from typing import NamedTuple, TypedDict, List

# 1. Dict - ง่ายสุด, ไม่มี type safety
user_dict = {
    "name": "Alice",
    "age": 30,
    "email": "alice@example.com"
}
# ❌ ไม่มี autocomplete
# ❌ ไม่รู้ว่า key อะไรมีบ้าง
# ❌ typo ไม่ error: user_dict["nema"]

# 2. TypedDict - dict ที่มี type
class UserTypedDict(TypedDict):
    name: str
    age: int
    email: str

user_td: UserTypedDict = {"name": "Alice", "age": 30, "email": "alice@example.com"}
# ✅ type safety
# ✅ IDE support
# ❌ ไม่มี methods
# ❌ ยังเป็น dict (mutable, hashable ไม่ได้)

# 3. NamedTuple - immutable, เหมือน tuple
class UserNamedTuple(NamedTuple):
    name: str
    age: int
    email: str

user_nt = UserNamedTuple("Alice", 30, "alice@example.com")
# ✅ immutable
# ✅ hashable
# ✅ unpacking: name, age, email = user_nt
# ✅ tuple methods: index(), count()
# ❌ ไม่มี default values ที่ยืดหยุ่น
# ❌ ไม่สามารถ inherit ได้ดี

# 4. Dataclass - ยืดหยุ่นที่สุด
@dataclass
class UserDataclass:
    name: str
    age: int
    email: str
    
    def is_adult(self) -> bool:
        return self.age >= 18
    
    @property
    def display_name(self) -> str:
        return self.name.title()

user_dc = UserDataclass("alice", 30, "alice@example.com")
# ✅ type safety
# ✅ methods
# ✅ properties
# ✅ inheritance
# ✅ default values
# ✅ mutable (ปกติ) หรือ frozen=True (immutable)

# เปรียบเทียบ
print("=== Dict ===")
print(user_dict["name"])

print("=== NamedTuple ===")
name, age, email = user_nt
print(f"{name}, {age}")

print("=== Dataclass ===")
print(user_dc.display_name)  # Alice
print(user_dc.is_adult())    # True

# สรุปว่าใช้อะไร:
comparison = {
    "Dict":         "เร็ว, flexible, ไม่ type-safe",
    "TypedDict":    "Dict + type safety, ไม่มี methods",
    "NamedTuple":   "Immutable tuple ที่อ่านง่าย",
    "Dataclass":    "OOP ที่สมบูรณ์, มี methods, inheritance",
    "frozen DC":    "Immutable dataclass, hashable"
}
for name, desc in comparison.items():
    print(f"  {name:15}: {desc}")
```

---

## 9. ตัวอย่างจริง: E-Commerce Order System

```python
from dataclasses import dataclass, field
from typing import List, Optional
from datetime import datetime
from enum import Enum
import uuid

class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

@dataclass(frozen=True)
class Money:
    """Immutable money value"""
    amount: float
    currency: str = "THB"
    
    def __add__(self, other: 'Money') -> 'Money':
        if self.currency != other.currency:
            raise ValueError(f"Currency mismatch: {self.currency} vs {other.currency}")
        return Money(self.amount + other.amount, self.currency)
    
    def __mul__(self, factor: float) -> 'Money':
        return Money(self.amount * factor, self.currency)
    
    def __str__(self) -> str:
        return f"{self.amount:,.2f} {self.currency}"

@dataclass
class OrderItem:
    product_id: str
    product_name: str
    unit_price: Money
    quantity: int
    
    @property
    def subtotal(self) -> Money:
        return self.unit_price * self.quantity
    
    def __str__(self) -> str:
        return f"{self.product_name} x{self.quantity} = {self.subtotal}"

@dataclass
class ShippingAddress:
    recipient_name: str
    street: str
    city: str
    province: str
    postal_code: str
    phone: str
    country: str = "Thailand"
    
    def format(self) -> str:
        return (f"{self.recipient_name}\n"
                f"{self.street}\n"
                f"{self.city}, {self.province} {self.postal_code}\n"
                f"{self.country}\n"
                f"Tel: {self.phone}")

@dataclass
class Order:
    customer_id: str
    shipping_address: ShippingAddress
    
    # Auto-generated
    order_id: str = field(default_factory=lambda: str(uuid.uuid4())[:8].upper())
    created_at: datetime = field(default_factory=datetime.now)
    status: OrderStatus = OrderStatus.PENDING
    items: List[OrderItem] = field(default_factory=list)
    notes: Optional[str] = None
    
    def add_item(self, item: OrderItem) -> None:
        self.items.append(item)
    
    @property
    def subtotal(self) -> Money:
        total = Money(0.0)
        for item in self.items:
            total = total + item.subtotal
        return total
    
    @property
    def shipping_fee(self) -> Money:
        # Free shipping ถ้ามากกว่า 500 บาท
        if self.subtotal.amount >= 500:
            return Money(0.0)
        return Money(50.0)
    
    @property
    def total(self) -> Money:
        return self.subtotal + self.shipping_fee
    
    def confirm(self) -> None:
        if self.status != OrderStatus.PENDING:
            raise ValueError(f"Cannot confirm order in {self.status.value} status")
        self.status = OrderStatus.CONFIRMED
    
    def ship(self) -> None:
        if self.status != OrderStatus.CONFIRMED:
            raise ValueError(f"Cannot ship order in {self.status.value} status")
        self.status = OrderStatus.SHIPPED
    
    def print_receipt(self) -> None:
        print(f"{'='*50}")
        print(f"ORDER #{self.order_id}")
        print(f"Date: {self.created_at.strftime('%Y-%m-%d %H:%M')}")
        print(f"Status: {self.status.value.upper()}")
        print(f"{'='*50}")
        print("ITEMS:")
        for item in self.items:
            print(f"  {item}")
        print(f"{'─'*50}")
        print(f"  Subtotal:     {self.subtotal}")
        print(f"  Shipping:     {self.shipping_fee}")
        print(f"  TOTAL:        {self.total}")
        print(f"{'='*50}")
        print("SHIPPING TO:")
        print(self.shipping_address.format())
        print(f"{'='*50}")

# ใช้งาน
address = ShippingAddress(
    recipient_name="สมชาย ใจดี",
    street="123 ถนนสุขุมวิท",
    city="กรุงเทพฯ",
    province="กรุงเทพมหานคร",
    postal_code="10110",
    phone="081-234-5678"
)

order = Order(customer_id="CUST001", shipping_address=address)

order.add_item(OrderItem("P001", "Python Book", Money(590.0), 1))
order.add_item(OrderItem("P002", "Mechanical Keyboard", Money(2500.0), 1))
order.add_item(OrderItem("P003", "USB Cable", Money(150.0), 2))

order.print_receipt()
order.confirm()
print(f"\nOrder status: {order.status.value}")
```

---

## 10. สรุป Part 028

✅ **@dataclass** สร้าง __init__, __repr__, __eq__ อัตโนมัติ  
✅ **field()** กำหนด metadata: default_factory, repr, compare, init  
✅ **__post_init__** สำหรับ validation และ computed fields  
✅ **frozen=True** ทำให้ immutable และ hashable  
✅ **order=True** เพิ่ม comparison operators  
✅ **Inheritance** รองรับ แต่ระวัง default value order  
✅ **asdict(), astuple(), replace()** utility functions  
✅ **vs dict/TypedDict/NamedTuple**: dataclass ยืดหยุ่นที่สุด  

---

## ➡️ ถัดไป: Part 029 - Abstract Base Classes

*Part 028/100+ | Python Course - Beginner to World-Class*
