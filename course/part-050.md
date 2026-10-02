# Part 050: Python Best Practices Review
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจและปฏิบัติตาม PEP 8
- Naming conventions ที่ถูกต้อง
- จัดระเบียบ code โดยใช้ SOLID principles
- Clean code principles
- Refactoring techniques
- Code review checklist

---

## 1. PEP 8 - Style Guide

```python
# === PEP 8 การจัดรูปแบบ Code ===

# ❌ ผิด
x=1
y = x+1
if x==1 : print("hello")
def myFunction(a,b,c):return a+b+c

# ✅ ถูก
x = 1
y = x + 1
if x == 1:
    print("hello")


def my_function(a, b, c):
    return a + b + c


# === Indentation: 4 spaces ===
# ❌
def bad_indent():
  return True  # 2 spaces

# ✅
def good_indent():
    return True  # 4 spaces


# === Line Length: 88 characters (black default) ===
# ❌ บรรทัดยาวเกินไป
result = some_function_with_long_name(argument_one, argument_two, argument_three, argument_four, keyword=value)

# ✅ ตัดบรรทัด
result = some_function_with_long_name(
    argument_one,
    argument_two,
    argument_three,
    argument_four,
    keyword=value,  # trailing comma ดี
)


# === Blank Lines ===
class MyClass:
    """Two blank lines before and after class."""
    
    def method_one(self):
        """One blank line between methods."""
        pass
    
    def method_two(self):
        pass


def standalone_function():
    """Two blank lines before and after top-level functions."""
    
    # One blank line inside function to separate logical groups
    x = 1
    y = 2
    
    result = x + y
    return result


# === Imports ===
# ❌
import os, sys
from os import *

# ✅ - เรียงตาม: stdlib, third-party, local
import os
import sys
from pathlib import Path
from typing import List, Optional  # stdlib

import requests  # third-party
from pydantic import BaseModel

from myapp.models import User  # local
from myapp.utils import format_date


# === String Quotes ===
# ใช้ double quotes (แนะนำ) หรือ single quotes ก็ได้ แต่ consistent
name = "Alice"
message = "Don't worry"       # double quotes สะดวกเมื่อมี apostrophe
code = 'print("hello")'       # single quotes สะดวกเมื่อมี double quotes


# === Whitespace ===
# ❌
spam( ham[ 1 ], { eggs : 2 } )
foo = (0 ,)
dict ['key'] = list [index]

# ✅
spam(ham[1], {eggs: 2})
foo = (0,)
dict['key'] = list[index]
```

---

## 2. Naming Conventions

```python
# === Python Naming Conventions ===

# snake_case สำหรับ functions, variables, modules
user_name = "Alice"
first_name = "Alice"
max_value = 100

def calculate_total_price(price, quantity):
    return price * quantity

def get_user_by_email(email: str):
    pass

# PascalCase สำหรับ Classes
class UserAccount:
    pass

class DatabaseConnection:
    pass

class HTTPRequestHandler:  # Acronym ใช้ UPPER
    pass

# UPPER_SNAKE_CASE สำหรับ constants
MAX_RETRY_COUNT = 3
DEFAULT_TIMEOUT = 30
API_BASE_URL = "https://api.example.com"
DATABASE_URL = "postgresql://..."

# Private: prefix ด้วย _
class BankAccount:
    def __init__(self, balance: float):
        self._balance = balance      # protected (convention)
        self.__pin = None           # private (name mangling)
    
    def _validate_amount(self, amount: float) -> bool:
        """Protected method"""
        return amount > 0
    
    def __check_pin(self, pin: str) -> bool:
        """Private method"""
        return pin == self.__pin
    
    @property
    def balance(self) -> float:
        """Public property"""
        return self._balance

# Double underscore สำหรับ dunder (magic) methods
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    def __repr__(self) -> str:
        return f"Vector({self.x}, {self.y})"
    
    def __str__(self) -> str:
        return f"({self.x}, {self.y})"
    
    def __add__(self, other: "Vector") -> "Vector":
        return Vector(self.x + other.x, self.y + other.y)
    
    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Vector):
            return NotImplemented
        return self.x == other.x and self.y == other.y
    
    def __len__(self) -> int:
        return 2
    
    def __iter__(self):
        yield self.x
        yield self.y


# ทดสอบ naming
account = BankAccount(1000)
print(f"Balance: {account.balance}")

v1 = Vector(1, 2)
v2 = Vector(3, 4)
v3 = v1 + v2
print(f"v1 + v2 = {v3}")
print(f"v3 == Vector(4, 6): {v3 == Vector(4, 6)}")
```

---

## 3. SOLID Principles ใน Python

```python
# === S - Single Responsibility Principle ===
# แต่ละ class/function ควรมีหน้าที่เดียว

# ❌ ผิด - class ทำหลายอย่าง
class UserManager_BAD:
    def create_user(self, name, email, password):
        # Hash password
        hashed = self._hash_password(password)
        # Save to DB
        user = {"name": name, "email": email, "password": hashed}
        self._save_to_db(user)
        # Send email
        self._send_welcome_email(email, name)
        # Log
        self._log_action("user_created", email)
        return user
    
    def _hash_password(self, pwd): pass
    def _save_to_db(self, data): pass
    def _send_welcome_email(self, to, name): pass
    def _log_action(self, action, email): pass


# ✅ ถูก - แยกหน้าที่
class PasswordHasher:
    def hash(self, password: str) -> str:
        # Hash logic
        return f"hashed_{password}"

class UserRepository:
    def save(self, user: dict) -> dict:
        # DB logic
        return {**user, "id": 1}

class EmailService:
    def send_welcome(self, to: str, name: str):
        print(f"Welcome email sent to {to}")

class Logger:
    def log(self, action: str, data: dict):
        print(f"[LOG] {action}: {data}")

class UserService:
    def __init__(
        self,
        password_hasher: PasswordHasher,
        repository: UserRepository,
        email_service: EmailService,
        logger: Logger
    ):
        self._hasher = password_hasher
        self._repo = repository
        self._email = email_service
        self._logger = logger
    
    def create_user(self, name: str, email: str, password: str) -> dict:
        hashed = self._hasher.hash(password)
        user = self._repo.save({"name": name, "email": email, "password": hashed})
        self._email.send_welcome(email, name)
        self._logger.log("user_created", {"email": email})
        return user


# === O - Open/Closed Principle ===
# เปิดสำหรับ extension, ปิดสำหรับ modification

from abc import ABC, abstractmethod
from decimal import Decimal

class DiscountStrategy(ABC):
    @abstractmethod
    def calculate(self, price: Decimal) -> Decimal:
        pass

class NoDiscount(DiscountStrategy):
    def calculate(self, price: Decimal) -> Decimal:
        return price

class PercentageDiscount(DiscountStrategy):
    def __init__(self, percent: float):
        self._percent = percent
    
    def calculate(self, price: Decimal) -> Decimal:
        return price * Decimal(str(1 - self._percent / 100))

class FixedDiscount(DiscountStrategy):
    def __init__(self, amount: Decimal):
        self._amount = amount
    
    def calculate(self, price: Decimal) -> Decimal:
        return max(Decimal("0"), price - self._amount)

class SeasonalDiscount(DiscountStrategy):
    """New discount type - ไม่ต้องแก้ code เดิม"""
    def __init__(self, multiplier: float):
        self._multiplier = multiplier
    
    def calculate(self, price: Decimal) -> Decimal:
        return price * Decimal(str(self._multiplier))


class PriceCalculator:
    def __init__(self, strategy: DiscountStrategy):
        self._strategy = strategy
    
    def final_price(self, base_price: Decimal) -> Decimal:
        return self._strategy.calculate(base_price)


# ทดสอบ OCP
price = Decimal("1000")
calc1 = PriceCalculator(PercentageDiscount(20))
calc2 = PriceCalculator(FixedDiscount(Decimal("150")))
calc3 = PriceCalculator(SeasonalDiscount(0.8))

print(f"20% off: {calc1.final_price(price)}")
print(f"150 off: {calc2.final_price(price)}")
print(f"Seasonal 20%: {calc3.final_price(price)}")


# === L - Liskov Substitution Principle ===
# Subclass ต้อง substitutable แทน superclass ได้

class Rectangle:
    def __init__(self, width: float, height: float):
        self._width = width
        self._height = height
    
    @property
    def width(self) -> float:
        return self._width
    
    @property
    def height(self) -> float:
        return self._height
    
    def area(self) -> float:
        return self._width * self._height


class Square(Rectangle):
    """Square เป็น Rectangle แต่ต้องระวัง LSP"""
    
    def __init__(self, side: float):
        super().__init__(side, side)  # ✅ ถ้าไม่ให้ set ค่าแยก
    
    # ❌ ถ้า Square อนุญาตให้ set width/height แยกกัน จะผิด LSP


# === I - Interface Segregation Principle ===
# ไม่ควรบังคับ implement methods ที่ไม่ได้ใช้

# ❌ Fat interface
class Animal_BAD(ABC):
    @abstractmethod
    def eat(self): pass
    @abstractmethod
    def sleep(self): pass
    @abstractmethod
    def fly(self): pass   # ไม่ใช่ทุก animal บินได้
    @abstractmethod
    def swim(self): pass  # ไม่ใช่ทุก animal ว่ายน้ำได้

# ✅ แยก interfaces
class Eatable(ABC):
    @abstractmethod
    def eat(self): pass

class Sleepable(ABC):
    @abstractmethod
    def sleep(self): pass

class Flyable(ABC):
    @abstractmethod
    def fly(self): pass

class Swimmable(ABC):
    @abstractmethod
    def swim(self): pass

class Dog(Eatable, Sleepable):
    def eat(self): print("Dog eating")
    def sleep(self): print("Dog sleeping")

class Bird(Eatable, Sleepable, Flyable):
    def eat(self): print("Bird eating")
    def sleep(self): print("Bird sleeping")
    def fly(self): print("Bird flying")

class Duck(Eatable, Sleepable, Flyable, Swimmable):
    def eat(self): print("Duck eating")
    def sleep(self): print("Duck sleeping")
    def fly(self): print("Duck flying")
    def swim(self): print("Duck swimming")


# === D - Dependency Inversion Principle ===
# Depend on abstractions, not concretions

# ❌ High-level depends on Low-level
class UserService_BAD:
    def __init__(self):
        self._db = MySQLDatabase()  # concrete dependency!
    
    def get_user(self, user_id: int):
        return self._db.query(f"SELECT * FROM users WHERE id = {user_id}")


# ✅ Depend on abstraction
class Database(ABC):
    @abstractmethod
    def find_by_id(self, table: str, id: int) -> dict:
        pass
    
    @abstractmethod
    def save(self, table: str, data: dict) -> dict:
        pass

class MockDatabase(Database):
    def __init__(self):
        self._data = {"users": {1: {"id": 1, "name": "Alice"}}}
    
    def find_by_id(self, table: str, id: int) -> dict:
        return self._data.get(table, {}).get(id)
    
    def save(self, table: str, data: dict) -> dict:
        if table not in self._data:
            self._data[table] = {}
        self._data[table][data.get("id")] = data
        return data

class UserService_Good:
    def __init__(self, database: Database):  # depends on abstraction
        self._db = database
    
    def get_user(self, user_id: int) -> dict:
        return self._db.find_by_id("users", user_id)


# ทดสอบ DIP
db = MockDatabase()
service = UserService_Good(db)
print(f"\nUser: {service.get_user(1)}")
```

---

## 4. Clean Code Principles

```python
# === Functions ที่ดี ===

# 1. ชื่อ function บอกสิ่งที่ทำ
# ❌
def process(x, flag=False):
    if flag:
        return x * 2
    return x + 1

# ✅
def double_value(value: float) -> float:
    return value * 2

def increment_by_one(value: float) -> float:
    return value + 1


# 2. Function ทำสิ่งเดียว
# ❌
def validate_and_save_and_notify_user(user_data: dict):
    # Validate
    if not user_data.get("email"):
        raise ValueError("Email required")
    # Save
    db.save(user_data)
    # Notify
    email.send(user_data["email"])

# ✅ แยกหน้าที่
def validate_user_data(user_data: dict) -> None:
    if not user_data.get("email"):
        raise ValueError("Email required")

def save_user(user_data: dict) -> dict:
    return db.save(user_data)

def notify_new_user(email: str) -> None:
    email_service.send_welcome(email)


# 3. Arguments: ไม่ควรเกิน 3-4 ตัว
# ❌
def create_user(name, email, age, phone, address, city, country):
    pass

# ✅ ใช้ dataclass หรือ dict
from dataclasses import dataclass

@dataclass
class CreateUserRequest:
    name: str
    email: str
    age: int
    phone: str = None
    address: str = None
    city: str = None
    country: str = "TH"

def create_user(request: CreateUserRequest):
    pass


# 4. ไม่มี Side Effects ที่ไม่คาดหวัง
# ❌ มี side effect ที่ไม่คาดหวัง
numbers = [3, 1, 4, 1, 5]

def get_sorted(lst):
    lst.sort()  # แก้ไข list เดิม! ❌
    return lst

# ✅ Return ค่าใหม่
def get_sorted_copy(lst):
    return sorted(lst)  # คืน list ใหม่ ไม่แก้ของเดิม


# 5. Avoid Magic Numbers
# ❌
def calculate_tax(price: float) -> float:
    return price * 0.07  # 0.07 คืออะไร?

# ✅
VAT_RATE = 0.07  # 7% VAT

def calculate_vat(price: float) -> float:
    return price * VAT_RATE


# 6. ใช้ Descriptive Variable Names
# ❌
d = 86400
t = time.time()
r = []

# ✅
SECONDS_IN_DAY = 86400
current_timestamp = time.time()
active_users = []


# 7. Comments ที่มีประโยชน์
# ❌ Comment บอกสิ่งที่ code ทำอยู่แล้ว
# Loop through users
for user in users:
    # Print user name
    print(user.name)

# ✅ Comment อธิบาย "ทำไม"
# Skip inactive users to reduce email unsubscribe rate
active_users = [u for u in users if u.is_active]

# Retry up to 3 times due to occasional Redis timeouts in production
for attempt in range(3):
    try:
        cache.set(key, value)
        break
    except RedisError:
        if attempt == 2:
            raise
```

---

## 5. Refactoring Techniques

```python
# === Extract Method ===
# ❌ ก่อน refactor
def process_order_before(order_data: dict) -> dict:
    # Validate
    if "customer_id" not in order_data:
        raise ValueError("customer_id required")
    if "items" not in order_data or not order_data["items"]:
        raise ValueError("items required")
    
    # Calculate total
    total = 0
    for item in order_data["items"]:
        item_total = item["price"] * item["quantity"]
        total += item_total
    
    # Apply discount
    if total > 1000:
        total *= 0.9
    
    # Create order
    return {
        "customer_id": order_data["customer_id"],
        "items": order_data["items"],
        "total": total,
        "status": "pending"
    }

# ✅ หลัง refactor
def validate_order_data(order_data: dict) -> None:
    """Validate order data"""
    if "customer_id" not in order_data:
        raise ValueError("customer_id required")
    if "items" not in order_data or not order_data["items"]:
        raise ValueError("items required")


def calculate_order_total(items: list) -> float:
    """Calculate total price from items"""
    return sum(item["price"] * item["quantity"] for item in items)


def apply_bulk_discount(total: float, threshold: float = 1000) -> float:
    """Apply 10% discount for orders over threshold"""
    if total > threshold:
        return total * 0.9
    return total


def process_order(order_data: dict) -> dict:
    validate_order_data(order_data)
    
    total = calculate_order_total(order_data["items"])
    discounted_total = apply_bulk_discount(total)
    
    return {
        "customer_id": order_data["customer_id"],
        "items": order_data["items"],
        "total": discounted_total,
        "status": "pending"
    }


# === Replace Conditional with Polymorphism ===
# ❌ ก่อน
def get_area(shape_type: str, **dims) -> float:
    if shape_type == "circle":
        import math
        return math.pi * dims["radius"] ** 2
    elif shape_type == "rectangle":
        return dims["width"] * dims["height"]
    elif shape_type == "triangle":
        return 0.5 * dims["base"] * dims["height"]
    else:
        raise ValueError(f"Unknown shape: {shape_type}")

# ✅ หลัง
import math
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        pass

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius
    
    def area(self) -> float:
        return math.pi * self.radius ** 2

class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height
    
    def area(self) -> float:
        return self.width * self.height

class Triangle(Shape):
    def __init__(self, base: float, height: float):
        self.base = base
        self.height = height
    
    def area(self) -> float:
        return 0.5 * self.base * self.height


# ทดสอบ
shapes = [Circle(5), Rectangle(4, 6), Triangle(3, 8)]
for shape in shapes:
    print(f"{shape.__class__.__name__}: area = {shape.area():.2f}")


# === Guard Clauses (Early Return) ===
# ❌ ก่อน - nested ifs
def process_payment_before(user, amount, card):
    if user:
        if user.is_active:
            if amount > 0:
                if card:
                    if card.is_valid:
                        # actual logic here
                        return {"status": "success"}
                    else:
                        return {"error": "invalid card"}
                else:
                    return {"error": "no card"}
            else:
                return {"error": "invalid amount"}
        else:
            return {"error": "inactive user"}
    else:
        return {"error": "no user"}

# ✅ หลัง - guard clauses
def process_payment(user, amount, card) -> dict:
    if not user:
        return {"error": "no user"}
    if not user.is_active:
        return {"error": "inactive user"}
    if amount <= 0:
        return {"error": "invalid amount"}
    if not card:
        return {"error": "no card"}
    if not card.is_valid:
        return {"error": "invalid card"}
    
    # actual logic - no nesting!
    return {"status": "success"}
```

---

## 6. Code Review Checklist

```python
"""
=== Code Review Checklist ===

📋 Correctness:
□ Logic ถูกต้องตาม requirements?
□ Edge cases ถูก handle?
□ Error conditions ถูก handle?
□ ไม่มี off-by-one errors?
□ Null/None ถูก handle?

📋 Testing:
□ มี unit tests?
□ Tests ครอบคลุม happy path?
□ Tests ครอบคลุม error cases?
□ Tests ครอบคลุม edge cases?
□ Coverage >= 80%?

📋 Readability:
□ ชื่อ variables/functions/classes บอกความหมาย?
□ Comments อธิบาย "ทำไม" ไม่ใช่ "ทำอะไร"?
□ Functions ทำสิ่งเดียว?
□ Functions ไม่ยาวเกิน 30 บรรทัด?
□ Nested levels ไม่เกิน 3?

📋 Performance:
□ ไม่มี N+1 query?
□ ใช้ appropriate data structures?
□ Cache ที่ควร cache?
□ Database indexes ที่จำเป็น?
□ ไม่ load data ที่ไม่จำเป็น?

📋 Security:
□ Input ถูก validate?
□ SQL Injection ป้องกัน?
□ XSS ป้องกัน?
□ Sensitive data ไม่ถูก log?
□ Secrets ไม่ hardcode?

📋 Maintainability:
□ DRY (Don't Repeat Yourself)?
□ SOLID principles?
□ Dependencies ที่จำเป็นเท่านั้น?
□ ง่ายต่อการ extend?
□ Type hints ครบถ้วน?

📋 Documentation:
□ Public APIs มี docstrings?
□ Complex logic มี comments?
□ README อัปเดต?
□ Changelog อัปเดต?
"""


# === ตัวอย่าง Code ที่ผ่าน Review ===
from typing import Optional, List
from datetime import datetime
from dataclasses import dataclass

# Type hints ครบถ้วน
# ชื่อบอกความหมาย
# Single responsibility

@dataclass
class Order:
    id: int
    customer_id: int
    items: List[dict]
    status: str
    created_at: datetime


class OrderRepository:
    """Repository สำหรับจัดการ Order persistence"""
    
    def __init__(self, db):
        self._db = db
    
    def find_by_id(self, order_id: int) -> Optional[Order]:
        """หา Order โดย ID"""
        # ใช้ parameterized query ป้องกัน SQL injection
        row = self._db.execute(
            "SELECT * FROM orders WHERE id = ?",
            (order_id,)
        ).fetchone()
        
        if not row:
            return None
        
        return self._row_to_order(row)
    
    def find_pending_orders(self) -> List[Order]:
        """หา orders ที่ pending ทั้งหมด"""
        rows = self._db.execute(
            "SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at"
        ).fetchall()
        
        return [self._row_to_order(row) for row in rows]
    
    def _row_to_order(self, row) -> Order:
        """Private method แปลง DB row เป็น Order"""
        import json
        return Order(
            id=row["id"],
            customer_id=row["customer_id"],
            items=json.loads(row["items"]),
            status=row["status"],
            created_at=datetime.fromisoformat(row["created_at"])
        )


class OrderProcessor:
    """Process orders - ทำสิ่งเดียว"""
    
    def __init__(self, order_repo: OrderRepository, payment_service, notification_service):
        # Dependencies ถูก inject
        self._repo = order_repo
        self._payment = payment_service
        self._notification = notification_service
    
    def process_order(self, order_id: int) -> dict:
        """
        Process a pending order.
        
        Returns:
            dict: Processing result with status and order details
        
        Raises:
            OrderNotFoundError: If order doesn't exist
            InvalidOrderStatusError: If order is not pending
        """
        # Guard clauses - early return
        order = self._repo.find_by_id(order_id)
        if not order:
            raise OrderNotFoundError(f"Order {order_id} not found")
        
        if order.status != "pending":
            raise InvalidOrderStatusError(
                f"Order {order_id} is {order.status}, expected pending"
            )
        
        # Calculate total
        total = sum(
            item["price"] * item["quantity"]
            for item in order.items
        )
        
        # Process payment
        payment_result = self._payment.charge(order.customer_id, total)
        
        if not payment_result.success:
            return {
                "success": False,
                "order_id": order_id,
                "error": payment_result.error_message
            }
        
        # Notify customer
        self._notification.send_order_confirmation(
            customer_id=order.customer_id,
            order_id=order_id,
            total=total
        )
        
        return {
            "success": True,
            "order_id": order_id,
            "total": total,
            "payment_id": payment_result.payment_id
        }


class OrderNotFoundError(Exception):
    pass

class InvalidOrderStatusError(Exception):
    pass
```

---

## 7. Type Hints Best Practices

```python
from typing import Optional, Union, List, Dict, Tuple, Any, Callable
from typing import TypeVar, Generic, Protocol
from collections.abc import Iterator, Generator
from functools import wraps

# === Basic Type Hints ===
def greet(name: str) -> str:
    return f"Hello, {name}!"


def process_items(items: List[int]) -> List[int]:
    return [i * 2 for i in items]


def find_user(user_id: int) -> Optional[dict]:
    # ส่งคืน None ถ้าไม่พบ
    return None


# === Union Types (Python 3.10+ ใช้ | ได้) ===
def parse_id(value: Union[str, int]) -> int:
    return int(value)

# Python 3.10+
def parse_id_modern(value: str | int) -> int:
    return int(value)


# === Callable ===
def apply(func: Callable[[int], int], value: int) -> int:
    return func(value)


# === TypeVar ===
T = TypeVar("T")
K = TypeVar("K")
V = TypeVar("V")

def first(items: List[T]) -> Optional[T]:
    return items[0] if items else None


# === Protocol (Structural Subtyping) ===
class Drawable(Protocol):
    def draw(self) -> None: ...

class Resizable(Protocol):
    def resize(self, factor: float) -> None: ...

def render(drawable: Drawable) -> None:
    drawable.draw()  # Duck typing ที่ type-safe

class Circle:  # ไม่ต้อง inherit Protocol!
    def draw(self) -> None:
        print("Drawing circle")

render(Circle())  # ✅ works because Circle has draw()


# === Generic Classes ===
class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: List[T] = []
    
    def push(self, item: T) -> None:
        self._items.append(item)
    
    def pop(self) -> T:
        if not self._items:
            raise IndexError("Stack is empty")
        return self._items.pop()
    
    def peek(self) -> Optional[T]:
        return self._items[-1] if self._items else None
    
    def __len__(self) -> int:
        return len(self._items)


# ทดสอบ Generic Stack
int_stack: Stack[int] = Stack()
int_stack.push(1)
int_stack.push(2)
print(f"Stack: {int_stack.pop()}")  # 2

str_stack: Stack[str] = Stack()
str_stack.push("hello")
str_stack.push("world")
print(f"Stack: {str_stack.pop()}")  # world
```

---

## 8. Testing Best Practices

```python
import pytest
from unittest.mock import Mock, patch, MagicMock

# === Test Naming ===
# format: test_{what}_{condition}_{expected}

def test_calculate_total_with_valid_items_returns_sum():
    items = [{"price": 100, "quantity": 2}, {"price": 50, "quantity": 3}]
    result = calculate_total(items)
    assert result == 350  # 100*2 + 50*3

def test_calculate_total_with_empty_list_returns_zero():
    result = calculate_total([])
    assert result == 0

def test_calculate_total_with_zero_quantity_returns_zero():
    items = [{"price": 100, "quantity": 0}]
    result = calculate_total([])
    assert result == 0


# === Fixtures ===
@pytest.fixture
def sample_user():
    """Reusable user data"""
    return {
        "id": 1,
        "name": "Alice",
        "email": "alice@example.com",
        "is_active": True
    }

@pytest.fixture
def mock_db():
    """Mock database"""
    db = Mock()
    db.find_by_id.return_value = {"id": 1, "name": "Alice"}
    return db


# === Test Organization ===
class TestUserService:
    """Group related tests"""
    
    def test_create_user_success(self, mock_db):
        service = UserService_Good(mock_db)
        user = service.get_user(1)
        assert user["name"] == "Alice"
    
    def test_get_nonexistent_user_returns_none(self, mock_db):
        mock_db.find_by_id.return_value = None
        service = UserService_Good(mock_db)
        user = service.get_user(999)
        assert user is None
    
    @pytest.mark.parametrize("user_id,expected_name", [
        (1, "Alice"),
        (2, "Bob"),
        (3, "Charlie"),
    ])
    def test_get_user_by_various_ids(self, user_id, expected_name, mock_db):
        mock_db.find_by_id.return_value = {"id": user_id, "name": expected_name}
        service = UserService_Good(mock_db)
        user = service.get_user(user_id)
        assert user["name"] == expected_name


def calculate_total(items: list) -> float:
    return sum(item["price"] * item["quantity"] for item in items)


# === Mocking ===
class TestEmailSending:
    @patch("smtplib.SMTP")
    def test_send_email_success(self, mock_smtp):
        # Arrange
        mock_smtp_instance = MagicMock()
        mock_smtp.return_value.__enter__.return_value = mock_smtp_instance
        
        email_service = EmailService()
        
        # Act
        email_service.send_welcome("alice@example.com", "Alice")
        
        # Assert
        mock_smtp_instance.sendmail.assert_called_once()
```

---

## 9. สรุป Part 050

✅ **PEP 8** - Indentation, line length, imports, whitespace  
✅ **Naming** - snake_case, PascalCase, UPPER_CASE, _ prefixes  
✅ **SOLID** - SRP, OCP, LSP, ISP, DIP  
✅ **Clean Code** - Functions ที่ดี, no side effects, guard clauses  
✅ **Refactoring** - Extract method, polymorphism, early return  
✅ **Code Review** - Checklist ครอบคลุม  
✅ **Type Hints** - Union, Protocol, Generic, TypeVar  
✅ **Testing** - Naming, fixtures, parametrize, mocking  

**สิ่งที่ควรจำ:**
- Code อ่านมากกว่าเขียน - เขียนให้อ่านง่าย
- SOLID ช่วยให้ code ยืดหยุ่น และ test ง่าย
- Test ก่อน refactor เสมอ
- Performance หลัง correctness เสมอ
- ทำให้ simple ก่อน แล้วค่อย optimize

## ➡️ ถัดไป: Part 051 - Django Framework Basics
*Part 050/100+ | Python Course - Beginner to World-Class*
