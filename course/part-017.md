# Part 017 - OOP Polymorphism (โพลิมอร์ฟิซึม)

## เป้าหมาย
- เข้าใจ Duck Typing ของ Python
- ใช้ Operator Overloading เพื่อให้คลาสทำงานกับ operators
- เข้าใจ Method Overriding
- ใช้ `isinstance()` และ `issubclass()` อย่างถูกต้อง
- สร้าง Protocols สำหรับ structural subtyping
- ใช้ `@dataclass` เพื่อลดโค้ดซ้ำซ้อน

---

## 1. Duck Typing

Python ใช้ "Duck Typing" - ถ้ามันเดินเหมือนเป็ด และร้องเหมือนเป็ด มันก็คือเป็ด

```python
# Duck Typing: ไม่สนว่า type อะไร สนแค่ว่ามี method ที่ต้องการไหม

class Duck:
    def speak(self):
        return "Quack!"
    
    def walk(self):
        return "Waddle waddle"


class Person:
    def speak(self):
        return "Hello!"
    
    def walk(self):
        return "Walk normally"


class Robot:
    def speak(self):
        return "Beep boop!"
    
    def walk(self):
        return "Mechanical walk"


def make_it_speak_and_walk(thing):
    """ฟังก์ชันนี้ทำงานกับ object ใดก็ได้ที่มี speak() และ walk()"""
    print(f"Speaking: {thing.speak()}")
    print(f"Walking: {thing.walk()}")
    print()


# Duck Typing ทำงานกับ object ทุกชนิดที่มี method ที่ต้องการ
for entity in [Duck(), Person(), Robot()]:
    make_it_speak_and_walk(entity)

# ตัวอย่างที่ใช้จริง: file-like objects
class StringBuffer:
    """Object ที่ทำงานเหมือน file"""
    
    def __init__(self):
        self._data = []
    
    def write(self, text: str):
        """เขียนข้อมูล"""
        self._data.append(text)
    
    def read(self) -> str:
        """อ่านข้อมูล"""
        return "".join(self._data)
    
    def close(self):
        """ปิด buffer"""
        self._data = []


def write_report(file_obj, data: list):
    """ฟังก์ชันนี้ทำงานกับ file จริง หรือ StringBuffer ก็ได้"""
    for item in data:
        file_obj.write(f"{item}\n")


# ใช้กับ StringBuffer
buffer = StringBuffer()
write_report(buffer, ["line 1", "line 2", "line 3"])
print(buffer.read())

# ใช้กับ file จริง
import io
real_file = io.StringIO()
write_report(real_file, ["line 1", "line 2", "line 3"])
real_file.seek(0)
print(real_file.read())
```

---

## 2. Operator Overloading (การ Overload Operators)

Python ให้เราใช้ dunder methods เพื่อกำหนดพฤติกรรมของ operators

```python
from __future__ import annotations
from typing import Union

class Vector2D:
    """Vector 2 มิติที่รองรับ arithmetic operations"""
    
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y
    
    # Arithmetic operators
    def __add__(self, other: Union[Vector2D, float]) -> Vector2D:
        """v1 + v2 หรือ v1 + scalar"""
        if isinstance(other, Vector2D):
            return Vector2D(self.x + other.x, self.y + other.y)
        elif isinstance(other, (int, float)):
            return Vector2D(self.x + other, self.y + other)
        return NotImplemented
    
    def __radd__(self, other: float) -> Vector2D:
        """scalar + v1 (reverse add)"""
        return self.__add__(other)
    
    def __sub__(self, other: Union[Vector2D, float]) -> Vector2D:
        """v1 - v2"""
        if isinstance(other, Vector2D):
            return Vector2D(self.x - other.x, self.y - other.y)
        elif isinstance(other, (int, float)):
            return Vector2D(self.x - other, self.y - other)
        return NotImplemented
    
    def __mul__(self, scalar: float) -> Vector2D:
        """v1 * scalar"""
        if isinstance(scalar, (int, float)):
            return Vector2D(self.x * scalar, self.y * scalar)
        return NotImplemented
    
    def __rmul__(self, scalar: float) -> Vector2D:
        """scalar * v1"""
        return self.__mul__(scalar)
    
    def __truediv__(self, scalar: float) -> Vector2D:
        """v1 / scalar"""
        if scalar == 0:
            raise ZeroDivisionError("Cannot divide vector by zero")
        return Vector2D(self.x / scalar, self.y / scalar)
    
    def __neg__(self) -> Vector2D:
        """-v1"""
        return Vector2D(-self.x, -self.y)
    
    def __abs__(self) -> float:
        """abs(v1) = magnitude"""
        import math
        return math.sqrt(self.x ** 2 + self.y ** 2)
    
    # Comparison operators
    def __eq__(self, other: object) -> bool:
        """v1 == v2"""
        if not isinstance(other, Vector2D):
            return NotImplemented
        return self.x == other.x and self.y == other.y
    
    def __lt__(self, other: Vector2D) -> bool:
        """v1 < v2 (เปรียบเทียบจาก magnitude)"""
        return abs(self) < abs(other)
    
    def __le__(self, other: Vector2D) -> bool:
        return abs(self) <= abs(other)
    
    def __gt__(self, other: Vector2D) -> bool:
        return abs(self) > abs(other)
    
    def __ge__(self, other: Vector2D) -> bool:
        return abs(self) >= abs(other)
    
    # Container-like behavior
    def __len__(self) -> int:
        """จำนวน dimensions"""
        return 2
    
    def __getitem__(self, index: int) -> float:
        """v[0] = x, v[1] = y"""
        if index == 0:
            return self.x
        elif index == 1:
            return self.y
        raise IndexError(f"Index {index} out of range for 2D vector")
    
    def __iter__(self):
        """iterate: for val in vector"""
        yield self.x
        yield self.y
    
    def __contains__(self, value: float) -> bool:
        """value in vector"""
        return value in (self.x, self.y)
    
    # Boolean behavior
    def __bool__(self) -> bool:
        """bool(v) = True ถ้าไม่ใช่ zero vector"""
        return self.x != 0 or self.y != 0
    
    # String representations
    def __str__(self) -> str:
        return f"({self.x}, {self.y})"
    
    def __repr__(self) -> str:
        return f"Vector2D({self.x!r}, {self.y!r})"
    
    # Hash (เพื่อใช้เป็น dict key หรือ set member)
    def __hash__(self) -> int:
        return hash((self.x, self.y))
    
    # Dot product
    def dot(self, other: Vector2D) -> float:
        """dot product"""
        return self.x * other.x + self.y * other.y
    
    def normalize(self) -> Vector2D:
        """normalize ให้ magnitude = 1"""
        magnitude = abs(self)
        if magnitude == 0:
            raise ValueError("Cannot normalize zero vector")
        return self / magnitude


# ทดสอบ Vector2D
v1 = Vector2D(3, 4)
v2 = Vector2D(1, 2)

print(f"v1 = {v1}")           # (3, 4)
print(f"v2 = {v2}")           # (1, 2)
print(f"v1 + v2 = {v1 + v2}") # (4, 6)
print(f"v1 - v2 = {v1 - v2}") # (2, 2)
print(f"v1 * 2 = {v1 * 2}")   # (6, 8)
print(f"3 * v2 = {3 * v2}")   # (3, 6)
print(f"|v1| = {abs(v1)}")    # 5.0
print(f"-v1 = {-v1}")         # (-3, -4)
print(f"v1 == v1: {v1 == v1}") # True
print(f"v1 > v2: {v1 > v2}")  # True

# iterate
print(list(v1))  # [3, 4]

# sorting
vectors = [Vector2D(5, 0), Vector2D(1, 1), Vector2D(3, 4)]
print(sorted(vectors))  # เรียงตาม magnitude
```

---

## 3. Operator Overloading กับ Money Class

```python
from decimal import Decimal
from typing import Union

class Money:
    """คลาส Money ที่รองรับ arithmetic operations"""
    
    CURRENCIES = {"THB": "฿", "USD": "$", "EUR": "€", "JPY": "¥"}
    
    def __init__(self, amount: Union[int, float, str, Decimal], 
                 currency: str = "THB"):
        self.amount = Decimal(str(amount))
        self.currency = currency.upper()
        if self.currency not in self.CURRENCIES:
            raise ValueError(f"Unsupported currency: {currency}")
    
    def _check_same_currency(self, other: Money):
        """ตรวจสอบว่าเป็นสกุลเงินเดียวกัน"""
        if self.currency != other.currency:
            raise ValueError(
                f"Cannot operate on different currencies: "
                f"{self.currency} and {other.currency}"
            )
    
    def __add__(self, other: Money) -> Money:
        self._check_same_currency(other)
        return Money(self.amount + other.amount, self.currency)
    
    def __sub__(self, other: Money) -> Money:
        self._check_same_currency(other)
        result = self.amount - other.amount
        if result < 0:
            raise ValueError("Money cannot be negative after subtraction")
        return Money(result, self.currency)
    
    def __mul__(self, multiplier: Union[int, float]) -> Money:
        return Money(self.amount * Decimal(str(multiplier)), self.currency)
    
    def __rmul__(self, multiplier: Union[int, float]) -> Money:
        return self.__mul__(multiplier)
    
    def __truediv__(self, divisor: Union[int, float]) -> Money:
        if divisor == 0:
            raise ZeroDivisionError("Cannot divide money by zero")
        return Money(self.amount / Decimal(str(divisor)), self.currency)
    
    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        return self.amount == other.amount and self.currency == other.currency
    
    def __lt__(self, other: Money) -> bool:
        self._check_same_currency(other)
        return self.amount < other.amount
    
    def __le__(self, other: Money) -> bool:
        self._check_same_currency(other)
        return self.amount <= other.amount
    
    def __gt__(self, other: Money) -> bool:
        self._check_same_currency(other)
        return self.amount > other.amount
    
    def __ge__(self, other: Money) -> bool:
        self._check_same_currency(other)
        return self.amount >= other.amount
    
    def __neg__(self) -> Money:
        raise ValueError("Money cannot be negative")
    
    def __abs__(self) -> Money:
        return Money(abs(self.amount), self.currency)
    
    def __round__(self, ndigits: int = 2) -> Money:
        return Money(round(self.amount, ndigits), self.currency)
    
    def __str__(self) -> str:
        symbol = self.CURRENCIES[self.currency]
        return f"{symbol}{self.amount:,.2f}"
    
    def __repr__(self) -> str:
        return f"Money({self.amount!r}, {self.currency!r})"
    
    def __format__(self, format_spec: str) -> str:
        if format_spec == "short":
            symbol = self.CURRENCIES[self.currency]
            return f"{symbol}{self.amount:.0f}"
        return str(self)
    
    @classmethod
    def zero(cls, currency: str = "THB") -> Money:
        """สร้าง Money ที่มีค่า 0"""
        return cls(0, currency)


# ทดสอบ Money
price = Money(100.50, "THB")
discount = Money(10.00, "THB")
tax_rate = 0.07

final_price = (price - discount) * (1 + tax_rate)
print(f"ราคา: {price}")
print(f"ส่วนลด: {discount}")
print(f"ราคาหลังภาษี: {final_price}")

# Sorting
prices = [Money(500), Money(100), Money(250), Money(75)]
print(sorted(prices))  # เรียงจากน้อยไปมาก

# Format
print(f"{price:short}")  # ฿101
```

---

## 4. isinstance() และ issubclass()

```python
from typing import Union

class Animal:
    pass

class Mammal(Animal):
    pass

class Dog(Mammal):
    def bark(self):
        return "Woof!"

class Cat(Mammal):
    def meow(self):
        return "Meow!"

class Fish(Animal):
    pass

class GoldFish(Fish):
    pass


# isinstance() - ตรวจสอบว่า object เป็น instance ของ class หรือ subclass
dog = Dog()
cat = Cat()
fish = GoldFish()

print(isinstance(dog, Dog))     # True
print(isinstance(dog, Mammal))  # True - Dog เป็น subclass ของ Mammal
print(isinstance(dog, Animal))  # True - Dog เป็น subclass ของ Animal
print(isinstance(dog, Cat))     # False

# isinstance กับ tuple of types
def process_animal(animal):
    """ประมวลผลตาม type"""
    if isinstance(animal, (Dog, Cat)):
        print(f"This is a pet: {type(animal).__name__}")
    elif isinstance(animal, Fish):
        print(f"This is a fish: {type(animal).__name__}")
    else:
        print(f"Unknown animal: {type(animal).__name__}")

process_animal(dog)   # This is a pet: Dog
process_animal(cat)   # This is a pet: Cat
process_animal(fish)  # This is a fish: GoldFish

# issubclass() - ตรวจสอบ class hierarchy
print(issubclass(Dog, Animal))    # True
print(issubclass(Dog, Mammal))    # True
print(issubclass(Mammal, Animal)) # True
print(issubclass(Dog, Fish))      # False
print(issubclass(Dog, Dog))       # True (class เป็น subclass ของตัวเอง)

# ใช้กับ built-in types
print(isinstance(42, int))        # True
print(isinstance(42, (int, float))) # True
print(isinstance("hello", str))   # True
print(isinstance([], list))       # True
print(issubclass(bool, int))      # True - bool เป็น subclass ของ int!
print(isinstance(True, int))      # True

# ใช้ isinstance อย่างถูกต้อง
def safe_divide(a: Union[int, float], b: Union[int, float]) -> float:
    """หารอย่างปลอดภัย"""
    if not isinstance(a, (int, float)):
        raise TypeError(f"a must be a number, got {type(a).__name__}")
    if not isinstance(b, (int, float)):
        raise TypeError(f"b must be a number, got {type(b).__name__}")
    if b == 0:
        raise ZeroDivisionError("Cannot divide by zero")
    return a / b

print(safe_divide(10, 3))  # 3.333...
```

---

## 5. Protocols (Structural Subtyping)

```python
from typing import Protocol, runtime_checkable
from typing import List, Optional

# Protocol กำหนด interface โดยไม่ต้อง inherit

@runtime_checkable
class Drawable(Protocol):
    """Protocol สำหรับ object ที่วาดได้"""
    
    def draw(self) -> str:
        """วาด object"""
        ...
    
    def get_position(self) -> tuple:
        """คืนค่า position"""
        ...


@runtime_checkable  
class Resizable(Protocol):
    """Protocol สำหรับ object ที่ resize ได้"""
    
    def resize(self, factor: float) -> None:
        """ปรับขนาด"""
        ...
    
    def get_size(self) -> float:
        """คืนค่าขนาด"""
        ...


class Circle:
    """วงกลม - implement Drawable และ Resizable"""
    
    def __init__(self, x: float, y: float, radius: float):
        self.x = x
        self.y = y
        self.radius = radius
    
    def draw(self) -> str:
        return f"Circle at ({self.x}, {self.y}) with radius {self.radius}"
    
    def get_position(self) -> tuple:
        return (self.x, self.y)
    
    def resize(self, factor: float) -> None:
        self.radius *= factor
    
    def get_size(self) -> float:
        return self.radius


class Square:
    """สี่เหลี่ยม - implement Drawable และ Resizable"""
    
    def __init__(self, x: float, y: float, side: float):
        self.x = x
        self.y = y
        self.side = side
    
    def draw(self) -> str:
        return f"Square at ({self.x}, {self.y}) with side {self.side}"
    
    def get_position(self) -> tuple:
        return (self.x, self.y)
    
    def resize(self, factor: float) -> None:
        self.side *= factor
    
    def get_size(self) -> float:
        return self.side


class TextLabel:
    """ข้อความ - implement Drawable เท่านั้น"""
    
    def __init__(self, x: float, y: float, text: str):
        self.x = x
        self.y = y
        self.text = text
    
    def draw(self) -> str:
        return f"Text '{self.text}' at ({self.x}, {self.y})"
    
    def get_position(self) -> tuple:
        return (self.x, self.y)


def render_all(drawables: List[Drawable]) -> None:
    """Render ทุก object ที่ implement Drawable"""
    for obj in drawables:
        print(obj.draw())


def resize_all(resizables: List[Resizable], factor: float) -> None:
    """Resize ทุก object ที่ implement Resizable"""
    for obj in resizables:
        obj.resize(factor)


# สร้าง objects
c = Circle(0, 0, 5)
s = Square(10, 10, 4)
t = TextLabel(5, 5, "Hello")

# ทดสอบ Protocol check
print(isinstance(c, Drawable))    # True
print(isinstance(s, Drawable))    # True
print(isinstance(t, Drawable))    # True
print(isinstance(t, Resizable))   # False - TextLabel ไม่มี resize()

# render ทุก object
shapes = [c, s, t]
render_all(shapes)

# resize เฉพาะที่ resize ได้
resizable_shapes = [obj for obj in shapes if isinstance(obj, Resizable)]
resize_all(resizable_shapes, 2.0)

print("\nAfter resize:")
render_all(shapes)
```

---

## 6. Dataclasses

```python
from dataclasses import dataclass, field, asdict, astuple
from typing import List, Optional
from datetime import datetime

# @dataclass สร้าง __init__, __repr__, __eq__ ให้อัตโนมัติ

@dataclass
class Point:
    """จุดใน 2D space"""
    x: float
    y: float
    
    def distance_to(self, other: Point) -> float:
        """คำนวณระยะห่าง"""
        import math
        return math.sqrt((self.x - other.x)**2 + (self.y - other.y)**2)
    
    def __add__(self, other: Point) -> Point:
        return Point(self.x + other.x, self.y + other.y)


@dataclass(order=True)  # สร้าง __lt__, __le__, __gt__, __ge__
class Student:
    """นักเรียน - ordered by grade"""
    # sort_index จะใช้สำหรับการเปรียบเทียบ
    sort_index: float = field(init=False, repr=False)
    
    name: str
    student_id: str
    grade: float
    courses: List[str] = field(default_factory=list)
    enrolled: datetime = field(default_factory=datetime.now)
    
    def __post_init__(self):
        """เรียกหลัง __init__"""
        # ตั้ง sort_index ตาม grade
        self.sort_index = self.grade
        # Validate grade
        if not 0 <= self.grade <= 4.0:
            raise ValueError(f"Grade must be between 0 and 4.0, got {self.grade}")
    
    def add_course(self, course: str) -> None:
        """เพิ่มวิชา"""
        if course not in self.courses:
            self.courses.append(course)
    
    def get_letter_grade(self) -> str:
        """แปลง GPA เป็นเกรดตัวอักษร"""
        if self.grade >= 3.5:
            return "A"
        elif self.grade >= 3.0:
            return "B+"
        elif self.grade >= 2.5:
            return "B"
        elif self.grade >= 2.0:
            return "C+"
        elif self.grade >= 1.5:
            return "C"
        elif self.grade >= 1.0:
            return "D"
        else:
            return "F"


@dataclass(frozen=True)  # immutable - ไม่สามารถแก้ไขได้หลังสร้าง
class Coordinate:
    """พิกัดที่เปลี่ยนแปลงไม่ได้"""
    latitude: float
    longitude: float
    
    def __post_init__(self):
        if not -90 <= self.latitude <= 90:
            raise ValueError(f"Invalid latitude: {self.latitude}")
        if not -180 <= self.longitude <= 180:
            raise ValueError(f"Invalid longitude: {self.longitude}")


# ทดสอบ dataclass
p1 = Point(1, 2)
p2 = Point(4, 6)
print(p1)                     # Point(x=1, y=2)
print(p1 + p2)                # Point(x=5, y=8)
print(p1.distance_to(p2))     # 5.0

s1 = Student("Alice", "S001", 3.8)
s2 = Student("Bob", "S002", 3.2)
s3 = Student("Charlie", "S003", 2.9)

s1.add_course("Python")
s1.add_course("Django")

print(s1)
print(f"{s1.name}: {s1.get_letter_grade()}")

# เรียงนักเรียน
students = [s1, s2, s3]
for student in sorted(students, reverse=True):
    print(f"{student.name}: {student.grade} ({student.get_letter_grade()})")

# แปลงเป็น dict หรือ tuple
print(asdict(s1))
print(astuple(p1))

# frozen dataclass
coord = Coordinate(13.7563, 100.5018)  # Bangkok
print(coord)
try:
    coord.latitude = 0  # AttributeError!
except AttributeError as e:
    print(f"Error: {e}")

# frozen dataclass สามารถใช้เป็น dict key ได้
locations = {
    Coordinate(13.7563, 100.5018): "Bangkok",
    Coordinate(35.6762, 139.6503): "Tokyo",
}
print(locations[Coordinate(13.7563, 100.5018)])  # Bangkok
```

---

## 7. Polymorphism ในการออกแบบ Real World

```python
from abc import ABC, abstractmethod
from typing import List, Dict, Any
from dataclasses import dataclass, field

# ตัวอย่าง: Payment System

@dataclass
class PaymentResult:
    """ผลลัพธ์การชำระเงิน"""
    success: bool
    transaction_id: str
    amount: float
    message: str


class PaymentGateway(ABC):
    """Abstract base สำหรับ payment gateway"""
    
    @abstractmethod
    def charge(self, amount: float, currency: str = "THB") -> PaymentResult:
        """เรียกเก็บเงิน"""
        pass
    
    @abstractmethod
    def refund(self, transaction_id: str, amount: float) -> PaymentResult:
        """คืนเงิน"""
        pass
    
    @abstractmethod
    def verify(self, transaction_id: str) -> bool:
        """ตรวจสอบ transaction"""
        pass
    
    def process_payment(self, amount: float) -> PaymentResult:
        """Template method: process payment with logging"""
        print(f"[{self.__class__.__name__}] Processing payment of {amount:.2f}")
        result = self.charge(amount)
        if result.success:
            print(f"[{self.__class__.__name__}] Payment successful: {result.transaction_id}")
        else:
            print(f"[{self.__class__.__name__}] Payment failed: {result.message}")
        return result


class StripeGateway(PaymentGateway):
    """Stripe payment gateway"""
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self._transactions = {}
    
    def charge(self, amount: float, currency: str = "THB") -> PaymentResult:
        """Charge via Stripe"""
        import uuid
        transaction_id = f"stripe_{uuid.uuid4().hex[:8]}"
        self._transactions[transaction_id] = amount
        return PaymentResult(
            success=True,
            transaction_id=transaction_id,
            amount=amount,
            message=f"Stripe charge successful"
        )
    
    def refund(self, transaction_id: str, amount: float) -> PaymentResult:
        """Refund via Stripe"""
        if transaction_id in self._transactions:
            return PaymentResult(
                success=True,
                transaction_id=f"refund_{transaction_id}",
                amount=amount,
                message="Stripe refund successful"
            )
        return PaymentResult(False, "", 0, "Transaction not found")
    
    def verify(self, transaction_id: str) -> bool:
        return transaction_id in self._transactions


class OmiseGateway(PaymentGateway):
    """Omise payment gateway (Thai)"""
    
    def __init__(self, public_key: str, secret_key: str):
        self.public_key = public_key
        self.secret_key = secret_key
        self._transactions = {}
    
    def charge(self, amount: float, currency: str = "THB") -> PaymentResult:
        """Charge via Omise"""
        import uuid
        transaction_id = f"omise_{uuid.uuid4().hex[:8]}"
        self._transactions[transaction_id] = amount
        return PaymentResult(
            success=True,
            transaction_id=transaction_id,
            amount=amount,
            message="Omise charge successful"
        )
    
    def refund(self, transaction_id: str, amount: float) -> PaymentResult:
        """Refund via Omise"""
        if transaction_id in self._transactions:
            return PaymentResult(
                success=True,
                transaction_id=f"refund_{transaction_id}",
                amount=amount,
                message="Omise refund successful"
            )
        return PaymentResult(False, "", 0, "Transaction not found")
    
    def verify(self, transaction_id: str) -> bool:
        return transaction_id in self._transactions


class QRCodeGateway(PaymentGateway):
    """PromptPay QR Code gateway"""
    
    def __init__(self, merchant_id: str):
        self.merchant_id = merchant_id
        self._transactions = {}
    
    def charge(self, amount: float, currency: str = "THB") -> PaymentResult:
        """Generate QR code and wait for payment"""
        import uuid
        transaction_id = f"qr_{uuid.uuid4().hex[:8]}"
        print(f"[QR Code] Please scan QR to pay {amount:.2f} THB")
        # Simulate successful payment
        self._transactions[transaction_id] = amount
        return PaymentResult(
            success=True,
            transaction_id=transaction_id,
            amount=amount,
            message="QR payment successful"
        )
    
    def refund(self, transaction_id: str, amount: float) -> PaymentResult:
        """QR payment ไม่รองรับ refund อัตโนมัติ"""
        return PaymentResult(
            success=False,
            transaction_id="",
            amount=0,
            message="Manual refund required for QR payments"
        )
    
    def verify(self, transaction_id: str) -> bool:
        return transaction_id in self._transactions


class PaymentProcessor:
    """ประมวลผลการชำระเงิน - ทำงานกับ gateway ใดก็ได้"""
    
    def __init__(self, gateway: PaymentGateway):
        self.gateway = gateway
        self.transaction_history: List[PaymentResult] = []
    
    def checkout(self, amount: float) -> PaymentResult:
        """Checkout และบันทึก history"""
        result = self.gateway.process_payment(amount)
        self.transaction_history.append(result)
        return result
    
    def get_total_collected(self) -> float:
        """ยอดรวมที่เก็บได้"""
        return sum(r.amount for r in self.transaction_history if r.success)


# ทดสอบ Polymorphism
stripe_processor = PaymentProcessor(StripeGateway("sk_test_xxx"))
omise_processor = PaymentProcessor(OmiseGateway("pkey_xxx", "skey_xxx"))
qr_processor = PaymentProcessor(QRCodeGateway("MERCHANT_123"))

# ใช้ polymorphism - code เดียวกัน ทำงานกับ gateway ต่างๆ
for processor in [stripe_processor, omise_processor, qr_processor]:
    result = processor.checkout(299.00)
    print(f"Result: {result.success}, ID: {result.transaction_id}")
    print()
```

---

## Exercises

### Exercise 1: Geometry Package
สร้าง package สำหรับ geometry ที่มี:
- `Shape` (ABC): `area()`, `perimeter()`, `scale(factor)`
- `__add__` สำหรับรวมพื้นที่
- `__lt__`, `__gt__` สำหรับเปรียบเทียบ
- Protocol `Colorable` สำหรับ shape ที่มีสี

### Exercise 2: Shopping Cart
สร้าง shopping cart ที่ใช้ operator overloading:
- `Cart + Product` = เพิ่มสินค้า
- `Cart - Product` = ลบสินค้า
- `Cart * quantity` = สั่งซื้อหลายชุด
- `len(cart)` = จำนวนสินค้า
- `for item in cart` = iterate

### Exercise 3: Notification System
สร้างระบบ notification ด้วย Protocol:
- Protocol `Notifiable`: `send(message, recipient)`
- EmailNotifier, SMSNotifier, LineNotifier
- `NotificationManager` ที่ส่งผ่าน gateway ใดก็ได้

---

## สรุป

| Concept | คำอธิบาย | ตัวอย่าง |
|---------|---------|---------|
| Duck Typing | ตรวจสอบ behavior ไม่ใช่ type | `if hasattr(obj, 'method')` |
| Operator Overloading | กำหนดพฤติกรรม operators | `__add__`, `__mul__`, `__eq__` |
| Method Overriding | override เมธอดจาก parent | `def method(self): ...` |
| `isinstance()` | ตรวจสอบ instance | `isinstance(obj, Class)` |
| `issubclass()` | ตรวจสอบ class hierarchy | `issubclass(Dog, Animal)` |
| Protocol | structural typing | `class P(Protocol):` |
| Dataclass | ลด boilerplate | `@dataclass` |

---

## ต่อไป

[Part 018 - Decorators](part-018.md) - Function decorators, class decorators, functools
