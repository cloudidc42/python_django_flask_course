# Part 029: Abstract Base Classes (ABC)
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Abstract Base Classes และประโยชน์ของมัน
- ใช้ ABC module สร้าง abstract classes
- กำหนด abstract methods และ abstract properties
- ใช้ ABC เป็น interface pattern
- Multiple inheritance กับ ABC
- isinstance() และ issubclass() กับ ABC
- ตัวอย่างการใช้งานจริง

---

## 1. Abstract Class คืออะไร?

```python
# ปัญหา: เราต้องการให้ subclasses implement method บางอย่าง
# แต่ Python ไม่บังคับตามปกติ

class Shape:
    def area(self):
        pass  # ไม่ได้บังคับให้ subclass implement
    
    def perimeter(self):
        pass

class BadCircle(Shape):
    pass  # ลืม implement แต่ไม่ error!

bad = BadCircle()
print(bad.area())  # None - ไม่ error แต่ผิด!

# ✅ แก้ด้วย ABC
from abc import ABC, abstractmethod

class Shape(ABC):
    """Abstract base class สำหรับ geometric shapes"""
    
    @abstractmethod
    def area(self) -> float:
        """คำนวณพื้นที่ - ต้อง implement ใน subclass"""
        ...
    
    @abstractmethod
    def perimeter(self) -> float:
        """คำนวณเส้นรอบรูป - ต้อง implement ใน subclass"""
        ...

# ❌ ไม่สามารถ instantiate abstract class ได้
try:
    s = Shape()
except TypeError as e:
    print(f"Error: {e}")
    # Error: Can't instantiate abstract class Shape 
    # with abstract methods area, perimeter

# ❌ ถ้า subclass ไม่ implement abstract methods ก็จะ error
class IncompleteCircle(Shape):
    def __init__(self, radius: float):
        self.radius = radius
    # ลืม implement area() และ perimeter()

try:
    c = IncompleteCircle(5.0)
except TypeError as e:
    print(f"Error: {e}")
    # Error: Can't instantiate abstract class IncompleteCircle 
    # with abstract methods area, perimeter
```

---

## 2. Implement Abstract Methods

```python
from abc import ABC, abstractmethod
import math
from typing import List

class Shape(ABC):
    """Abstract base class สำหรับ shapes"""
    
    def __init__(self, color: str = "white") -> None:
        self.color = color
    
    @abstractmethod
    def area(self) -> float: ...
    
    @abstractmethod
    def perimeter(self) -> float: ...
    
    # Non-abstract method (concrete) - สืบทอดได้เลย
    def describe(self) -> str:
        return (f"{self.__class__.__name__}("
                f"color={self.color}, "
                f"area={self.area():.2f}, "
                f"perimeter={self.perimeter():.2f})")
    
    def scale(self, factor: float) -> 'Shape':
        """Abstract method ที่ return ตัวเอง"""
        raise NotImplementedError

class Circle(Shape):
    def __init__(self, radius: float, color: str = "white") -> None:
        super().__init__(color)
        self.radius = radius
    
    def area(self) -> float:
        return math.pi * self.radius ** 2
    
    def perimeter(self) -> float:
        return 2 * math.pi * self.radius
    
    def scale(self, factor: float) -> 'Circle':
        return Circle(self.radius * factor, self.color)

class Rectangle(Shape):
    def __init__(self, width: float, height: float, color: str = "white") -> None:
        super().__init__(color)
        self.width = width
        self.height = height
    
    def area(self) -> float:
        return self.width * self.height
    
    def perimeter(self) -> float:
        return 2 * (self.width + self.height)
    
    def scale(self, factor: float) -> 'Rectangle':
        return Rectangle(self.width * factor, self.height * factor, self.color)

class Triangle(Shape):
    def __init__(self, a: float, b: float, c: float, color: str = "white") -> None:
        super().__init__(color)
        self.a = a
        self.b = b
        self.c = c
        if not self._is_valid():
            raise ValueError(f"Invalid triangle: {a}, {b}, {c}")
    
    def _is_valid(self) -> bool:
        a, b, c = self.a, self.b, self.c
        return a + b > c and b + c > a and a + c > b
    
    def area(self) -> float:
        # Heron's formula
        s = self.perimeter() / 2
        return math.sqrt(s * (s-self.a) * (s-self.b) * (s-self.c))
    
    def perimeter(self) -> float:
        return self.a + self.b + self.c

# ใช้งาน
shapes: List[Shape] = [
    Circle(5.0, "red"),
    Rectangle(4.0, 6.0, "blue"),
    Triangle(3.0, 4.0, 5.0, "green"),
]

for shape in shapes:
    print(shape.describe())
# Circle(color=red, area=78.54, perimeter=31.42)
# Rectangle(color=blue, area=24.00, perimeter=20.00)
# Triangle(color=green, area=6.00, perimeter=12.00)

# Total area
total = sum(s.area() for s in shapes)
print(f"Total area: {total:.2f}")  # Total area: 108.54

# Scale ทุก shapes
scaled = [s.scale(2.0) for s in shapes]
for s in scaled:
    print(f"Scaled: {s.describe()}")
```

---

## 3. Abstract Properties

```python
from abc import ABC, abstractmethod

class Vehicle(ABC):
    """Abstract class สำหรับ vehicles"""
    
    def __init__(self, make: str, model: str, year: int) -> None:
        self.make = make
        self.model = model
        self.year = year
        self._speed = 0.0
    
    @property
    @abstractmethod
    def fuel_type(self) -> str:
        """Type of fuel (abstract property)"""
        ...
    
    @property
    @abstractmethod
    def max_speed(self) -> float:
        """Maximum speed in km/h (abstract property)"""
        ...
    
    @abstractmethod
    def start_engine(self) -> str: ...
    
    @abstractmethod
    def stop_engine(self) -> str: ...
    
    # Concrete methods
    def accelerate(self, amount: float) -> None:
        self._speed = min(self._speed + amount, self.max_speed)
        print(f"Speed: {self._speed:.1f} km/h")
    
    def brake(self, amount: float) -> None:
        self._speed = max(self._speed - amount, 0.0)
        print(f"Speed: {self._speed:.1f} km/h")
    
    def __str__(self) -> str:
        return f"{self.year} {self.make} {self.model} ({self.fuel_type})"

class GasCar(Vehicle):
    @property
    def fuel_type(self) -> str:
        return "Gasoline"
    
    @property
    def max_speed(self) -> float:
        return 200.0
    
    def start_engine(self) -> str:
        return "🔊 Vroom! Engine started"
    
    def stop_engine(self) -> str:
        return "🔇 Engine stopped"

class ElectricCar(Vehicle):
    def __init__(self, make: str, model: str, year: int, battery_kwh: float) -> None:
        super().__init__(make, model, year)
        self.battery_kwh = battery_kwh
    
    @property
    def fuel_type(self) -> str:
        return "Electric"
    
    @property
    def max_speed(self) -> float:
        return 250.0
    
    def start_engine(self) -> str:
        return "⚡ Whirr... Electric motor engaged"
    
    def stop_engine(self) -> str:
        return "🔋 Motor stopped, regenerative braking active"
    
    @property
    def range_km(self) -> float:
        return self.battery_kwh * 6  # ~6 km per kWh

class Bicycle(Vehicle):
    @property
    def fuel_type(self) -> str:
        return "Human Power"
    
    @property
    def max_speed(self) -> float:
        return 40.0
    
    def start_engine(self) -> str:
        return "🚴 Start pedaling!"
    
    def stop_engine(self) -> str:
        return "🛑 Stop pedaling"

# ใช้งาน
vehicles = [
    GasCar("Toyota", "Camry", 2023),
    ElectricCar("Tesla", "Model 3", 2024, battery_kwh=82),
    Bicycle("Giant", "Escape", 2023),
]

for v in vehicles:
    print(v)
    print(f"  {v.start_engine()}")
    v.accelerate(100)
    v.brake(40)
    print(f"  {v.stop_engine()}")
    print()
```

---

## 4. ABC เป็น Interface Pattern

```python
from abc import ABC, abstractmethod
from typing import List, Optional

# Interface สำหรับ data storage
class Repository(ABC):
    """Interface สำหรับ data repository"""
    
    @abstractmethod
    def save(self, entity: dict) -> dict: ...
    
    @abstractmethod
    def find_by_id(self, id: int) -> Optional[dict]: ...
    
    @abstractmethod
    def find_all(self) -> List[dict]: ...
    
    @abstractmethod
    def update(self, id: int, data: dict) -> Optional[dict]: ...
    
    @abstractmethod
    def delete(self, id: int) -> bool: ...

# In-Memory implementation
class InMemoryRepository(Repository):
    def __init__(self) -> None:
        self._store: dict = {}
        self._next_id = 1
    
    def save(self, entity: dict) -> dict:
        entity = {**entity, "id": self._next_id}
        self._store[self._next_id] = entity
        self._next_id += 1
        return entity
    
    def find_by_id(self, id: int) -> Optional[dict]:
        return self._store.get(id)
    
    def find_all(self) -> List[dict]:
        return list(self._store.values())
    
    def update(self, id: int, data: dict) -> Optional[dict]:
        if id not in self._store:
            return None
        self._store[id].update(data)
        return self._store[id]
    
    def delete(self, id: int) -> bool:
        if id in self._store:
            del self._store[id]
            return True
        return False

# SQLite implementation (stub)
class SQLiteRepository(Repository):
    def __init__(self, db_path: str, table_name: str) -> None:
        self.db_path = db_path
        self.table_name = table_name
    
    def save(self, entity: dict) -> dict:
        # TODO: implement with sqlite3
        print(f"SQLite: Saving to {self.table_name}")
        return entity
    
    def find_by_id(self, id: int) -> Optional[dict]:
        print(f"SQLite: SELECT FROM {self.table_name} WHERE id={id}")
        return None
    
    def find_all(self) -> List[dict]:
        print(f"SQLite: SELECT * FROM {self.table_name}")
        return []
    
    def update(self, id: int, data: dict) -> Optional[dict]:
        print(f"SQLite: UPDATE {self.table_name} SET ... WHERE id={id}")
        return None
    
    def delete(self, id: int) -> bool:
        print(f"SQLite: DELETE FROM {self.table_name} WHERE id={id}")
        return True

# Service layer - ไม่สนใจว่า repository เก็บข้อมูลที่ไหน
class UserService:
    def __init__(self, repository: Repository) -> None:
        self.repo = repository  # Dependency injection
    
    def create_user(self, name: str, email: str) -> dict:
        # Validate
        if not name or not email:
            raise ValueError("Name and email required")
        if '@' not in email:
            raise ValueError("Invalid email")
        
        return self.repo.save({"name": name, "email": email})
    
    def get_user(self, user_id: int) -> dict:
        user = self.repo.find_by_id(user_id)
        if not user:
            raise KeyError(f"User {user_id} not found")
        return user
    
    def list_users(self) -> List[dict]:
        return self.repo.find_all()

# ทดสอบด้วย InMemory (ไม่ต้องใช้ database จริง)
repo = InMemoryRepository()
service = UserService(repo)

alice = service.create_user("Alice", "alice@example.com")
bob = service.create_user("Bob", "bob@example.com")
print(alice)  # {'name': 'Alice', 'email': 'alice@example.com', 'id': 1}

users = service.list_users()
print(f"Total users: {len(users)}")  # Total users: 2

# เปลี่ยนเป็น SQLite โดยไม่ต้องแก้ UserService
# sqlite_repo = SQLiteRepository("users.db", "users")
# service = UserService(sqlite_repo)
```

---

## 5. Multiple Inheritance กับ ABC

```python
from abc import ABC, abstractmethod
from typing import List

# หลาย interfaces
class Serializable(ABC):
    """ต้อง serialize ได้"""
    
    @abstractmethod
    def to_dict(self) -> dict: ...
    
    @abstractmethod
    def to_json(self) -> str: ...
    
    @classmethod
    @abstractmethod
    def from_dict(cls, data: dict) -> 'Serializable': ...

class Validatable(ABC):
    """ต้อง validate ได้"""
    
    @abstractmethod
    def validate(self) -> bool: ...
    
    @abstractmethod
    def get_errors(self) -> List[str]: ...

class Loggable(ABC):
    """ต้อง log ได้"""
    
    @abstractmethod
    def log_entry(self) -> str: ...

# Class ที่ implement หลาย interfaces
import json

class User(Serializable, Validatable, Loggable):
    def __init__(self, username: str, email: str, age: int) -> None:
        self.username = username
        self.email = email
        self.age = age
    
    # Serializable
    def to_dict(self) -> dict:
        return {
            "username": self.username,
            "email": self.email,
            "age": self.age
        }
    
    def to_json(self) -> str:
        return json.dumps(self.to_dict())
    
    @classmethod
    def from_dict(cls, data: dict) -> 'User':
        return cls(data["username"], data["email"], data["age"])
    
    # Validatable
    def validate(self) -> bool:
        return len(self.get_errors()) == 0
    
    def get_errors(self) -> List[str]:
        errors = []
        if not self.username or len(self.username) < 3:
            errors.append("Username must be at least 3 characters")
        if '@' not in self.email:
            errors.append("Invalid email format")
        if not 13 <= self.age <= 120:
            errors.append(f"Age {self.age} is not valid (13-120)")
        return errors
    
    # Loggable
    def log_entry(self) -> str:
        return f"[USER] {self.username} <{self.email}> age={self.age}"

# ทดสอบ
user = User("alice", "alice@example.com", 25)
print(user.to_json())
# {"username": "alice", "email": "alice@example.com", "age": 25}

print(user.validate())    # True
print(user.log_entry())   # [USER] alice <alice@example.com> age=25

# Restore จาก dict
data = {"username": "bob", "email": "bob@example.com", "age": 30}
bob = User.from_dict(data)
print(bob.validate())  # True

# Invalid user
invalid = User("ab", "notanemail", 10)
print(invalid.validate())    # False
print(invalid.get_errors())
# ['Username must be at least 3 characters', 'Invalid email format', 'Age 10 is not valid (13-120)']

# isinstance ตรวจสอบ
print(isinstance(user, Serializable))   # True
print(isinstance(user, Validatable))    # True
print(isinstance(user, Loggable))       # True
print(isinstance(user, User))           # True
```

---

## 6. isinstance() และ issubclass() กับ ABC

```python
from abc import ABC, abstractmethod, ABCMeta
from typing import List

class Drawable(ABC):
    @abstractmethod
    def draw(self) -> str: ...

class Clickable(ABC):
    @abstractmethod
    def on_click(self) -> None: ...

class Button(Drawable, Clickable):
    def __init__(self, label: str) -> None:
        self.label = label
    
    def draw(self) -> str:
        return f"[{self.label}]"
    
    def on_click(self) -> None:
        print(f"Button '{self.label}' clicked!")

class Label(Drawable):
    def __init__(self, text: str) -> None:
        self.text = text
    
    def draw(self) -> str:
        return f"<{self.text}>"

# isinstance() checks
btn = Button("OK")
lbl = Label("Hello")

print(isinstance(btn, Button))    # True
print(isinstance(btn, Drawable))  # True - เพราะ Button extend Drawable
print(isinstance(btn, Clickable)) # True
print(isinstance(lbl, Drawable))  # True
print(isinstance(lbl, Clickable)) # False - Label ไม่ implement Clickable

# issubclass() checks
print(issubclass(Button, Drawable))   # True
print(issubclass(Button, Clickable))  # True
print(issubclass(Label, Drawable))    # True
print(issubclass(Label, Clickable))   # False

# ใช้ใน function
def render_all(items: List[Drawable]) -> None:
    for item in items:
        print(item.draw())

def handle_click(widget) -> None:
    if isinstance(widget, Clickable):
        widget.on_click()
    else:
        print(f"{type(widget).__name__} is not clickable")

widgets = [Button("OK"), Label("Status"), Button("Cancel")]
render_all(widgets)
# [OK]
# <Status>
# [Cancel]

for w in widgets:
    handle_click(w)
# Button 'OK' clicked!
# Label is not clickable
# Button 'Cancel' clicked!

# __subclasses__() - ดู subclasses ทั้งหมด
print(Drawable.__subclasses__())  # [<class 'Button'>, <class 'Label'>]

# register() - virtual subclass (ไม่ต้อง inherit)
class Point:
    """ไม่ได้ inherit Drawable แต่จะ register"""
    def draw(self) -> str:
        return "• Point"

# Register เป็น virtual subclass
Drawable.register(Point)

p = Point()
print(isinstance(p, Drawable))  # True (เพราะ registered)
print(p.draw())                  # • Point
```

---

## 7. ABC กับ __subclasshook__

```python
from abc import ABC, abstractmethod

class Sized(ABC):
    """ABC ที่ check ด้วย __len__ method"""
    
    @abstractmethod
    def __len__(self) -> int: ...
    
    @classmethod
    def __subclasshook__(cls, subclass):
        """กำหนด logic สำหรับ isinstance() check"""
        if cls is Sized:
            # ถ้า subclass มี __len__ ก็ถือว่าเป็น Sized
            if any("__len__" in B.__dict__ for B in subclass.__mro__):
                return True
        return NotImplemented

# Classes ที่ไม่ได้ inherit แต่มี __len__
class MyList:
    def __init__(self, items):
        self.items = items
    
    def __len__(self):
        return len(self.items)

class MyString:
    def __init__(self, text):
        self.text = text
    
    def __len__(self):
        return len(self.text)

class NoLen:
    pass

ml = MyList([1, 2, 3])
ms = MyString("hello")
nl = NoLen()

print(isinstance(ml, Sized))   # True (มี __len__)
print(isinstance(ms, Sized))   # True (มี __len__)
print(isinstance(nl, Sized))   # False (ไม่มี __len__)
print(isinstance([1,2], Sized))  # True (list มี __len__)
print(isinstance("hi", Sized))   # True (str มี __len__)
print(isinstance(42, Sized))     # False (int ไม่มี __len__)

# ตัวอย่าง: ABC สำหรับ plugin system
class Plugin(ABC):
    """Base class สำหรับ plugins"""
    
    @property
    @abstractmethod
    def name(self) -> str: ...
    
    @property
    @abstractmethod
    def version(self) -> str: ...
    
    @abstractmethod
    def execute(self, *args, **kwargs): ...
    
    def __init_subclass__(cls, **kwargs):
        """เรียกเมื่อมี subclass ถูกสร้าง"""
        super().__init_subclass__(**kwargs)
        print(f"Plugin registered: {cls.__name__}")

class ImagePlugin(Plugin):
    @property
    def name(self) -> str: return "ImagePlugin"
    
    @property
    def version(self) -> str: return "1.0.0"
    
    def execute(self, image_path: str) -> str:
        return f"Processing image: {image_path}"

class VideoPlugin(Plugin):
    @property
    def name(self) -> str: return "VideoPlugin"
    
    @property
    def version(self) -> str: return "2.1.0"
    
    def execute(self, video_path: str, quality: str = "HD") -> str:
        return f"Processing video: {video_path} ({quality})"

# ใช้งาน plugins
plugins = [ImagePlugin(), VideoPlugin()]
for p in plugins:
    print(f"{p.name} v{p.version}")
    result = p.execute("test_file.mp4")
    print(f"  {result}")
```

---

## 8. ตัวอย่างจริง: Payment Gateway Interface

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional
from enum import Enum

class PaymentStatus(Enum):
    SUCCESS = "success"
    FAILED = "failed"
    PENDING = "pending"
    REFUNDED = "refunded"

@dataclass
class PaymentResult:
    status: PaymentStatus
    transaction_id: str
    amount: float
    currency: str
    message: str
    gateway: str

class PaymentGateway(ABC):
    """Abstract interface สำหรับ payment gateways"""
    
    @property
    @abstractmethod
    def gateway_name(self) -> str: ...
    
    @abstractmethod
    def charge(
        self,
        amount: float,
        currency: str,
        card_token: str,
        description: str = ""
    ) -> PaymentResult: ...
    
    @abstractmethod
    def refund(
        self,
        transaction_id: str,
        amount: Optional[float] = None
    ) -> PaymentResult: ...
    
    @abstractmethod
    def verify(self, transaction_id: str) -> PaymentResult: ...
    
    def charge_thb(self, amount_thb: float, card_token: str) -> PaymentResult:
        """Convenience method สำหรับ THB"""
        return self.charge(amount_thb, "THB", card_token)

# Stripe implementation
import uuid

class StripeGateway(PaymentGateway):
    def __init__(self, api_key: str) -> None:
        self.api_key = api_key
    
    @property
    def gateway_name(self) -> str:
        return "Stripe"
    
    def charge(self, amount, currency, card_token, description="") -> PaymentResult:
        # Simulate API call
        txn_id = f"stripe_{uuid.uuid4().hex[:8]}"
        print(f"[Stripe] Charging {amount} {currency}...")
        return PaymentResult(
            status=PaymentStatus.SUCCESS,
            transaction_id=txn_id,
            amount=amount,
            currency=currency,
            message="Payment successful",
            gateway=self.gateway_name
        )
    
    def refund(self, transaction_id, amount=None) -> PaymentResult:
        ref_id = f"stripe_ref_{uuid.uuid4().hex[:8]}"
        print(f"[Stripe] Refunding {transaction_id}...")
        return PaymentResult(
            status=PaymentStatus.REFUNDED,
            transaction_id=ref_id,
            amount=amount or 0.0,
            currency="THB",
            message="Refund processed",
            gateway=self.gateway_name
        )
    
    def verify(self, transaction_id) -> PaymentResult:
        return PaymentResult(
            status=PaymentStatus.SUCCESS,
            transaction_id=transaction_id,
            amount=0.0,
            currency="THB",
            message="Transaction verified",
            gateway=self.gateway_name
        )

# Omise (Thai payment gateway)
class OmiseGateway(PaymentGateway):
    def __init__(self, public_key: str, secret_key: str) -> None:
        self.public_key = public_key
        self.secret_key = secret_key
    
    @property
    def gateway_name(self) -> str:
        return "Omise"
    
    def charge(self, amount, currency, card_token, description="") -> PaymentResult:
        txn_id = f"chrg_{uuid.uuid4().hex[:8]}"
        print(f"[Omise] Charging {amount} {currency}...")
        return PaymentResult(
            status=PaymentStatus.SUCCESS,
            transaction_id=txn_id,
            amount=amount,
            currency=currency,
            message="สำเร็จ",
            gateway=self.gateway_name
        )
    
    def refund(self, transaction_id, amount=None) -> PaymentResult:
        ref_id = f"refn_{uuid.uuid4().hex[:8]}"
        return PaymentResult(
            status=PaymentStatus.REFUNDED,
            transaction_id=ref_id,
            amount=amount or 0.0,
            currency="THB",
            message="คืนเงินแล้ว",
            gateway=self.gateway_name
        )
    
    def verify(self, transaction_id) -> PaymentResult:
        return PaymentResult(
            status=PaymentStatus.SUCCESS,
            transaction_id=transaction_id,
            amount=0.0,
            currency="THB",
            message="ยืนยันแล้ว",
            gateway=self.gateway_name
        )

# Checkout service - ไม่สนใจว่าใช้ gateway ไหน
class CheckoutService:
    def __init__(self, gateway: PaymentGateway) -> None:
        self.gateway = gateway
    
    def process_order(
        self,
        order_id: str,
        amount: float,
        card_token: str
    ) -> bool:
        print(f"\nProcessing order #{order_id}")
        print(f"Amount: {amount:.2f} THB")
        print(f"Gateway: {self.gateway.gateway_name}")
        
        result = self.gateway.charge_thb(amount, card_token)
        
        if result.status == PaymentStatus.SUCCESS:
            print(f"✅ Payment successful: {result.transaction_id}")
            return True
        else:
            print(f"❌ Payment failed: {result.message}")
            return False

# ทดสอบกับ gateways ต่างๆ
stripe = StripeGateway("sk_test_xxx")
omise = OmiseGateway("pkey_test_xxx", "skey_test_xxx")

# ใช้ Stripe
service = CheckoutService(stripe)
service.process_order("ORD001", 1500.0, "tok_visa")

# เปลี่ยนเป็น Omise ง่ายมาก
service = CheckoutService(omise)
service.process_order("ORD002", 2990.0, "tokn_test_xxx")
```

---

## 9. สรุป Part 029

✅ **ABC** บังคับให้ subclasses implement abstract methods  
✅ **@abstractmethod** กำหนด methods ที่ต้อง implement  
✅ **@property + @abstractmethod** สำหรับ abstract properties  
✅ **ไม่สามารถ instantiate** abstract class ได้โดยตรง  
✅ **Interface pattern** ด้วย ABC + Dependency Injection  
✅ **Multiple inheritance** รองรับได้  
✅ **isinstance/issubclass** ทำงานกับ ABC ได้  
✅ **register()** สำหรับ virtual subclasses  
✅ **__subclasshook__** กำหนด logic สำหรับ isinstance check  

---

## ➡️ ถัดไป: Part 030 - Design Patterns

*Part 029/100+ | Python Course - Beginner to World-Class*
